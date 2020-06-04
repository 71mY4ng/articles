---
layout: article
title:  SnowFlake 分布式ID算法理解
tags: binary, distribute id generator, SnowFlake
---

* (理解分布式id生成算法SnowFlake)[https://segmentfault.com/a/1190000011282426?utm_source=tag-newest#articleHeader3]
* (snowflake 时间回拨异常)[https://blog.csdn.net/X5fnncxzq4/article/details/79549514]

规定位数划分ID中的信息区域，一般用：
* 执行时间戳
* 工作机器ID，可以将数据中心或者运行机器用序列或者其它方法进行标识
* 序列号
组合而成

执行时间戳是一个毫秒数相对于配置好的初始时间戳的偏移量，既
timstamp - epoch_timestamp
epoch_timestamp是个常量，代表开端时间

各字段的位数是可以按需求控制的，例如：

执行时间戳毫秒数用 41 位表示，那么它可以表示 2 ^ (41) 个二进制数，
marchineId 可以由 datacenterId和 workerId组成，前面指代哪一台机器，后者指代哪一个线程


<!--more-->

## 文章中一些实用二进制处理思路

### n 位能表示的最大数值：

```java
long nBit = 5L;
long nMaxBin = -1L ^ (-1L << nBit);
```

* -1 在二进制中表示1的补码，既一长串 1；
* 将 -1 向右位移 N 位，产生了N个0位
* -1 用异或与 上一条结果运算，将不同的转为1位，既将之前的串转成 N个1的串

最后得到了 $2 ^ {N} - 1$ // 2 的N次方
例如 N=5 时， 2^5 - 1 = 31，既 2^4 + 2^3 + 2^2 + 2^1 + 2^0，用二进制表示就是11111


### 二进制序列的自增循环：

```java
long seqMask = -1L ^ (-1L << 12L); // 12位二进制表示的最大数
seqId = (seqId + 1) & seqMask;
```


位与运算符`^`，在10进制计算中经常用于求余数。
seqId 自增后对mask取余，如果小于seqMask则得到自己本身，大于seqMask则是一个新的循环。这样则实现了seqId的取值范围在 0 ~ seqMask;


### 位运算来拼接结果

如果一串二进制由，19位执行时间戳timestamp，10位处理线程标识workerId，12自增序列sequence，首位拼接而成。

那么 workerId的开始位需要偏移的长度就是 sequence的长度，既 workerIdShift = sequnceBit

同理 timestamp 的开始位需要偏移的长度是 sequence + workerId的长度，既 timestampShift = workerIdBits + sequenceBits

将各字段结果进行位移

```java
long timstamp = (timstamp << timestampShift); // 向右位移10 + 12 位
long workerId = (workerId << workerIdShift); // 向右位移12 位
```

后进行拼接，因为位移腾出来的位数都为0，用位与运算`|`，如果原位是1，那么就是1，原位位0，则是0。

那么由此实现了几个字段的首位相接

```java
long id = timestamp | workerId | sequence;
```