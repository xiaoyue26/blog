---
title: ThreadLocal的线性探测
date: 2020-01-12 18:01:31
tags: 
- java
categories: 
- java

---

参考:
https://www.liaoxuefeng.com/wiki/1252599548343744/1306581251653666
http://songkun.me/2019/10/26/2019-10-26-java-threadlocal-hash-clash/
https://juejin.im/post/5d43e415e51d4561db5e39ed

# 避免ThreadLocal的内存泄露

参考上述资料,ThreadLocal是存在线程自己的变量,只不过用threadLocalMap把这个线程所有的ThreadLocal变量都集中存储了,key用的是ThreadLocal对象,value用的是对象T,也就是实际的数据。
由于key用的是弱引用,因此使用ThreadLocal时,使用结束时一定要进行`remove()`,否则会导致内存泄露。

这里的内存泄露有3重含义:
## 局部ThreadLocal变量
这个数据语义上已经不用了,而且语法上ThreadLocal变量也已经离开作用域，也回收了,key变成了null;但由于ThreadLocal设计实现上的缺陷,ThreadLocalMap中的key变成null,value却不是null,依然强引用了数据,所以不能自动回收,需要程序员手动处理:
```java
try {
    threadLocalUser.set(user);
    ...
} finally {
    threadLocalUser.remove();
}
```

## static ThreadLocal变量
这个数据语义上已经不用了,但语法上这个ThreadLocal是static，没有离开作用域,
因此理论上还能访问到,设计上的生命周期太过于长了。 这种属于设计缺陷。也需要程序员手动处理,代码同上,也是调用`remove()`方法。

## 搭配线程池的ThreadLocal变量
这个数据本意上已经不用了,  如果不是线程池的线程，如果只是上述两种情况，也只会内存泄露一段时间，等到线程销毁的时候，就会释放相应的threadlocalMap和内存。
但如果搭配线程池使用，这里的ThreadLocalMap和ThreadLocal变量都会继续使用,永不释放内存。
因此程序语义变为不同任务都共享这个threadLocal变量，很可能有语义错误。
保险起见,也是同上,调用`remove()`方法即可。


# remove方法背后: ThreadLocalMap的实现机制
remove方法的语义很简单,就是将ThreadLocalMap的对应key/value都删除。
实际底层实现却并不简单,涉及到性能考虑=>线性探测法=>删除时rehash的连锁反应,导致实现较为特殊。

## 斐波那契哈希
ThreadLocalMap希望尽可能提高性能,因此使用斐波那契哈希,使用魔法数0x61c88647,为每一个ThreadLocal变量涉及了尽可能均匀、理论无碰撞的完美哈希，同时辅助以rehash扩容，来保证碰撞和线性探测死循环尽可能不发生。

## 线性探测与删除
ThreadLocalMap处理哈希冲突时使用的是线性探测法,因此删除key的时候不能直接简单把entry置为null; 它采用的方法是把后续每个不为null的entry进行rehash, 放在合适的位置,保证不会因为删除导致线性探测失效中断。这里会把整个哈希表的数组看作循环队列,一直rehash到null的entry,因为当前的entry一定是null,而且负载因子不会太高,而且是单线程,因此一定不会死循环,有终止条件。

```java
private int expungeStaleEntry(int staleSlot) {
  Entry[] tab = table;
  int len = tab.length;

  // 删除对应位置的entry
  tab[staleSlot].value = null;
  tab[staleSlot] = null;
  size--;

  Entry e;
  int i;

  // rehash过程,直到entry为null
  for (i = nextIndex(staleSlot, len);(e = tab[i]) != null; i = nextIndex(i, len)) {
    ThreadLocal<?> k = e.get();
    // k为空,证明已经被垃圾回收了
    if (k == null) {
        e.value = null;
        tab[i] = null;
        size--;
    } else {
        int h = k.threadLocalHashCode & (len - 1);
        // 判断当前元素是否处于"真正"应该待的位置
        if (h != i) {
            tab[i] = null;
            // 线性探测
            while (tab[h] != null)
                h = nextIndex(h, len);
            tab[h] = e;
        }
    }
  }
  return i;
}

```
综上, 理论上每次删除都会进行局部链的rehash,但是由于斐波那契哈希设计上是绝对均匀，因此这个"局部链"的长度理论上是非常短甚至是0.
此外, 除了这种rehash的方法,还可以对entry进行delete标记来确保线性探测不会中断。





