# ArrayBlockingQueue
## 前言
#### 本文讲一下ArrayBlockingQueue(下文简称ABQ)的一些原理，整理一下之前有点混乱的几个方法，具体ABQ是什么我就不详细说了。
## 构造
#### 照例先看看怎么创建这个我们需要的这个数据结构。有三个构造函数，简单看一下可以看到重点的是ArrayBlockingQueue(int , boolean)方法。其他两个都是之后调用这个方法。所以直接看看这个方法
```
    public ArrayBlockingQueue(int capacity, boolean fair) {
    	// 校验容量不能是负数
        if (capacity <= 0)
            throw new IllegalArgumentException();
        //
        this.items = new Object[capacity];
        lock = new ReentrantLock(fair);
        notEmpty = lock.newCondition();
        notFull =  lock.newCondition();
    }
```
