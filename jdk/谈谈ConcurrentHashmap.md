# 谈谈ConcurrentHashmap
## 前文
#### 本系列终于对ConcurrentHashmap下手了，为了解决HashMap的线程不安全的问题。java还提供了一种允许多线程同时高效访问的类似HashMap的数据结构，也就是ConcurrentHashmap，并在jdk 1.8版本中做了一个较大的改变。在jdk 1.7中，一直采用的都是对每个segment进行分段锁方法进行加锁，所以有几个segment就允许多少线程同时进行读写。但是在jdk 1.8中，完全抛弃了这种分段锁的方法，取而代之的是用CAS+synchronized的方式，对多线程进行控制。现在直接进入正文，看看内部到底是怎么进行操作的。
## 正文
#### 先看看构造函数，可以看到和HashMap的构造函数非常类似
```
    public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
        if (initialCapacity < concurrencyLevel)   // Use at least as many bins
            initialCapacity = concurrencyLevel;   // as estimated threads
        long size = (long)(1.0 + (long)initialCapacity / loadFactor);
        int cap = (size >= (long)MAXIMUM_CAPACITY) ?
            MAXIMUM_CAPACITY : tableSizeFor((int)size);
        this.sizeCtl = cap;
    }
```
# TODO 构造函数 concurrencyLevel this.sizeCtl
### put
#### 现在看看put方法里做了什么。为什么可以接受多并发。
```
	// 和hashMap差不多，也是调用了putVal方法，第三个参数是onlyIfAbsent，在后面会用到
    public V put(K key, V value) {
        return putVal(key, value, false);
    }

    /** Implementation for put and putIfAbsent */
    final V putVal(K key, V value, boolean onlyIfAbsent) {
    	// 不接受key或者value为null的情况，这和HashMap也是不一样
        if (key == null || value == null) throw new NullPointerException();
        // 得到key对应的Hash值，也是高16位和低16位做^操作
        int hash = spread(key.hashCode());
        int binCount = 0;
        // 永真循环的重试
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            // 若底层table还没初始化，就需要先初始化
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            ...
        }
    }
    
    private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
        	// 若sizeCtl小于0，表示其他线程正在初始化table。只能有一个线程进行初始化，所以当前线程先挂起
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
            // 用CAS的方式将sizeCtl的值置为-1，若成功，则代表当前线程抢占到了锁，可以进行初始化table了
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if ((tab = table) == null || tab.length == 0) {
                    	// 若sizeCtl没有设置，则直接用默认值16
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        // sc = 3n / 4 ,即默认值为12
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
```
#### 底层就是利用CAS来控制在多线程中选择一个唯一的线程来创建底层table，在ConcurrentHashmap中会大量出现CAS的方法来控制并发。接下来看初始化成功之后做了什么
```
	// 类似HashMap中，利用hash来确定在底层的桶位值
	else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
		// 若当前的桶位值还没有被其他节点占据，则直接新创建一个Node，也是用CAS的方式放在对应的位置
        if (casTabAt(tab, i, null,new Node<K,V>(hash, key, value, null)))
            break;                   // no lock when adding to empty bin
    }
    
    // 以下的CAS方式都是根据每个变量在内存的地址直接置换，不再细述了
    // Unsafe mechanics
    private static final sun.misc.Unsafe U;
    private static final long ABASE;
    private static final int ASHIFT;

    static {
        try {
            U = sun.misc.Unsafe.getUnsafe();
            Class<?> ak = Node[].class;
            ABASE = U.arrayBaseOffset(ak);
            int scale = U.arrayIndexScale(ak);
            if ((scale & (scale - 1)) != 0)
                throw new Error("data type scale not a power of two");
            ASHIFT = 31 - Integer.numberOfLeadingZeros(scale);
        } catch (Exception e) {
            throw new Error(e);
        }
    }
	
    static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
    }

    static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                        Node<K,V> c, Node<K,V> v) {
        return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
    }
    
```
#### 这一段也是和HashMap一样，直接在底层table对应的桶位放入Node元素。但是由于用了CAS的操作，保证了线程安全。
```
    else if ((fh = f.hash) == MOVED)
        tab = helpTransfer(tab, f);
```
#### 这个if判断是hash值为MOVED，也就是-1。这是由于集群正在扩容以及对原有的节点进行移位。这在后面会再讲到,先看到下面这个if判断,这个判断就是若table桶位值已经存在了节点，那么就需要在后面接节点组成链表，链表长度超过阈值的时候，也要转成红黑树。在jdk 1.7的时候并不是这种处理方式的，但是在jdk 1.8的时候变成了和HashMap同样的处理方式。
```
	else {
        V oldVal = null;
        // 需要先锁定桶位的首节点
        synchronized (f) {
            if (tabAt(tab, i) == f) {
                // 若节点的hash值>=0,则表明是链表节点
                if (fh >= 0) {
                	// 作为记录链表的长度
                    binCount = 1;
                    for (Node<K,V> e = f;; ++binCount) {
                        K ek;
                        if (e.hash == hash &&
                            ((ek = e.key) == key ||
                             (ek != null && key.equals(ek)))) {
                            oldVal = e.val;
                            // 只有在值不存在的时候才会去插入
                            if (!onlyIfAbsent)
                                e.val = value;
                            break;
                        }
                        Node<K,V> pred = e;
                        // 遍历链表
                        if ((e = e.next) == null) {
                            pred.next = new Node<K,V>(hash, key,
                                                      value, null);
                            break;
                        }
                    }
                }
                // 若节点的hash值<0,则表明是树节点
                else if (f instanceof TreeBin) {
                    Node<K,V> p;
                    binCount = 2;
                    // 利用红黑树的方式插入
                    if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                   value)) != null) {
                        oldVal = p.val;
                        if (!onlyIfAbsent)
                            p.val = value;
                    }
                }
            }
        }
        // 若链表的长度此时已大于TREEIFY_THRESHOLD，默认值为8，则需要将链表转为红黑树
        if (binCount != 0) {
            if (binCount >= TREEIFY_THRESHOLD)
                treeifyBin(tab, i);
            if (oldVal != null)
                return oldVal;
            break;
        }
    }
```
#### 基本逻辑和HashMap非常相似，只是加了synchronized修饰符，保证线程安全。put方法先讲到这边，留了很多坑，例如红黑树怎么转化，以及最重要的怎么进行扩容。这个先放着，下面先讲讲怎么进行get，之后再回来填坑。
### get
```
    public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        // 同样的得到hash值，为了计算得到后续的桶位值
        int h = spread(key.hashCode());
        // 验证此时table不为空，且已初始化，并在对应的桶位有节点已经放置节点
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
            // 首先验证首节点是否满足条件，是就返回
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            // 若首节点不满足条件，且hash<0，则证明此时已经转化为红黑树，则调用红黑树的方式去寻找
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;
            // 若上面都不符合，则遍历链表，得到满足条件的值
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }
```
