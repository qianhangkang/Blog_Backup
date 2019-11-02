---
title: JDK9中HashMap关键变量与方法的分析
date: 2018-04-06 13:13:30
tags: HashMap
---

> 最近准备面试，将自己的一些关于HashMap的理解记录于此，同时附上在Java9中对于HashMap的源码的分析。

<!-- more -->

# 原理图

![](https://user-gold-cdn.xitu.io/2017/12/19/1606cca4f5b48e63?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

## 概述

> 引用极客学院对于hashmap的摘要：
>
> HashMap 是基于哈希表的 Map 接口的非同步实现。此实现提供所有可选的映射操作，并允许使用 null 值和 null 键。此类不保证映射的顺序，特别是它不保证该顺序恒久不变。
>
> 此实现假定哈希函数将元素适当地分布在各桶之间，可为基本操作（get 和 put）提供稳定的性能。迭代 collection 视图所需的时间与 HashMap 实例的“容量”（桶的数量）及其大小（键-值映射关系数）成比例。所以，如果迭代性能很重要，则不要将初始容量设置得太高或将加载因子设置得太低。
>
> 需要注意的是：Hashmap 不是同步的，如果多个线程同时访问一个 HashMap，而其中至少一个线程从结构上（指添加或者删除一个或多个映射关系的任何操作）修改了，则必须保持外部同步，以防止对映射进行意外的非同步访问。




# 变量

## 实例变量

### table

```java
/**
 * The table, initialized on first use, and resized as
 * necessary. When allocated, length is always a power of two.
 * (We also tolerate length zero in some operations to allow
 * bootstrapping mechanics that are currently not needed.)
 */
/**
*这个table，在第一次使用时需要初始化并且会在需要的时候重新扩容。
*当在内存中分配时，这个table的长度总是2的幂次方。
*（为了允许引导机制在某些操作中我们也容忍零长度，但目前这些并不需要）
*/
transient Node<K,V>[] table;
```

table是存储Node的数组，相当于HashMap的地基，而Node是一个静态内部类，继承了Map.Entry<K,V>这个接口，如下
```java
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;//哈希值
        final K key;//键
        V value;//值
        Node<K,V> next;//下一个节点
		//构造函数
        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }
        
		
        /**
        *通过key的hashcode与value的hashcode 异或 得到新的hash值
        */
        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

        public final boolean equals(Object o) {
            if (o == this)
                return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) &&
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
    }
```

### entrySet
```java
    /**
     * Holds cached entrySet(). Note that AbstractMap fields are used
     * for keySet() and values().
     */
     /**
     *保存缓存的entrySet()。注意，AbstractMap字段被用于keySet()和values()
     */
    transient Set<Map.Entry<K,V>> entrySet;
```

### size
```java
    /**
     * The number of key-value mappings contained in this map.
     */
	/**
	*该HashMap中保存的键值对的数量
	*/
    transient int size;
```

### modCount
```java
    /**
     * The number of times this HashMap has been structurally modified
     * Structural modifications are those that change the number of mappings in
     * the HashMap or otherwise modify its internal structure (e.g.,
     * rehash).  This field is used to make iterators on Collection-views of
     * the HashMap fail-fast.  (See ConcurrentModificationException).
     */
     
     /**
     *这个HashMap被结构化修改的次数
     *结构化修改是例如在这个HashMap中修改映射关系的数量或者修改其内部结构（例如 rehash）。这个变量被用	  于在收集视图的快速失败中构建迭代器
     */
    transient int modCount;
```
> `fail-fast` 快速失败是Java集合的一种错误检测机制。当多个线程对集合进行结构上的改变的操作时，有可能会产生fail-fast机制。记住是有可能，而不是一定。例如：假设存在两个线程（线程1、线程2），线程1通过Iterator在遍历集合A中的元素，在某个时候线程2修改了集合A的结构（是结构上面的修改，而不是简单的修改集合元素的内容），那么这个时候程序就会抛出 ConcurrentModificationException 异常，从而产生fail-fast机制。

### threshold

```java
    /**
     * The next size value at which to resize (capacity * load factor).
     *
     * @serial
     */
    // (The javadoc description is true upon serialization.
    // Additionally, if the table array has not been allocated, this
    // field holds the initial array capacity, or zero signifying
    // DEFAULT_INITIAL_CAPACITY.)
    int threshold;
```

当达到这个threshold（capacity * load factor）后，table会重新扩容
意味着默认情况构造情况下，当你存够12个时，table会第一次扩容


### loadFactor

``` java
/**
 * The load factor for the hash table.
 *
 * @serial
 */
final float loadFactor;
```

相关定义与解释请看下面的静态变量中的[DEFAULT_LOAD_FACTOR](../JDK9中HashMap的分析/#DEFAULT-LOAD-FACTOR)



## 静态变量

###DEFAULT_INITIAL_CAPACITY

```java
/**
 * The default initial capacity - MUST be a power of two.
 */
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
```

**默认初始化容量**，默认值为16，且必须是2的幂次方。不知道读者有没有注意到 ` // aka 16 ` 这个注释，很奇怪aka是什么意思，Google上也没有合理的解释，知道的小伙伴们可以留言下：）

AND 还有个细节，为什么这个默认值不直接写成 16，而是写成了 1 << 4。

StackOverflow上给出了一个合理的解释

>It's to emphasize that the number is a power of two, and not a completely arbitrary choice. It thus nudges developers experimenting with different numbers to change it to other numbers in the pattern (e.g., `1 << 3` or `1 << 5`, rather than `25`) so they don't break the code.
>
>蹩脚的翻译：
>
>这是为了强调DEFAULT_INITIAL_CAPACITY是2的幂次方而不是完全任意的选择。因此，这促使开发者尝试使用不同的数字来将其改变为符合其模式的数字（e.g.，...），这样不会让他们破坏代码。



### DEFAULT_LOAD_FACTOR

>```java
>/**
> * The load factor used when none specified in constructor.
> */
>static final float DEFAULT_LOAD_FACTOR = 0.75f;
>```

**默认载入因子**，默认值为0.75。

在这里提出两个问题：

- 什么是载入因子？


  载入因子是表示Hsah表中元素的填满的程度。
  加载因子越大,填满的元素越多,空间利用率越高，但Hash冲突的机会加大了。
  反之,加载因子越小,填满的元素越少,Hash冲突的机会减小,但空间浪费多了。
  Hash冲突的机会越大,则查找的成本越高。反之,查找的成本越小。

- 为什么默认值为0.75？


源码中有这么一段

> Because TreeNodes are about twice the size of regular nodes, we use them only when bins contain enough nodes to warrant use (see TREEIFY_THRESHOLD). And when they become too small (due to removal or resizing) they are converted back to plain bins.  In usages with well-distributed user hashCodes, tree bins are rarely used.  Ideally, under random hashCodes, the frequency of nodes in bins follows a Poisson distribution (http://en.wikipedia.org/wiki/Poisson_distribution) with a parameter of about 0.5 on average for the default resizing threshold of 0.75, although with a large variance because of resizing granularity. Ignoring variance, the expected occurrences of list size k are (exp(-0.5) * pow(0.5, k) / factorial(k)).The first values are:
>
> - 0:    0.60653066
> - 1:    0.30326533
> - 2:    0.07581633
> - 3:    0.01263606
> - 4:    0.00157952
> - 5:    0.00015795
> - 6:    0.00001316
> - 7:    0.00000094
> - 8:    0.00000006
> - more: less than 1 in ten million

大意就是在理想情况下,使用随机哈希码,节点出现的频率在hash桶中遵循[泊松分布](http://www.ruanyifeng.com/blog/2015/06/poisson-distribution.html)，同时给出了桶中元素个数和概率的对照表。从上面的表中可以看到当桶中元素到达8个的时候，概率已经变得非常小，也就是说用0.75作为加载因子，每个碰撞位置的链表长度超过８个是几乎不可能的。

**总而言之，0.75能够让HashMap在 "冲突的机会"与"空间利用率"之间寻找一种平衡与折衷，这样既不会出现太大的冲突，又能够很好的利用空间。**




# 方法

## getNode(int hash, Object key)

大致思路如下

1. 先查找tab[(n - 1) & hash]，命中则返回，否则继续


2. 若tab[(n - 1) & hash]节点为树，则在树中查找，时间复杂度为O(logn)

3. 若为链表，则在链表中查找，时间复杂度为O(n)

   

```java
    /**
     * Implements Map.get and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @return the node, or null if none
     */
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        //当table不为null，长度大于0并且根据(n - 1) & hash 得到对应的节点Node不为null
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            //如果传进来的hash与key与索引对应的链表（红黑树）的Node的key与hash相同，则表明找到了
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            //当存在下一个节点时
            if ((e = first.next) != null) {
                //如果第一个节点属于红黑树，那么在红黑树中进行查找
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                //链表查找
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

* (n - 1) & hash 的作用是什么？


相当于%（取模）操作，防止hash out of range.
StackOverflow（OK，又是你~）上也有外国朋友问了[该问题](https://stackoverflow.com/questions/27230938/why-hashmap-insert-new-node-on-index-n-1-hash)

* 接着上一题问，那为什么不用%而用了&？

首先，对于现代的处理器来说，除法和求余数（模运算）是最慢的动作。

其次，由于在HashMap中，容量一定为2的幂次方。所以当b为2的幂次方时，如下等式成立 `a % b == (b-1) & a`

## putVal(int hash, K key, V value, boolean onlyIfAbsent,boolean evict)
![](https://tech.meituan.com/img/java-hashmap/hashMap%20put%E6%96%B9%E6%B3%95%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B%E5%9B%BE.png)

```java
    /**
     * Implements Map.put and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @param value the value to put
     * @param onlyIfAbsent if true, don't change existing value
     * @param evict if false, the table is in creation mode.
     * @return previous value, or null if none
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        //如果table为null或长度为0，那么扩容
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        //如果对应的节点Node为null，新建一个Node节点并插入键值对，此时没有冲突
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        //若存在冲突
        else {
            Node<K,V> e; K k;
            //如果第一个节点刚好匹配key，直接跳出循环，在循环外覆盖旧值
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            //如果为树节点
            else if (p instanceof TreeNode)
                //红黑树插入键值对
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            //如果为链表
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        //先插入键值对
                        p.next = newNode(hash, key, value, null);
                        //如果链表长度大于TREEIFY_THRESHOLD（8），那么将该链表构造为红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    //如果在链表中匹配到了key，直接跳出循环，在循环外覆盖旧值
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            //如果能够匹配key
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                //将新的值替换旧的值
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        //判断是否需要resize即是否超过threshold = capacity * load factor
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```



## hash(Object key)

```java
    /**
     * Computes key.hashCode() and spreads (XORs) higher bits of hash
     * to lower.  Because the table uses power-of-two masking, sets of
     * hashes that vary only in bits above the current mask will
     * always collide. (Among known examples are sets of Float keys
     * holding consecutive whole numbers in small tables.)  So we
     * apply a transform that spreads the impact of higher bits
     * downward. There is a tradeoff between speed, utility, and
     * quality of bit-spreading. Because many common sets of hashes
     * are already reasonably distributed (so don't benefit from
     * spreading), and because we use trees to handle large sets of
     * collisions in bins, we just XOR some shifted bits in the
     * cheapest possible way to reduce systematic lossage, as well as
     * to incorporate impact of the highest bits that would otherwise
     * never be used in index calculations because of table bounds.
     */

	/**  		Google Translation~
          计算key.hashCode（）并散布（XOR）高位散列
          降低。 由于该表使用幂的两个掩码，
          散列值只会在当前掩码之上的位上发生变化
          总是碰撞。 （在已知的例子中是Float键的集合
          在小表中保持连续的整数）。所以我们
          应用扩展高位影响的变换
          向下。 在速度，效用和方法之间进行权衡
          比特扩散的质量。 因为许多常见的哈希集合
          已经合理分配（所以不会从中受益
          传播），并且因为我们使用树来处理大量的
          在箱中发生碰撞，我们只是在XOR中异或
          最便宜的方法来减少系统损失，以及
          合并否则会影响最高位
          由于表的边界，决不会用于索引计算。
     */
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

一句话解释为什么要这么做？**为了更好的让存储的下标均匀分布，尽可能的减少hash碰撞**

WHY？

> 首先，假设有一种情况，对象 A 的 hashCode 为 1000010001110001000001111000000，对象 B 的 hashCode 为 0111011100111000101000010100000。
>
> 如果数组长度是16，也就是 15 与运算这两个数， 你会发现结果都是0。这样的散列结果太让人失望了。很明显不是一个好的散列算法。
>
> 但是如果我们将 hashCode 值右移 16 位，也就是取 int 类型的一半，刚好将该二进制数对半切开。并且使用位异或运算（如果两个数对应的位置相反，则结果为1，反之为0），这样的话，就能避免我们上面的情况的发生。
>
> 总的来说，使用位移 16 位和 异或 就是防止这种极端情况。但是，该方法在一些极端情况下还是有问题，比如：10000000000000000000000000(33554432) 和 10000000000100000000000000(33570816) 这两个数，如果数组长度是16，那么即使右移16位，在异或，hash 值还是会重复。但是为了性能，对这种极端情况，JDK 的作者选择了性能。毕竟这是少数情况，为了这种情况去增加 hash 时间，性价比不高。



## resize()

分析resize()函数之前，先提出一个问题：

* 如果重新扩容，那么势必会导致tab[(n-1)&hash]中的index不同，以致于无法找到对应的键值对，怎么办？

1. 对所有的键值对重新执行putVal(int hash, K key, V value, boolean onlyIfAbsent,boolean evict)操作。这么做开销很大，只在JDK8之前这么做
![](https://tech.meituan.com/img/java-hashmap/jdk1.7%E6%89%A9%E5%AE%B9%E4%BE%8B%E5%9B%BE.png)
2. 经过观测可以发现，我们使用的是2次幂的扩展(指长度扩为原来2倍)，所以，元素的位置要么是在原位置，要么是在原位置再移动2次幂的位置。如下图可以明白这句话的意思，n为table的长度，图（a）表示扩容前的key1和key2两种key确定索引位置的示例，图（b）表示扩容后key1和key2两种key确定索引位置的示例，其中hash1是key1对应的哈希与高位运算结果。


![](https://tech.meituan.com/img/java-hashmap/hashMap%201.8%20%E5%93%88%E5%B8%8C%E7%AE%97%E6%B3%95%E4%BE%8B%E5%9B%BE1.png)

​	元素在重新计算hash之后，因为n变为2倍，那么n-1的mask范围在高位多1bit(红色)，因此新的index就会发生这样的变化：

![](https://tech.meituan.com/img/java-hashmap/hashMap%201.8%20%E5%93%88%E5%B8%8C%E7%AE%97%E6%B3%95%E4%BE%8B%E5%9B%BE2.png)

​	因此，我们在扩充HashMap的时候，不需要像JDK1.7的实现那样重新计算hash，只需要看看原来的hash值新增的那个bit是1还是0就好了，是0的话索引没变，是1的话索引变成“原索引+oldCap”，可以看看下图为16扩充为32的resize示意图：

![](https://tech.meituan.com/img/java-hashmap/jdk1.8%20hashMap%E6%89%A9%E5%AE%B9%E4%BE%8B%E5%9B%BE.png)

​	这个设计确实非常的巧妙，既省去了重新计算hash值的时间，而且同时，由于新增的1bit是0还是1可以认为是随机的，因此resize的过程，均匀的把之前的冲突的节点分散到新的newTable了



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

	/**
	*初始化或者加倍table的大小。如果为null，那么分配空间【这段实在不知道该怎么翻译了：）】
	*否则，由于我们使用了幂级别的扩展，那么在table中每个桶中的元素要么保持相同的索引，要么移动两倍的偏移
	*量
	*/
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            //如果容量超过最大值2^30，无法扩容了，那么只能让其碰撞，
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            //没超过最大值，那么扩大为原来的2倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                //threshold也需要扩大为2倍，否则下次触发resize时，用的还是原先的threshold，因此会提前扩容
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                //原先table上的Node不为空
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    //如果仅仅存在一个Node，重新计算index并放置进newTab中
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        //红黑树分裂
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    //节点后为链表
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            // 原索引
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            // 原索引 + oldCap
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        //原索引放到newTab里
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        //原索引 + oldCap 放到newTab里
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```



# 总结

>1. 扩容是一个特别耗性能的操作，所以当程序员在使用HashMap的时候，估算map的大小，初始化的时候给一个大致的数值，避免map进行频繁的扩容。
>2. 负载因子是可以修改的，也可以大于1，但是建议不要轻易修改，除非情况非常特殊。
>3. HashMap是线程不安全的，不要在并发的环境中同时操作HashMap，建议使用ConcurrentHashMap。
>4. JDK8之后引入红黑树大程度优化了HashMap的性能。






# 参考链接

https://my.oschina.net/weiweiblog/blog/612812

https://www.jianshu.com/p/dff8f4641814

https://yikun.github.io/2015/04/01/Java-HashMap%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%8F%8A%E5%AE%9E%E7%8E%B0/

https://blog.csdn.net/qq_38182963/article/details/78940047#comments

https://tech.meituan.com/java-hashmap.html




> 大部分正义感都是虚伪的 聊胜于无   —————————————沃德彭尤·胡某

