# Twitter 新一代流处理利器——Heron 论文笔记之Storm Limitations

标签（空格分隔）： Streaming-Processing

---
## Storm Problems
scalability, debug-ability, manageability, and efficient sharing of cluster resources with other data services。

##Storm Worker Architecture: Limitations
> 1. Storm的worker就是一个JVM进程，每个worker可以跑多个executor，目前根据Storm现有的调度机制，我们无法确定那个task被分配到了哪个worker上，哪台物理机器上。
> 2. 由于不知道task被分配到哪个worker上，有可能是同一个，考虑join的情况，一个join task和一个output 到 DB Store或其他存储的task被分配到同一个worker，这样性能可能无法保证
> 3. 当前正在跑的topology如果重启的话，之前分派在同一个worker的task由于toplogy重启，可不能不会再被分配到同一个worker上，这给debug带来了困难。
> 4. Storm 提供自己实现的isolate 调度，但是要交于开发人员来分配集群资源是个及其不好的做法。
> 5. 资源分配浪费。Storm假设每个worker都是homogenous，这种做法经常会造成在资源预的超额分配。例如3个spouts和1个bolt，加入每个spout和bolt各自需要5G和10G内存，这样的话，topoogy必须为每个worker预留15G的内存来跑一个spout和一个bolt，如果用户设置worker数为2，那么两个worker就要总共预留30G内存，但是实际上只需要 3*5 + 1 *10 = 25G内存，这样就浪费了5G。
> 6. 如果对一个worker进行heap dump时，可能会阻塞worker hearbeats的发送，导致supervisor认为该worker心跳超时，kill 和重启了该worker
> 7. worker用thread和queue来做tuple的接收和发送，每个worker有一个receive-thread接收上游tuple，一个全局send-thread负责往下游发送tuple，然后executor有一个logic-thread来执行用户的代码逻辑，最后有一个本地的send-thread来做logic-thread和全局send-thread做数据通信，到这里，一个tuple需要从进入一个worker到出来总共要通过4个thread转发。

## Issues with the Storm Nimbus
Storm的NImbus任务很多很艰巨，包括调度，监听，分发JAR等等，topology多的时候，Nimbus将变成瓶颈。
> 1. Nimbus调度器不支持worker细粒度的resource reservation和isolation。不同topology的worker被分配到了同一个物理node上，很有可能会相互影响。
> 2. Storm利用Zookeeper来存储worker和supervisor以及executor的心跳信息。如果topology很多，每个topology的并发很多，这样Zookeeper就是瓶颈。
> 3. 就是老生常谈的nimbus单点故障，Nimbus不是HA。

##Lack of Backpressure
Storm没有backpressure机制，如果下游接收数据的component没有及时处理数据的话，发送者就会drop message。这是一种fail-fast机制，也很简单，但是有以下缺点：

 1. If acknowledgements are disabled, this mechanism will resultin unbounded tuple drops, making it hard to get visibility about these drops.
 2. Work done by upstream components is lost.
 3. System behavior becomes less predictable.

##Efficiency

 - Suboptimal replays
 - Long Garbage Collection cycles
 - Queue contention

未完待续，下次讲述Twitter的新利器——Heron的架构以及是如何解决上述Storm存在的问题的。

##Reference
[Twitter Heron: Stream Processing at Scale ](http://delivery.acm.org/10.1145/2750000/2742788/p239-kulkarni.pdf?ip=140.207.158.107&id=2742788&acc=OPENTOC&key=4D4702B0C3E38B35%2E4D4702B0C3E38B35%2E4D4702B0C3E38B35%2E9F04A3A78F7D3B8D&CFID=680099850&CFTOKEN=82958124&__acm__=1433325314_450661652f71fdf59bb8bc0eafcc8806)
 
 
