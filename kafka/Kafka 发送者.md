# Kafka 发送者

重点需要讲一下Sender这个类，这个类本质上是一个runnable类。看一下sender类在producer中被构造时传入的参数，作为我们分析的入口。

```java
this.sender = new Sender(logContext, // 日志封装
        client, // 封装与broker的通信的网络客户端
        this.metadata, // 
        this.accumulator, // 消息数据缓冲池，作为消息传递的媒介，供两个线程通信
        maxInflightRequests == 1, // 标识消息是否保持完全有序的布尔值，通过maxInflightRequests值是否为1判断，这个值之后会说到
        config.getInt(ProducerConfig.MAX_REQUEST_SIZE_CONFIG), // 请求的最大大小，默认为1M
        acks,
        retries,
        metricsRegistry.senderMetrics, 
        Time.SYSTEM,
        this.requestTimeoutMs,
        config.getLong(ProducerConfig.RETRY_BACKOFF_MS_CONFIG),
        this.transactionManager, // 全局的事务管理器，默认情况下为null
        apiVersions); // 全局的版本控制
String ioThreadName = NETWORK_THREAD_PREFIX + " | " + clientId;
this.ioThread = new KafkaThread(ioThreadName, this.sender, true); // 构建线程并启动
this.ioThread.start();
```

既然是一个Runnable的实现类，所以我们直接关注到run方法

```java
public void run() {
    log.debug("Starting Kafka producer I/O thread.");

    // main loop, runs until close is called
    // 在没被主动close的情况下，持续永真循环
    while (running) {
        try {
            run(time.milliseconds());
        } catch (Exception e) {
            log.error("Uncaught error in kafka producer I/O thread: ", e);
        }
    }

    log.debug("Beginning shutdown of Kafka producer I/O thread, sending remaining records.");
}
```

sender流程

1. 当线程运行时，始终循环run方法
2. 当线程中断，但非forceClose时，处理剩余请求
3. 当forceClose时，直接中断



为了便于梳理整体流程，先省略transactionManager事务管理的部分，所以代码简略如下

```java
void run(long now) {
    long pollTimeout = sendProducerData(now);
    client.poll(pollTimeout, now);
}
```

我们先看到sendProducerData方法的上半部分。

```java
private long sendProducerData(long now) {
    Cluster cluster = metadata.fetch();

    // get the list of partitions with data ready to send
    RecordAccumulator.ReadyCheckResult result = this.accumulator.ready(cluster, now);

    // if there are any partitions whose leaders are not known yet, force metadata update
    if (!result.unknownLeaderTopics.isEmpty()) {
      // The set of topics with unknown leader contains topics with leader election pending as well as
      // topics which may have expired. Add the topic again to metadata to ensure it is included
      // and request metadata update, since there are messages to send to the topic.
      for (String topic : result.unknownLeaderTopics)
        this.metadata.add(topic);

      log.debug("Requesting metadata update due to unknown leader topics from the batched records: {}", result.unknownLeaderTopics);

      this.metadata.requestUpdate();
    }

  	... ...
    return pollTimeout;
}
```



- 获取最新的cluster信息
- 获取消息缓存池中准备好发送的消息以及对应的topic和partition

直接看到如何和消息缓存池中沟通

```java
public ReadyCheckResult ready(Cluster cluster, long nowMs) {
    Set<Node> readyNodes = new HashSet<>();
    long nextReadyCheckDelayMs = Long.MAX_VALUE;
    Set<String> unknownLeaderTopics = new HashSet<>();

    boolean exhausted = this.free.queued() > 0;
    for (Map.Entry<TopicPartition, Deque<ProducerBatch>> entry : this.batches.entrySet()) {
        TopicPartition part = entry.getKey();
        Deque<ProducerBatch> deque = entry.getValue();

        Node leader = cluster.leaderFor(part);
        synchronized (deque) {
            if (leader == null && !deque.isEmpty()) {
                // This is a partition for which leader is not known, but messages are available to send.
                // Note that entries are currently not removed from batches when deque is empty.
                unknownLeaderTopics.add(part.topic());
            } else if (!readyNodes.contains(leader) && !isMuted(part, nowMs)) {
                ProducerBatch batch = deque.peekFirst();
                if (batch != null) {
                    long waitedTimeMs = batch.waitedTimeMs(nowMs);
                    boolean backingOff = batch.attempts() > 0 && waitedTimeMs < retryBackoffMs;
                    long timeToWaitMs = backingOff ? retryBackoffMs : lingerMs;
                    boolean full = deque.size() > 1 || batch.isFull();
                    boolean expired = waitedTimeMs >= timeToWaitMs;
                    boolean sendable = full || expired || exhausted || closed || flushInProgress();
                    if (sendable && !backingOff) {
                        readyNodes.add(leader);
                    } else {
                        long timeLeftMs = Math.max(timeToWaitMs - waitedTimeMs, 0);
                        // Note that this results in a conservative estimate since an un-sendable partition may have
                        // a leader that will later be found to have sendable data. However, this is good enough
                        // since we'll just wake up and then sleep again for the remaining time.
                        nextReadyCheckDelayMs = Math.min(timeLeftMs, nextReadyCheckDelayMs);
                    }
                }
            }
        }
    }

    return new ReadyCheckResult(readyNodes, nextReadyCheckDelayMs, unknownLeaderTopics);
}
```

