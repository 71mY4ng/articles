---
layout: default
colorspace: purple
title: ns-3 tracing 笔记
tags: jekyll, github
---

## 分析Ascii Traces

AsciiTrace指的是使用tracing系统生成ASCII文件的输出结果，例如`out.tr`这种文件。一般输出ASCII traces的脚本代码在文件中有如下操作：

```cpp
AsciiTraceHelper ascii;
pointToPoint.EnableAsciiAll(ascii.CreateFileStream("mySecond.tr");
```
打开Ascii traces文件你可看到十分密集的信息，下面是如何从这些信息中识别模拟过程的参考
首先ascii traces中的每一行对应了一个trace事件，如下代码块显示

```bash
r
2.01761
/NodeList/0/DeviceList/0/$ns3::PointToPointNetDevice/MacRx 
ns3::PppHeader 
(Point-to-Point Protocol: IP (0x0021))
ns3::Ipv4Header
(tos 0x0 DSCP Default ECN Not-ECT ttl 63 id 0 protocol 17 offset (bytes) 0 flags [none] 
length: 1052 10.1.2.4 > 10.1.1.1)
ns3::UdpHeader (length: 1032 9 > 49153)
Payload (size=1024)
```

1. `r` 开头第一个单独字符具有如下含义：

| 符号 | English          | 中文 |
|:-----|:-------------------------------------------------|:------|
|`+`   |An enqueue operation occurred on the device queue | 设备队列中的入队操作|
|`-`   |A dequeue operation occurred on the device queue  | 设备队列中的出队操作|
|`d`   |A packet was dropped, typically because the queue was full | 数据包被丢弃，通常因为队列已满|
|`r`   |A packet was received by the net device | 网络设备接收到数据包|

2. `2.01761` 指的是单位为秒的仿真时间
3. `/NodeList/0/DeviceList/0/$ns3::PointToPointNetDevice/MacRx` 这个是发起tracing事件的发送端，以tracing命名空间表示。这段trace的含义是：由根命名空间是NodeList的第0个节点，也就是节点0（`NodeList/0`）；此节点下的第0个设备（`DeviceList/0`），是`$ns3::PointToPointNetDevice`类型的；并且它是数据包的接收trace端（`MacRx`），这也与首字符`r`（接收到数据包的事件标识符）所对应。
相似的还有`TxQueue/Enqueue`对应了入队操作，也就是首字符是`+`的trace事件。
4. `ns3::PppHeader (Point-to-Point Protocol: IP (0x0021))` 直观显示了数据包被封装成了点对点协议。
5. `ns3::Ipv4Header (tos 0x0 ... length: 1052 10.1.2.4 > 10.1.1.1)` 直观显示了数据包IP版本（Ipv4），接收路径（从10.1.2.4到10.1.1.1），以及包大小等信息。
6. `ns3::UdpHeader (length: 1032 9 > 49153)` 显示数据包的Udp头
7. `Payload (size=1024)` 数据包数据量为，1024bytes，Payload(有效载荷，装载货物)

> *有效载荷指的是数据包中的全体有效数据大小，有效数据，定义是“需要被传输的数据”，比如说一个域的值。*

另外展示入队事件的trace例子：

```bash
+
2
/NodeList/0/DeviceList/0/$ns3::PointToPointNetDevice/TxQueue/Enqueue
ns3::PppHeader (
  Point-to-Point Protocol: IP (0x0021))
  ns3::Ipv4Header (
    tos 0x0 ttl 64 id 0 protocol 17 offset 0 flags [none]
    length: 1052 10.1.1.1 > 10.1.1.2)
    ns3::UdpHeader (
      length: 1032 49153 > 9)
      Payload (size=1024)
/*来源： https://www.nsnam.org/docs/release/3.25/tutorial/singlehtml/index.html
*/
```

[首页](../){: .button}