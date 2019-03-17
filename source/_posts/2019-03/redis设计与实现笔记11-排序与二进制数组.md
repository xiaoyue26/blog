---
title: redis设计与实现笔记11-排序与二进制数组
date: 2019-03-17 20:35:13
tags:
- redis
categories:
- redis

---


排序命令`SORT`:
```shell
RPUSH numbers 5 3 1 4 2
SORT numbers
# 结果:
1
2
3
4
5
```
也可以在末尾加上`alpha`参数，指定按字母序排序。
还可以在末尾加上`By`参数，指定按某个集合定义的权重进行排序。


## sort <key>命令的实现
{% img /images/2019-03/sort_key.png 800 1200 sort_key %}
实现sort的时候为被排序的key创建一个与排序目标长度相同的数组，数组中的节点的obj字段存放实际数据指针，score字段是double，存放每个对象的排序值。
实际排序对这个新创建的数组进行即可。

## sort <key> By *-price命令的实现
```shell
MSEST apple-price 8 banana-price 5.5 cherry-price 7
SORT fruits BY *-price
```

不同选项顺序：(执行顺序和编程时书写顺序无关)
1. 先`by *-price`；
2. 后`alpha`

## limit命令的实现
`sort fruits limit <offset> <count>`
实现上是先排序后，然后在跳转到`offset`上选`count`个。

## STORE命令
可以用`STORE`命令保存`SORT`的结果:
```shell
Sort fruits STORE sorted_fruits
```
实现上，首先排序，然后：
1. 检查要保存的键`sorted_fruits`是否存在,存在则删除;
2. 创建`sorted_fruits`列表；
3. 将辅助排序数组依次压入`sorted_fruits`.


# 第22章-二进制数组
单个数组操作:
```shell
SETBIT bit 0 1 # 第0位设置为1
GETBIT bit 3   # 获取第3位的值
BITCOUNT bit   # 获取bit数组中1的个数
```
多个数组之间操作:
```shell
BITOP AND and-result x y z # x,y,z求与的结果放and-result
类似的操作还有OR,XOR,NOT,也就是与、或、非、异或、取反。
```

## 底层存储
用SDS字符串存储二进制数组。
对二进制数组的操作： 也借用SDS的字符串函数。

因为SDS是二进制安全的(也就是存储的数据中有\0也没关系，因为SDS中有存长度)，所以可以用来存储二进制数组。

### 逆序存储
{% img /images/2019-03/sds_binary.png 800 1200 sds_binary %}
这里保存的二进制数组是`0100 1101`;
图中SDS保存的是逆序的:`1011 0010`，之所以逆序存储的原因为了简化`SETBIT`命令的实现。
#### offset、高位、低位
这里要强调一下二进制的offset、高位、低位的概念，方便理解为什么要逆序存储以及因为逆序存储所以扩展时不需要移动。
二进制数组是`0100 1101`：
高位、低位: `0100`是高位,`1101`是低位,(想象十进制数字9876的高位是9).
offset: offset为0的是1，offset为1的为0.

`GETBIT 7`是0
`SETBIT 11 1`其中第11位不存在，因此扩展后会变成`1000 0100 1101`。

## GETBIT实现O(1)
由于前面是逆序存储的二进制数组，因此可以从左边开始数offset了（原来是从右边往左数，负负得正）。
由于用SDS存储二进制，当我们想要获取第n位0，1值的时候，相当于需要获取两个坐标：
1. 位于哪个byte:    `n/8`
2. 位于byte的第几位:`n%8+1` 。
换句话说所有值的坐标计算公式为: `(n/8，n%8+1)`。
{% img /images/2019-03/locate_binary.png 800 1200 locate_binary %}
而`GETBIT <bitarray> 10`的结果是:
`(10/8，10%8+1)`也就是byte[1][3-1]。
（从左到右第3位）。
其实如果统一从0开始计数，公式可以简化为：
`(n/8,n%8)`

{% img /images/2019-03/locate_binary10.png 800 1200 locate_binary10 %}
GETBIT命令算法复杂度为O(1)。

## SETBIT命令实现O(1)
命令格式：
```shell
SETBIT <bitarray> <offset> <value>
```
这个命令会做3件事：
1. 设置新值
2. 返回旧值；
3. 如果offset超出原有数组长度，会拓展原数组，并且把新扩展空间的值设置为0.
这里的扩展应当注意到SDS的空间预分配策略：
```
总长度len<1MB: 总空间为2*len+1;
总长度len>=1MB: 总空间为len+1MB+1。
换句话说，预分配的空间上限是1MB，尽量为len。
```

### 逆序存储与扩展不需要移动
由于扩展是扩展高位，而高位经过逆序存储后，放在了buf数组的末尾，因此扩展时就不需要移动原来的数据了。


## BITCOUNT命令实现
有几种实现算法：
(1) 遍历：最慢；
(2) 查表：空间换时间，二进制数组的排列是有穷的，可以预先存下不同排列对应的1数量；
(3) SWAR算法: 计算汉明码问题，Hamming Weight。 
如果cpu不支持直接进行汉明码计算，可以使用SWAR算法:
```c
uint32_t swar(uint32_t i){
    i = (i& 0x55555555)+((i>>1)&0x55555555);
    i = (i& 0x33333333)+((i>>2)&0x33333333);
    i = (i& 0x0F0F0F0F)+((i>>4)&0x0F0F0F0F);
    i = (i*(0x01010101)>>24);
    return i;
}
```
比遍历快32倍，比查8位的表快4倍，比查16位的表快2倍，无需额外内存。
由于SWAR算法缓存的值较少而且规整，是缓存友好的。
(4)redis的二进制位统计算法：
结合查8位的表和SWAR算法。
如果n< 128: 直接用查表；
如果n>=128: 使用SWAR，每次载入128位，调用4次SWAR。

## BITOP OR\XOR\NOT命令实现
对于每个byte调用c函数操作。

