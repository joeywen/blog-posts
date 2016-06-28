## Optimization
1. Esper跑在JVM上，需要tuning jvm , 使用time window或batch时，占用内存需要考虑一下
2. The processing of output events that your listener or subscriber performs temporarily blocks the thread until the processing completes, and may thus reduce throughput.
3.
