---
title: 浅谈tcp协议第一篇
date: 2017-07-28 17:39:07
categories: tcp
tags: tcp
---

## 源起

> 关于`TCP`建立连接的具体步骤自己并不太了解,一直处于一知半解的状态,写这篇文章主要因为以下几个方面。
> 1. 最近项目中使用`netty`作为底层的通讯组件,因此有必要了解TCP连接的建立和数据传输。
> 2. 对TCP连接的建立和数据传输一直停留在初级阶段，想通过这篇文章让自己更加深入的了解`TCP协议`。
> 3. 在`微服务`大行其道的今天,一个完整的系统被从`业务角度`或者`技术角度`拆分成多个微服务,而微服务之间的数据调用都使用了`TCP`建立连接。
> 4. 对于未来随着业务量的不断增加,开源框架对系统的限制还是存在的,而修改源码对框架的稳定性构成了一定的威胁,所以可以考虑编写自己的`RPC`框架。

## 什么是TCP?

TCP位于`OSI/ISO 7层网络模型`第`4`层,位于`TCP/IP 4层网络模型`的第`3`层。

    值得一提的是`OSI/ISO 7层网络模型`只是停留在设计者的图纸上,在实际网络环境中使用的是`TCP/IP 4层网络模型`.

下图给出了 `OSI/ISO 7层网络模型`和`TCP/IP 4层网络模型`的图示。

**1. 网络模型**

`OSI/ISO 7层网络模型`

![image](http://wx2.sinaimg.cn/large/74b07056ly1fgkodfqeibj20o70h577g.jpg)

`TCP/IP 4层网络模型`

![image](http://wx4.sinaimg.cn/large/74b07056ly1fgkodfbba1j20hs088ab0.jpg)

从上面的两张模型图中我们能知道`tcp/ip`模型将`数据链路层`和`物理层`统一为`网络接入层`，将`应用层`，`表示层`，`会话层`统一为`应用层`。

**2. TCP协议头**

当数据包传输到传输层时，传输层在现有的数据包结构上包装一个`TCP协议头`，协议头结构如图所示：

![image](http://wx4.sinaimg.cn/large/74b07056ly1fgkodhf6kvj20m8090jsz.jpg)

- Source Port 是源端口
- Destination Port 是目的端口
- Sequence Number 是包的序号，用来解决网络包乱序（reordering）问题。
- Acknowledgement Number就是ACK——用于确认收到，用来解决不丢包的问题。
- Window又叫Advertised-Window，也就是著名的滑动窗口（Sliding Window），用于解决流控的
- TCP Flag ，TCP状态标志，主要是用于操控TCP的状态机的。
- TCP Options，是tcp的可选项，其中包括一些优化选项，例如TCP SACK Permitted Option 该选项用于优化tcp数据的重传机制。

下图是通过`Wireshark`抓取的`tcp协议头`

![image](http://wx3.sinaimg.cn/large/74b07056ly1fgkodevityj211b08g0sz.jpg)

**3. TCP FLAGS**

TCP状态标志的对应关系如下：

- S -> Syn
- A -> ACK
- F -> FIN
- R -> RST
- P -> PUSH


## TCP vs UDP 

在`传输层`我们可以看到既有`TCP`,还有`UDP`。相比于`TCP`而言`UDP`在数据传输时是不可靠的,在`UDP`协议中并没有双端建立连接的`3`次握手，以及断开连接的`4`次握手。所以在对数据可靠性要求不高的场景下,使用`UDP`做为数据传输协议可以提高数据传输性能。相反的如果对数据可靠性较高的场景下我们必须要使用`TCP`来保证数据的完整性和可靠性，但频繁的建立`TCP`连接会非常消耗系统资源。

## 连接的建立与断开

**1. 三次握手**

- 第一步：client 发送 syn 到server 发起握手；
- 第二步：server 收到 syn后回复syn+ack给client；
- 第三步：client 收到syn+ack后，回复server一个ack表示收到了server的syn+ack


- 图示建立三次握手的示意图

![image](http://wx4.sinaimg.cn/large/74b07056ly1fgkodg9royj20ob0r40ut.jpg)

为了更好的理解`tcp`建立连接的过程，我在本地与远程`mysql`建立链接，然后查询一条数据，最后断开连接，整个过程使用`Wireshark`抓取数据包。建立连接的代码如下：
```
public static void main(String[] args) throws SQLException {
		int count = 0;
		Connection connection = DriverManager.getConnection(URL, USERNAME, PASSWORD);
		Statement stat = connection.createStatement();
		ResultSet rs = stat.executeQuery("SELECT count(1) FROM iot_update.iu_job_info");
		while (rs.next()) {
			count = rs.getInt(1);
		}
		System.out.println(count);
		connection.close();
	}
```

使用`Wireshark`抓取建立连接的数据包如下：

![image](http://wx1.sinaimg.cn/large/74b07056ly1fgkodhwt2tj210s01qt8s.jpg)

1. 第一步client向server发送SYN,seq = 0;
2. 第二步server响应client发送SYN + ACK, ack = seq+1 , seq = 0;
3. 第三步client响应server发送ACK, ack = seq+1, seq;

值得一提的是 ack = seq + 数据包大小，在建立链接过程中由于没有数据包的传输，所以在seq后面默认加1。

- win (Window size value) 滑动窗口大小
- mss (Maximum Segment Size) 数据段最大大小
- SACK_PERM (TCP SACK Permitted) 是否支持Selective ack(用户优化重传效率）

**2. 为啥是三次握手?**

为什么是3次握手不是2次呢??不是5次呢? 按照我自己的理解是第一步client向server发送SYN，如果server接收到请求就能证明，server的接收网络是能正常工作的。第二步server响应client发送SYN + ACK，如果client接收到回馈信息，就表示cilent的接收网络是能正常工作的，以及client发送网络是能正常工作的。这时已经经过两次握手了，但是还没有确定server的发送网络是否能正常工作。所以需要第三次握手来让server认定自己的发送网络是没有问题的。所以经历过三次握手连接后能过确保数据传输通道的稳定性和可靠性。

**3. 四次挥手**

- 第一步： client主动发送fin包给server
- 第二步： server回复ack（对应第一步fin包的ack）给client，表示server知道client要断开了
- 第三步： server发送fin包给client，表示server也可以断开了
- 第四步： client回复ack给server，表示既然双发都发送fin包表示断开，那么就真的断开吧

为了更好的理解挥手过程我使用了`Wireshark`抓取断开链接的数据包。

![image](http://wx1.sinaimg.cn/large/74b07056ly1fgkodibclcj211a031q36.jpg)

1. 第一步client向server发送FIN ACK,seq = x ack = y;
2. 第二步server响应client发送FIN ACK, seq = y ack = x+1;
3. 第三步client响应server发送ACK, seq ack = y+1;

**4. 三次挥手还是四次挥手**

说到这里你可能有疑问为什么理论上的4次挥手，到实际操作中变成了3次挥手。我的理解是这样的。第一步client向server发送FIN ACK时表示,client主动要求断连。这时server接受到断开请求后从OS通知上层应用client要断开连接，同时应用程序需要为断开连接做一些准备工作。重点就在这段应用程序的准备时间，如果准备时间很长的情况下，server会先向client回馈ack，当应用程序准备完成后再发送fin。如果准备时间很短的情况下,server会同时回馈 fin + ack，两步并一步。从而减少挥手次数提高效率。

**5. 半连接队列与全连接队列**

半连接对列保存着第一次握手阶段请求client的信息。全连接队列同样保存着连接client信息，不同的是全队列的client信息是从半队列出栈后再入栈到全连接队列中，下图展示了半连接队列与全连接队列。

![image](http://wx4.sinaimg.cn/large/74b07056ly1fgkvt2zguwj20sg0krjt1.jpg)

在高并发情况下，有时间歇性出现建立不上连接。
