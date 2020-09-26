---
title: HashMap原理
date: 2020-09-23 16:32:32
categories: 集合框架
tags:
---

## 序
学习新知识，绝不是一蹴而就的，得需要一个循循渐进的过程。在分析 HashMap 源码之前咱先聊聊，HashMap怎么用？有哪些特点？

文章中的源码分析基于`JDK 1.8.0_191-b12`
## HashMap 了解
### HashMap 功能及其特性
HashMap 支持 Key - value 键值对储存，其中 key 一般采用 String 类型，若采用 Object 类型需要复写 hashCode() 方法和 equals()方法。HashMap 默认初始容量为 16，且支持动态扩容。HashMap 查找时间复杂度很低，为常数级别。
### HashMap 数据结构
HashMap 底层数据结构为：数组 + 链表 + 红黑树

正是由于底层数据结构为数组，所以 HashMap 的查询效率很高，由于数组容量有限，当发生 hash 冲突时，采用拉链法解决冲突，在数组的“桶”节点上生成链表，当链表过长时，该链表会转化为红黑树。
## HashMap 进阶
在深入 HashMap get() 方法、put() 方法之前，得先了解一些基础知识。还是那句话，学习知识得循循渐进，切莫着急！

### HashMap 构造方法
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
上面三个构造方法中，并没有初始化数组空间，因此大致可以猜想 HashMap 的初始化采用的懒加载方式。

主要看第一个构造方法中的`tableSizeFor()`方法，这个是计算数据容量的方法，计算的结果暂存在`threshold`中。

```
	static final int tableSizeFor(int cap) {
	    int n = cap - 1;
	    n |= n >>> 1;
	    n |= n >>> 2;
	    n |= n >>> 4;
	    n |= n >>> 8;
	    n |= n >>> 16;
	    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
	}
```

大致解释下，该方法的目的是为了计算出大于等于`cap`的最小2的幂。
### HashMap 寻址算法
一般的寻址算法可以采用取模的方式，即 hash 值模以数组大小。而源码中采用`(n - 1) & hash`方式，其中 n 为数组大小。

为什么要采用位运算呢？

位运算效率优于取模运算。

位运算方式与取模方式有什么区别呢？

当 n 为 2 的幂时，位运算与取模的方式等效。
### HashMap hash() 方法
```
	static final int hash(Object key) {
	    int h;
	    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
	}
```

`JDK 1.8`中对`hash()`方法进行了优化，增加了异或操作，可以减少 hash 冲突，但是为什么呢？

因为寻址算法中，会进行 hash & (n - 1) 运算，一般情况下 (n - 1) 比较小，所以在与运算后 hash 值的高位会被直接丢弃掉。而 hash 算法中，把高位和低位进行异或运算，保留了高位的特征，后面进行寻址算法的时候，高位部分也能参与进来。
## HashMap 深入分析
清除前面进阶部分的知识后，理解 HashMap 后面源码就很简单了。
### HashMap get() 方法
```
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
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

`get()`大致流程，红黑树过于复杂，分析过程不包括红黑树

1. 利用寻址算法定位的数组下标中的元素
2. 若当前元素非空，检查节点的 key 是否相等
3. 若第一个节点 key 不相等，且 next 节点非空，遍历链表找到元素为止

以上流程中，判断 key 是否相等的逻辑。

```
first.hash == hash && ((k = first.key) == key || (key != null && key.equals(k)))
```
以 String 类型 key 为例，先判断引用值是否相等，不相等再判断 `equals()` 方法是否相等。这也是为什么用 Object 类作为 key 需要复写 `equals()` 方法的原因。

### HashMap put() 方法
```
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```
`put()`方法看起来一大坨，其实核心代码没几行，简单说下流程

1. 扩容 当数组为空时，需要扩容（初始化发生在`put()`方法中）
2. 查询 寻址算法定位到数组下标 
3. 插入 
	* 数组槽位为空，直接插入
	* 数组槽位非空，根据 key 的引用和 key.equals() 方法进行匹配，匹配不到就插入，匹配到再决定是否替换
4. 扩容 插入之后，判断 hash 的实际容量是否大于阈值，决定是否需要扩容

### HashMap 如何扩容
扩容步骤：

1. 数组容量翻倍
2. 遍历 hash 表中数组，对每个Node进行 rehash （重新计算其在新数组中的下标 ）

`JDK 1.8` rehash 的技巧：扩容之后，需要重新计算 hash 值在新数组中的下标，而寻址算法 hash & (newCap - 1) 有个特点，(newCap - 1) 的二进制值比 (oldCap - 1) 多一个 "1"，因此 rehash 计算出新数组中的下标只有两种可能：

* 保持不变
* 原位置 + oldCap