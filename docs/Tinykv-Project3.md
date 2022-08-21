# Project3

​	这一部分我们需要实现一个带有balance scheduler的基于multi-raft的kv服务器，需要实现领导人转移和成员变更，需要在raftstore上实现配置变更和region split，最后实现调度器

## Project3a

### 主要任务

​	这里是要在raft算法层实现领导人转移和成员变更

### 介绍与思路

​	这一部分基本也是面向测试集编程，难度其实不大。

​	先来看一下领导人转移的实现，这里需要使用Raft结构体中的一个量，如下：

- `leadTransferee`，这个量在领导人转移的时候使用，当它的值不是0的时候就是当前Lead需要转移的对象的id。

​	同时引入了两个新的message，如下：

- `MessageType_MsgTransferLeader`，根据项目文档该信息应该仅作用于领导人，但是在测试中该信息也被发给了Follower，这个会在后面的测试环节提到.该信息表示应该进行领导人转移
- `MessageType_MsgTimeoutNow`，该信息发送给领导人转移的对象，就是让接收方直接开始一次选举

​	接下来我们看一下具体的实现.首先根据文档，看一下`raftnode.go`中的代码，可以看到使用如下方式让底层进行领导人转移，并且该信息的From就是转移对象的id.

```go
func (rn *RawNode) TransferLeader(transferee uint64) {
	_ = rn.Raft.Step(pb.Message{MsgType: pb.MessageType_MsgTransferLeader， From: transferee})
}
```

​	然后看一下关于`MsgTransferLeader`这个信息的处理

1. Leader应该检查转移对象是否满足转移的要求，比如转移对象是否在集群里，日志是否足够新.
2. 如果满足了上面的要求，直接向转移对象发送`MsgTimeoutNow`信息即可
3. 如果转移对象在集群里，但是日志不够新，则Leader需要帮助转移对象，也就是向它发送一条Append信息

​	接下来看一下`MsgTimeoutNow`这个信息的处理，其实很简单，首先需要判断当前节点是否在集群里，如果在的话直接Step一个MsgHup信息开启新一轮选举即可

​	最后，根据文档里的提示，这块也有一些小细节需要改，如下

1. 前面提到在领导人转移的时候如果转移对象的日志不够新需要由Leader帮助转移对象.这里其实还需要考虑什么时候继续进行上面的领导权转移.其实只需要在MsgAppendResponse信息的时候看一下发送方和当前Lead的leadTransfer是否相同就行，如果相同的话就发一条MsgTransferLeader，继续尝试领导人转移
2. 在Propose的时候，如果正在进行领导人转移，就拒绝这次操作

​	接下来要实现成员变更，在Project3a这部分要实现的是很简单的，只需要增加对PendingIndex的处理以及实现两个函数即可，如下

1. `PendingConfIndex`，根据定义，只需要在Propose阶段进行更新即可
2. `addNode`，这个方法很简单，只需要增加Prs中的一个量即可
3. `removeNode`，这个方法也很简单，需要注意的是在删除后应该尝试提交日志，因为删除一个节点之后有可能可以推进日志提交的进度

### 测试与分析

​	这一部分基本也是面向测试集编程，主要遇到了如下问题

1. 删除一个节点之后有可能会导致有的日志可以提交，应该重新尝试，这个其实我最开始仅仅只是在Prs里删除了节点
2. 如果一个Follower接收到了一个TransferLeader，应该转发给自己的Leader，这个感觉有点奇怪，因为文档里说TransferLeader是一个local信息，不会通过网络传播

​	最终成功通过了这部分的测试

![image-20220820090950792](https://cdn.jsdelivr.net/gh/ysjyx7/img@main/image-20220820090950792.png)

## Project3b

### 主要任务

​	这里需要在Raftstore这一层实现对领导人转移和成员变更的支持，同时需要实现region split

### 介绍与思路

​	这一部分需要实现的信息全部被封装在AdminRequest中，整体的一个流程和之前是类似的

​	首先先来看一下对于TransferLeader的支持，由于TransferLeader是本地信息，不会提交到别的节点，所以其实在收到这样一个请求之后，直接调用RaftGroup的TransferLeader函数即可，然后进行回复就行，整个调用链也很清晰

​	然后看一下对于成员变更的支持，这里涉及到了一些新的量，如下

```go
type RegionEpoch struct {
	// Conf change version, auto increment when add or remove peer
	ConfVer uint64 `protobuf:"varint，1，opt，name=conf_ver，json=confVer，proto3" json:"conf_ver，omitempty"`
	// Region version, auto increment when split or merge
	Version              uint64   `protobuf:"varint，2，opt，name=version，proto3" json:"version，omitempty"`
	XXX_NoUnkeyedLiteral struct{} `json:"-"`
	XXX_unrecognized     []byte   `json:"-"`
	XXX_sizecache        int32    `json:"-"`
}
type ConfChange struct {
	ChangeType ConfChangeType `protobuf:"varint，1，opt，name=change_type，json=changeType，proto3，enum=eraftpb.ConfChangeType" json:"change_type，omitempty"`
	// node will be add/remove
	NodeId               uint64   `protobuf:"varint，2，opt，name=node_id，json=nodeId，proto3" json:"node_id，omitempty"`
	Context              []byte   `protobuf:"bytes，3，opt，name=context，proto3" json:"context，omitempty"`
	XXX_NoUnkeyedLiteral struct{} `json:"-"`
	XXX_unrecognized     []byte   `json:"-"`
	XXX_sizecache        int32    `json:"-"`
}
```

​	前面的RegionEpoch大概就是用来标志进行成员变更和后面的Splitregion的版本的，后面的ConfChange存放配置变更的相关信息，其中Context存放待处理的Peer的编码

​	下面说一下成员变更的大体流程

1. 传过来一个ConfChange的信息，这里调用ProposeConfChange进行Propose，其他的和之前写的处理方式一样，这就是在proposeRaftCommand函数处的修改，需要注意的是只有AppliedIndex大于等于PendConfIndex的时候才允许进行ConfChange，这是之前的注释里提到的
2. Apply这条信息，需要做如下工作
   - 如果是AddNode的话，首先查一下待添加Peer是否已经存在，如果不存在就继续进行下面的步骤
     1. 对刚才编码的Context进行解码得到Peer的信息
     2. 修改Peers，具体就是增加这个新的Peer
     3. 修改regionepoch，storemeta等信息，同时写入到底层存储中
     4. 调用insertPeercache，这个Peercache其实就是在发送信息的时候用来获取storeid的
     5. 另外，
   - 如果是RemoveNode的话，首先查一下待删除Peer是否已经存在，如果不存在就继续进行下面的步骤
     1. 如果要删除的是自己的话直接调用destroyPeer即可
     2. 其余整个流程和AddNode基本是完全镜像的，此处掠过
   - 最后调用ApplyConfChange进行应用即可

​	最后说一下Split Region。

​	简单来说，Split Region就是在一个store上的region过大时对其进行分裂，首先重新看一下Region的定义

```go
type Region struct {
	Id uint64 `protobuf:"varint，1，opt，name=id，proto3" json:"id，omitempty"`
	// Region key range [start_key， end_key).
	StartKey             []byte       `protobuf:"bytes，2，opt，name=start_key，json=startKey，proto3" json:"start_key，omitempty"`
	EndKey               []byte       `protobuf:"bytes，3，opt，name=end_key，json=endKey，proto3" json:"end_key，omitempty"`
	RegionEpoch          *RegionEpoch `protobuf:"bytes，4，opt，name=region_epoch，json=regionEpoch" json:"region_epoch，omitempty"`
	Peers                []*Peer      `protobuf:"bytes，5，rep，name=peers" json:"peers，omitempty"`
	XXX_NoUnkeyedLiteral struct{}     `json:"-"`
	XXX_unrecognized     []byte       `json:"-"`
	XXX_sizecache        int32        `json:"-"`
}
```

​	其中StartKey和EndKey就是Region所负责的数据，而Split Region消息提供了一个SplitKey，通过这个Key来将原本的区域进行切分

​	先来看一下触发split region的流程

1. 首先在`./kv/raftstore/peer_msg_handler.go`中，由`OnTick`函数驱动逻辑时钟，进而调用`OnSplitRegionCheckTick`函数
2. 进行一系列SplitRegionCheck，通过简单的Check之后向`splitCheckTaskSender`发送一个SplitCheckTask
3. 该信息被发到`./kv/raftstore/runner/split_check.go`中进行SplitCheck，通过之后发送一个`MsgTypeSplitRegion`
4. 该信息传到 `peer_msg_handler.go` ， 然后`HandleMsg()` 方法调用 `onPrepareSplitRegion()`，发送一个 `SchedulerAskSplitTask` 请求到 `scheduler_task.go` 中，申请分配相应的新region id和peer id。然后就会发一个Splitregionrequest

​	然后看一下我们需要增加的东西

1. 首先在Propose阶段，和之前基本是一样的，不同的是需要检查一下splitkey是否在目标region里，因为目标region在这段时间里可能发生了分裂
2. 在Apply阶段，大致流程如下
   - 根据文档，首先需要做三项检查，即检查一下region是否对应，splitkey是否在region里，regionepoch是否对应，这些检查可以参考peer_msg_handler.go文件中的别的函数，很多地方都进行了这些检查.此外，在后面测试的时候偶尔会出现NewPeerId和当前Peers的数量不一致的情况，这里我选择如果不一致就直接停止这次操作了
   - 创建新的Region和peer，需要调用`CreatePeer`函数
   - 更新原本region的endkey和regionepoch信息，更新storemeta的信息，将新的region和旧的region都插入进去，同时将这些数据写入到底层
   - 把新的peer注册到router，并且启动peer

### 测试与分析

​	这一块出现的问题非常的多，在网上参考了很多别人的文档，参考了他们出现的bug和解决的思路，更改了很多地方，下面简单的说一下遇到的一些bug和一些小坑

1. 在测试TestBasicConfChange时死循环，这是因为如果remove的是自己的话应该在destroyPeer后退出

2. panic: [region 1] 2 meta corruption detected，发现这个错误是在destroyPeer中报的，发现似乎是删除BTree上的节点的时候报错了，随后看了网上的文档，说是需要修改maybeCreatePeer，在最后加上`ReplaceOrInsert`方法，如下

   ```go
   func (d *storeWorker) maybeCreatePeer(regionID uint64， msg *rspb.RaftMessage) (bool， error) {
   	......
   	meta.regionRanges.ReplaceOrInsert(&regionItem{region: peer.Region()})
   	return true， nil
   }
   ```

3. 在peer_storage.go中，applysnapshot时，在清除之前的信息的时候需要先调用ps.isInitialized，因为对于新建的peer，startkey和endkey都为空，需要一个snapshot来获取数据，而如果不调用的话会导致之前的旧数据被全部清空，从而产生各种各样的不一致的问题

4. 在ConfChange的时候有一种情况就是集群里就剩下两个节点并且删除的节点是Leader，然后网络设置的unreliable，Leader给另外一个节点的心跳丢了，然后那个节点没有执行removenode操作，结果Leader自毁了，然后就一直死循环.这时候只需要在propose阶段加入判断即可，也就是如果被移除的刚好是Leader，就拒绝这次propose，然后把领导权给另一个节点就行，客户端会继续发送同一条指令.另外这里也有网上的一个大佬让Leader在自己被remove之前重复发送多个心跳到目标节点，但是我发现在这里我这样做的话有时候会出现requesttimeout的问题，所以没有采用。

5. 在resp的时候，测试函数会对返回消息进行检查，在confchange相关测试很容易出现返回的类型不是snap的情况，在split相关测试里，大部分时候返回的resp的response字段压根没有东西，后续重写了callback的逻辑，解决了这一问题

6. 报错region is not split.其实不是很清楚为什么报了这个错，split命令没有传到我这一层，应该是最开始splitcheck的时候就没有通过，后来改了不少地方，基本也没出现过这个错误了

7. 日志错误.最开始apply的时候是把所有的待Apply的entry全部处理过之后再写入的，随后改成了每处理一个entry就写入一次

8. no region。这个问题也经常遇到，大概的原因就是cluster向PD请求region信息的时候没有得到响应，具体解决方法就是在split之后直接发送一个heartbeat task，如下

   ```go
   func (d *peerMsgHandler) sendHeartbeat(region *metapb.Region， peer *peer) {
   	tempRegion := &metapb.Region{}
   	err := util.CloneMsg(region， tempRegion)
   	if err != nil {
   		panic(err)
   	}
   	d.ctx.schedulerTaskSender <- &runner.SchedulerRegionHeartbeatTask{
   		Region:          tempRegion，
   		Peer:            peer.Meta，
   		PendingPeers:    peer.CollectPendingPeers()，
   		ApproximateSize: peer.ApproximateSize，
   	}
   }
   ```

​	最后通过记录如下

![image-20220821085320937](https://cdn.jsdelivr.net/gh/ysjyx7/img@main/image-20220821085320937.png)

## Project3c

### 主要任务

​	这一部分需要实现一个上层的调度器.这一部分只需要实现收集心跳的函数和一个转移region的scheduler.

​	对于scheduler，包含了一个集群的各种信息，每个region都会定期向scheduler发送心跳，具体可以参见peer_msg_handler.go中的Ontick函数.scheduler通过心跳来获取集群的全新信息，这就是我们需要实现的第一个函数的功能.要实现的第二个函数主要是要对split的region进行调动，将其转移到合适的store中

### 介绍与思路

​	这一部分官方给的文档是非常详细的，只需要跟着文档一点一点写就行了

​	首先先看一下需要实现的`processRegionHeartbeat`函数，参数定义如下

```go
type RegionInfo struct {
	meta            *metapb.Region
	learners        []*metapb.Peer
	voters          []*metapb.Peer
	leader          *metapb.Peer
	pendingPeers    []*metapb.Peer
	approximateSize int64
}
```

​	RegionInfo包含了一个心跳中的全部信息	

​	根据文档，可以梳理出下面的具体实现流程

1. 判断是否能够利用这个RegionInfo来更新scheduler的信息，具体流程如下
   1. 如果当前scheduler中有Region，对比RegionInfo中的Regionepoch和scheduler中的regionepoch，如果传进来的是过时的心跳就返回Err
   2. 如果当前scheduler中没有Region，则需要扫描所有和心跳中的region负责的范围有重叠的 region，同样还是检查regionepoch是否过时，如果都没有，就可以接受传来的心跳，进入下一个阶段
2. 由于文档中说冗余更新不会影响正确性，这里就直接经过上面检查之后直接进行更新了，具体就按照文档中说的调用PutRegion和updatestorestatus即可

​	然后看一下要实现的第二个函数`Schedule`，具体流程如下

1. 获取`suitableStore`，主要就是选出其Downtime小于maxDowntime的store，并对这些store按regionsize从大到小排序
2. 遍历`suitableStore`，以此使用GetPendingRegionsWithLock，GetFollowersWithLock，GetLeadersWithLock直到获取到suitableRegion
3. 接下来找到要转移到的store，这次从小到大遍历到一个suitableregion不在的store
4. 看一下suitableStore和targetStore的regionsize的关系，应该满足相差大于等于suitable的ApproxiamateSize才行
5. 创建新的Peer和MovePeeropt，并返回

​	简单来说，第二个函数其实就是把regionsize最大的store中的一个合适的region放到regionsize最小的store中.

### 测试与分析

​	有一个文档里没提到的点，就是regionstore的数目不能小于maxreplicas，加一个特判即可，最终成功通过测试

![image-20220820121905254](https://cdn.jsdelivr.net/gh/ysjyx7/img@main/image-20220820121905254.png)