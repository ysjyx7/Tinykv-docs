# Project1

## 任务介绍

​	这一部分主要是要实现一个支持列族(Column Family)的KV数据库。

- `列族 ColumnFamily`，列族是HBase中的一个概念，将多个列作为一个整体去进行管理，可以大幅提高读取效率，此外也由于列族中列的数据格式相同，可以进行压缩存储。
- `gRPC`，这是之前没有接触过的东西，在网上简单查了资料看了一下，gRPC是一个RPC框架，对于RPC，大概指的就是一种可以在一个服务器A上的一个应用调用部署在另一个服务器B上的另一个应用提供的方法的这样一个过程。

​	整个服务提供四种操作，即Put/Delete/Get/Scan。

​	我们需要实现以下两部分:

1. 单机存储引擎
2. 原生服务接口

## 实现思路

### Implement standalone storage engine

​	首先，可以参考一下文件中给出的整体的架构图

![image-20220808031336501](https://cdn.jsdelivr.net/gh/ysjyx7/img@main/image-20220808031336501.png)

​	从这个图中可以看到我们要实现的Standalone Storage和已经实现的RaftStorage都是对底层的Engine进行了一层封装，可以认为把RaftStorage中Raft相关的部分去掉就基本是StandaloneStorage，所以这里基本可以完全参照RaftStorage的实现，代码在`./kv/storage/standalone_storage/standalone_storage.go`中实现。

1. 首先需要实现StandAloneStorage结构以及其初始化
2. 然后实现该结构体的四种方法，主要的在于Reader和Write方法，Write方法需要对Modify中的一系列操作进行处理，只需要进行一次遍历即可。
3. 在实现Reader方法时，需要返回storage.StorageReader，这里也是参考着RaftStorage中返回的RegionReader的实现进行的。

​	这一小部分并不复杂，基本都是参考着RaftStorage来的，最后测试的时候也没有出现什么问题

### Implement service handlers

​	这里需要实现我们之前提到过的Put/Delete/Scan/Get四种API，代码在`。/kv/server/raw_api。go`中实现，这四种API是要对我们刚刚实现的Standalone Storage进行操作

1. `Put/Delete`，这两个操作都需要对数据进行更改，实现方法是类似的。由于我们上面实现的Write函数是对Modify中的操作进行的一系列处理，所以这里我们也仅仅只需要将对应的操作放到Modify中然后调用Write函数进行写入即可
2. `Get/Scan`，这两个操作都需要读取数据，也就是需要我们上边实现的Reader函数。对于Get操作，只需要调用GetCF函数进行读取即可。对于Scan操作，实现起来相对复杂一点，可以说是Project1最难的一个操作，需要去读取一系列的数据，注释中也给我们提示了需要借用reader。IterCF，大概流程就是通过Reader获取一个itor，然后从StartKey开始进行读取即可，在engine_util中定义了像Seek()，Value()，Valid()等等比较有用的方法，需要注意读取的范围是由请求中的limit决定的。

## 测试

​	在实现了上述几个部分之后，可以发现我们成功通过了测试。

![image-20220808110136778](https://cdn.jsdelivr.net/gh/ysjyx7/img@main/image-20220808110136778.png)