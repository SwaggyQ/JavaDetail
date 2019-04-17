# Semaphore 详解
## 正文
#### 因为之前详细看过了AQS的代码。所以今天再简单过一下基于AQS的一些子类实现，今天看到这个Semaphore
#### 先看一下代码结构，可以看到内部实现了一个基于AQS的Sync类，还有两个基于Sync类的公平和非公平的子类实现。那么就先看到这两个子类实现
```
	static final class NonfairSync extends Sync {
        private static final long serialVersionUID = -2694183684443567898L;

        NonfairSync(int permits) {
            super(permits);
        }

        protected int tryAcquireShared(int acquires) {
            return nonfairTryAcquireShared(acquires);
        }
        
        final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
    }

    /**
     * Fair version
     */
    static final class FairSync extends Sync {
        private static final long serialVersionUID = 2014338818796000944L;

        FairSync(int permits) {
            super(permits);
        }

        protected int tryAcquireShared(int acquires) {
            for (;;) {
                if (hasQueuedPredecessors())
                    return -1;
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
    }

```
#### 很明显看到公平和非公平的实现方式的区别就在tryAcquireShared方法中，公平实现逻辑中，先调用了一下hasQueuedPredecessors方法。这个方法是AQS中自带的方法，逻辑实现如下
```
    public final boolean hasQueuedPredecessors() {
        // The correctness of this depends on head being initialized
        // before tail and on head.next being accurate if the current
        // thread is first in queue.
        Node t = tail; // Read fields in reverse initialization order
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }
```
#### 就是检查在等待者队列中，是不是存在有其他线程已经在队列中，且排在当前线程的前面。所以公平锁的做法是，如果有其他线程排在前面，则返回-1，代表竞争失败，需要排在队列中继续等待。而非公平锁的做法就是，即使有其他线程排在前面了，可以直接插队获取锁。
#### 现在回到Semaphore类中，从构造方法中我们也可以看出，默认是非公平锁的实现。然后从构造方法中，传入一个permits参数。再看到acquire和release方法中，竞争的资源个数都是1，也就是可以看做是获取资源的线程数了。通过AQS的方式，控制获得资源的线程个数。