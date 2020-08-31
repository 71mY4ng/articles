# java Collection

## Array 和 Arraylist

array 是java 的数组基本类型，初始化的时候需要定长，使用过程中无法方便地动态扩容，对于数组元素的操作比较简单

arrayList 是 java 提供的 Collection 类, 基于array实现的, 是可以动态扩容的。

由于arrayList 是依靠维护内部的一个内容数组的，所以其内存空间是连续的。

ArrayList 不是线程安全的集合，如果要使用线程安全的列表，可以使用 `Collections.synchronizedList(List<T>)`, 其产生的对象带有 `SynchronizedCollection` 实现，在每个内容操作方法上嵌套了 synchronized 的同步块。也可以使用 `CopyOnWriteArrayList`, 后者使用了 `ReentralLock`, 对操作加锁，每次更新的时候会把内部的数组复制一份。

## LinkedList

ArrayList 是基于数组的，所以存储的内存空间是连续的顺序表结构，但如果我们希望实现在任意位置上快速的插入和删除操作，就需要使用到链表结构，在java 中就提供了 `LinkedList`。



## HashMap

HashMap 的实现就是 哈希表 + 链表

几个重要的常量

```java
    /**
     * The default initial capacity - MUST be a power of two.
     */
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

    /**
     * The maximum capacity, used if a higher value is implicitly specified
     * by either of the constructors with arguments.
     * MUST be a power of two <= 1<<30.
     */
    static final int MAXIMUM_CAPACITY = 1 << 30;

    /**
     * The load factor used when none specified in constructor.
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    /**
     * The bin count threshold for using a tree rather than list for a
     * bin.  Bins are converted to trees when adding an element to a
     * bin with at least this many nodes. The value must be greater
     * than 2 and should be at least 8 to mesh with assumptions in
     * tree removal about conversion back to plain bins upon
     * shrinkage.
     */
    static final int TREEIFY_THRESHOLD = 8;

    /**
     * The bin count threshold for untreeifying a (split) bin during a
     * resize operation. Should be less than TREEIFY_THRESHOLD, and at
     * most 6 to mesh with shrinkage detection under removal.
     */
    static final int UNTREEIFY_THRESHOLD = 6;

    /**
     * The smallest table capacity for which bins may be treeified.
     * (Otherwise the table is resized if too many nodes in a bin.)
     * Should be at least 4 * TREEIFY_THRESHOLD to avoid conflicts
     * between resizing and treeification thresholds.
     */
    static final int MIN_TREEIFY_CAPACITY = 64;

```

* `DEFAULT_INITIAL_CAPACITY` 初始化大小默认值16，需要是2次幂，原因是可以将取模运算转成位运算以提高性能
* `MAXIMUM_CAPACITY` 哈希表最大长度，2^30
* `DEFAULT_LOAD_FACTOR` 默认加载因子, 0.75f, 扩容的时候用到, threshold = table.capacity * LOAD_FACTOR, 所以默认情况下，put的k-v 个数超过12 就会扩容
* `TREEIFY_THRESHOLD` 将哈希桶从链表转为红黑树的桶内元素个数阈值 8
* `UNTREEIFY_THRESHOLD` 扩容(resize)后会有对桶上的内容进行拆分(split)的操作, 拆分完的子链表长度小于6就从TreeNode变成普通链表
* `MIN_TREEIFY_CAPACITY` 哈希表的容量超出64就进行桶上的链表转树过程，否则还是扩容

### put 操作

1. 哈希桶数组 table 为空时，通过 resize() 方法进行初始化，为了保证拿到table.length 作为散列 index 的计算
2. 若上一步计算出的桶index位置为空，将k-v构造成一个新的链表Node填入此位置
3. 如果桶index位置不为空，说明这一步出现了hash冲突，需要在桶上找到位置
4. 待插入的 key 是桶上的头节点，覆盖 value，注意如果用的是putIfAbsent, 这里只会在对应value == null 的时候才会写值
5. 桶上的结构是红黑树结构，那么要去红黑树里遍历找空位置，进行balanceInsertion，如果key存在还是覆盖value
6. 桶上结构是普通链表，顺序遍历链表，找到空位（尾插），或者存在key就覆盖value
7. 普通链表插入完成后需要判断一下所在桶的长度，如果超出了 TREEIFY_THRESHOLD 那么需要进行从链表到红黑树的转换`treeifyBin()`，或是 `resize()`

__modCount__

另外注意最后这个变量modCount:

```java
    /**
     * The number of times this HashMap has been structurally modified
     * Structural modifications are those that change the number of mappings in
     * the HashMap or otherwise modify its internal structure (e.g.,
     * rehash).  This field is used to make iterators on Collection-views of
     * the HashMap fail-fast.  (See ConcurrentModificationException).
     */
    transient int modCount;
```

在涉及到对HashMap 数据的新增或修改的操作都会增加这个modCount, 意义在于 Iterator 遍历的时候如果出现了结构性修改，即modCount 变化了，那么将会抛出 `ConcurrentModificationException`。这符合fail-fast 的原则，与其让Iterator 考虑结构可能变化的逻辑，不如在出现并发修改的情况直接抛出异常结束流程。

### HashMap 扩容 (resize)

```java
    /**
     * Initializes or doubles table size.  If null, allocates in
     * accord with initial capacity target held in field threshold.
     * Otherwise, because we are using power-of-two expansion, the
     * elements from each bin must either stay at same index, or move
     * with a power of two offset in the new table.
     *
     * @return the table
     */
    final Node<K,V>[] resize() {
        ...
    }
```

resize方法的代码比较长而且比较复杂，但上下可以分为“扩容空间”和“数据搬迁”两个部分。

1. 扩容空间的部分

主要还是计算新的 __容量(capacity)__ 和 __resize上限(threshold)__

oldCapacity 超出了 MAXIMUM_CAPACITY 就不再扩容了，直接原样返回哈希表。

正常情况下，按原来的两倍来扩容，这个2倍的由来是: `(newCap = oldCap << 1)`, threshold 一般也是两倍: `newThr = oldThr << 1;`, 但如果用户指定了一个小于默认capacity的初始化容量，后面按新的capacity重新计算，公式还是 `threshold = capacity * LOAD_FACTOR`。

由此，hashMap 得出了新空间的容量和上限，创建一个新的hashtable 空间:

```java
Node<K,V>[] oldTab = table;
Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
table = newTab;
```

2. 数据搬迁

由于hashtable 的 capacity 被增大了1倍，hashtable 上的key 的位置就可能会出现变化了。

于是遍历旧table 上每个桶里的数据，重新计算在newTable 上的散列位置:

```java
for (int j = 0; j < oldCap; ++j) {
    Node<K,V> e;
    if ((e = oldTab[j]) != null) {
        oldTab[j] = null;
        if (e.next == null)                         // 请注意这里只是对未冲突的桶
            newTab[e.hash & (newCap - 1)] = e;      // 重新散列
        ...
    }
}
```

__原来桶上有冲突的Node怎么处理？__

resize 是将capacity 按原来的2倍增大的，这样原来元素的hash值在oldCapacity 上溢出的部分会到新扩的空间上, 没有溢出的元素则还是在原位。可以看源代码这部分，作者命名了两个链表结构，loHead (low, 低位链表头指针), hiHead (high, 高位链表头指针):

```java
// 拆链表，拆成两个子链表并保持原有顺序
Node<K,V> loHead = null, loTail = null;
Node<K,V> hiHead = null, hiTail = null;
Node<K,V> next;
do {
  next = e.next;
  // 原位置不变的子链表
  if ((e.hash & oldCap) == 0) {
    if (loTail == null)
      loHead = e;
    else
      loTail.next = e;
    loTail = e;
  }
  // 原位置偏移 oldCap 的子链表
  else {
    if (hiTail == null)
      hiHead = e;
    else
      hiTail.next = e;
    hiTail = e;
  }
} while ((e = next) != null);
```

推演一下，假设默认的HashMap的一个bin上面存在4个hash冲突的key, 经过一次resize

```java
oldCapacity = 16;   // 0001 0000
newCapacity = 32;   // 0010 0000 (16 << 1)

hashCode1 = 90；    // 0101 1010 
hashCode2 = 58;     // 0011 1010
hashCode3 = 10;     // 0000 1010
hashCode4 = 76;     // 0100 1010
// index = hashCode & (capacity - 1)
old_index = 10;     // 0000 1010 冲突了
new_index1 = 90 & 31 = 26;  // 0101 1010 & 0001 1111 = 0001 1010
new_index2 = 58 & 31 = 26;  // 0011 1010 & 0001 1111 = 0001 1010
new_index3 = 10 & 31 = 10;  // 0000 1010 & 0001 1111 = 0000 1010
new_index4 = 76 & 31 = 10;  // 0100 1010 & 0001 1111 = 0000 1010
```

可见原本在oldCapacity 上冲突的4个key, 有两个hash值因为newCapacity的扩容下移动到了高位上。而且新index 和旧index 的变化量是 oldCapacity, 这样代码里面只需要看oldCapacity 这个二进制位上是否有溢出就可以判断key的移动了，如果`(e.hash & oldCap) == 0` 说明位置不变，如果 `(e.hash & oldCap) == 1` 说明要移动到 index = oldIndex + oldCapacity的位置上。

另外，由于这部分是链表的顺序操作，用的是尾插法，所以移动到新的桶后也能保证子链表上的原顺序。

__原桶上的结构是红黑树__

```java
else if (e instanceof TreeNode)
    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
```

TreeNode 的split其实仅比上面多了根据low_count和high_count 与 TREEIFY_THRESHOLD 和 UNTREEIFY_THRESHOLD 的比较，判断是否要进行 untreeification 或是 treeification, 即低于建树阈值从红黑树变成普通链表。

同样也是按newCapacity 计算新的散列index, 分出lowHead, highHead。但分别根据低位和高位链表的长度 lc 和 hc, 判断子链表是用红黑树还是链表;

```java
    for (TreeNode<K,V> e = b, next; e != null; e = next) {
        next = (TreeNode<K,V>)e.next;
        e.next = null;
        if ((e.hash & bit) == 0) {
            ... // e 在原位 (loTail)
            ++lc;
        }
        else {
            ... // e 在高位 (hiTail)
            ++hc;
        }
    }

    if (loHead != null) {
        if (lc <= UNTREEIFY_THRESHOLD)
            tab[index] = loHead.untreeify(map);
        else {
            tab[index] = loHead;
            if (hiHead != null) // (else is already treeified)
                loHead.treeify(tab);
        }
    }
    if (hiHead != null) {
        if (hc <= UNTREEIFY_THRESHOLD)
            tab[index + bit] = hiHead.untreeify(map);
        else {
            tab[index + bit] = hiHead;
            if (loHead != null)
                hiHead.treeify(tab);
        }
    }
```

__为什么hashmap 要将超出阈值的桶转成红黑树？__

如果桶的长度超出了定义的阈值，说明哈希冲突的情况比较严重，接下来索引冲突的键值对需要去链表进行O(n) 复杂度的遍历，转换成红黑树可以把这个过程的时间复杂度降低到O(logN) 级别

## LinkedHashMap

LinkedHashMap 和 HashMap 的使用区别是：HashMap 是无序散列的，而 LinkedHashMap是有序的

根据 LinkedHashMap 内的 accessOrder 标识，可以分别采用“插入顺序”和“访问顺序”的排列方式。

```java
    /**
     * The iteration ordering method for this linked hash map: <tt>true</tt>
     * for access-order, <tt>false</tt> for insertion-order.
     *
     * @serial
     */
    final boolean accessOrder;
```

### linkedHashMap 如何实现其有序性?

查看源码可以发现, LinkedHashMap 许多方法继承自 HashMap，在HashMap 的k-v元素Node 增加了 before, after 指针，成为了双向链表:

```java
    /**
     * HashMap.Node subclass for normal LinkedHashMap entries.
     */
    static class Entry<K,V> extends HashMap.Node<K,V> {
        Entry<K,V> before, after;
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }
```

原 HashMap 的Node

```java
    /**
     * Basic hash bin node, used for most entries.  (See below for
     * TreeNode subclass, and in LinkedHashMap for its Entry subclass.)
     */
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;
        ...
    }
```

在 `newNode` 和 `newTreeNode` 后加入 `linkNodeLast(p)` 操作，将 after, before, tail指针进行维护，使得双向链表的逻辑顺序上，新插入的结点总是排列在已插入结点之后，这样实现了插入顺序的记录。

在 `afterNodeAccess` 方法增加了将访问结点的before, after, tail, head进行修改，使得双向链表的逻辑顺序上，被访问的结点被移动到顺序链表的最后，这样实现了最近访问的结点在最末尾的顺序记录，前提是初始化的时候 `accessOrder` 为 true。

简言之，LinkedHashMap 为 HashMap 另外建立了一个逻辑双向链表来记录k-v插入顺序和访问顺序。

### LRU

LinkedHashMap 提供了实现LRU 的方法，HashMap 提供了 `afterNodeInsertion` 后置方法, LinkedHashMap则将其覆写使得其具有LRU功能：

```java
    void afterNodeInsertion(boolean evict) { // possibly remove eldest
        LinkedHashMap.Entry<K,V> first;
        if (evict && (first = head) != null && removeEldestEntry(first)) {
            K key = first.key;
            removeNode(hash(key), key, null, false, true);
        }
    }
```

默认情况下 `removeEldestEntry` 是返回 false 的，也即是默认不开启 LRU 功能，如果希望使用LRU, 可以将 `removeEldestEntry` 方法覆写。

## TreeMap

> A Red-Black tree based `NavigableMap` implementation.
> The map is sorted according to the `Comparable` natural ordering of its keys, or by a  `Comparator` provided at map creation time, depending on which constructor is used.

TreeMap 是基于 NavigableMap 而实现的红黑树结构。

因为其 NavigableMap 接口的特性，具有以下API

* ceilling 大于等于给定项目的最小值
* descending 返回逆序视图
* floor 小于等于给定项的最大值
* head 返回远小于给定项的元素集合, `[head, toE]`
* tail 返回远大于给定项的元素集合, `[toE, tail]`
* higher 返回严格大于给定项的元素, 
* lower 返回严格小于给定项的元素