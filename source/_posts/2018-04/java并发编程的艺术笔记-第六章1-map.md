---
title: java并发编程的艺术笔记-第六章1-map
date: 2018-04-07 21:49:47
tags: 
- java
- 并发
categories:
- java
- 并发
---


# 第六章 java并发容器和框架
主要内容是concurrent包中并发性能较好的几个容器，逻辑上是：
1. map： `concurrentHashMap`;
2. 队列: `concurrentLinkedQueue`;
3. 阻塞队列.
此外还有`Fork/Join`工作窃取框架（`java8 stream api`的底层）。

## 6.1 ConcurrentHashMap

### 6.1.1 背景
`HashTable`缺点：并发性能低，直接锁整个table。
`HashMap`缺点：非线程安全，并发时产生死循环。
多线程put操作可能导致环形链表。

- `ConcurrentHashMap`：
分段加锁，并发效率高。

### 6.1.2 具体实现
`concurrentHashMap`在不同jdk版本中实现不同：

### jdk1.6中的实现
- 基本思路
将哈希表分成几个段，然后分段加锁。
把原来的一维数组，变成二维数组。（类似于二级索引，先寻址到第一级，获得锁以后再访问第二级具体数据）

{% img /images/concurrentClass.jpg 400 600 concurrentClass %}
如类图所示：
`ConcurrentHashMap`->`Segment`->`HashEntry`
{% img /images/concurrentStruct.jpg 400 600 concurrentStruct %}

第一维是`Segment`数组，每个`Segment`里存一个`HashEntry`数组，最后拉链表。
`HashEntry`源码：

```java
static final class HashEntry<K,V> {
        final int hash;
        final K key;
        volatile V value;
        volatile HashEntry<K,V> next;
```

 
**特点**
1. 实际并发度为2的n次方。（离散取值，而不是连续）
=>原因：
`Segment`数组的长度需要为2的n次方
=>原因：
寻址的时候使用移位来提高性能，需要数组长度为2的n次方：
hash值的高位确定落在哪个`Segment`里，低位确定落在`Segment`中的位置。

```java
hash >>> segmentShift) & segmentMask//定位Segment所使用的hash算法
int index = hash & (tab.length - 1);// 定位HashEntry所使用的hash算法
// 当tab.length为2的n次方时，index的计算等价于= hash % tab.length
```
并发度(`Concurrency Level`)实际就是由多少锁，也就是`Segment`数组的长度。
如果设定并发度为14，实际上会创建长度为16的`Segment`数组，也就是实际并发度为16.

2. **弱一致性**
写入的数据并不一定能马上读到，或者不一定能读到最新数据。
// `get`,`containsKey`,`clear`方法和迭代器都是弱一致性的
原因：
并发的时候使用了`volatile`代替锁提高性能，牺牲了一致性换取性能。
```java
static final class HashEntry<K,V> {
        final int hash;
        final K key;
        volatile V value;
        volatile HashEntry<K,V> next;
```
由于遍历期间其他线程可能对链表结构做了调整，因此`get和containsKey`返回的可能是过时的数据。如在get执行`UNSAFE.getObjectVolatile(segments, u)`之后，其他线程若执行了`clear`操作，则`get`将读到失效的数据。

由于`clear`没有使用全局的锁，当清除完一个`segment`之后，开始清理下一个`segment`的时候，已经清理`segments`可能又被加入了数据，因此`clear`返回的时候，`ConcurrentHashMap`中是可能存在数据的。因此，`clear`方法是弱一致的。

如果加锁的话，就能保证强一致性了。


### jdk1.8中的实现
- 基本思路
//把链表改成红黑树，提高效率。
第一维：数组
第二维：链表(<8)/红黑树
用CAS操作代替大范围加锁。
锁的粒度缩小到`Node`级别。（linux中很多并发相关的数据结构都是红黑树，在如epoll，防火墙中应用）

- 具体实现
1. 最小粒度是`Node`,成员变量如下:
```java
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        volatile V val;
        volatile Node<K,V> next;
```
可以看出与原来的`HashEntry`基本一致，多了一个继承。

其他的容器成员：
```java
    /**
     * The array of bins. Lazily initialized upon first insertion.
     * Size is always a power of two. Accessed directly by iterators.
     */
    transient volatile Node<K,V>[] table;

    /**
     * 平时为null。扩容时,为下一个表。
     */
    private transient volatile Node<K,V>[] nextTable;

    /**
     * Base counter value, used mainly when there is no contention,
     * but also as a fallback during table initialization
     * races. Updated via CAS.
     */
    private transient volatile long baseCount;
    private transient volatile int sizeCtl;

    /**
     * The next table index (plus one) to split while resizing.
     */
    private transient volatile int transferIndex;

    /**
     * Spinlock (locked via CAS) used when resizing and/or creating CounterCells.
     */
    private transient volatile int cellsBusy;

    /**
     * Table of counter cells. When non-null, size is a power of 2.
     */
    private transient volatile CounterCell[] counterCells;

    // views
    private transient KeySetView<K,V> keySet;
    private transient ValuesView<K,V> values;
    private transient EntrySetView<K,V> entrySet;

```
修饰符复习：
`transient`: 序列化的时候忽略。（一般会自定义相关的序列化操作）一般用于一些运行时的状态变量。


**sizeCtl**
`表初始化和扩容时的控制状态量。`

- 负数时: 
    - -1: 正在初始化;   
    - -2: 正在扩容,有1个活跃的扩容线程数;
    - -3: 正在扩容,有2个活跃的扩容线程数;
    - -N: 正在扩容,有N-1个活跃的扩容线程数;
以此类推...

- 0: 还没初始化,未指定容量。   
- 正数时: 
    - table=null: 还没初始化,目标初始化容量。
    - table!=null: sizeCtrl=0.75*n(扩容阈值,抵达这个值就要进行扩容了)


2. `Node`节点会从链表节点转化为红黑树节点`TreeNode`:
```java
/**
* Nodes for use in TreeBins
*/
static final class TreeNode<K,V> extends Node<K,V> {
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;
```

3. `TreeBin`作为红黑树的根节点，封装相关的操作，并且有读写锁:
```java
static final class TreeBin<K,V> extends Node<K,V> {
    TreeNode<K,V> root;
    volatile TreeNode<K,V> first;
    volatile Thread waiter;
    volatile int lockState;
    // values for lockState
    static final int WRITER = 1; // set while holding write lock
    static final int WAITER = 2; // set when waiting for write lock
    static final int READER = 4; // increment value for setting read lock
    
    /**
     * Acquires write lock for tree restructuring.
    进行平衡梳理的时候,也会lockRoot
     */
    private final void lockRoot() {
        if (!U.compareAndSwapInt(this, LOCKSTATE, 0, WRITER))
            contendedLock(); // offload to separate method
    }

    /**
     * Releases write lock for tree restructuring.
     */
    private final void unlockRoot() {
        lockState = 0;
    }
    
    /**
     * Finds or adds a node.
     * @return null if added
     */
    final TreeNode<K,V> putTreeVal(int h, K k, V v) {

```

4. `tabAt`方法,获取i位置上的`Node`:
```
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}
```

5. `casTabAt`方法,CAS设定i位置上的`Node`:
```java
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
Node<K,V> c, Node<K,V> v) {
return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}
```

6. `setTabAt`方法,直接设定i位置上的`Node`(已经占据锁的时候使用),
利用`volatile`:
```
static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
}
```

简单总结:
`get`方法: 利用`volatile`,不加锁,并发访问;(弱一致性)
`put`方法: 利用CAS方法,轮询,大部分时候不加锁.如果是红黑树，调用它的`putTreeVal`方法。
