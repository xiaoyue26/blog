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
```
private transient volatile Node<E>tail = head;
```

## simple解法(假想方案)
一种simple的实现方案是这样的: 
- tail定义:
```
将tail定义成永远是队列最后一个节点:
```

```
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
1次volatile读;(tail)
2次CAS操作;
2次volatie写.(tail.next和tail)
tail.next实际上也是volatile变量。

## 实际解法
这个思路最重要的一点就是tail不一定是最后一个节点.
由于volatile写的开销远远高于volatile读,jdk中的解法是:
```
1. 假设队列中最后一个节点为:realTail;
2. tail 不一定等于 realTail;
3. 仅当tail与realTail距离大于等于HOPS时,才更新tail=realTail.
```
这样设计以后,可以通过增加volatile读减少volatile写。

具体实现:
```
public boolean offer(E e) {
    checkNotNull(e);
    final Node<E> newNode = new Node<E>(e);

    for (Node<E> t = tail, p = t;;) {
        Node<E> q = p.next;
        if (q == null) {
            // p is last node
            if (p.casNext(null, newNode)) {
                // Successful CAS is the linearization point
                // for e to become an element of this queue,
                // and for newNode to become "live".
                if (p != t) // hop two nodes at a time
                    casTail(t, newNode);  // Failure is OK.
                return true;
            }
            // Lost CAS race to another thread; re-read next
        }
        else if (p == q)
            // We have fallen off list.  If tail is unchanged, it
            // will also be off-list, in which case we need to
            // jump to head, from which all live nodes are always
            // reachable.  Else the new tail is a better bet.
            p = (t != (t = tail)) ? t : head;
        else
            // Check for tail updates after two hops.
            p = (p != t && t != (t = tail)) ? t : q;
    }
}
```

