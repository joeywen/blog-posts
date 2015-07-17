# 关于分布式程序 java的内存管理浅谈

标签（空格分隔）： 分布式 内存管理  java

---

##Preface

当前全球正处于互联网时代，是个信息大爆炸时代。对于商家来说，每一天信息都是宝贵的，都可以转换成money的。所以对数据的处理要求也变的越来越严格，从以前的hadoop/MapReduce 的离线处理，到现在的准实时和实时处理，都是由数据需求而引起的技术革命。数据的处理快慢取决于很多因素。现在主流的解决方法，像Spark，Flink，Pular，包括腾讯，阿里，百度的诸多为开源的框架都是基于分布式内存的大数据处理计算框架，所以内存的使用变的很关键。由于各个框架都是基于JVM语言开发的，所以JVM的内存管理问题被提上日程。

##Java AutoBoxing and UnBoxing
提到Java大家都会想到AutoBoxing和UnBoxing，这项技术最早是在jdk1.5引入进来的。AutoBoxing是java编译器的自动转换过程，针对primitive和与之对应的object wrapper类之间的自动转换，例如把int转换为Integer，double转换为Double等等。反之呐就是Unboxing。
虽然Boxing技术给开发人员带来了很多方便，但是也带来了问题------内存的问题

所有的primitive 的Object 类型都是继承于Object类的，如果调用new Object()， object没有任何其他可存储的成员，那么它也会占用8个字节的内存。

大家都知道C语言中字节是4字节对齐的，那么java也是字节对齐的，不过是8字节对齐的。类的成员变量声明的先后顺序也能影响该类所占的内存大小。

在现今的大数据处理时代，为了能够提高实时性，很多数据直接在内存中不落地的，所以内存占用很大，也会造成一定效率问题。

这个autoboxing的问题很早之前就已经有人提出来了，有个开源的lib叫fastutil是针对java的autoboxing和unboxing做了优化的。读者可以自行下载源码看看。

## Programming based on Memory
上学时图书馆有本书貌似叫《Programming based on Limited Memory》，内容讲的是在移动设备上编程，很老的一本书，那个时候的移动设备内存最大才1M，一般都是512K居多，现在这个Programming based on Limited Memory概念仍然是需要的。

之前Spark提出来基于内存RDD的计算模式，在其documents上对于Memory Tuning有几条建议：
    >  There are three considerations in tuning memory usage: the amount of memory used by your    objects (you may want your entire dataset to fit in memory), the cost of accessing those objects, and the overhead of garbage collection (if you have high turnover in terms of objects).
    >  
    > By default, Java objects are fast to access, but can easily consume a factor of 2-5x more space than the “raw” data inside their fields. This is due to several reasons:
    >> - Each distinct Java object has an “object header”, which is about 16 bytes and contains information such as a pointer to its class. For an object with very little data in it (say one Int field), this can be bigger than the data.
    >> - Java Strings have about 40 bytes of overhead over the raw string data (since they store it in an array of Chars and keep extra data such as the length), and store each character as two bytes due to String’s internal usage of UTF-16 encoding. Thus a 10-character string can easily consume 60 bytes.
    >> - Common collection classes, such as HashMap and LinkedList, use linked data structures, where there is a “wrapper” object for each entry (e.g. Map.Entry). This object not only has a header, but also pointers (typically 8 bytes each) to the next object in the list.
    >> - Collections of primitive types often store them as “boxed” objects such as java.lang.Integer.
    
摘自[Tuning Spark](http://spark.apache.org/docs/latest/tuning.html)

还有一点关于In-Process Memory 和Distributed Memory的对比：

 1. Consistency
    > - ***In-Process Memory***: While using an in-process cache, your cache elements are local to a single instance of your application. Many medium-to-large applications, however, will not have a single application instance as they will most likely be load-balanced. In such a setting, you will end up with as many caches as your application instances, each having a different state resulting in inconsistency. State may however be eventually consistent as cached items time-out or are evicted from all cache instances.
    > - ***Distributed caches***, although deployed on a cluster of multiple nodes, offer a single logical view (and state) of the cache. In most cases, an object stored in a distributed cache cluster will reside on a single node in a distributed cache cluster. By means of a hashing algorithm, the cache engine can always determine on which node a particular key-value resides. Since there is always a single state of the cache cluster, it is never inconsistent.
    > - ***Commnets*** If you are caching immutable objects, consistency ceases to be an issue. In such a case, an in-process cache is a better choice as many overheads typically associated with external distributed caches are simply not there. If your application is deployed on multiple nodes, you cache mutable objects and you want your reads to  always be consistent rather than eventually consistent, a distributed cache is the way to go.

2.Overheads
    >> - ***In-Process Memory***, This dated but very descriptive article describes how an in-process cache can negatively effect performance of an application with an embedded cache primarily due to garbage collection overheads. Your results however are heavily dependent on factors such as the size of the cache and how quickly objects are being evicted and timed-out.
    >> - ***Distributed caches***, A distributed cache will have two major overheads that will make it slower than an in-process cache (but better than not caching at all): network latency and object serialization
    >> - ***Commnets*** , As described earlier, if you are looking for an always-consistent global cache state in a multi-node deployment, a distributed cache is what you are looking for (at the cost of performance that you may get from a local in-process cache).
    

3.Reliability
    > - ***In-Process Memory***, An in-process cache makes use of the same heap space as your program so one has to be careful when determining the upper limits of memory usage for the cache. If your program runs out of memory there is no easy way to recover from it.
    > - ***Distributed caches***,A distributed cache runs as an independent processes across multiple nodes and therefore failure of a single node does not result in a complete failure of the cache. As a result of a node failure, items that are no longer cached will make their way into surviving nodes on the next cache miss. Also in the case of distributed caches, the worst consequence of a complete cache failure should be degraded performance of the application as opposed to complete system failure.    
    > - ***Commnets***, An in-process cache seems like a better option for a small and predictable number of frequently accessed, preferably immutable objects. For large, unpredictable volumes, you are better off with a distributed cache.

摘自 In-Process Caching vs. Distributed Caching

## Open Project Solutions
1. Apache Flink 利用Java的Unsafe API来自行管理程序内存
    ![此处输入图片的描述][1]
    详细请看 [Juggling with Bits and Bytes](http://flink.apache.org/news/2015/05/11/Juggling-with-Bits-and-Bytes.html)，代码可以去github上去clone一下，关于内存管理这块在flink-runtime module里

2. 开源的Lib，fastutil： [http://fastutil.di.unimi.it/](http://fastutil.di.unimi.it/)

3. Facebook Presto: Slice(这是facebook利用github上的一个牛人开源项目) [github](https://github.com/airlift/slice) 


>>>上述如有不对的地方还请指正，小弟将不胜感激。 转载请注明出处，谢谢

##Reference

 1. [Autoboxing and Unboxing](https://docs.oracle.com/javase/tutorial/java/data/autoboxing.html)
 2. [Dalvik和Java字节码的对比](http://www.importnew.com/596.html)
 3. [一个Java对象到底占多大内存？](http://www.importnew.com/14948.html)
 4. [Java对象内存结构](http://www.importnew.com/1305.html)
 5. [Facebook Presto presentation](http://www.slideshare.net/GuorongLIANG/facebook-presto-presentation?qid=a0e2e63f-2615-4234-bf94-b5ccd3581868&v=qf1&b=&from_search=26)
 6. [Presto 在美团](http://www.slideshare.net/chenchunss/presto-33806651?qid=a0e2e63f-2615-4234-bf94-b5ccd3581868&v=qf1&b=&from_search=29)
 7. [Understanding Presto](http://www.slideshare.net/frsyuki/understanding-presto-presto-meetup-tokyo-1?qid=a0e2e63f-2615-4234-bf94-b5ccd3581868&v=qf1&b=&from_search=1)
 8. [Presto Meetup @ Facebook (2014-05-14)](http://www.slideshare.net/dain1/presto-meetup-20140514-34731104?qid=a0e2e63f-2615-4234-bf94-b5ccd3581868&v=qf1&b=&from_search=9)


  [1]: http://flink.apache.org/img/blog/memory-mgmt.png