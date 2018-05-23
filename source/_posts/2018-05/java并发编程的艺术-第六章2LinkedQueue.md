---
title: java并发编程的艺术-第六章2LinkedQueue
date: 2018-05-19 20:32:03
tags: 
- java
- 并发
categories:
- java
- 并发

---

# ConcurrentLinkedQueue

目标: 线程安全的队列，并发性能好。
实现: 基于链表的队列,`ConcurrentLinkedQueue`,使用`volatile`,`CAS`和`HOPS`变量（LockFree）,性能较好,而且可以通过`HOPS`变量调节性能.

{% img /images/2018-05/1.jpg 400 600 类图 %}


初始时:
```java
private transient volatile Node<E>tail = head;
private transient volatile Node<E> tail;
public ConcurrentLinkedQueue() {
        head = tail = new Node<E>(null);
        // 数据为null. 相当于一个dump头节点.
}
```

## simple解法(假想方案)
一种simple的实现方案是这样的: 
- tail定义:
```
将tail定义成永远是队列最后一个节点:
```

```java
public <E> boolean offer(E e) {
    if (e == null) {
        throw new NullPointerException();
    }
    Node<E> n = new Node(e);
    for (; ; ) {
        Node<E> t = tail;
        if (t.casNext(null, n) && casTail(t, n)) {
        // 1. casNext: 把t的next从null换成n;
        // 2. casTail: 把tail从t换成n。
            return true;
        }
    }
}
```
回想我们的目标: 
1. 安全: 由于tail是`volatile`的,因此可以通过CAS操作保证并发写的安全;
2. 性能: 每次入列至少需要:
```
1次 volatile读;(tail)
2次 CAS操作;
2次 volatie写.(tail.next和tail)
// tail.next实际上也是volatile变量。
```

## jdk注释
直接阅读源码有可能误解了作者的意思,而且源码确实比较trick,看不太懂,注释里声明的几个基本点:
```
1. 仅有一个节点的next为null,它是队列的最后一个节点. 想访问最后一个节点的话,可以通过tail节点往后找,可以通过O(1)时间找到(最多4次); 
// 注意tail不一定指向最后一个节点

2. 队列没有null的实际数据(item = null),也就是容器不接受空元素。
// dumpHead头节点的item为null; 移除节点的时候把item置为null,所以null是一个很重要的标志量,所以不允许实际数据有null造成冲突;

3. 为了满足上一条,更为了gc,过期的节点的next指向自己;
//  所以很多节点遇到p==p.next的时候,会跳转到head重新进入有效节点范围,或者进行过期逻辑处理;

4. 把item置为null相当于移除了该数据,迭代器iterator会跳过item为null的节点.
```




## 实际解法
这个思路最重要的有两点:
一点就是updateHead函数,离群节点的next置为自己;
另一点就是`tail`不一定是最后一个节点.
由于`volatile`写的开销远远高于`volatile`读,jdk中的解法是:
```
1. 假设队列中最后一个节点为:realTail;
2. tail 不一定等于 realTail;
3. 仅当tail与realTail距离大于等于HOPS时,才更新tail=realTail.
```
这样设计以后,可以通过增加volatile读减少volatile写。

具体实现:

## updateHead方法
```java
// 如果当前的head是h,把它换成p
final void updateHead(Node<E> h, Node<E> p) {
        if (h != p && casHead(h, p))
        // 这个是最重要的一个地方
        // 将离群h节点的next置为自己
            h.lazySetNext(h); // 不急,但是迟早进行离群标记
    }
```

## offer方法
```java
public boolean offer(E e) {
    checkNotNull(e);
    final Node<E> newNode = new Node<E>(e);

    for (Node<E> t = tail, p = t;;) {
        Node<E> q = p.next;
        if (q == null) {// 如果p是尾节点
            if (p.casNext(null, newNode)) {// 把item加入队列,让newNode变成活节点
                if (p != t) // hop two nodes at a time
                    casTail(t, newNode);  
                    // 更新tail失败也没关系,因为tail仅仅用于优化性能,并不影响正确性
                return true;
            }
        }
        else if (p == q) // p==q意味着p已经是一个离群节点
            p = (t != (t = tail)) ? t : head;
            // 首先我们一定过时了:
            // 如果t过时了,尝试从tail重连;
            // 如果t没过时,说明tail也过时了,从head重连。
        else // Check for tail updates after two hops.
            p = (p != t && t != (t = tail)) ? t : q;
            // 写竞争失败了而已,把p往后挪就好了. 
    }
}
```

其中有一种trick：
```java
t != (t = tail)
// 可以展开成:
if(t!=tail){
t= tail;
// return true;
}
else{
// return false;
}
```
因此`p = (t != (t = tail)) ? t : head;`的实际含义就是:
```java
if(t!=tail){
    t=tail;
    p=t;
}
else{
    p=head;
}
// 这段代码实际意思就是,已知p节点过时(离群,已经被删除)了,我们需要重新连接到queue中可用的节点,然后重新定位到尾节点.
1. 如果t!=tail,说明t过时了,tail可能还可用,因此尝试从tail重连;(t=tail,p=t);

2. 如果t==tail,说明t和tail都过时了,尝试从head重连。(p=head)


```
使用这个trick的目的不得而知,个人猜测是因为效率更高,代码更精炼。毕竟非阻塞并发存在的意义就是性能,而不是可读性,如果想要可读性,就去用阻塞队列了。
可读性下降了.但如果用得熟练的话,肉眼parse功力上涨,也可能对于大牛来说比if else阅读起来更快。


### 节点定义
```java
private static class Node<E> {
        volatile E item;
        volatile Node<E> next;

        Node(E item) {
            UNSAFE.putObject(this, itemOffset, item);
        }

        boolean casItem(E cmp, E val) {
            return UNSAFE.compareAndSwapObject(this, itemOffset, cmp, val);
        }

        void lazySetNext(Node<E> val) {
            UNSAFE.putOrderedObject(this, nextOffset, val);
        }

        boolean casNext(Node<E> cmp, Node<E> val) {
            return UNSAFE.compareAndSwapObject(this, nextOffset, cmp, val);
        }

        // Unsafe mechanics
        private static final sun.misc.Unsafe UNSAFE;
        private static final long itemOffset;
        private static final long nextOffset;

        static {
            try {
                UNSAFE = sun.misc.Unsafe.getUnsafe();
                Class<?> k = Node.class;
                itemOffset = UNSAFE.objectFieldOffset
                    (k.getDeclaredField("item"));
                nextOffset = UNSAFE.objectFieldOffset
                    (k.getDeclaredField("next"));
            } catch (Exception e) {
                throw new Error(e);
            }
        }
    }
```


## poll方法
```java
public E poll() {
    restartFromHead:
    for (;;) {
        for (Node<E> h = head, p = h, q;;) {
            E item = p.item;
            if (item != null && p.casItem(item, null)) {
                // 成功找到第一个数据节点(item!=null)并移除item
                if (p != h) 
                // p!=h说明p往后找过,说明队首有至少2个非数据节点,更新head
                    updateHead(h, ((q = p.next) != null) ? q : p);
                return item;
            }
            else if ((q = p.next) == null) {// 往后找数据节点
                updateHead(h, p); // 已经是空队列
                return null;
            }
            else if (p == q) // 离群了,重连
                continue restartFromHead;
            else
                p = q;
        }
    }
}
```

## succ方法
```java
/*找next.如果next与自身相等,说明离群了,返回head以便重连*/
final Node<E> succ(Node<E> p) {
    Node<E> next = p.next;
    return (p == next) ? head : next;
}
```



## remove方法
```java
public boolean remove(Object o) {
        if (o != null) {
            Node<E> next, pred = null;
            for (Node<E> p = first(); p != null; pred = p, p = next) {
                boolean removed = false;
                E item = p.item;
                if (item != null) {
                    if (!o.equals(item)) {
                        next = succ(p);
                        continue;
                    }
                    removed = p.casItem(item, null);
                }

                next = succ(p);
                if (pred != null && next != null) // unlink
                    pred.casNext(p, next);
                if (removed)
                    return true;
            }
        }
        return false;
    }
```

