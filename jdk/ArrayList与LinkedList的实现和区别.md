# ArrayList与LinkedList的实现和区别

## LinkedList
#### LinkedList实现了Deque的接口，即代表可以在首尾均可进行操作。类中有两个属性，first和last，分别指向链表的首和尾。
#### 在类内部还实现了一个链表节点Node，存储val，pre，next
```
	private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```

#### linkLast() 方法,在最尾端插入节点
```
    void linkLast(E e) {
        final Node<E> l = last;
        final Node<E> newNode = new Node<>(l, e, null);
        last = newNode;
        if (l == null)
            first = newNode;
        else
            l.next = newNode;
        size++;
        modCount++;
    }
```
#### linkFirst() 方法，在最首端插入节点
```
	void linkBefore(E e, Node<E> succ) {
        // assert succ != null;
        final Node<E> pred = succ.prev;
        final Node<E> newNode = new Node<>(pred, e, succ);
        succ.prev = newNode;
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        size++;
        modCount++;
    }	
```
#### linkBefore() 方法，在指定位置插入节点
```
	void linkBefore(E e, Node<E> succ) {
        // assert succ != null;
        final Node<E> pred = succ.prev;
        final Node<E> newNode = new Node<>(pred, e, succ);
        succ.prev = newNode;
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        size++;
        modCount++;
    }
```

#### 以上三个就是类中的灵魂方法，其他的新增节点的方法均会调用到以上三个，例如
#### add()方法，等同于offer方法。直接在尾部插入节点，直接调用linkLast
```
	public boolean add(E e) {
        linkLast(e);
        return true;
    }
```

#### add(int index, E element)方法，在指定的位置插入节点，会先判断插入的位置是不是已经大于了链表长度了，再调用linkBefore
```
	public void add(int index, E element) {
        checkPositionIndex(index);

        if (index == size)
            linkLast(element);
        else
            linkBefore(element, node(index));
    }
    
    // 从前向后，或者从后向前
    Node<E> node(int index) {
        // assert isElementIndex(index);

        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```
#### 再看到对首部进行操作。即push方法，或者offerFirst,最终会调用linkFirst方法
```
	 public void push(E e) {
        addFirst(e);
    }
    
    public void addFirst(E e) {
        linkFirst(e);
    }
```