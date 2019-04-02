# Disruptor的写入
#### 本文主要讲一下Disruptor的写入部分，也就是生产者怎么将数据成功的写入RingBuffer 
                                   
    


#### 写入RingBuffer需要关注的几个问题

- 1:如何避免生产者的生产速度过快而造成的新消息覆盖了未被消费的旧消息的问题 
- 2:如何解决多个生产者抢占生产位的问题

#### 带着问题去看代码，会更有目的性，所以本文主要会围绕这两个问题作出解答，在解答的过程中，我们都会分为单生产者和多生产者两种情况进行解答。 
#### 首先看第一个问题: 
           这个问题是Disruptor特有的，因为他的核心RingBuffer是一个首尾相连的有界的连续数组，所以如果生产者生产过快，一定会使得生产的消息超过了未消费的消息一圈，从而使未被消费的消息被覆盖而丢失。那Disruptor是如何解决这个问题的呢？ 
           在这之前，要先插播一下生产者生产消息写入RingBuffer的主要流程。大致可以分为两步。 
### 一 : 占位
          因为RingBuffer中的每个存储单元都是带序号的数据容器，所以生产者在往里写入数据时，第一步就要先抢占到这个位置，这个抢占包括两部分，首先是如果是多生产者的情况，需要考虑多生产者同时抢占的问题，然后要考虑消费者的消费速度问题，不能抢占还未被消费的位置。 
          
### 第二，填装数据之后发布
          填装数据好理解，因为RingBuffer在初始化时已经分配好了所有数据容器，我们要做的就是往这个容器中填入我们的数据，做个比喻就是思考一下摩天轮这个场景。摩天轮的每个箱子都是一直存在的，游客做的就是不停的往箱子里坐人和下来给下一波人腾位置。那发布又是什么意思呢?这里的发布指的是当生产者成功填装好数据之后，怎么确认这一次的生产行为，这个过程中单生产者和多生产者的处理方式是不一样的，这个我们会在之后详细讲到。 
        我总是喜欢把这个写入的过程比作摩天轮，希望这个图能让你有一个大致的印象 
 
           好，现在回到之前没说完的这个问题，生产者在写入时，怎么感知消费者的消费速度是否允许我这一次新数据的写入呢？这个就和我们上面提到的第一步，占位有关。 
           为了方便读者快速的先有一个画面感，所以先讲到单生产者的情况. 

单生产者占位------------------------------ 

           单生产者的时候，因为不会有其他生产者跟他抢占位置，所以他只需要知道他的消费者的消费到哪里了就行。Disruptor处理这个问题的方法是每个消费者加入消费时，都会调用生产者的以下这个方法 
Java代码  收藏代码
/** 
    * Add the specified gating sequences to this instance of the Disruptor.  They will 
    * safely and atomically added to the list of gating sequences. 
    * 
    * @param gatingSequences The sequences to add. 
    */  
   void addGatingSequences(Sequence... gatingSequences);  

           从这个方法的名字也能看出来，这个方法是向生产者添加一个门控序列，那什么门控序列呢，这个可以从生产者抢占位子的代码里获得灵感，对此的解释我都会按照注释的方式也在代码里，其余也不过多赘述了。 
Java代码  收藏代码
/** 
     * @see Sequencer#next(int) 
     */  
    @Override  
    public long next(int n)    //生产者进行抢占时调用的方法，next(1)即为抢占已生产过的序号的下一个位置  
    {  
        if (n < 1)  
        {  
            throw new IllegalArgumentException("n must be > 0");  
        }  
  
        long nextValue = this.nextValue;      
  
        long nextSequence = nextValue + n;   // 例如已经生产到了8号，则调用next(1)即为抢占生产9号的权利  
        long wrapPoint = nextSequence - bufferSize;     
        long cachedGatingSequence = this.cachedValue;   // 这个值即为我们记录的  
  
        if (wrapPoint > cachedGatingSequence || cachedGatingSequence > nextValue)  
        {  
            cursor.setVolatile(nextValue);  // StoreLoad fence  
  
            long minSequence;  
            while (wrapPoint > (minSequence = Util.getMinimumSequence(gatingSequences, nextValue)))  
            {  
                LockSupport.parkNanos(1L); // TODO: Use waitStrategy to spin?  
            }  
  
            this.cachedValue = minSequence;  
        }  
  
        this.nextValue = nextSequence;  
  
        return nextSequence;  
    }  

          方法里的cachedGatingSequence即为之前说的通过addGatingSequence方法记录的所有门控序列的最小值，所以门控序列在这里起到的作用就是让生产者可以感知到所有消费者中最慢的消费者现在所在的序号，若生产速度已经赶上了这个最慢消费者，那生产者就要park一段时间来等待消费者进行消费 

单生产者的装填数据和发布过程------------------------------ 

           单生产者的装填数据和发布过程需要注意的点没有很多，像我之前说的，装填数据主要就是往数据容器中装填数据的过程，这边主要想提一下发布的过程。发布的过程，其实就是移动生产者已生产的游标和通知消费者的过程。 
Java代码  收藏代码
/** 
     * @see Sequencer#publish(long) 
     */  
    @Override  
    public void publish(long sequence)  
    {  
        cursor.set(sequence);  
        waitStrategy.signalAllWhenBlocking();  
    }  

          上述代码中的cursor记录了该生产者目前生产到的序号，例如若为8，则可以认为此时生产者已经成功生产了8个序号，接下来要占位生产的是9号序号，为什么着重讲一下移动游标的这句代码呢，因为这边的处理逻辑和多生产者的时候有一些不同，所以先在这边提一句，之后在讲多生产者的时候，会再进行对比。移动游标之后要通过waitStrategy通知等待数据的消费者，waitStrategy这个类在我们下一篇讲消费者的时候会讲到，此时可以先理解为消费者没数据时会一直等候数据，而生产者在生产完新数据后会去通知消费者。 

多生产者占位------------------------------ 

           好的，现在来看一下稍微复杂一点的多生产者是怎么进行占位处理的 
之前说过占位需要考虑两方面因素，消费者的消费速度和生产者之间的抢占关系。第一个因素在单生产者和多生产者之间都是类似的，主要就是根据门控序列来感知当前最慢消费者的位置。所以现在来分析一下多生产者在抢占同一个生产位的时候是怎么做的。 
当有两个或者两个生产者以上的时候，难免会出现抢占的情况，而在原来的阻塞队列里面，面对这种情况下处理的方法就是进行加锁操作，而加锁是必然会降低性能的，所以在之前的文章也提到过当阻塞队列中有多生产者或者多消费者的时候，性能会急剧下降。那Disruptor是怎么做的呢？ 
Java代码  收藏代码
/** 
    * @see Sequencer#next(int) 
    */  
   @Override  
   public long next(int n)  
   {  
       ...  
       do  
       {  
           current = cursor.get();    // #1   
           next = current + n;          // #2  
  
           long wrapPoint = next - bufferSize;  
           long cachedGatingSequence = gatingSequenceCache.get();  
  
           if (wrapPoint > cachedGatingSequence || cachedGatingSequence > current)  
           {  
               ... //和单生产者的代码类似，不展开讲了  
           }  
           else if (cursor.compareAndSet(current, next))  
           {  
               break;  
           }  
       }  
       while (true);  
  
       return next;  
   }  

           因为和单生产者的情况有部分代码类似，为了节省篇幅我省去了这一部分，主要看else if()的那句话，可以看到里面用了CAS操作,这边描述起来可能有点麻烦，所以我会用一个例子进行说明。 
           假设，此时有两个生产者，如果现在整个RingBuffer的生产游标已经到了12这个位置，那两个生产者都会调用next(1)这个方法，去竞争13这个位置，先看生产者A，进入这段代码时在代码#1位置，拿到current游标为12，next游标为13，那经过上面和最慢消费者位置的比较后，如果发现这个位置是可以生产的，那么会进入CAS这句代码，那这句话实际执行就是cursor.compareAndSet(12,13)，与此同时，生产者B也进行了刚才同样的操作，也进行到了这一步也执行cursor.compareAndSet(12,13)，那此时我们会发现根据CAS的特性，只可能有一个生产者修改游标成功，假设是生产者A好了，那生产者B的这句话就会执行失败，再次进入永真循环的#1代码处，而此时cusor已经变成了13，所以生产者B会执行cursor.compareAndSet(13,14)，将游标改到14。所以结果就是生产者A获得了13号位置的生产权，生产者B获得了14号位置的生产权。 
           根据我这个例子，我想大家应该能理解我想表达的意思，多生产者抢占到了各自的位置，但是回想我们单生产者发布的时候，我们移动游标后就相当于向消费者宣告这个位子是有消息可消费的，但是我们现在两个生产者只是抢到位子还没将真正数据放入就已经移动了游标，那是不是会有问题呢？ 
           关于这个问题，Disruptor的作者也想到，所以他在多生产者的时候，另外设计了一个和RingBuffer大小一致数组，作为标志位数组。只有当生产者真正将数据写入到相应的位子后，才会将这个标志位数组相应的位置置为可消费状态，所以在多生产者的情况下，消费者在寻找可消费的位置时，除了要看cusor这个游标移动的情况，还有看对应的这个标志位是否已经被修改成功。这个标志位修改的逻辑我们看下一个步骤 

多生产者的装填数据和发布------------------------------ 

           装填数据不多说了，和单生产一样。主要讲讲发布的过程。多生产者的发布主要是设置这个标志位数组。 
Java代码  收藏代码
/** 
    * @see Sequencer#publish(long) 
    */  
   @Override  
   public void publish(final long sequence)  
   {  
       setAvailable(sequence);  
       waitStrategy.signalAllWhenBlocking();  
   }  
   private void setAvailable(final long sequence)  
   {  
       setAvailableBufferValue(calculateIndex(sequence), calculateAvailabilityFlag(sequence));  
   }  
  
   private void setAvailableBufferValue(int index, int flag)  
   {  
       long bufferAddress = (index * SCALE) + BASE;  
       UNSAFE.putOrderedInt(availableBuffer, bufferAddress, flag);  
   }  
  
   private int calculateAvailabilityFlag(final long sequence)   // indexShift = log2(bufferSize)     
   {  
       return (int) (sequence >>> indexShift);  
   }  
  
   private int calculateIndex(final long sequence)  
   {  
       return ((int) sequence) & indexMask;     //indexMask = bufferSize -1   
   }  

           主要想讲讲这个设置标志位的过程，一开始我也不太理解这个过程这么写是为什么。 
           

例如假设我们的bufferSize是8，现在消费者A已经将数据成功写进了13号位置，然后调用了publish这个方法宣告写入成功，然后我们按整个逻辑走一下，计算index和flag的结果分别为5，1，所以会按照第5个int的计算地址方法，用UNSAFE类将该地址的值设置为1，这个值主要记录的是现在是第几圈。那么之后，在消费者去查询13号是不是可以消费时，若按照一样的算法去查询第5个位置的值是不是还是1，如果还是1，则证明可以消费，若不是1，也就是生产者还没修改这个标志位，则证明数据还没有准备好，这个位置不能消费。 
           以上，我们大致分析了一下生产者将数据写入RingBuffer的过程，不管是单生产者还是多生产者，主要过程还是离不开占位和装填数据之后发布这两个步骤。回到开头我们提出的两个问题，现在试着给出一个答案。 
           Q1:如何避免生产者的生产速度过快而造成的新消息覆盖了未被消费的旧消息的问题 
           A1:单生产者和多生产者对于这个问题的解决办法都是一样的，就是当消费者注册时，就会以门控序列的方式保留序号供生产者查看，那么当生产者要占位消费时，就可以根据所有消费者中最慢的消费者来确定这个位子是不是可以用于生产，是不是会覆盖到未被消费的数据 

           Q2:如何解决多个生产者抢占生产位的问题 
           A2:这个问题Disruptor采用了一个CAS+等大的标志数组的方式，用CAS来竞争移动生产游标，用标志位数组确定是否装填数据完毕 
