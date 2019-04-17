# 聊聊 Synchronize 关键字
## 正文
#### Synchronize是java中提供的一个并发控制的一个关键字。主要是为了控制多线程并发情况下对共享资源的竞争情况。加锁有以下三种情况
1：修饰普通方法，锁的对象是当前实例
2：修饰形态方法，锁的对象是当前类class对象
3：修饰同步代码块，锁的对象是括号中的内容，可以是实例也可以是class对象

#### 以上三种情况，简单归纳可以分为两类，一个是同步方法，一个是同步代码块。在java中，这两种的实现方式是不一样的，先说说同步方法，如果我们把我们写的关于同步方法的代码反编译，会发现在反编译代码中，同步方法的代码部分，会是这样子的
```
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
```
#### 就代表这个方法是public的，而且是同步的。那么同步代码块是怎么样的呢？同样反编译，我们可以得到答案。
```
	3: monitorenter  //注意此处，进入同步方法
	...      
    21: monitorexit //注意此处，退出同步方法
```
#### 可以很直观的看出来，是用了两个指令来控制的，注意一下两个指令必须是成对出现的，代表获得锁和释放锁的过程。
#### 再看到，每个实例和类对象都有自己单独的监视器，所以两个线程可以同时访问普通的Synchronize以及静态的Synchronize方法。所以该修饰符可能会有以下几个特性
1：当线程A正在访问某个实例对象的普通Synchronize方法，或者同步代码块的括号中为一个实例对象是，其他线程不但不能访问这个同步方法，而且其他同步方法也不能方法。但是静态的同步方法是可以访问的，因为之前说过实例对象和类对象是有自己不同的监视器的

#### 之前说了很多次监视器，那么监视器到底是什么呢?在java中，这部分是用c++实现的，对应的对象内容如下
```
ObjectMonitor() {
    _count        = 0; //用来记录该对象被线程获取锁的次数
    _waiters      = 0;
    _recursions   = 0; //锁的重入次数
    _owner        = NULL; //指向持有ObjectMonitor对象的线程 
    _WaitSet      = NULL; //处于wait状态的线程，会被加入到_WaitSet
    _WaitSetLock  = 0 ;
    _EntryList    = NULL ; //处于等待锁block状态的线程，会被加入到该列表
  }
```

#### 这一块的解释，我引用被人写的一个整个的属性转换的过程
> 对于一个synchronized修饰的方法(代码块)来说：
当多个线程同时访问该方法，那么这些线程会先被放进_EntryList队列，此时线程处于blocking状态
当一个线程获取到了实例对象的监视器（monitor）锁，那么就可以进入running状态，执行方法，此时，ObjectMonitor对象的_owner指向当前线程，_count加1表示当前对象锁被一个线程获取
当running状态的线程调用wait()方法，那么当前线程释放monitor对象，进入waiting状态，ObjectMonitor对象的_owner变为null，_count减1，同时线程进入_WaitSet队列，直到有线程调用notify()方法唤醒该线程，则该线程重新获取monitor对象进入_Owner区
如果当前线程执行完毕，那么也释放monitor对象，进入waiting状态，ObjectMonitor对象的_owner变为null，_count减1

#### 现在再讲讲关于锁的一些膨胀的机制。其实在jdk 1.6之前，是只有重量锁，在jdk 1.6之后，引用了轻量锁，和偏向锁。在一些竞争不激烈的情况下，减轻锁开销，这些锁的分类都是基于对象头。

#### 先讲到重量锁
重量级锁的状态下，对象的mark word为指向一个堆中monitor对象的指针。


可以看到锁信息也是存在于对象的mark word中的。当对象状态为偏向锁（biasable）时，mark word存储的是偏向的线程ID；当状态为轻量级锁（lightweight locked）时，mark word存储的是指向线程栈中Lock Record的指针；当状态为重量级锁（inflated）时，为指向堆中的monitor对象的指针。




