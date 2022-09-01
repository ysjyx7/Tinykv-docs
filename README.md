# Tinykv-docs
​	Tinykv的四个Project的文档

​	另外Project3其实有个小的bug一直没修，主要是因为本身没有想到什么很好的方法去解决，其实也是因为搞了这么久有点佛了:joy:

​	具体表现如下：在AddNode之后，Peer需要收到一个snapshot进行同步，但是在还没有创建好peer的时候snapshot就发了出去，导致snapshot丢失，然后生成snapshot的时间开销又很大，所以最后就超时了，具体日志上的表现就是一直有一个Peer在Request snapshot，直到超时

---
  2022/8/31
  成绩出来了，最终得分93.25，很满意
