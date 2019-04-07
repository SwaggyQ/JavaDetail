# canal源码分析 (六) eventStore的多线程模式
## 前言
#### 上一章说了一下单线程模式，这章会继续说一下多线程模式。因为多线程模式主要用到了Disruptor，所以本文也会详细说一下Disruptor的内容

## 正文
### 多线程模式
#### 回到AbstractEventParser中
```
	if (parallel) {
        // build stage processor
        multiStageCoprocessor = buildMultiStageCoprocessor();
        ...
    }
    
   
    // AbstractMysqlEventParser.java
    protected MultiStageCoprocessor buildMultiStageCoprocessor() {
        MysqlMultiStageCoprocessor mysqlMultiStageCoprocessor = new MysqlMultiStageCoprocessor(parallelBufferSize,
            parallelThreadSize,
            (LogEventConvert) binlogParser,
            transactionBuffer,
            destination);
        mysqlMultiStageCoprocessor.setEventsPublishBlockingTime(eventsPublishBlockingTime);
        return mysqlMultiStageCoprocessor;
    }
```
#### 可以大概看到，多线程模式下会先构建一个MultiStageCoprocessor，这个可以类比为单线程情况下的SinkHandler。直接看到GTID模式下的dump方法
```
	multiStageCoprocessor.start();
    erosaConnection.dump(gtidSet, multiStageCoprocessor);
```
#### 这次我们先看到dump方法
```
	@Override
    public void dump(GTIDSet gtidSet, MultiStageCoprocessor coprocessor) throws IOException {
        updateSettings();
        loadBinlogChecksum();
        sendBinlogDumpGTID(gtidSet);
        ((MysqlMultiStageCoprocessor) coprocessor).setConnection(this);
        ((MysqlMultiStageCoprocessor) coprocessor).setBinlogChecksum(binlogChecksum);
        DirectLogFetcher fetcher = new DirectLogFetcher(connector.getReceiveBufferSize());
        try {
            fetcher.start(connector.getChannel());
            while (fetcher.fetch()) {
                accumulateReceivedBytes(fetcher.limit());
                LogBuffer buffer = fetcher.duplicate();
                fetcher.consume(fetcher.limit());
                if (!coprocessor.publish(buffer)) {
                    break;
                }
            }
        } finally {
            fetcher.close();
        }
    }
```
#### 整体逻辑和单线程模式下都是类似的。重点看到
```
	coprocessor.publish(buffer)
```
#### 好了，现在专心回到MultiStageCoprocessor类中。我们先要看到start方法，再看到publish方法。
#### 先看到start方法里面做了什么。
```
	this.disruptorMsgBuffer = RingBuffer.createSingleProducer(new MessageEventFactory(),
            ringBufferSize,
            new BlockingWaitStrategy());
```
#### 可以看到，先是创建了一个单生产者的RingBuffer，作为存储的数据结构。简单跟进去看一下
```
public static <E> RingBuffer<E> createSingleProducer(
        EventFactory<E> factory,
        int bufferSize,
        WaitStrategy waitStrategy)
    {
        SingleProducerSequencer sequencer = new SingleProducerSequencer(bufferSize, waitStrategy);

        return new RingBuffer<E>(factory, sequencer);
    }
```
#### 可以看到，创建了一个单生产者的Sequencer，然后等待策略是采用了BlockingWait的方式。然后返回了RingBuffer的结构。具体关于Disruptor的内部逻辑，可以看我另一篇文章。
#### 紧接着创建了两个线程池
```
	// 这边的线程池大小取的是当前处理器核数的60%
	int tc = parserThreadCount > 0 ? parserThreadCount : 1;
    this.parserExecutor = Executors.newFixedThreadPool(tc, new NamedThreadFactory("MultiStageCoprocessor-Parser-"
                                                                                  + destination));

    this.stageExecutor = Executors.newFixedThreadPool(2, new NamedThreadFactory("MultiStageCoprocessor-other-"
                                                                                + destination));
      
```
#### 再往下看，可以看到有几段逻辑非常类似的代码。这边抽一段看一下
```
	SequenceBarrier sequenceBarrier = disruptorMsgBuffer.newBarrier();
        ExceptionHandler exceptionHandler = new SimpleFatalExceptionHandler();
        // stage 2
        this.logContext = new LogContext();
        simpleParserStage = new BatchEventProcessor<MessageEvent>(disruptorMsgBuffer,
            sequenceBarrier,
            new SimpleParserStage(logContext));
        simpleParserStage.setExceptionHandler(exceptionHandler);
        disruptorMsgBuffer.addGatingSequences(simpleParserStage.getSequence());

```
#### 这样的逻辑还有重复的几段。都是先disruptorMsgBuffer.newBarrier()，再disruptorMsgBuffer.addGatingSequences。
#### 其实这一部分就是注册消费者，消费者可以是单线程，也可以是多线程。每个消费者之间都有逻辑关系，例如 A 先于 B ，那么一份数据进来，必须先是被A处理完，再交由B处理。这边归纳一下代码中有几处消费者。