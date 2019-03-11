# 谈谈HashMap
## 前文
#### 最近正在复习jdk中的一些源码的细节。今天会讲讲Map接口下的HashMap。这个数据结构我们平时肯定也会经常用到，今天会基于jdk 1.8讲讲他底层是怎么实现的，并会对比jdk 1.7看看这两个版本做了什么修改
## 正文
#### 按我们平时的使用习惯，一般就是直接new一个HashMap的实例对象，供之后的操作。那这个构造函数中，HashMap初始化了什么参数呢？
```
   public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }

    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
```
#### 总共有三个不同实现的构造函数。可以看到第1，2种可以看做是同一种实现，因为2中也是调用了1。那我们看到1中也是对传入的参数进行了一些校验，重点看最后一句
```
     this.threshold = tableSizeFor(initialCapacity);
```
#### 我们传入了初始容量大小的值，但是构造函数中并没有对应的接收，而只是用这个变量去计算了thershold，也就是我们之后会用到的容量阈值。tableSizeFor方法的逻辑就是，得到最小的大于入参的2的幂次方。 例如传入7会得到8，传入13会返回16。 这样就做了初始化的动作。注意前两个构造函数初始化了loadFactor，和threshold，而第三个构造方法中值初始化了loadFactor一个方法。这个细节在后面也是会被用到。
#### 既然是HashMap，那最大的作用应该是存储我们给定的键值对。那我们现在看看他是怎么高效存储数据。
```
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
```
#### 这个put方法是常用的方法。可以看到里面真正起作用的是putVal方法，但是里面还有一个hash()方法
```
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```
#### 这个方法是通过给定的key得到散列算法算出的hash值。这个散列方法只有一行，就是得到key的hashcode，然后前16位 ^ 后16位。当然若key是null也是可以接受的，就是直接返回0号位。这个返回的值就是之后会用到的每个key属于哪个桶位的编号。我们接下去看putVal方法。
```
	if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
```
#### 在方法的一开头就先判断底层的数组是否已经初始化成功，如果还没初始化，就需要通过resize方法进行初始化。这个方法通过名字就可以看出应该还有改变大小的作用，这个作用会在方法的最后提到。这边先只关注初始化数组的作用。方法内部的初始化部分这边先不说了，核心就是这一句。new了一个Node的数组。
```
     Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
```
#### 那这个Node是什么呢？这边插播一下Node的定义
```
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
    }    
```
#### 可以看到里面除了一些hash，key，vaule之外，还有一个Node类型的引用，所以可以合理推断这个Node后面会组成一个链表。所以模糊的可以确认出，HashMap的底层是数组+链表的格式组成的。
#### 好了，回到正题。看回原来的putVal方法。接下来的代码格式就是一个if-else的逻辑。
```
		 // (n-1) & hash 计算出给定的key应该保存在数组的第几个位置中
		 // 如果数组的这个位置现在还是空的，则可以直接new一个Node作为第一个占位的节点，放在给定的位置
	     if ((p = tab[i = (n - 1) & hash]) == null)
 	           tab[i] = newNode(hash, key, value, null);
         else {
         }
```
#### 如果数组对应的位置已经有了元素，需要以原有的元素作为链表的头节点往后延伸。但是这里也有几种情况区分。
- 新加入的key已经在数组存在了，这个put操作就成为了用新数据更新旧数据的操作

	```
	if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
        e = p;
	```
- 在数组中存在的链表中继续添加新节点。注意当链表长度大于8的时候，链表会转化为红黑树。这也是jdk 1.8中新增的功能，避免了1.7版本中链表长度过长造成的查询效率低的问题。

```
	for (int binCount = 0; ; ++binCount) {
			// 到达了链表的末尾，则只需要将原尾节点的后继指针指向新加入的节点
            if ((e = p.next) == null) {
                p.next = newNode(hash, key, value, null);
                // 当链表长度大于7的时候，就会红黑树化
                if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                    treeifyBin(tab, hash);
                break;
            }
            // 若在链表中已存在key
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k))))
                break;
            p = e;
     }
     
     // 将原数组中的对应位置的链表转为红黑树
     final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
        // 如果原有的数组的容量小于64，则直接去扩容
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();
        // 已达到转化的阈值，则开始转换
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            TreeNode<K,V> hd = null, tl = null; // 树的首尾节点
            do {
            	// 将原有的Node节点，转为TreeNode
                TreeNode<K,V> p = replacementTreeNode(e, null);
                if (tl == null)
                    hd = p;
                else {
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;
            } while ((e = e.next) != null);
            if ((tab[index] = hd) != null)
                hd.treeify(tab);
        }
    }
    
    TreeNode<K,V> replacementTreeNode(Node<K,V> p, Node<K,V> next) {
        return new TreeNode<>(p.hash, p.key, p.value, next);
    }
```
	
- 在数组中的链表已经转为了红黑树

#### 好了，经过上面的操作。我们就已经成功有序的将元素放在了HashMap的底层数组中，不管是以链表的形式，还是红黑树的方式。可以看到在putVal的最后，会判断一下size是否已经超过了之前设定的阈值。若超过需要调用resize方法进行重新散列。
```
	if (++size > threshold)
            resize();
```
#### 在重新散列方法中，这边就先跳过方法前面部分的一些判断方法，只需要知道若真需要扩容，则将原table的容量大小x2，就得到了新的容量大小。回想到我们之前初始化的时候，容量大小必须是2的幂次方，那么x2之后肯定也会是2的幂次方。为什么要取这个方式呢？我们可以从移动原有数组中的元素到新数组中得到答案。直接看到移动链表的部分
```
	else { // preserve order
		// 这边定义了两对首尾指针，作为之后移动的两个不同链表的首节点入口
        Node<K,V> loHead = null, loTail = null;
        Node<K,V> hiHead = null, hiTail = null;
        Node<K,V> next;
        do {
            next = e.next;
            // 这边看到主要判断了e.hash & oldCap为0还是1
            if ((e.hash & oldCap) == 0) {
                if (loTail == null)
                    loHead = e;
                else
                    loTail.next = e;
                loTail = e;
            }
            else {
                if (hiTail == null)
                    hiHead = e;
                else
                    hiTail.next = e;
                hiTail = e;
            }
        } while ((e = next) != null);
        if (loTail != null) {
            loTail.next = null;
            newTab[j] = loHead;
        }
        if (hiTail != null) {
            hiTail.next = null;
            newTab[j + oldCap] = hiHead;
        }
    }
```
#### 看代码里可以看到主要的逻辑判断是  (e.hash & oldCap)这个值为0还是为1。回想一下，在putVal方法的时候，是用hash & (n-1)的方法来确定元素所在的桶位值，假设n现在是16，此时来了两个hash分别为为15和31的key，那么通过计算
```
	01111 & 1111  = 1111   也就是 (15 & (16-1)) = 15，也就是桶位值为15
   11111 & 1111  = 1111   也就是 (31 & (16-1)) = 15,桶位值同样是15
```
#### 通过上面的计算也可以看出，最左边的bit被忽略了，为0或者1都会被定位到同一个桶位中连接成一个链表。那么当此时容量大小扩大了两倍，那么结果就会变成
```
	01111 & 11111  = 01111   也就是 (15 & (32-1)) = 15，也就是桶位值为15
   11111 & 11111  = 11111   也就是 (31 & (32-1)) = 31,桶位值变成了31
```
#### 也就是最左边的bit值起到了作用，原本同一桶位的两个key可能会被分散在两个不同的桶位中，而根据最左边为0或者为1，就可以轻松的定位到新的桶位

#### 那现在HashMap就已经完成了他一半的实名，接下来他需要做的就是可以让我们能方便的取出我们的数据。直接看到他的get方法
```
    public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
```
#### 可以看到，主要还是要得到key在底层数组中对应的Node。得到Node之后，想得到值就非常容易了。
```
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            // 找到底层数组对应的桶位的第一个节点，直接查看是否满足要求，是的话可以直接返回
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
            	// 若节点已经是红黑树的一部分，则用查找红黑树的方式得到想要的节点
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                // 若是链表，则可以直接按照链表来查找
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```
