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
- simpleParserStage
- dmlParserSequenceBarrier
- sinkStoreStage

#### 所以上面说了生产过程 publish方法，还有几个消费者的阶段。所以先看到publish方法的过程,先看到方法里的这一句关键的代码
```
	long next = disruptorMsgBuffer.tryNext();
```
#### 这一句其实就是生产者去竞争生产位置的地方，跟进去看一下。因为我们现在的情况是单生产者情况，所以直接看到这个子类
```
	@Override
    public long tryNext(int n) throws InsufficientCapacityException
    {
        if (n < 1)
        {
            throw new IllegalArgumentException("n must be > 0");
        }
		// 返回值为true，代表有空位生产，返回false代表已经超过了最慢的消费者一圈，必须等待
        if (!hasAvailableCapacity(n, true))
        {
            throw InsufficientCapacityException.INSTANCE;
        }
		// 得到下一位的生产序号
        long nextSequence = this.nextValue += n;

        return nextSequence;
    }
    
    // 查询目前是否有空位供存储
    private boolean hasAvailableCapacity(int requiredCapacity, boolean doStore)
    {
        long nextValue = this.nextValue;
		
        long wrapPoint = (nextValue + requiredCapacity) - bufferSize;
        // 初始值为-1，在之后的运行过程中，会得到较小值
        long cachedGatingSequence = this.cachedValue;

        if (wrapPoint > cachedGatingSequence || cachedGatingSequence > nextValue)
        {
            if (doStore)
            {
                cursor.setVolatile(nextValue);  // StoreLoad fence
            }
			// 赋值位点
            long minSequence = Util.getMinimumSequence(gatingSequences, nextValue);
            this.cachedValue = minSequence;
			// 如果需要生产的位点，已经超过了被消费的位点，则代表不能往里生产数据
            if (wrapPoint > minSequence)
            {
                return false;
            }
        }

        return true;
    }
```
#### 以上就得到了生产者可以向RingBuffer生产数据的序号值，在得到生产序号之后，就可以放数据了
```
	// UNSAFE的方式获取在RingBuffer对应位置的数据容器，打个比方就是把RingBuffer看成一个摩天轮，每个箱体就是一个装载数据的容器，每个游客就是放入的数据
	MessageEvent data = disruptorMsgBuffer.get(next);
    if (buffer != null) {
        data.setBuffer(buffer);
    } else {
        data.setEvent(event);
    }
    disruptorMsgBuffer.publish(next);
```
#### 再回到单生产者的publish方法
```
    public void publish(long sequence)
    {
        cursor.set(sequence);
        waitStrategy.signalAllWhenBlocking();
    }
```
#### 上面就是单生产者往其中放置数据的过程，重要的就是主要到那几个cursor位点的移动。现在我们再来看一下sinkStoreStage阶段的消费者。先回去看到这个消费者的创建过程
```
	// 注意传入的这个参数，是上一阶段的Sequence，所以得到的SequenceBarrier就会以上一阶段的Sequence作为界限，效果就是一份数据只有在上一阶段消费之后，才会被后续的阶段消费
	SequenceBarrier sinkSequenceBarrier = disruptorMsgBuffer.newBarrier(sequence);
	// 创建一个BatchEventProcessor，可以批次处理
	// 传入SinkStoreStage，作为eventHandler
    sinkStoreStage = new BatchEventProcessor<MessageEvent>(disruptorMsgBuffer,
        sinkSequenceBarrier,
        new SinkStoreStage());
    sinkStoreStage.setExceptionHandler(exceptionHandler);
    disruptorMsgBuffer.addGatingSequences(sinkStoreStage.getSequence());
```
#### BatchEventProcessor是一个Runnable的类，所以接下来会启动BatchEventProcessor
```
	public void run()
    {
    	// CAS修改运行状态
        if (running.compareAndSet(IDLE, RUNNING))
        {
            sequenceBarrier.clearAlert();
			// 会通知eventHandler，已经启动。并调用他的onStart方法
            notifyStart();
            try
            {
                if (running.get() == RUNNING)
                {
                	// 内部永真循环，处理event
                    processEvents();
                }
            }
            finally
            {
            	// 调用eventHandler的onShutdown方法
                notifyShutdown();
                // 修改状态
                running.set(IDLE);
            }
        }
        else
        {
            // This is a little bit of guess work.  The running state could of changed to HALTED by
            // this point.  However, Java does not have compareAndExchange which is the only way
            // to get it exactly correct.
            if (running.get() == RUNNING)
            {
                throw new IllegalStateException("Thread is already running");
            }
            else
            {
                earlyExit();
            }
        }
    }
```
#### 继续看到processEvents方法
```
	private void processEvents()
    {
        T event = null;
        // 初始值为-1，记录消费者消费到的位点
        long nextSequence = sequence.get() + 1L;

        while (true)
        {
            try
            {
            	// 尝试获取给定位点的最靠近的可用序列，内部会调用到之前设定的等待策略
                final long availableSequence = sequenceBarrier.waitFor(nextSequence);
                if (batchStartAware != null)
                {
                    batchStartAware.onBatchStart(availableSequence - nextSequence + 1);
                }
				// 依次处理数据
                while (nextSequence <= availableSequence)
                {
                    event = dataProvider.get(nextSequence);
                    // 调用实际的eventHandler的处理逻辑
                    eventHandler.onEvent(event, nextSequence, nextSequence == availableSequence);
                    nextSequence++;
                }
				// 将消费位点移到最新位置
                sequence.set(availableSequence);
            }
            catch (final TimeoutException e)
            {
                notifyTimeout(sequence.get());
            }
            catch (final AlertException ex)
            {
                if (running.get() != RUNNING)
                {
                    break;
                }
            }
            catch (final Throwable ex)
            {
                exceptionHandler.handleEventException(ex, nextSequence, event);
                sequence.set(nextSequence);
                nextSequence++;
            }
        }
    }
```