# 聊一聊HashTable
## 前言
#### 关于HashTable，今天想深入的看一下源码实现，通过其暴露的接口方法，看看他内部实现的原理。并和HashMap简单对比一下
## 正文
#### 大家都知道HashTable简单来说，就是一个存储kv键值对的一个数据结构。用put方法放进去，用get方法拿出来。那内部到底是做了什么样的操作，可以做到这个方便又快速的存储呢？
#### 这边用一个小例子作为入口，我们可以进去一探究竟
```
    public static void main(String[] args) {
        // 初始化
        Hashtable hb = new Hashtable();
        // 存储
        hb.put("1",new Integer(1));
        hb.put("2",new Integer(2));
        hb.put("3",new Integer(3));
        // 取值
        System.out.println(hb.get("1"));
        System.out.println(hb.get("2"));
        System.out.println(hb.get("3"));


    }
```
#### 这边写了一个最简单的例子。例子包括三部分，在注释中也有指出。
- 初始化 
- 存储
- 取值

#### 这三个步骤也是我们平时用到的流程，本文就会根据这三个小块来探究一下内部的实现
## 初始化
#### 当我们想要使用一个HashTable的时候，最显而易见的就是我们会先new一个实例化对象出来。这样我们才会有对象可操作，那么就先看看这个构造函数。
# TODO 构造函数截图
#### 从这个图中也可以看到，总共提供了四种不同的构造函数。其实当我们看过实现之后我们可以发现，其实四个不同的方法最终都是调用了Hashtable(int, float) 这个方法，其他构造方法只是提供了一些默认参数而已。所以我们直接先看到这个方法。
```
    public Hashtable(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal Load: "+loadFactor);

        if (initialCapacity==0)
            initialCapacity = 1;
        this.loadFactor = loadFactor;
        // 上面的代码主要对参数的格式做了一些检验，例如initialCapacity不能为负数，而且最小也必须是1，
        // loadFactor 必须是大于零的浮点数
        
        // 再根据initialCapacity参数创建一个Entry数组
        table = new Entry<?,?>[initialCapacity];
        // 初始化threshold，(两个参数的乘积)和(Int最大值-8)中的较小值
        threshold = (int)Math.min(initialCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
    }
```
#### 这样我们就完成了HashTable的部分参数初始化的工作。可以看到主要是创建了一个Entry的数组。另外如果我们使用的是无参数的构造函数，那么这两个的参数的默认值为11, 0.75f。
## 存储
#### 现在我们看一下当我们put的时候，内部发生了什么
```
	// 可以看到这个方法名上加了synchronized，这和在HashMap上是有不一样的地方
    public synchronized V put(K key, V value) {
        // Make sure the value is not null
        if (value == null) {
            throw new NullPointerException();
        }

        // Makes sure the key is not already in the hashtable.
        Entry<?,?> tab[] = table;
        // 会取传入的键的Hashcode，所以每个字符串的Hashcode都是一样的
        int hash = key.hashCode();
        // 通过计算得到当前键在Entry数组中的下标
        int index = (hash & 0x7FFFFFFF) % tab.length;
        @SuppressWarnings("unchecked")
        // 取出在数组中指定下标已存在的值，指定下标取出来可能是一个链表，在Entry中存有链表后继结点的引用
        Entry<K,V> entry = (Entry<K,V>)tab[index];
        for(; entry != null ; entry = entry.next) {
        	// 必须双重判断hash值一样，而且键值内容一样，才能说明传入的key已存在
            if ((entry.hash == hash) && entry.key.equals(key)) {
            	// 若已存在，则替换新传入的值，并返回旧值
                V old = entry.value;
                entry.value = value;
                return old;
            }
        }
		// 若key在原结构中不存在，则需要新增
        addEntry(hash, key, value, index);
        return null;
    }
```
#### 继续看看往存储结构中新增一个key的时候做了什么操作
```
    private void addEntry(int hash, K key, V value, int index) {
    	// 这个字段作为修改的一个version值做后续的版本控制
        modCount++;

        Entry<?,?> tab[] = table;
    	// count是当前数组中已有的Entry的数目
    	// 如果比原定的threshold要大，则需要重新进行一次hash重分配
        if (count >= threshold) {
            // Rehash the table if the threshold is exceeded
            rehash();

            tab = table;
            hash = key.hashCode();
            index = (hash & 0x7FFFFFFF) % tab.length;
        }

        // Creates the new entry.
        @SuppressWarnings("unchecked")
        // 在Entry数组对应的index的位置创建一个值，并把可能原有的值作为新增节点在链表中的后继结点存储，所以在链表中是新的节点排在链表前面
        Entry<K,V> e = (Entry<K,V>) tab[index];
        tab[index] = new Entry<>(hash, key, value, e);
        count++;
    }
```
#### 新增部分也容易理解，主要还是数组+链表的格式。可以看到在过程中已经限定了k，v都不能为空。这边留了一个概念没讲，就是这个rehash()的过程。在初始化的时候，我们定义了threshold这个int类型的参数。这个的概念就是，假设我们数组的最大可容纳数目为11个，loadFactor为0.75，那么当数目达到了8个的时候，我们就要进行一次rehash，避免数目过大影响性能。现在具体看看rehash过程。以下过程内容，都按照默认值11，0.75为前提。
```
	protected void rehash() {
		// old容量 默认为11
        int oldCapacity = table.length;
        Entry<?,?>[] oldMap = table;

        // overflow-conscious code
        // new容量 = (11 << 1) + 1 = 23 
        int newCapacity = (oldCapacity << 1) + 1;
        // MAX_ARRAY_SIZE 是Int.max - 8， 如果新的容量比这个数值还要大的话，要进行进一步的考量
        if (newCapacity - MAX_ARRAY_SIZE > 0) {
            if (oldCapacity == MAX_ARRAY_SIZE)
                // Keep running with MAX_ARRAY_SIZE buckets
                return;
            newCapacity = MAX_ARRAY_SIZE;
        }
        // 用新的容量创建一个新的数组
        Entry<?,?>[] newMap = new Entry<?,?>[newCapacity];

        modCount++;
        // 重新计算threshold
        threshold = (int)Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
        table = newMap;
		// 从原有数组的末尾，从后往前扫描每一个节点，按照新的容量计算新的index。
        for (int i = oldCapacity ; i-- > 0 ;) {
            for (Entry<K,V> old = (Entry<K,V>)oldMap[i] ; old != null ; ) {
                Entry<K,V> e = old;
                old = old.next;

                int index = (e.hash & 0x7FFFFFFF) % newCapacity;
                e.next = (Entry<K,V>)newMap[index];
                newMap[index] = e;
            }
        }
    }
```
#### 好的，现在也大概吧put梳理了一下。整体大概就是用一种数组+链表的方式存储我们的节点，并在必要时刻进行重新整理，接下来我们看看取值的过程
## 取值
```
	// 同样是一个同步的方法
    public synchronized V get(Object key) {
    	// 整个方法是不是似曾相识？在我们第二步的存储的代码中，已经包含了这一部分代码。当时是为了确定存入的值是否已经存在。所以这边也不赘述了
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        for (Entry<?,?> e = tab[index] ; e != null ; e = e.next) {
            if ((e.hash == hash) && e.key.equals(key)) {
                return (V)e.value;
            }
        }
        return null;
    }
```
## 总结
#### 大致梳理了一下HashTable的几个关键流程，有一些细节还没有讲到。之后会再写一篇关于HashMap的文章，在那边可以横向对比一下这两者之间的区别。
