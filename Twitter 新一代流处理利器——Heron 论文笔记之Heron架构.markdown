# Twitter 新一代流处理利器——Heron 论文笔记之Heron架构

标签（空格分隔）： Streaming-process realtime-process

---

## Heron Architecture
Heron 架构如下图：

用户编写发布topoloy到Aurora调度器。每一个topology都作为一个Aurora的job在运行。每一个job包括几个container，这些container由Aurora来分配和调度。第一个container作为Topology Master，其他的Container作为Stream Manager。所有的元数据信息包括谁提交的job，job的运行信息，启动时间等等都会保存在Zookeeper中。
每个Heron Instance都是用java写的，且都是JVM进程。Heron进程之间用protocol buffers进行通信。

##Heron Instance
值得一提的是每个HI都是只跑一个task，即要么是spout要么是bolt。这样有利于debug。
这种设计也为以后数据的复杂性考虑，当以后数据复杂性变高的时候，我们还可以考虑用其他语言来实现HI。

HI的设计有以下两种：

 - 单线程
 - 双线程
### Single-threaded approach
主线程有一个TCP channel与本地的SM通信，等待tuple的到来，一旦tuple来了，就会调用用户的逻辑代码来执行，如果需要输出，该线程就会缓存数据，直到达到阈值，然后输出到downstream的SM。

这种设计简单，但是也有一些缺点，由于某些原因，用户的逻辑可能被block：

 - Invoking the sleep system call for a finite duration of time
 - Using read/write system calls for file or socket I/O
 - Calling thread synchronization primitives


### Two-threaded approach
顾名思义，两个thread：Gateway thread 和Task Execution thread，如下图：

Gateway thread负责数据的输入输出和通信
Task Execution thread则负责运行用户逻辑代码

Gateway thread要和Task Execution thread要进行数据通信，他们之间通过如上图的三种queue来通信。Gateway thread用data-in往Task Execution thread输入数据，Task Execution thread用data-out往Gateway thread，metrics-out是用Task Execution thread用来收集metric然后往Gateway thread发送。

## Toplogy Master
TM（Topology Master）主要负责topology的throughout，在startup的时候，TM把信息存放在Zookeeper上，以便其他进程能够发现TM。所以TM有如下两个目的：

 - 阻止多个TM的产生
 - 允许其他属于该topology的进程发现该TM

## Topology Backpressure
Heron提供一种背压机制来动态调整数据流动的速率。这种机制可以让topology中的各个components以不同speed来跑。也可以动态更改它的speed。

###TCP Backpressure
这个策略利用TCP窗口的机制来梳理HI（Heron Instance）和其他Componet的背压。所有的消息都是通过TCP sockets来做通信，如果某个HI处理缓慢，那么它的本地接收buffer就会被装满，在这个HI上游和数据通信的SM也会发现这个然后填满发送的buffer，这样该HI的处理速度就加快了。

###Spout backpressure
这个背压策略是和TCP背压策略协同使用的，当SM发现它本地的HI运行慢时，SM就会通知本地的SPout停止读取数据，那么往该Spout发送数据的SM的buffer就会阻塞以致fill up，这是受影响的SM就会发送一条start backpressure的msg到其他与之相连的SM，当其他SM收到该msg时就会告诉他们本地的Spout不再读取数据，当上游缓慢的HI速度赶上来之后，SM再发一个stop backpressure的msg到下游，然后停止backpressure。

>当topoloy处于backpressure模式时，它的运行速度取决于最慢的那个HI。


## Architecture Features: Summary

 1. First, the provisioning of resources (e.g. for containers and even the Topology Master) is cleanly abstracted from the duties of the cluster manager, thereby allowing Heron to “play nice” with the rest of the (shared) infrastructure.
 2. Second, since each Heron Instance is executing only a single task (e.g. running a spout or bolt), it is easy to debug that instance by simply using tools like jstack and heap dump with that process.
 3. Third, the design makes it transparent as to which component of the topology is failing or slowing down, as the metrics collection is granular, and lets us easily map an issue unambiguously to a specific process in the system.
 4. Fourth, by allowing component-level resource allocation, Heron allows a topology writer to specify exactly the resources for each component, thereby avoiding unnecessary over-provisioning.
 5. Fifth, having a Topology Master per topology allows each topology to be managed independently of each other (and other systems in the underlying cluster). In additional, failure of one topology (which can happen as user-defined code often gets run in the bolts) does not impact the other topologies.
 6. Sixth, the backpressure mechanism allows us to achieve a consistent rate of delivering results, and a precise way to reason about the system. It is also a key mechanism that allows migrating topologies from one set of containers to another (e.g. to an upgraded set of machines).
 7. Finally, we now do not have any single point of failure.

## Performance
直接看图吧

  
 
