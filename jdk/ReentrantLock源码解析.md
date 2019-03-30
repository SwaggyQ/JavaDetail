# ReentrantLock源码解析
## 前文
#### 之前的文章我们分析了AQS这个并发框架的逻辑，本文再对同在JUC包中的一种锁的具体实现进行一下分析。
## 正文
#### ReentrantLock实现了Lock的接口，用于对代码中的并发控制，内部实现了一个AQS的子类实现。有公平和非公平两种模式。模式选择在初始化的时候就已经选定
```
    public ReentrantLock() {
        sync = new NonfairSync();
    }

    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```
#### 可以看到默认实现是非公平的。所以源码的重点就是这两个类FairSync和NonfairSync。进一步可以看到，这两个类都是继承自内部类Sync。所以我们先看到这个统一的父类
```
	// 继承实现了AQS，使用了独占模式
 	abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = -5179523762034025860L;

        abstract void lock();
		// 这是非公平锁独有的抢占方法
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            // 如果当前资源为0，则代表可以开始抢占
            if (c == 0) {
            	// 用CAS的方式将资源改成抢占后的值，一般都是+1
                if (compareAndSetState(0, acquires)) {
                	// 表示当前线程已经独占了锁
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            // 如果当前锁已经被抢占，而且正是被当前线程抢占，则可以直接再次进入锁
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
		// 继承实现了独占模式的释放资源的抽象方法
        protected final boolean tryRelease(int releases) {
        	// 在ReentrantLock中，得到资源之后会将state + 1;释放之后 再 -1
        	// 所以得到的c的值表示是否已经将所有加锁的资源释放
        	// 例如 初始state = 0，此时 线程A第一次得到了锁，则state += 1，同理多次得到会多次+1
        	// 然后依次释放，依次-1
            int c = getState() - releases;
            // 只有当前线程是抢占到资源的线程时才可以释放
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            // 标志是否完全可以供抢占
            boolean free = false;
            if (c == 0) {
            	// 如上分析，c==0，代表线程完全释放了资源，
                free = true;
              // 表示当前没有线程抢占了资源
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }

        protected final boolean isHeldExclusively() {
            // While we must in general read state before owner,
            // we don't need to do so to check if current thread is owner
            return getExclusiveOwnerThread() == Thread.currentThread();
        }
    }
```
#### 在ReentrantLock的内部类Sync的实现就是如上。接下来看到他的子类，公平锁和非公平锁的实现。先看到非公平锁
```
	static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        /**
         * Performs lock.  Try immediate barge, backing up to normal
         * acquire on failure.
         */
        // 抢占锁 
        final void lock() {
        	// 直接先尝试用CAS的方式，改变资源的状态
            if (compareAndSetState(0, 1))
            	// 如果CAS成功，则看做已经得到了锁，直接将独占线程改为自己
                setExclusiveOwnerThread(Thread.currentThread());
            else
            	// 如果CAS没有成功，则进入正常的AQS抢占流程
                acquire(1);
        }
		// 抽象方法直接使用了Sync中的非公平抢占实现
        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }
```
#### 再看看公平锁
```
	static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;

        final void lock() {
        	// 直接进行AQS抢占流程
            acquire(1);
        }

        /**
         * Fair version of tryAcquire.  Don't grant access unless
         * recursive call or no waiters or is first.
         */
         // 抽象方法的实现，可以看到和非公平的方式基本相同
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
            		// 只有多了一个这个判断
            		// 就是判断在AQS的等待队列中，是否已经有其他节点在等待
            		// 如果没有，则直接调用
            		// 也就是公平锁代表不能插队
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }
```
#### 所以看到公平锁和非公平锁的区别就体现在，在尝试获取资源的时候，是否可以插队获取。
#### 进一步看到这个tryAcquire方法，也就是公平锁和非公平锁具体差异的地方。假设此时在AQS内部已经有一个等待者队列吗。head节点释放了资源，应该去唤醒后续的节点开始竞争，假设此时非公平锁开始插队竞争，而且成功的话。那么在等待者队列中的节点将没有资源再Park了，就会一直不停的永真循环去尝试获取资源。