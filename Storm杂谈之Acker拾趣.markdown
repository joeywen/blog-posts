
# Storm杂谈之Acker拾趣

本文所讲内容并非storm的acker机制，如果想看acker机制的让您失望了，不过在此奉上徐明明大牛的blog：
[Twitter Storm源代码分析之acker工作流程](https://xumingming.sinaapp.com/410/twitter-storm-code-analysis-acker-merchanism/)
[Twitter Storm如何保证消息不丢失](http://xumingming.sinaapp.com/127/twitter-storm%E5%A6%82%E4%BD%95%E4%BF%9D%E8%AF%81%E6%B6%88%E6%81%AF%E4%B8%8D%E4%B8%A2%E5%A4%B1/)

或者查看《[storm源码分析](http://item.jd.com/11567931.html)》（又给京狗打链接）第12章-storm的acker系统，里面会详细说明storm的acker机制，笔者在此就不多述（多述都是废话，还不一定有人家讲的好）了。

这篇主要讲一下，关于开acker和不开acker的区别。
首先说一下，BasicBolt和RichBolt的区别，RichBolt会帮我们自动ack tuple的，basicbolt不会，所以如果继承的是basicBolt的话，就需要自己outputcollecter调ack方法了。

一般Storm有个配置项

```
	/**
     * How many executors to spawn for ackers.
     *
     * <p>If this is set to 0, then Storm will immediately ack tuples as soon
     * as they come off the spout, effectively disabling reliability.</p>
     */
    public static final String TOPOLOGY_ACKER_EXECUTORS = "topology.acker.executors";
```
该配置项是配置Acker Bolt数目的，大于0则spout每发一条msg，都会把相应的信息<rootid, msgid>发送给Acker Bolt进行跟踪。AckerBolt跟踪这个消息进行可靠性处理。

但是如果TOPOLOGY_ACKER_EXECUTORS配置为0的话，bolt和spout之间的ack又会是怎样的呐？？

先看下面的executor.clj中*(defmethod mk-threads :spout [executor-data task-datas]*方法的部分代码

```
pending (RotatingMap.
                 2 ;; microoptimize for performance of .size method
                 (reify RotatingMap$ExpiredCallback
                   (expire [this msg-id [task-id spout-id tuple-info start-time-ms]]
                     (let [time-delta (if start-time-ms (time-delta-ms start-time-ms))]
                       (fail-spout-msg executor-data (get task-datas task-id) spout-id tuple-info time-delta)
                       ))))
        tuple-action-fn (fn [task-id ^TupleImpl tuple]
                          (let [stream-id (.getSourceStreamId tuple)]
                            (condp = stream-id
                              Constants/SYSTEM_TICK_STREAM_ID (.rotate pending)
                              Constants/METRICS_TICK_STREAM_ID (metrics-tick executor-data (get task-datas task-id) tuple)
                              (let [id (.getValue tuple 0)
                                    [stored-task-id spout-id tuple-finished-info start-time-ms] (.remove pending id)]
                                (when spout-id
                                  (when-not (= stored-task-id task-id)
                                    (throw-runtime "Fatal error, mismatched task ids: " task-id " " stored-task-id))
                                  (let [time-delta (if start-time-ms (time-delta-ms start-time-ms))]
                                    (condp = stream-id //这里，根据stream id 来对msg进行ack或fail
                                      ACKER-ACK-STREAM-ID (ack-spout-msg executor-data (get task-datas task-id)
		                                                                         spout-id tuple-finished-info time-delta)
                                      ACKER-FAIL-STREAM-ID (fail-spout-msg executor-data (get task-datas task-id)
                                                                           spout-id tuple-finished-info time-delta)
                                      )))
                                ;; TODO: on failure, emit tuple to failure stream
                                ))))
        receive-queue (:receive-queue executor-data)
        event-handler (mk-task-receiver executor-data tuple-action-fn)
        has-ackers? (has-ackers? storm-conf)
        emitted-count (MutableLong. 0)
        empty-emit-streak (MutableLong. 0)
```
再看acker.clj的execute代码就清晰多了，acker也是根据stream-id来对tuple的msgid进行异或处理的，

```
(^void execute [this ^Tuple tuple]
             (let [^RotatingMap pending (.getObject pending)
                   stream-id (.getSourceStreamId tuple)]
               (if (= stream-id Constants/SYSTEM_TICK_STREAM_ID)
                 (.rotate pending)
                 (let [id (.getValue tuple 0)
                       ^OutputCollector output-collector (.getObject output-collector)
                       curr (.get pending id)
                       curr (condp = stream-id
                                ACKER-INIT-STREAM-ID (-> curr
                                                         (update-ack (.getValue tuple 1))
                                                         (assoc :spout-task (.getValue tuple 2)))
                                ACKER-ACK-STREAM-ID (update-ack curr (.getValue tuple 1))
                                ACKER-FAIL-STREAM-ID (assoc curr :failed true))]
                   (.put pending id curr)
                   (when (and curr (:spout-task curr))
                     (cond (= 0 (:val curr))
                           (do
                             (.remove pending id)
                             (acker-emit-direct output-collector
                                                (:spout-task curr)
                                                ACKER-ACK-STREAM-ID
                                                [id]
                                                ))
                           (:failed curr)
                           (do
                             (.remove pending id)
                             (acker-emit-direct output-collector
                                                (:spout-task curr)
                                                ACKER-FAIL-STREAM-ID
                                                [id]
                                                ))
                           ))
                   (.ack output-collector tuple)
                   ))))
      (^void cleanup [this]
        )
```
那么bolt是怎样调用acker的呐，上面说了，继承richbolt时会自动调用ack方法，那么ack方法到底做了哪些事情呐，按executor.clj中的mk-threads :bolt部分代码就清晰多了

```
(.prepare bolt-obj
                    storm-conf
                    user-context
                    (OutputCollector. //实现了匿名内部类，实现IOutputCollector的emit，emitDirect，ack和fail方法
                     (reify IOutputCollector
                       (emit [this stream anchors values]
                         (bolt-emit stream anchors values nil))
                       (emitDirect [this task stream anchors values]
                         (bolt-emit stream anchors values task))
                       (^void ack [this ^Tuple tuple]
                         (let [^TupleImpl tuple tuple
                               ack-val (.getAckVal tuple)]
                           (fast-map-iter [[root id] (.. tuple getMessageId getAnchorsToIds)]
                                          (task/send-unanchored task-data
                                                                ACKER-ACK-STREAM-ID
                                                                [root (bit-xor id ack-val)])
                                          ))
                         (let [delta (tuple-time-delta! tuple)]
                           (task/apply-hooks user-context .boltAck (BoltAckInfo. tuple task-id delta))
                           (when delta
                             (builtin-metrics/bolt-acked-tuple! (:builtin-metrics task-data)
                                                                executor-stats
                                                                (.getSourceComponent tuple)                                                      
                                                                (.getSourceStreamId tuple)
                                                                delta)
                             (stats/bolt-acked-tuple! executor-stats
                                                      (.getSourceComponent tuple)
                                                      (.getSourceStreamId tuple)
                                                      delta))))
                       (^void fail [this ^Tuple tuple]
                         (fast-list-iter [root (.. tuple getMessageId getAnchors)]
                                         (task/send-unanchored task-data
                                                               ACKER-FAIL-STREAM-ID
                                                               [root]))
                         (let [delta (tuple-time-delta! tuple)]
                           (task/apply-hooks user-context .boltFail (BoltFailInfo. tuple task-id delta))
                           (when delta
                             (builtin-metrics/bolt-failed-tuple! (:builtin-metrics task-data)
                                                                 executor-stats
                                                                 (.getSourceComponent tuple)   
```
从上面可以看出，prepare方法中实现了匿名内部类，实现IOutputCollector的emit，emitDirect，ack和fail方法，然后把OutputCollector传给用户所实现的bolt，即bolt重载prepare方法中的OutputCollector，当用利用OutputCollector调用ack方法时就是调用上面clojure代码中匿名实现的ack方法，该方法会发送一个ACKER-ACK-STREAM-ID 的streamid，然后Spout就可以接收到该stream进行ack操作，这也是你在WebUI上看到spout后面的ack count数目了。

总结一下

 - 开了Acker，只是对发送的消息进行跟踪处理，处理成功或失败会发送stream给spout，spout回调ack或fail方法同时进行metric统计，然后具体的重发逻辑用户可以自己定义。
 - 不开Acker， Bolt调用ack方法也会发送stream给spout，spout接收stream做metric统计。

Spout只是根据Ack stream id来判断是否进行ack或fail，跟具体哪个bolt没有任何关系。所以大家不要认为storm的消息可靠性是来源spout和acker bolt之间的通信，其实不然，Bolt也是会发消息给spout的。

OK，到此结束。
PS：storm版本是0.9.4






