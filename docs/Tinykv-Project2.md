# Project2

## Project2a

### 主要任务

​	这一部分我们首先需要根据Raft论文实现Raft算法，然后再为Raft算法提供一个封装，提供一系列上层调用的接口，其中Project2aa和Project2ab实现了Raft算法，Project2ac实现了一些供上层调用的接口

### 2aa&2ab介绍与思路

​	这两部分我个人其实都主要是面向测试集编程，由于2aa部分仅仅实现领导人选举，没有牵扯到论文中安全性一节添加的限制，所以本身过测试集是非常简单的，但是在2ab部分增加了日志复制，会导致2aa部分可能会有的地方挂掉，所以最后我是每过一个2ab的测试集就重新测一遍2aa.这一部分也没有出现一些很大的问题，往往根据测试函数就可以很快找到自己有哪些地方考虑不当.

​	这一部分重点需要关注`./raft/raft.go`和`./raft/log.go`中的代码，主要涉及了`Raft`和`RaftLog`两个结构体，`RaftLog`存放了一个节点的日志信息，`Raft`存放了一个节点的信息.

​	整个Raft系统是被`Tick()`函数驱动的，上层不断调用该函数，每调用一次就相当于过了一个单位时间，需要依次去更新时间量，并且检查是否到达临界条件，比如是否需要发起一次选举.`Step()`函数用来驱动节点执行传进来的信息，在处理完一个消息之后，有可能会产生一些别的消息，这个时候是不会调用Step函数去处理的，而是将其放到Raft结构中的msgs数组中.

​	先来简单看一下`Raft`结构体的定义

```go
type Raft struct {
	id uint64

	Term uint64
	Vote uint64

	// the log
	RaftLog *RaftLog

	// log replication progress of each peers
	Prs map[uint64]*Progress

	// this peer's role
	State StateType

	// votes records
	votes map[uint64]bool

	// msgs need to send
	msgs []pb.Message

	// the leader id
	Lead uint64

	// heartbeat interval, should send
	heartbeatTimeout int
	// baseline of election interval
	electionTimeout int
	// number of ticks since it reached last heartbeatTimeout.
	// only leader keeps heartbeatElapsed.
	heartbeatElapsed int
	// Ticks since it reached last electionTimeout when it is leader or candidate.
	// Number of ticks since it reached last electionTimeout or received a
	// valid message from current leader when it is a follower.
	electionElapsed int
	label bool
}
```

​	我在这里列出了在Project2中会设计到的量.其实注释已经基本说的很明白了，我在这里说几个我最开始实现的时候有一些疑惑的量

1. 几个时间相关的量.注释中已经说明了这四个量的含义，但是我最开始在将这些值初始化成多少的时候有一点疑惑，因为论文中明确要求了选举时间应该是随机的.后来看了测试函数，发现经过两个electiontimeout一定会发生一次选举，所以只需要根据这个去设置就行，我这里是将初始的electionElapsed设置成了一个`[-eletiontimeout，0]`之间的随机数
2. `msgs`，这个数组就是用来存放一次Step之后产生的一系列新的信息，等待被处理
3. `Prs`，这个数组存放了论文中提到的nextIndex和matchIndex，我们也需要通过这个数组去向集群中的其他节点发送信息
4. `label`，这个是我自己添加的一个量，因为在测试的时候发现会出现"瓜分"的情况，也就是一次选举没有Leader产生，这里就是作为一个标记，表示当前这次选举是否是因为瓜分产生的，如果是的话可以直接更新这个Candidate的任期.

​	下面看一下`RaftLog`结构体

```go
type RaftLog struct {
	// storage contains all stable entries since the last snapshot.
	storage Storage

	// committed is the highest log position that is known to be in
	// stable storage on a quorum of nodes.
	committed uint64

	// applied is the highest log position that the application has
	// been instructed to apply to its state machine.
	// Invariant: applied <= committed
	applied uint64

	// log entries with index <= stabled are persisted to storage.
	// It is used to record the logs that are not persisted by storage yet.
	// Everytime handling `Ready`, the unstabled logs will be included.
	stabled uint64

	// all entries that have not yet compact.
	entries []pb.Entry

	// the incoming unstable snapshot, if any.
	// (Used in 2C)
	pendingSnapshot *pb.Snapshot

	// Your Data Here (2A).
}
type Entry struct {
	EntryType            EntryType `protobuf:"varint,1,opt,name=entry_type,json=entryType,proto3,enum=eraftpb.EntryType" json:"entry_type,omitempty"`
	Term                 uint64    `protobuf:"varint,2,opt,name=term,proto3" json:"term,omitempty"`
	Index                uint64    `protobuf:"varint,3,opt,name=index,proto3" json:"index,omitempty"`
	Data                 []byte    `protobuf:"bytes,4,opt,name=data,proto3" json:"data,omitempty"`
	XXX_NoUnkeyedLiteral struct{}  `json:"-"`
	XXX_unrecognized     []byte    `json:"-"`
	XXX_sizecache        int32     `json:"-"`
}
```

​	这里注释其实也都说的非常清楚了，也没有什么特别让人费解的量.下面贴一张在网上找到的图，感觉画的还是很清楚的

![2-2.drawio.png](https://github.com/Smith-Cruise/TinyKV-White-Paper/blob/main/image/2-2.drawio.png?raw=true)

​	需要注意的是这里RaftLog结构中的索引和每个Entry的Index不是一回事，需要借助firstIndex去转化.

​	然后就是这里的Raft结构进行的一系列操作所需要的信息都被存到了Message中，这也是我们这一部分主要实现的Step函数在做的事情，也就是分门别类的去执行不同的Message，下面简单说一下Project2a中遇到的Message

1. `MessageType_MsgHup`，该信息作用于Follower和Candidate，当选举时间超时时用来开启新一轮的选举，具体的选举逻辑和论文中的一致，如下
   - 当Follower收到该信息之后，首先变成Candidate，然后广播一次请求投票，当然如果集群中只有这一个节点的话可以直接变成Leader
   - 当Candidate收到该信息之后，需要根据上面设置的label标志位决定是否自增任期，然后请求投票.
2. `MessageType_MsgBeat`，该信息作用于Leader，用来提醒Leader当心跳时间结束时向集群中所有节点发送一次心跳.这里主要是要实现raft.go中的sendHeartbeat函数，逻辑也非常简单，就是把心跳信息append到当前节点的msgs数组中
3. `MessageType_MsgPropose`，该信息作用于Leader，主要是上层用来向Leader追加日志的，具体逻辑如下
   - 首先将该信息中包含的Entry追加到RaftLog的后面，这里也需要实现LastIndex方法
   - 然后更新当前Leader自身的Prs值
   - 最后发送AppendRPC给peer，需要实现sendAppend方法，该方法和论文中的AppendRPC包含的内容是一样的.
4. `MessageType_MsgAppend`，该信息作用于三种状态的节点，其实就是sendAppend函数发送的信息类型，需要实现函数handleAppendEntries，逻辑和论文基本完全一致
   1. 当发送信息的任期小于接收者的任期时，直接拒绝
   2. 如果接收者的日志的LastIndex小于信息中的Index的话，也直接拒绝，因为这个时候肯定违反了论文中对应部分的第一条约定，也就是肯定找不到接收者日志的Index和Term都和信息中的相对应的一条日志，同时将返回信息中的Index设置成此时的LastIndex+1，便于Leader重新发送
   3. 获取信息中的PrevLogIndex对应的term，如果不对应的话，在接收者的日志中找到任期相对应的日志索引，然后拒绝这次请求.这里是做了论文中提到的一个优化，也就是直接将需要的日志的索引返回给Leader，而不是像论文最开始说的一样让Leader发送的信息的Index每次递减重复尝试
   4. 经过了上述条件，现在就可以舍弃掉冲突的日志了，然后接受这次的请求
5. `MessageType_MsgAppendResponse`，该信息作用于Leader，就是上面的handleAppendEntries处理后产生的信息，Leader主要就是在这里判断某个日志是否被复制到的大多数节点，然后判断这个日志是否能被提交
   - 首先如果收到了拒绝，就不需要处理了
   - 在收到同意的时候，首先根据传进来的信息来更新Prs的信息，然后需要判断当前日志是否被复制到了大部分节点，我是通过求Match这一信息的中位数来进行判断的
6. `MessageType_MsgRequestVote`，该信息作用于三种状态的节点，我单独写了一个handleRequestVote来进行处理，基本逻辑和论文是一致的
   - 当信息的任期小于接收者的任期时，直接拒绝，如果当前信息任期大于接收者任期的话，将接收者变成Follower
   - 然后我们判断日志是否匹配，如果不匹配就直接拒绝
   - 如果匹配的话，再判断接收者是否已经把票投给了别人，如果是的话同样拒绝
   - 给发送方投票
7. `MessageType_MsgRequestVoteResponse`，该信息作用于Candidate，主要就是看是否获取了大多数节点的选票，以此决定自己是否能够成为Leader
   - 统计是否获得半数投票
   - 如果是就变成Leader，否则变成Follower
   - 如果刚好获得一半，说明发生了瓜分，将上面自己定义的label设置成true
8. `MessageType_MsgHeartbeat`，该信息作用于三种状态的节点，主要需要实现handleHeartbeat函数
   - 当信息的任期小于接收者的任期时，直接拒绝，如果当前信息任期大于接收者任期的话，将接收者变成Follower
   - 接受心跳
9. `MessageType_MsgHeartbeatResponse`，该信息作用于Leader，直接向发送方发一次AppendRPC即可

### 2ac介绍与思路

​	这一部分主要是要实现三个函数，也就是`HasReady`，`Ready`，`Advance`.Ready这个结构体里存放的就是，经过一次处理之后会产生一系列需要更新和处理的信息，只需要根据Ready结构体里的字段进行填充即可.对于HasReady，其实就是判断新的状态和之前的状态是否相同，所以我们也需要在RawNode中存放之前的HardState和SoftState.

### 运行与测试

​	先来提几个2aa和2ab遇到的bug，由于本身这里是面向测试编程，所以也没有遇到太大的问题

1. RaftLog中的entries数组的索引和entry的index不是一回事，需要借助firstIndex去转化，2ab部分出错的很大原因是这里搞混了
2. 在初始化Raft结构体时，需要从storage中拿到HardState，里面保存了Vote等信息，需要在最开始进行初始化，有一个测试集就测了这一点
3. 每选举成了一个新的Leader，需要发送一个空entry，这里我是直接step了一个Propose信息
4. 在handleAppendEntries函数中，最开始如果r的lead信息是空的话，可以将它设置成m.From，个人感觉这样对最终结果没有什么影响，只是为了通过测试集

​	对于2ac，这里没有遇到什么问题，毕竟就两个测试集，而且测试集测得感觉也不是很完善，因为后来我写2B的时候发现我2ac中有一个函数写的完全是错的，但是没有测出来

​	最后进行测试，运行结果如下

![image-20220809103812981](https://cdn.jsdelivr.net/gh/ysjyx7/img@main/image-20220809103812981.png)

![image-20220809103836500](https://cdn.jsdelivr.net/gh/ysjyx7/img@main/image-20220809103836500.png)

![image-20220809104038394](https://cdn.jsdelivr.net/gh/ysjyx7/img@main/image-20220809104038394.png)

## Project2b

### 主要任务

​	这里是要用Project2A中实现的Raft模块构建一个可容错的键值存储系统，只需要实现`./kv/raftstore/peer_msg_handler.go`和`./kv/raftstore/peer_storage.go`两个文件中的两个函数即可，具体编写的代码量很小，但是需要阅读大量代码来理解整个项目的架构

### 介绍与思路

​	先来说一下对这一部分出现的几个名词的理解

1. `Store`，`Region`，`Peer`，三者的关系大致如下图

   ![img](https://github.com/Smith-Cruise/TinyKV-White-Paper/raw/main/image/2-0.png)

   从这张图里可以看出，一个Store中有多个Region，一个Region有多个Peer，一个Region处理某一个范围里的数据，同时一个Region里所有的peer里的数据是一致的，用于增加容错，不过在这一部分里一个Store只有一个Region，

2. 关于编码，很容易注意到在这一部分很多地方都调用了像`Marshal`和`Unmarshal`这样的编解码函数.由于相关知识太过于匮乏，最开始我也不清楚这是在干什么，后来也是觉得把需要传递的内容编码成字节序列是更能便于通信的，并且合理的编码也可以提高效率

​	下面说一下整个存储系统的一个启动流程

> 首先在`./kv`目录下的main函数中，调用了`./kv/storage/raft_storage`中的`start`函数启动`RaftStorage`，其封装了`RaftStore`，RaftStore中又包含了一系列`RaftWorker`，RaftStore主要就是启动了这些Worker，同时载入Peers.在启动Worker之后，就开始调用RaftWorker的`run`方法，也就是文档中提到过的很重要的一个方法，如下
>
> ```go
> func (rw *raftWorker) run(closeCh <-chan struct{}, wg *sync.WaitGroup) {
> 	defer wg.Done()
> 	var msgs []message.Msg
> 	for {
> 		msgs = msgs[:0]
> 		select {
> 		case <-closeCh:
> 			return
> 		case msg := <-rw.raftCh:
> 			msgs = append(msgs, msg)
> 		}
> 		pending := len(rw.raftCh)
> 		for i := 0; i < pending; i++ {
> 			msgs = append(msgs, <-rw.raftCh)
> 		}
> 		peerStateMap := make(map[uint64]*peerState)
> 		for _, msg := range msgs {
> 			peerState := rw.getPeerState(peerStateMap, msg.RegionID)
> 			if peerState == nil {
> 				continue
> 			}
> 			newPeerMsgHandler(peerState.peer, rw.ctx).HandleMsg(msg)
> 		}
> 		for _, peerState := range peerStateMap {
> 			newPeerMsgHandler(peerState.peer, rw.ctx).HandleRaftReady()
> 		}
> 	}
> }
> ```
>
> 可以看到，这里主要就是从管道`raftCh`中拿到需要处理的信息，然后调用HandleMsg函数和HandleRaftReady函数去进行处理，HandleMsg函数是已经实现好的函数，这个函数主要的作用就是根据信息种类的不同调用不同的方法去实现它，HandleRaftReady是一个待实现的函数

​	在2B中主要实现的是peerstorage中的两个函数和peermsghandler中的两个函数

​	先看一下peerstorage中的两个函数

1. `append`，作为一个辅助函数，将ready中的entries持久化，更新raftLocalstate的部分内容，删除永远不会被提交的entries。主要就是使用Writebatch进行操作。关于这个删除永远不会被提交的日志，首先在此次append之前已经有了一些entry，这些entry的最后的索引是lastindex，然后这次append的一系列entry的也有一系列索引，如果这些索引发生冲突的话，就要把之前的发生冲突的entries删除掉。
2. `SaveReadyState`，将entries和raftstate持久化，所以只需要调用append函数然后更新一下Hardstate即可。

​	然后说一下指令执行的流程

> 1. 从上层传进来的指令经过HandleMsg函数进行分流。如果是对数据进行交互的指令就会调用`proposeRaftCommand`函数，这是这部分需要实现的第一个函数，主要作用是，先检查目前符不符合发送的条件，如果符合就将指令封装成entry，propose给raft中其他所有的节点，并且在peer的proposals数组中append一个新propose，等待之后callback。
> 2. 在经过一系列的处理之后，这些日志会被commit，然后会调用`handleRaftReady`函数，主要作用就是去应用指令，apply，然后更新状态，即applystate。主要流程如下
>    - 通过 `d.RaftGroup.HasReady()` 方法判断是否有新的Ready，没有的话不做处理。
>    - 如果有Ready，需要先持久化Ready中的一些内容，主要就是调用上面已经实现的`SaveReadyState()`函数。
>    - 然后将Ready中保存的一些Msg发送给其他节点。需要调用`d.Send()`方法。
>    - 应用Ready中的commitentries，主要有如下几步
>      - 对一些操作进行应用，比如Put和Delete。
>      - 从peer的proposals数组中拿到对应的callback，向callback中放入不同请求的response，返回到上层。
>    - 调用`Advance`方法

### 运行与测试

​	这一部分随机测试，主要遇到了如下的bug

#### 非典型bug

​	这一部分的bug主要是在TestBasic2B出现的问题.

1. find no region for 302030303030

   打日志发现始终没有Leader产生，然后发现投票阶段出现问题，最后发现Raft节点中的Prs数组为空，在初始化的时候需要注意

2. panic: runtime error: index out of range [1192] with length 1192

   2A中的Term函数写的有问题，加一个边界判断就行了

#### 典型bug

​	下面是后面进行重复测试的时候出现的bug，主要就是下面这两类

1. [fatal] get wrong value， client 1
   - got的条目数比want少1个，打日志发现最后一次SetCF时集群的Lead是0，发现becomeLeader的时候需要将自己的Lead设置成自己的id
   - got的条目数比want少很多，从某个地方开始被截断了。打日志发现从某次分区更换过后，原本一直进行写入的Leader直接变成了Follower，这显然是不符合安全性这一要求的。后来发现是在handleheartbeat函数中，如果接收到的信息任期大于自己，应该直接变成Follower，我写成了在大于等于的时候直接变成Follower
   - 还有一些别的情况，后来修了request timeout这个问题之后也就再也没有出现过，怀疑是有的proposal没有拿到对应的回复然后就出错了，也就是和出现的request timeout的原因类似吧
2. panic: request timeout
   - 在TestBasic2B函数中出现这个问题。在初始化Ready的时候，把数据存放到ready之后，应该把raft节点中的msgs清空
   - 在后面的很多测试函数中都出现了这个问题，发现是在request函数中报了这个错误，进一步应该是在WaitRespWithTimeout中超时，也就是cb.done中没有东西。大致调用链为上层Client发起请求->调用proposeRaftCommand->propose经过相关流程之后，被apply到状态机->调用process函数处理entry，并在处理后调用p.cb.Done，也就是说有的proposal没有拿到对应的回复，最后逐步排查有哪种情况没有对应上即可.

#### 运行结果

​	最后也是基本无bug的过了.

![image-20220809225047574](https://cdn.jsdelivr.net/gh/ysjyx7/img@main/image-20220809225047574.png)

## Project2c

### 主要任务

​	这一部分我们需要在前两部分的基础上实现对快照的支持，快照就是为了防止日志无限制增长下去所采取的一种方法

### 介绍与思路

​	根据任务书，这一部分需要分别在原来的Raft模块和RaftStore模块添加相关的内容，从而能够支持快照

​	我们先来看一下对Raft模块的修改。

1. 这里涉及到一个新的消息`MessageType_MsgSnapshot`，这里可以参考论文中的`InstallSnapshotRPC`的逻辑进行实现，需要实现`handleSnapshot`函数，在此之前先看一下snapshot结构体里的内容

   ```go
   type Snapshot struct {
   	Data                 []byte            `protobuf:"bytes,1,opt,name=data,proto3" json:"data,omitempty"`
   	Metadata             *SnapshotMetadata `protobuf:"bytes,2,opt,name=metadata" json:"metadata,omitempty"`
   	XXX_NoUnkeyedLiteral struct{}          `json:"-"`
   	XXX_unrecognized     []byte            `json:"-"`
   	XXX_sizecache        int32             `json:"-"`
   }
   type SnapshotMetadata struct {
   	ConfState            *ConfState `protobuf:"bytes,1,opt,name=conf_state,json=confState" json:"conf_state,omitempty"`
   	Index                uint64     `protobuf:"varint,2,opt,name=index,proto3" json:"index,omitempty"`
   	Term                 uint64     `protobuf:"varint,3,opt,name=term,proto3" json:"term,omitempty"`
   	XXX_NoUnkeyedLiteral struct{}   `json:"-"`
   	XXX_unrecognized     []byte     `json:"-"`
   	XXX_sizecache        int32      `json:"-"`
   }
   ```

   文档中说Snapshot里的Data并不代表实际的状态机数据，不过这个暂时好像也不是很重要，对于Metadata，这里的Index和Term就是快照中的最后一个日志的索引和任期号，而ConfState中包含了Raft集群信息，根据论文我们需要重新加载集群配置

   下面看一下handleSnapshot的实现

   1. 首先看信息的任期是否小于接收者的任期号，如果是就直接拒绝操作，然后再看一下元数据中的Index和接收者已提交日志索引的大小关系，如果小于等于的话也直接拒绝，因为这说明，当前节点不需要这个快照同步
   2. 然后我们开始利用这个Snapshot进行一系列更新，主要的更新其实都在RaftLog这一块，更新一下各种类型的索引，以及更新一下RaftLog中的Entries这个数组，如果元数据的Index大于RaftLog的LastIndex，直接将Entries置为空即可，否则如果元数据的Index比firstindex大，就取原本数组的一部分即可.另外，这里也有一个小坑，就是如果此时RaftLog中的Entries数组为空的话，就append进去一个新的量，这个是测试样例有这个要求.
   3. 最后，利用元数据中的ConfState更新当前节点的集群配置信息

   感觉这里相比于论文中的InstallSnapshotRPC，要少了很多东西

2. 接下来需要考虑另外一个问题，也就是**什么时候发送Snapshot**，Snapshot本身也是去进行日志同步的一个东西，之前我们都是用AppendRPC来进行同步，也就是sendAppend那里，现在由于我们会进行日志压缩，如果我们没能找到要发送的日志，那就只能把整个快照发送过去让对方进行同步了，所以需要对sendAppend函数进行一些修改，也就是如果最开始我们没能根据PrevLogIndex找到对应的Term的话，就需要发送快照.也就是说我们需要对返回的error进行判断来决定是否需要发送快照即可

3. 然后，我们需要对之前实现过的RaftLog相关方法进行修改.

   - `FirstIndex`，由于之前没有快照，所以原则上不管是调用storage的firstindex来获取index还是用entris[0].Index作为FirstIndex都是可以的，但是现在不行了，所以可以封装一个FirstIndex方法
   - `Term`，如果直接用storage的Term返回的err的话，看它的实现似乎很大部分都是返回的`ErrUnavailable`，所以这里我们需要自己加一个判断逻辑，看看是否有快照，有的话基本就可以将err更改成`ErrCompacted`
   - `maybeCompact`，这个函数我最开始就没写，好像也没测出什么问题，其实也就是把一部分肯定没有用的日志删掉就行了

4. 最后，还需要更新RaftNode的相关逻辑，主要就是加入了对于快照的判断和更新

​	接下来看一下如何对RaftStore进行修改，先来简单根据任务书看一下相关的工作流程

1. 上层传过来一个Tick信号之后，就会调用OnTick，进而会触发`onRaftGCLogTick`，当检查到需要一个CompactLog的时候就会将这条信息封装到AdminRequest，然后交给proposeRaftCommand去执行，而这是我们上面实现的函数，在这里需要进行修改，也就是需要仿照之前对于正常的Request的支持，增加对于AdminRequest的支持，直接Propose即可
2. 在Propose之后，需要Apply这一日志，也就是在process函数中增加对AdminRequest的支持，只需要根据任务书中提到的，更改ApplyState，然后调用`ScheduleCompactLog`函数即可，这个函数就是把一个RaftLogGCTask发送出去

​	最后，只需要更改一下peer_storage.go中的ApplySnapshot函数，然后修改一下SaveReadyState函数即可

1. `ApplySnapshot`函数，只需要根据注释进行相应的修改即可
   - 调用`ClearMeta`和`ps.clearExtraData`删除过时的状态
   - 利用快照信息更新`raftState`，`ApplyState`，`SnapState`等等
   - 发送一个RegionTaskApply到regionSched去处理
2. `SaveReadyState`函数，其实只需要看看有没有快照，如果有的话就调用上面的函数进行处理即可

### 运行与测试

​	还是谈一谈遇到的几个主要的bug

1. 各种越界的报错，这里有时候是因为在获取firstindex的时候，有的地方没有改导致的问题，但总体改起来是很快的

2. failed to apply snap!!!. err: missing snapshot file invalid size 0 for snapshot cf file /tmp/test-raftstore705821829/snap/rev_1_20_118_default.sst, expected 680

   这个bug出现的频率很低，打日志也感觉不是很好定位出现在哪。后来把整个流程过了一边，发现有可能是在handleSnapshot那里，初始化集群配置的时候，没有把具体的Next和Match加进去，加进去之后也再没出现过这个问题了

3. panic: runtime error: index out of range [-1]

   这个我也没有追溯到出现的原因，并且出现的频率也很低，不过在我更改了上面的那个问题之后也没怎么出现过

​	最终成功基本无错

![image-20220810112549655](https://cdn.jsdelivr.net/gh/ysjyx7/img@main/image-20220810112549655.png)
