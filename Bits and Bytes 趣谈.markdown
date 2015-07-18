Bits and Bytes 趣谈

现在很多开源框架都是跑在JVM上，如Apache SPark，Apache Drill，还有Apache Flink。基于JVM的数据分析引擎普面临的一个问题就是在内存中要存储大量的数据——为了缓存和更有效的数据处理如数据的排序和join连接运算。然而JVM的内存管理在各个系统上都是一个很大的难题，很难配置，难以预测的可靠性，性能问题等等。

在这篇博客中，我们讨论flink的内存管理，自己实现的序列化/反序列化框架，flink如何操作binary数据

## Data Objects？ Lets put them on the heap ！



原文：http://flink.apache.org/news/2015/05/11/Juggling-with-Bits-and-Bytes.html