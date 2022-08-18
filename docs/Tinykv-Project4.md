# Project4

​	这一部分需要读一篇新的论文，学习Percolator，并且在Project4a中实现MVCC，在Project4b和Project4c中实现一些API，这一部分基本也是面向论文和测试集编程。

## Project4a

### 主要任务

​	这一部分需要实现MVCC，编写的代码位于`./kv/transaction/mvcc/transaction.go`，需要编写MVCCTxn结构体的几个方法

### 介绍与思路

​	先看一下MvccTxn结构体的定义

```go
type MvccTxn struct {
	StartTS uint64
	Reader  storage.StorageReader
	writes  []storage.Modify
}
```

​	MvccTxn结构体中的成员含义也比较显然，`StartTS`就是论文中的时间戳，而`Reader`和`write`和Project1中的含义应该是类似的，前者用来读取数据，后者用来暂存做的更改，便于以后对这些更改进行处理。

​	另外，在Tinykv中使用三个列族，如下

- `CfDefault`，保存用户值，或者说暂存key对应value值，由相关机制决定后续是Commit还是回滚
- `CfLock`，保存锁，如果某个key存在lock，则说明被上锁
- `CfWrite`，用来记录更改

​	还有一些别的概念，如下

- 编解码，这个我们在之前已经用到过Marshal函数去进行编码，这里也是类似的道理。将用户密钥和时间戳混合编码
- 存储顺序，存储顺序是按Key升序，如果Key相同则按ts降序

​	下面看一下这一部分需要实现的几个方法

- `PutWrite`，`PutLock`，`DeleteLock`，`PutValue`，`DeleteValue`，这几个函数都差不多，都是对相应的列族进行修改，更具体一点其实就是和Project1一样直接在writes数组后面附加新的项即可，注意相应列族需要对应，同时有的项需要进行编码

- `GetValue`，`GetLock`，`CurrentWrite`，`MostRecentWrite`，这几个函数都是需要借助Reader读取数据，同时也需要借助IterCF()，Seek()，Valid()，Next()等等函数进行遍历，下面依次去介绍

  - `GetValue`，这个函数是需要获取在这个事物开始前已提交的key对应的最新的value，因此我们不能通过遍历CfDefault去查找想要的值，因为这个列族里可能有没能提交的value，所以我们需要去遍历CfWrite，大致遍历的代码如下

    ```go
    iter := txn.Reader.IterCF(engine_util.CfWrite)
    for iter.Seek(EncodeKey(key， txn.StartTS)); iter.Valid(); iter.Next() {
    	if bytes.Equal(DecodeUserKey(iter.Item().Key())， key) {
    		break
    	}
    }
    ```

    然后我们再进行解码，并加几次判断即可.比如我们要求查找到的value的类型需要是Put，否则返回nil就行了

  - `GetLock`，这个直接利用GetCF拿到数据即可

  - `CurrentWrite`，这个是当write的StartTS与MvccTxn的StartTS相同并且key相同时返回write，也就是同样的遍历方法，然后先看key是否相同，如果相同再获取Value，解码出StartTS，看看是否相同，相同就直接返回即可

  - `MostRecentWrite`，用来获取给定Key的最新的write，考虑到数据存储是按Key升序ts降序，也就是直接找到第一次和给定Key相同的Key即可返回了。

### 运行与测试

在完成上述内容之后，我们就可以开始测试了，在测试过程中主要出现了如下问题

1. TestGetLock4A，Expected nil, but got: &errors.errorString{s:"mvcc: error parsing lock， not enough input， found 0 bytes"}

   随后发现这是没有对用GetCF获取到的内容是否为空做判断，然后在后续的ParseLock函数中报了错.随后在所有编写的函数中都增加了对得到的内容是否为空的判断

​			最终顺利通过Project4A的所有测试函数	

![image-20220812093615836](https://cdn.jsdelivr.net/gh/ysjyx7/img@main/image-20220812093615836.png)

## Project4B

### 主要任务

​	这一部分需要在Project4A的基础上实现KvGet，KvPrewrite，KvCommit三个请求，代码位于`./kv/server/server.go`，大体流程可以参考论文中的伪代码

### 介绍与思路

​	这一部分编写server结构的三个方法，server结构定义如下

```go
type Server struct {
	storage storage.Storage

	// (Used in 4A/4B)
	Latches *latches.Latches

	// coprocessor API handler， out of course scope
	copHandler *coprocessor.CopHandler
}
```

​	可以看到这个结构体封装了Storage，同时根据文档我们可以知道这里的Latches的作用类似于锁，不过实际实现的时候不管这个Latches好像也没啥事

​	下面依次介绍三个待实现的方法

1. `KvGet`，作用是要获取给定Key对应的Value，参数与返回值定义如下

   ```go
   type GetRequest struct {
   	Context              *Context `protobuf:"bytes，1，opt，name=context" json:"context，omitempty"`
   	Key                  []byte   `protobuf:"bytes，2，opt，name=key，proto3" json:"key，omitempty"`
   	Version              uint64   `protobuf:"varint，3，opt，name=version，proto3" json:"version，omitempty"`
   	XXX_NoUnkeyedLiteral struct{} `json:"-"`
   	XXX_unrecognized     []byte   `json:"-"`
   	XXX_sizecache        int32    `json:"-"`
   }
   type GetResponse struct {
   	RegionError *errorpb.Error `protobuf:"bytes，1，opt，name=region_error，json=regionError" json:"region_error，omitempty"`
   	Error       *KeyError      `protobuf:"bytes，2，opt，name=error" json:"error，omitempty"`
   	Value       []byte         `protobuf:"bytes，3，opt，name=value，proto3" json:"value，omitempty"`
   	// True if the requested key doesn't exist; another error will not be signalled.
   	NotFound             bool     `protobuf:"varint，4，opt，name=not_found，json=notFound，proto3" json:"not_found，omitempty"`
   	XXX_NoUnkeyedLiteral struct{} `json:"-"`
   	XXX_unrecognized     []byte   `json:"-"`
   	XXX_sizecache        int32    `json:"-"`
   }
   ```

   可以知道GetRequest中的Context主要是用来创建Reader的，Version应该就是给定的时间戳，GetReponse中有两个Error，根据文档可以推断出RegionError应该是在创建Reader失败的时候返回的Error，基本对于所有的Error都需要进行判断，逻辑如下

   ```go
   if err != nil {
   		if regionErr， ok := err.(*raft_storage.RegionError); ok {
   			resp.RegionError = regionErr.RequestErr
   			return resp， nil
   		}
   		return nil， err
   	}
   ```

   该方法具体流程如下

   1. 根据请求中的Context创建Reader，然后根据Reader创建一个事务txn
   2. 获取锁，如果有锁并且锁的时间戳小于等于请求的时间戳，说明在此之前有未被提交的请求，则此次请求失败，返回Error，为锁冲突
   3. 获取value，如果获取的value为空的话应该把NotFound置为true

2. `KvPrewrite`，这里就是Percolator的2PC中的第一阶段，主要功能就是对此事物涉及的所有key上锁，过程中需要检查是否有冲突，然后写入value，先看一下涉及到的参数和返回值定义

   ```go
   type PrewriteRequest struct {
   	Context   *Context    `protobuf:"bytes，1，opt，name=context" json:"context，omitempty"`
   	Mutations []*Mutation `protobuf:"bytes，2，rep，name=mutations" json:"mutations，omitempty"`
   	// Key of the primary lock.
   	PrimaryLock          []byte   `protobuf:"bytes，3，opt，name=primary_lock，json=primaryLock，proto3" json:"primary_lock，omitempty"`
   	StartVersion         uint64   `protobuf:"varint，4，opt，name=start_version，json=startVersion，proto3" json:"start_version，omitempty"`
   	LockTtl              uint64   `protobuf:"varint，5，opt，name=lock_ttl，json=lockTtl，proto3" json:"lock_ttl，omitempty"`
   	XXX_NoUnkeyedLiteral struct{} `json:"-"`
   	XXX_unrecognized     []byte   `json:"-"`
   	XXX_sizecache        int32    `json:"-"`
   }
   type Mutation struct {
   	Op                   Op       `protobuf:"varint，1，opt，name=op，proto3，enum=kvrpcpb.Op" json:"op，omitempty"`
   	Key                  []byte   `protobuf:"bytes，2，opt，name=key，proto3" json:"key，omitempty"`
   	Value                []byte   `protobuf:"bytes，3，opt，name=value，proto3" json:"value，omitempty"`
   	XXX_NoUnkeyedLiteral struct{} `json:"-"`
   	XXX_unrecognized     []byte   `json:"-"`
   	XXX_sizecache        int32    `json:"-"`
   }
   type PrewriteResponse struct {
   	RegionError          *errorpb.Error `protobuf:"bytes，1，opt，name=region_error，json=regionError" json:"region_error，omitempty"`
   	Errors               []*KeyError    `protobuf:"bytes，2，rep，name=errors" json:"errors，omitempty"`
   	XXX_NoUnkeyedLiteral struct{}       `json:"-"`
   	XXX_unrecognized     []byte         `json:"-"`
   	XXX_sizecache        int32          `json:"-"`
   }
   ```

   返回值里只有RegionError和Errors，毕竟根据论文里的伪代码，这一步需要返回的也就是是否有冲突.参数中的`PrimaryLock`就是我们选出的那个Primary，Mutation中包含对于每一个Key进行的操作.主要流程如下

   1. 根据请求中的Context创建Reader，然后根据Reader创建一个事务txn
   2. 遍历请求中的所有Mutation，查看是否有冲突，主要需要使用Project4a中实现的`MostRecentWrite`函数，如果查到的最近的Write的时间戳大于等于请求开始的时间戳，就说明发生了冲突，初始化WriteConflict然后附加到当前的Errors数组后即可，然后再查看是否有锁冲突，如果有的话同样也是初始化一个Locked信息，然后附加到当前的Errors数组后即可，如果上面两种冲突都没有发生的话，就根据当前mutation的类型去进行对Value的更改，然后上锁
   3. 上面的循环结束后，如果没有发现Errors的话，就写入此次的操作，调用storage的Write方法即可，最后返回相关的值即可

3. `KvCommit`，这里就是Percolator的2PC中的第二阶段，主要就是删除锁并且写入事物提交记录，主要就是对CFLOCK和CFWRITE两个列族进行操作，涉及到的参数和返回值定义如下

   ```go
   type CommitRequest struct {
   	Context *Context `protobuf:"bytes，1，opt，name=context" json:"context，omitempty"`
   	// Identifies the transaction， must match the start_version in the transaction's
   	// prewrite request.
   	StartVersion uint64 `protobuf:"varint，2，opt，name=start_version，json=startVersion，proto3" json:"start_version，omitempty"`
   	// Must match the keys mutated by the transaction's prewrite request.
   	Keys [][]byte `protobuf:"bytes，3，rep，name=keys" json:"keys，omitempty"`
   	// Must be greater than start_version.
   	CommitVersion        uint64   `protobuf:"varint，4，opt，name=commit_version，json=commitVersion，proto3" json:"commit_version，omitempty"`
   	XXX_NoUnkeyedLiteral struct{} `json:"-"`
   	XXX_unrecognized     []byte   `json:"-"`
   	XXX_sizecache        int32    `json:"-"`
   }
   type CommitResponse struct {
   	RegionError          *errorpb.Error `protobuf:"bytes，1，opt，name=region_error，json=regionError" json:"region_error，omitempty"`
   	Error                *KeyError      `protobuf:"bytes，2，opt，name=error" json:"error，omitempty"`
   	XXX_NoUnkeyedLiteral struct{}       `json:"-"`
   	XXX_unrecognized     []byte         `json:"-"`
   	XXX_sizecache        int32          `json:"-"`
   }
   ```

   具体含义就不多说了，和之前是完全类似的，然后看一下KvCommit的执行流程

   1. 根据请求中的Context创建Reader，然后根据Reader创建一个事务txn
   2. 调用WaitForLatches和ReleaseLatches
   3. 遍历请求中设计到的Keys，对于每一个Key，获取他的Lock，如果没有锁就遍历下一个(事实上这是错的，参考运行与测试部分)，如果锁的时间戳和请求的StartTS不相同的话应该让client重试，因为现在进入到了Commit阶段就表示PrimaryKey已经完成了相关检查，进而要保证数据库以后要提交成功，直接返回即可.然后就可以把此次提交写入到CfWrite，然后删除锁
   4. 最后只需要整体写入到storage中即可

### 运行与测试

主要有下面的问题:

1. 最开始测试的时候会出现各种奇奇怪怪的错误，后来发现是没有对storage进行写入

2. 关于regionerr，最开始不知道怎么处理，参考了server.go中其他地方关于err的处理

3. 在测试函数TestCommitConflictRollback4B和测试函数TestCommitConflictRepeat4B，这两个函数都没有对任何key加锁，那讲道理在KvCommit函数那里应该都是直接返回一个空的resp，但是对于Error这个成员，前者要求非空，后者要求为空，最开始我以为测试函数写错了，后来在Tinykv提交版本里发现是专门有一次提交把原本要求为空的判断改成了非空，看了一下对应的issue，发现如果是回滚的话应该返回一个Error，也就是说，如果没有锁的话，可能是以下两种情况:

   - 这个key已经成功提交，锁是已经被删除了，但是由于种种原因导致没有收到成功提交的信号，这是在重试，或者被抢先提交了，这种情况就不用管了
   - 该事务发生了回滚，这个时候需要返回一个Error

   具体判断是不是回滚的方法也很简单，只需要调用一下CurrentWrite，如果返回的write的类型是回滚的话，就说明这个事务被回滚了.

   感觉文档里好像也没有特别明确的提到过发生回滚时应该返回Error

​	最终成功通过了4B的测试

![image-20220812173137906](https://cdn.jsdelivr.net/gh/ysjyx7/img@main/image-20220812173137906.png)

## Project4C

### 主要任务

​	这一部分要实现`KvScan`，`KvCheckTxnStatue`，`KvBatchRollBack`和`KvResolveLock`四个方法，编写的代码在`./kv/server/server.go`

### 介绍与思路

​	直接介绍一下这四个方法的功能和流程吧

1. `KvScan`，它的功能其实就是批量读取数据，根据文档中的提示，我们需要首先编写`./kv/transaction/mvcc/scanner.go`中的scanner结构体及其方法，另外在扫描时某个Key出现错误不会导致整个流程停止，而是选择去为这个Key记录错误.

   - 先来看一下Scanner的实现，主要需要填充结构体，然后实现`Next`和`Close`方法，关键在于`Next`方法，也就是要返回下一个数据。所以Scanner中要有两个成员，也就是一个迭代器DBIterator，同时也要包含当前事务.进而可以写出NewScanner函数，需要注意的是**我们需要把迭代器定位到第一个可以读取到的数据**，也就是Seek到参数StartKey和txn.StartTS编码到的位置即可.对于Close方法，也很简单，直接调用成员的Close方法即可.Next方法比较复杂，主要功能是要拿到当前kv对，并且让scanner的迭代器指向下一个kv对，需要去考虑我们上面谈到的存储顺序，也就是按Key升序，对于同一个Key，又按时间戳降序.也就是说，在拿到当前的kv对之后，我们需要用循环使得scanner中的迭代器一直指向下一个内容，直到解码出来的key不同才会返回.需要注意的是在这个循环之前我们需要先看一下当前的Key在当前事务的时间戳有没有数据

   - 在实现过Scanner之后，KvScan的实现就非常简单了，先看一下参数和返回值定义

     ```go
     type ScanRequest struct {
     	Context  *Context `protobuf:"bytes，1，opt，name=context" json:"context，omitempty"`
     	StartKey []byte   `protobuf:"bytes，2，opt，name=start_key，json=startKey，proto3" json:"start_key，omitempty"`
     	// The maximum number of values read.
     	Limit                uint32   `protobuf:"varint，3，opt，name=limit，proto3" json:"limit，omitempty"`
     	Version              uint64   `protobuf:"varint，4，opt，name=version，proto3" json:"version，omitempty"`
     	XXX_NoUnkeyedLiteral struct{} `json:"-"`
     	XXX_unrecognized     []byte   `json:"-"`
     	XXX_sizecache        int32    `json:"-"`
     }
     type ScanResponse struct {
     	RegionError *errorpb.Error `protobuf:"bytes，1，opt，name=region_error，json=regionError" json:"region_error，omitempty"`
     	// Other errors are recorded for each key in pairs.
     	Pairs                []*KvPair `protobuf:"bytes，2，rep，name=pairs" json:"pairs，omitempty"`
     	XXX_NoUnkeyedLiteral struct{}  `json:"-"`
     	XXX_unrecognized     []byte    `json:"-"`
     	XXX_sizecache        int32     `json:"-"`
     }
     
     type KvPair struct {
     	Error                *KeyError `protobuf:"bytes，1，opt，name=error" json:"error，omitempty"`
     	Key                  []byte    `protobuf:"bytes，2，opt，name=key，proto3" json:"key，omitempty"`
     	Value                []byte    `protobuf:"bytes，3，opt，name=value，proto3" json:"value，omitempty"`
     	XXX_NoUnkeyedLiteral struct{}  `json:"-"`
     	XXX_unrecognized     []byte    `json:"-"`
     	XXX_sizecache        int32     `json:"-"`
     }
     ```

     这个limit就是读到的最大的数据量，其他的成员的含义都很明显了.大概流程如下

     1. 根据请求中的Context创建Reader，然后根据Reader创建一个事务txn，然后创建scanner
     2. 循环读取limit个数据，每次循环都调用Next方法获取k，v，err，然后进行相关的判断，和Project4B中实现的KvGet类似，这里就不多说了

2. `KvCheckTxnStatus`，该函数事务状态，并且处理一些回滚超时的锁。首先还是先看一下参数和返回值定义

   ```go
   type CheckTxnStatusRequest struct {
   	Context              *Context `protobuf:"bytes，1，opt，name=context" json:"context，omitempty"`
   	PrimaryKey           []byte   `protobuf:"bytes，2，opt，name=primary_key，json=primaryKey，proto3" json:"primary_key，omitempty"`
   	LockTs               uint64   `protobuf:"varint，3，opt，name=lock_ts，json=lockTs，proto3" json:"lock_ts，omitempty"`
   	CurrentTs            uint64   `protobuf:"varint，4，opt，name=current_ts，json=currentTs，proto3" json:"current_ts，omitempty"`
   	XXX_NoUnkeyedLiteral struct{} `json:"-"`
   	XXX_unrecognized     []byte   `json:"-"`
   	XXX_sizecache        int32    `json:"-"`
   }
   
   type CheckTxnStatusResponse struct {
   	RegionError *errorpb.Error `protobuf:"bytes，1，opt，name=region_error，json=regionError" json:"region_error，omitempty"`
   	// Three kinds of txn status:
   	// locked: lock_ttl > 0
   	// committed: commit_version > 0
   	// rolled back: lock_ttl == 0 && commit_version == 0
   	LockTtl       uint64 `protobuf:"varint，2，opt，name=lock_ttl，json=lockTtl，proto3" json:"lock_ttl，omitempty"`
   	CommitVersion uint64 `protobuf:"varint，3，opt，name=commit_version，json=commitVersion，proto3" json:"commit_version，omitempty"`
   	// The action performed by TinyKV in response to the CheckTxnStatus request.
   	Action               Action   `protobuf:"varint，4，opt，name=action，proto3，enum=kvrpcpb.Action" json:"action，omitempty"`
   	XXX_NoUnkeyedLiteral struct{} `json:"-"`
   	XXX_unrecognized     []byte   `json:"-"`
   	XXX_sizecache        int32    `json:"-"`
   }
   type Action int32
   
   const (
   	Action_NoAction Action = 0
   	// The lock is rolled back because it has expired.
   	Action_TTLExpireRollback Action = 1
   	// The lock does not exist， TinyKV left a record of the rollback， but did not
   	// have to delete a lock.
   	Action_LockNotExistRollback Action = 2
   )
   ```

   这部分的注释给了三种事务状态的定义:

   - `locked`，LockTtl>0
   - `committed`，CommitVersion>0
   - `rolled back`，LockTtl==0&&CommitVersion==0

   然后看一下这个函数的流程，其实注释里也已经说的很明白了

   1. 根据请求中的Context创建Reader，然后根据Reader创建一个事务txn
   2. 利用CurrentWrite函数获取事务开始时间戳时候的write，如果他的类型不是回滚的话，就说明已提交，直接返回相应信息即可
   3. 否则获取PrimaryKey的锁
   4. 如果没有上锁，则说明由于某些原因导致锁被移除了，但是我们还没有作出相应的反映.这个时候我们需要写入一条回滚的write，并且将返回的Action置为kvrpcpb.Action_LockNotExistRollback
   5. 如果已经上了锁，我们需要判断是否发生了超时，根据文档提示需要借助`PhysicalTime`函数，如果超时的话需要删除过时的锁和数据，并且将返回的Action置为kvrpcpb.Action_TTLExpireRollback

3. `KvBatchRollback`，该函数检查key是否被当前事务上锁，如果是，则删除锁，删除相关value并将回滚指示符保留为write。首先还是先看一下参数和返回值

   ```go
   type BatchRollbackRequest struct {
   	Context              *Context `protobuf:"bytes，1，opt，name=context" json:"context，omitempty"`
   	StartVersion         uint64   `protobuf:"varint，2，opt，name=start_version，json=startVersion，proto3" json:"start_version，omitempty"`
   	Keys                 [][]byte `protobuf:"bytes，3，rep，name=keys" json:"keys，omitempty"`
   	XXX_NoUnkeyedLiteral struct{} `json:"-"`
   	XXX_unrecognized     []byte   `json:"-"`
   	XXX_sizecache        int32    `json:"-"`
   }
   type BatchRollbackResponse struct {
   	RegionError          *errorpb.Error `protobuf:"bytes，1，opt，name=region_error，json=regionError" json:"region_error，omitempty"`
   	Error                *KeyError      `protobuf:"bytes，2，opt，name=error" json:"error，omitempty"`
   	XXX_NoUnkeyedLiteral struct{}       `json:"-"`
   	XXX_unrecognized     []byte         `json:"-"`
   	XXX_sizecache        int32          `json:"-"`
   }
   ```

   这部分的参数和返回值定义没有什么好说的，这个操作也是如果一个Key失败，那么所有的Key都失败，不会进行写入，整体函数的流程在注释里也说的很明白了，我们在下面简单梳理一下

   1. 根据请求中的Context创建Reader，然后根据Reader创建一个事务txn
   2. 遍历所有的Key，对于每一个Key，如果已经被提交了，就回滚失败，这部分主要是看CurrentWrite是否存在，如果存在且类型不是回滚就直接返回失败，然后获取锁，如果这个锁是别的事务上的，也是直接返回失败(这里感觉怪怪的，按照注释来说，应该就是直接失败了，但是如果在这里直接返回失败的话，有个测试函数一直过不了，所以只是忽略了锁被别的事务上的这种情况，具体似乎就是不管这个锁了)，具体判断方法就是看时间戳能不能对应上.最后正常来说就是删除锁和相应值并进行回滚即可
   3. 写入到storage中

4. `KvResolveLock`，该方法就是检查一批被锁定的Key，然后将他们全部回滚或全部提交.首先还是先看一下它的参数和返回值

   ```go
   type ResolveLockRequest struct {
   	Context              *Context `protobuf:"bytes，1，opt，name=context" json:"context，omitempty"`
   	StartVersion         uint64   `protobuf:"varint，2，opt，name=start_version，json=startVersion，proto3" json:"start_version，omitempty"`
   	CommitVersion        uint64   `protobuf:"varint，3，opt，name=commit_version，json=commitVersion，proto3" json:"commit_version，omitempty"`
   	XXX_NoUnkeyedLiteral struct{} `json:"-"`
   	XXX_unrecognized     []byte   `json:"-"`
   	XXX_sizecache        int32    `json:"-"`
   }
   type ResolveLockResponse struct {
   	RegionError          *errorpb.Error `protobuf:"bytes，1，opt，name=region_error，json=regionError" json:"region_error，omitempty"`
   	Error                *KeyError      `protobuf:"bytes，2，opt，name=error" json:"error，omitempty"`
   	XXX_NoUnkeyedLiteral struct{}       `json:"-"`
   	XXX_unrecognized     []byte         `json:"-"`
   	XXX_sizecache        int32          `json:"-"`
   }
   ```

   这里也是按照注释中给的逻辑去写就行，关键就在于CommitVersion，为0时回滚全部锁，大于0时提交所有的锁，下面看一下具体实现流程

   1. 根据请求中的Context创建Reader，然后根据Reader创建一个事务txn
   2. 用迭代器遍历，获取所有上锁的Key，注意是被当前事务上锁的Key
   3. 根据CommitVersion决定是全部回滚还是全部提交，需要调用之前实现的KvCommit方法和KvBatchRollback方法.

### 运行与测试

​	在测试过程中主要出现了如下问题

1. 编写scanner时，没有先seek到StartKey的位置

​	最后成功通过了测试

![image-20220812221051353](https://cdn.jsdelivr.net/gh/ysjyx7/img@main/image-20220812221051353.png)
