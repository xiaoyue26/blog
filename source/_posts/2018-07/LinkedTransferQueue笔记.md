---
title: LinkedTransferQueue笔记
date: 2018-07-14 20:48:42
tags: 
- java
- 并发
categories:
- java
- 并发

---

本文主要讨论两个容器: `SynchronousQueue`和`LinkedTransferQueue`。重点是`LinkedTransferQueue`.
背景:
> https://stackoverflow.com/questions/5102570/implementation-of-blockingqueue-what-are-the-differences-between-synchronousque
有了LinkedBlockingQueue,为啥还要使用SynchronousQueue呢? 
队列元素较少情况下: SynchronousQueue优于LinkedBlockingQueue
队列元素较多情况下: LinkedBlockingQueue优于SynchronousQueue。


`LinkedTransferQueue`的意义在于兼顾了可读性,易用性和高性能。
> 高性能,无锁: `SynchronousQueue`,`ConcurrentLinkedQueue`
可读性,易用性,有锁: `LinkedBlockingQueue`
兼顾上述优点: `LinkedTransferQueue`

`BlockingQueue`的实现:
大多是锁整个队列，并发量大的时候，锁比较耗费资源和时间。

`SynchronousQueue`:
无锁，只是传递元素，性能远高于`LinkedBlockingQueue`。

`ConcurrentLinkedQueue`:
无锁,性能高用CAS实现非阻塞的并发,但没有实现`BlockingQueue`接口,因此不能当阻塞队列使用(不能直接用于生产者消费者场景下).


具体来说:
`ConcurrentLinkedQueue`是`Queue`接口的一个实现.
`SynchronousQueue`是`BlockingQueue`接口的一个实现:
`LinkedTransferQueue`是`TransferQueue`接口的一个实现:
```java
LinkedTransferQueue->TransferQueue->BlockingQueue
```
具体定义(jdk1.8):
```java
public class LinkedTransferQueue<E> extends AbstractQueue<E>
    implements TransferQueue<E>, java.io.Serializable {
```

## SynchronousQueue
它本身不存在容量,只能进行线程之间的元素传送。由于对于传递性场景进行了某种充分的优化，其中最重要的是不需要锁，因此在只需要同步，不需要大量存储元素的场景下吞吐量很高。
> 优势: 无锁,吞吐量大；
缺点: 无容量,只能用于传递性场景。

*todo: 具体是什么样神奇的优化。*

## ConcurrentLinkedQueue
详见之前的博客:
[java并发编程的艺术-第六章2LinkedQueue][1]

## LinkedTransferQueue
本质上是多了`TransferQueue`接口的实现:
```java
void transfer(E e) throws InterruptedException;// 阻塞
boolean tryTransfer(E e);
boolean tryTransfer(E e, long timeout, TimeUnit unit) throws InterruptedException;// 一段时间阻塞
boolean hasWaitingConsumer();
int getWaitingConsumerCount();

/**一般来说，不抛`InterruptedException`异常的是不阻塞的方法;
抛`InterruptedException`异常的是阻塞,或者定时阻塞的方法.
*/
```
综合了`ConcurrentLinkedQueue`和`SynchronousQueue`的高吞吐量优点和`LinkedTransferQueue`的能存储大量元素的优点。

1. `transfer(E e)`：若当前存在一个正在等待获取的消费者线程，即立刻移交之；否则，会插入当前元素e到队列尾部，并且等待进入阻塞状态，到有消费者线程取走该元素。
2. `tryTransfer(E e)`：若当前存在一个正在等待获取的消费者线程（使用take()或者poll()函数），使用该方法会即刻转移/传输对象元素e；若不存在，则返回false，并且不进入队列。这是一个不阻塞的操作。
3. `tryTransfer(E e, long timeout, TimeUnit unit)`：若当前存在一个正在等待获取的消费者线程，会立即传输给它;否则将插入元素e到队列尾部，并且等待被消费者线程获取消费掉；若在指定的时间内元素e无法被消费者线程获取，则返回false，同时该元素被移除。
4. `hasWaitingConsumer()`：判断是否存在消费者线程。
5. `getWaitingConsumerCount()`：获取所有等待获取元素的消费线程数量。


需要看看xfer的4个参数:
```java
private static final int NOW   = 0; // 即使没有可以用的空间或者可用的数据,也立即返回。
// for 不计时的poll和tryTransfer

private static final int ASYNC = 1; // 异步/非阻塞/立即返回. 
// for offer, put, add

private static final int SYNC  = 2; // 同步/阻塞/无限制等待.
// for transfer, take

private static final int TIMED = 3; // 可以等待一定时间.
// for 计时的poll和 tryTransfer
```
这里要注意`take`和`poll`的区别,`take`阻塞到有元素,`poll`没元素的话,直接返回`null`即可,不会死等。

参考资料: 
http://www.cnblogs.com/rockman12352/p/3790245.html
http://cmsblogs.com/?p=2433


  [1]: http://xiaoyue26.github.io/2018/05/19/2018-05/java%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%E7%9A%84%E8%89%BA%E6%9C%AF-%E7%AC%AC%E5%85%AD%E7%AB%A02LinkedQueue/

