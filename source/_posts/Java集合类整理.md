---
title: Java集合类整理
date: 2018-04-12 10:44:28
tags: 集合类
---

Java集合类关系挺复杂的，这一次先从集合类开始整理，陆续会整理下线程池、二叉树、JVM和GC



<!--more-->

# Java集合类图

![Java集合类图](https://upload-images.jianshu.io/upload_images/5543739-d576aa33540ee84a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/561)



咋一看感觉挺混乱的~~~但是多看两遍后这个图的结构还是很清晰的



# Collection接口

```java
public interface Collection<E> extends Iterable<E>{
    int size();
    boolean isEmpty();
    boolean contains(Object o);
    Iterator<E> iterator();
    Object[] toArray();
    <T> T[] toArray(T[] a);
    boolean add(E e);
    boolean remove(Object o);
    boolean containsAll(Collection<?> c);
    boolean addAll(Collection<? extends E> c);
    boolean removeAll(Collection<?> c);
    boolean retainAll(Collection<?> c);
    void clear();
    boolean equals(Object o);
    int hashCode();
}
```





Collection接口是集合类接口的root接口。一个集合代表了一组对象，即**elements**。一些集合允许集合重复，一些不允许。一些能进行排序，一些不行。JDK没有<直接>提供任何这个接口的实现，但是它提供了更多的特定的实现这个接口的子接口像Set、List。这个接口通常用于传递集合并在需要最大通用性的情况下对其进行操作。



# Set接口

```java
/**
这个接口实现了Collection接口，并且在JDK9之后多了static <E> Set<E> of(...)方法，该方法返回了一个构造好的final Set
*/
public interface Set<E> extends Collection<E>
```





一个不包含重复元素的集合。更正式的说，这个集合不包含元素e1、e2，使得e1.equals(e2)，同时，该集合最多含有一个null元素。正如它的名字所暗示的，这个接口模拟了数学的集合抽象。



## HashSet

```java
public class HashSet<E>
    extends AbstractSet<E>
    implements Set<E>, Cloneable, java.io.Serializable
```





这个类实现了Set接口，底层是由hash table支持的（事实上就是一个HashMap实例）。它不保证集合迭代的顺序，特别是，它不保证顺序会随着时间保持不变。这个类允许null元素。

总而言之，在源码中，HashSet类中只有两个变量

1. private transient HashMap<E,Object> map;

2. static final Object PRESENT = new Object();

   这个PRESENT源码上的注释是*Dummy value to associate with an Object in the backing Map*，意思就是用来填充HashMap中的value值。这个也可以从源码的add方法中可以看出。

   ```java
   public boolean add(E e) {
       return map.put(e, PRESENT)==null;
   }
   ```

   



### LinkedHashSet

```java
public class LinkedHashSet<E>
    extends HashSet<E>
    implements Set<E>, Cloneable, java.io.Serializable
```





LinkedHashSet是Set接口的hash表和链表的实现，具有可预测迭代顺序。LinkedHashSet与HashSet不同的地方在于LinkedHashSet维护了一个双向列表`双向链表也叫双链表，是链表的一种，它的每个数据结点中都有两个指针，分别指向直接后继和直接前驱`，它贯穿了所有entries。这个链表定义了迭代的顺序，这个顺序是元素被插入到集合中的顺序。需要注意的是，插入元素的顺序不会收到影响如果一个元素被重新插入到这个集合中。比如，当s.contains(e)在调用之前立即返回true，那么s.add(e)被调用时，元素e会被重新插入到集合s中。



## TreeSet

```java
public class TreeSet<E> 
	extends AbstractSet<E>
    implements NavigableSet<E>, Cloneable, java.io.Serializable
```





TreeSet实现了NavigableSet接口，NavigableSet接口继承自SortedSet接口。TreeSet是基于TreeMap是实现的。这些元素将会使用Comparable自然排序，或者使用在集合创建时指定的Comparable进行排序，具体取决于使用哪个构造方法。TreeSet保证了基本操作（add,remove,contains）的时间复杂度，为log(n)。



# SortedSet接口

```java
public interface SortedSet<E> extends Set<E>{
    Comparator<? super E> comparator();
    SortedSet<E> subSet(E fromElement, E toElement);
    SortedSet<E> headSet(E toElement);
    SortedSet<E> tailSet(E fromElement);
    E first();
    E last();
    @Override
    default Spliterator<E> spliterator() {
        return new Spliterators.IteratorSpliterator<E>(
                this, Spliterator.DISTINCT | Spliterator.SORTED | Spliterator.ORDERED) {
            @Override
            public Comparator<? super E> getComparator() {
                return SortedSet.this.comparator();
            }
        };
    }
}
```





SortedSet接口继承自Set接口，它进一步提供了对其中的元素进行完整的排序的功能。这些元素将会使用Comparable自然排序，或者使用在集合创建时指定的Comparable进行排序，具体取决于使用哪个构造方法。（The set's）该集合的迭代器将会以递增的顺序遍历该集合。同时它提供了额外的操作来进行排序。



# List接口

```java
public interface List<E> extends Collection<E>{
    int size();
    boolean isEmpty();
    boolean contains(Object o);
    Iterator<E> iterator();
    Object[] toArray();
    <T> T[] toArray(T[] a);
    boolean add(E e);
    boolean remove(Object o);
    boolean containsAll(Collection<?> c);
    boolean addAll(Collection<? extends E> c);
    boolean removeAll(Collection<?> c);
    boolean retainAll(Collection<?> c);
    void clear();
    boolean equals(Object o);
    int hashCode();
    //属于List的抽象方法
    E get(int index);
    E set(int index, E element);
    void add(int index, E element);
    E remove(int index);
    int indexOf(Object o);
    int lastIndexOf(Object o);
    ListIterator<E> listIterator();
    ListIterator<E> listIterator(int index);
    List<E> subList(int fromIndex, int toIndex);
    //以及类似Set接口中的static <E> Set<E> of(...)方法
    static <E> List<E> of(...);
}
```





List是一个有序集合接口。使用这个接口的用户可以精确的控制每个元素在列表中的位置。使用者可以通过它们的整数索引（列表中的位置）来访问这些元素，并且可以搜索列表中的元素。

与集合不同，列表通常允许重复的元素。更加正式的说，列表通常允许一对这样的元素`e1，e2，并且e1.equals(e2)`，它们通常允许多个null元素如果它们允许的话。有人可能希望通过在用户尝试插入时抛出运行时异常来实现禁止重复的列表，这并不是不可想象的，但我们预计这种用法很少见。



## ArrayList

```java
public class ArrayList<E> 
	extends AbstractList<E>
	implements List<E>, RandomAccess, Cloneable,java.io.Serializable
```





ArrayList是List接口的可调整大小的实现，底层采用数组。它是实现了所有可选列表的操作，并且允许存放所有的元素，包括null。除了实现List接口外，ArrayList提供了操作数组大小的方法用于内存存储列表。（ArrayList大致与Vector相同，除了ArrayList不是同步的）。

## Vector

```java
public class Vector<E>
    extends AbstractList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```





类似ArrayList，不过它是线程安全的。



### Stack

```java
class Stack<E> extends Vector<E>
```





Stack 继承了Vector，是一个先进后出的队列。



## LinkedList

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
```





LinkedList是List和Deque的双向链表的实现。LinkedList实现了所有可选列表的操作，并且允许存放所有的元素，包括null。



# Map接口

```java
public interface Map<K, V>{
    int size();
    boolean isEmpty();
    boolean containsKey(Object key);
    boolean containsValue(Object value);
    V get(Object key);
    V put(K key, V value);
    V remove(Object key);
    void putAll(Map<? extends K, ? extends V> m);
    void clear();
    Set<K> keySet();
    Collection<V> values();
    Set<Map.Entry<K, V>> entrySet();
    boolean equals(Object o);
    int hashCode();
    //以及类似Set接口中的static <E> Set<E> of(...)方法
    static <K, V> Map<K, V> of(...)
}
```





这是一个将键与值相匹配的对象。一个map不能包含重复的键；每个键最多可以映射一个值。这个接口代替了一个完全是抽象类而不是一个接口`Dictionary`的类



## HashMap

这个见[链接](../Java集合类整理)~



## HashTable

```java
public class Hashtable<K,V>
    extends Dictionary<K,V>
    implements Map<K,V>, Cloneable, java.io.Serializable
```





HashTable类实现了一个将键值相匹配的hash表，这个类不允许键或者值为null。为了成功存储和检索哈希表中的对象，用作键的对象必须实现hashCode()和equals()方法。



## TreeMap

```java
public class TreeMap<K,V>
    extends AbstractMap<K,V>
    implements NavigableMap<K,V>, Cloneable, java.io.Serializable
```





TreeMap是基于NavigableMap实现的**红黑树**。这个Map根据其键的Comparable或关键时提供的Comparable来实现自然排序，这取决于使用哪个构造函数。TreeMap保证了containsKey,get, put,remove的时间复杂度为log(n)。

TreeMap中由4个变量：

```
private final Comparator<? super K> comparator;

private transient Entry<K,V> root;

/**
 * The number of entries in the tree
 */
private transient int size = 0;

/**
 * The number of structural modifications to the tree.
 */
private transient int modCount = 0;
```

`Entry<K,V>`是一个静态内部类：

```java
  	static final class Entry<K,V> implements Map.Entry<K,V> {
        K key;
        V value;
        Entry<K,V> left;
        Entry<K,V> right;
        Entry<K,V> parent;
        boolean color = BLACK;
        
        //...
	}
```







# SortedMap接口

```java
public interface SortedMap<K,V> extends Map<K,V>{
    Comparator<? super K> comparator();
    SortedMap<K,V> subMap(K fromKey, K toKey);
	SortedMap<K,V> headMap(K toKey);
    SortedMap<K,V> tailMap(K fromKey);
    K firstKey();
    K lastKey();
    Set<K> keySet();
    Collection<V> values();
    Set<Map.Entry<K, V>> entrySet();
}
```





SortedMap是一个Map，它在键上进一步提供了完整的排序。这个Map根据其键的Comparable或关键时提供的Comparable来实现自然排序，这取决于使用哪个构造函数。

这个顺序反映了在什么时候迭代有序的map集合视图。同时它额外的提供了几个操作来利用这个排序



# 区别

## ArrayList和LinkedList的区别

1. ArrayList底层采用的是动态数组，LinkedList采用了双向链表。
2. 对于访问来说，前者更快；对于增、删来说后者更快。



## HashTable与HashMap的区别

1. Hashtable是线程安全的，也就是说是同步的，而HashMap是线程不安全的，不是同步的。
2. HashMap允许存在一个为null的key，多个为null的value 。
3. hashtable的key和value都不允许为null。





# 参考链接

https://segmentfault.com/a/1190000008522388#articleHeader14

https://www.jianshu.com/p/50e19038e361



> 大部分正义感都是虚伪的 聊胜于无   —————————————沃德彭尤·胡某











