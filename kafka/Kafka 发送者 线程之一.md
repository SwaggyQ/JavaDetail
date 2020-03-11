# Kafka 发送者 线程之一

省略key和value的serial过程，直接看到KafkaProducer的doSend方法的以下代码

```java
int partition = partition(record, serializedKey, serializedValue, cluster);
TopicPartition tp = new TopicPartition(record.topic(), partition);
RecordAccumulator.RecordAppendResult result = accumulator.append(tp, timestamp, serializedKey,
        serializedValue, headers, interceptCallback, remainingWaitMs);
```

accmulator看名字可以知道是一个累加器，其实就是在producer中的消息的累加器，一个线程往其中写数据，另外一个sender线程从中批量获取数据。sender线程后面会重点说明，本文重点看一下往其中累计记录的过程。继续往下看代码。

```java
public RecordAppendResult append(TopicPartition tp,
                                 long timestamp,
                                 byte[] key,
                                 byte[] value,
                                 Header[] headers,
                                 Callback callback,
                                 long maxTimeToBlock) throws InterruptedException {
    try {
        // check if we have an in-progress batch
        Deque<ProducerBatch> dq = getOrCreateDeque(tp);
        synchronized (dq) {
            if (closed)
                throw new KafkaException("Producer closed while send in progress");
            RecordAppendResult appendResult = tryAppend(timestamp, key, value, headers, callback, dq);
            if (appendResult != null)
                return appendResult;
        }
    }
}

private Deque<ProducerBatch> getOrCreateDeque(TopicPartition tp) {
    Deque<ProducerBatch> d = this.batches.get(tp);
    if (d != null)
      return d;
    d = new ArrayDeque<>();
    Deque<ProducerBatch> previous = this.batches.putIfAbsent(tp, d);
    if (previous == null)
      return d;
    else
      return previous;
}


```

看到以上简化版的代码，步骤如下

- 根据TopicPartition得到Deque<ProducerBatch>。
  - 每个不同的TopicPartition(由topic和partition双重决定)都对应一个唯一的双向队列
  - 双向队列中的元素为ProducerBatch，而不是ProducerRecord。看名字也可以看出，Batch其实就是Record的聚合体，后续也会看到记录如何按一定规则的聚集。

- 得到每个TopicPartition唯一对应的队列之后，如果生产者没有close的话，会锁定队列并尝试向队列中append记录。

```java
private RecordAppendResult tryAppend(long timestamp, byte[] key, byte[] value, Header[] headers,
                                     Callback callback, Deque<ProducerBatch> deque) {
    ProducerBatch last = deque.peekLast();
    if (last != null) {
        FutureRecordMetadata future = last.tryAppend(timestamp, key, value, headers, callback, time.milliseconds());
        if (future == null)
            last.closeForRecordAppends();
        else
            return new RecordAppendResult(future, deque.size() > 1 || last.isFull(), false);
    }
    return null;
}
```

可以看到

- 如果成功append，会返回一个非空的RecordAppendResult，其中包含record的一些offset信息
- 如果append失败，会返回null，并且把这个batch置为不可再append，因为之后会新创建一个producerBatch

继续回到一开始的代码，看看如果第一次尝试append失败后会做什么操作。

```java
// we don't have an in-progress record batch try to allocate a new batch
byte maxUsableMagic = apiVersions.maxUsableProduceMagic();
int size = Math.max(this.batchSize, AbstractRecords.estimateSizeInBytesUpperBound(maxUsableMagic, compression, key, value, headers));
log.trace("Allocating a new {} byte message buffer for topic {} partition {}", size, tp.topic(), tp.partition());
buffer = free.allocate(size, maxTimeToBlock);
synchronized (dq) {
    // Need to check if producer is closed again after grabbing the dequeue lock.
    if (closed)
        throw new KafkaException("Producer closed while send in progress");

    RecordAppendResult appendResult = tryAppend(timestamp, key, value, headers, callback, dq);
    if (appendResult != null) {
        // Somebody else found us a batch, return the one we waited for! Hopefully this doesn't happen often...
        return appendResult;
    }

    MemoryRecordsBuilder recordsBuilder = recordsBuilder(buffer, maxUsableMagic);
    ProducerBatch batch = new ProducerBatch(tp, recordsBuilder, time.milliseconds());
    FutureRecordMetadata future = Utils.notNull(batch.tryAppend(timestamp, key, value, headers, callback, time.milliseconds()));

    dq.addLast(batch);
    incomplete.add(batch);

    // Don't deallocate this buffer in the finally block as it's being used in the record batch
    buffer = null;

    return new RecordAppendResult(future, dq.size() > 1 || batch.isFull(), true);
}
```

梳理一下这段代码做了什么操作。

- 用apiVersion做版本控制，避免并发问题
- 估算需要分配的byteBuffer的大小，主要根据默认的batchSize以及估算的key-value需要的最大size来共同得到
- 从byteBuffer缓冲区中，分配得到对应大小的byteBuffer
- 锁定对应的队列后double-check尝试直接append
- 失败后先构建一个recordsBuilder，相当于之后的producerBatch的统一内存分配指挥官，可供分配的内存上限就是刚才从byteBuffer缓冲区中分配得到的byteBuffer
- 向新创建的producerBatch中append本次的record，并将新创建的batch加到队列的末尾，并添加至待发送的batch列表中
- 返回RecordAppendResult

按上面的描述，我们就梳理了一下将我们创建的producerRecord登记在案的操作，但是还只是做了保存，并没有真正完成发送的操作。并且我们可以看到我们append方法返回了RecordAppendResult，我们继续看看这个result起到了什么作用。

```java
RecordAccumulator.RecordAppendResult result = accumulator.append(tp, timestamp, serializedKey,
        serializedValue, headers, interceptCallback, remainingWaitMs);
if (result.batchIsFull || result.newBatchCreated) {
    log.trace("Waking up the sender since topic {} partition {} is either full or getting a new batch", record.topic(), partition);
    this.sender.wakeup();
}
return result.future;
```

看一下我们刚才的过程有构建result的地方。

```java
public final static class RecordAppendResult {
  public final FutureRecordMetadata future;
  public final boolean batchIsFull;
  public final boolean newBatchCreated;

  public RecordAppendResult(FutureRecordMetadata future, boolean batchIsFull, boolean newBatchCreated) {
    this.future = future;
    this.batchIsFull = batchIsFull;
    this.newBatchCreated = newBatchCreated;
  }
}


new RecordAppendResult(future, dq.size() > 1 || batch.isFull(), true);

new RecordAppendResult(future, deque.size() > 1 || last.isFull(), false);
```

两处分别是向原有的producerBatch中append和新创建producerBatch后append的result返回。可以看到主要区别就是最后一个布尔值newBatchCreated的区别。并且如果batchIsFull或者newBatchCreated两者有一个满足条件，就需要调用

```java
log.trace("Waking up the sender since topic {} partition {} is either full or getting a new batch", record.topic(), partition);
this.sender.wakeup();
```

也就是文章一开始的时候说的，发送者中的另一个线程，作用就是从accumulator中获取batch按批次发送。