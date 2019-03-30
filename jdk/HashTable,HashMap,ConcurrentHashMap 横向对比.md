# HashTable,HashMap,ConcurrentHashMap 横向对比
## hash算法
### HashTable
	int index = (key.hashcode & 0x7fffffff) % tab.length
### HashMap
	int index = (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
	最终的桶位值还需要 index & (tab.length -1)
### ConcurrentHashMap
	int index = (key.hashCode() ^ (key.hashCode() >>> 16)) & 0x7fffffff;
	最终的桶位值还需要 index & (tab.length -1)

## k，v能否为null
### HashTable
	k，v均不能为null，v是因为在代码中判断，k是因为null.hashcode()会报错
### HashMap
	k,v均可以为null。 且k为null时，会自动分配其为0号桶位
### ConcurrentHashMap
	k，v均不能为null，强行校验


## 扩容机制
### HashTable
	在put值到底层tab前，会先进行一次count>=threshold的判断，若是，则先直接进行rehash(),再将k,v放到扩容后的tab中
	扩容后的newsize = (oldsize * 2) +1
	倒序遍历oldtab中的每一个桶位，并用首插法的方式，将元素插入到newtab中
	整个过程由synchronized强加锁，避免并发问题
### HashMap
	在put值到tab之后，才会进行容量检查，若(++size > threshold)，则进行扩容
	扩容后的newsize = (oldsize * 2)，保持每次底层tab的容量都为2的幂次方
	顺序遍历oldtab，然后将元素放置在相应的位置，改为尾插法
	且根据bit直接计算对应所在的位置
	整个过程，不做并发控制，所以会有并发问题
### ConcurrentHashMap
	
