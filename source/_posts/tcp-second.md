---
title: 浅谈tcp协议第二篇
date: 2017-08-06 20:34:49
categories: tcp
tags: tcp
---

> 这篇文章将继续上一次的tcp内容继续说说。

### TCP 状态

下图是tcp连接过程的状态流转。

![image](http://wx3.sinaimg.cn/large/74b07056ly1fgkodgukbpj20fm0m90tw.jpg)

**1. 建立连接阶段**

- 当接收端开启监听服务，状态变更为listen。
- 发送端发送syn到接收端，状态变更为syn-sent。
- 接收端接收到发送端发送的syn状态变更为syn-received，同时发送syn+ack到发送端。
- 发送端接收到接收端发来的syn+ack状态变更为establish，同时发送ack到接收端。
- 接收端接收到响应信息后将状态变更为establish。

可以使用如下命令查看当前系统tcp连接所在的状态和在该状态下的连接个数
```
netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
```

**2. 半连接队列和全连接队列**

接着上一篇文章我们继续的深入的理解这两个队列。

![image](http://wx4.sinaimg.cn/large/74b07056ly1fgkvt2zguwj20sg0krjt1.jpg)

- 当接收方状态处于syn-received的时候，接收方维护了一个半连接队列，这个队列的大小阈值 = `max(64, /proc/sys/net/ipv4/tcp_max_syn_backlog)`。
- 当接收方状态处于establish的时候，接收方将连接对象从半连接队列出队列，再进全连接队列。 全队列大小阈值 = `min(backlog, somaxconn)` backlog是在socket创建的时候传入的，somaxconn是一个os级别的系统参数，这个参数的配置在`/proc/sys/net/core/somaxconn`
- 当server调用accept()函数，表示从accpt_queue取出连接。

我们可以使用以下命令查看全连接队列和半连接队列的使用情况
```
ss -lnt
```

![image](http://wx2.sinaimg.cn/large/74b07056ly1fgru9uhnedj20xc03tjrd.jpg)

上面看到的第二列Send-Q 表示第三列的listen端口上的全连接队列最大为100，第一列Recv-Q为全连接队列当前使用了多少。java建立SocketServer时默认使用的全连接队列大小为50。

当全连接队列溢出后对于之后连接进来的连接如何处理可以根据以下参数进行处理。
```
/proc/sys/net/ipv4/tcp_abort_on_overflow

0 - 表示如果三次握手第三步的时候全连接队列满了那么server扔掉client 发过来的ack（在server端认为连接还没建立起来），同时根据以下参数进行重试，重新发送syn+ack到client

/proc/sys/net/ipv4/tcp_synack_retries

1 - 1表示第三步的时候如果全连接队列满了，server发送一个reset包给client，表示废掉这个握手过程和这个连接（本来在server端这个连接就还没建立起来）。
```



**半连接攻击**

比如syn floods 攻击就是针对半连接队列的，攻击方不停地建连接，但是建连接的时候只做第一步，第二步中攻击方收到server的syn+ack后故意扔掉什么也不做，导致server上这个队列满其它正常请求无法进来。

**2. 断开连接**

- 发送端发送syn到接收端状态变更为fin-wait1。
- 接收端接收到syn后通知上层应用进行连接关闭，响应ack给发送端，接收端状态变更为close-wait。
- 发送端接收到ack后状态变更为fin-wait2。
- 当数据接收端应用关闭准备完成，os发送syn-fin到发送端，状态变更为last-ack。
- 接收端接收到syn-fin后，响应ack给发送端，状态变更为time-wait。
- 接收端接收到发送端发来的最后一个ack后状态变更为closed。

我们可以看到断开连接中有一个closing状态，该状态的出现是在接收端发送syn(fin)+ack后接收端出现的状态，即三次挥手。之后数据接收端发送ack给对端，状态变更成time-wait。

**time-wait状态**

当主动要求断开连接的一端，会一直处于time-wait状态，当超过预定的阈值的时候，连接状态会变更为closed.

time-wait被设计长时间占有是必然的。

1. 为了防止接收端没有接收到最后的ack响应，重新发送syn(fin)+ack到发送端，然后发送端可以在当前没有断开的连接上进行ack响应。
2. 是因为防止路由器将新连接接入到当前即将关闭的连接通道上，与即将关闭的链接通道进行隔离。

优化time-wait持有策略。
```
/proc/sys/net/ipv4/tcp_tw_reuse = 1 表示开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭；
/proc/sys/net/ipv4/tcp_tw_recycle = 1 表示开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭。
```

### tcp 重试机制

tcp的可靠性离不开其自身的重试机制。

接收端给发送端的Ack确认只会确认最后一个连续的包，比如，发送端发了1,2,3,4,5一共五份数据，接收端收到了1，2，于是回ack 3，然后收到了4（注意此时3没收到），此时的TCP会怎么办？我们要知道，因为正如前面所说的，SeqNum和Ack是以字节数为单位，所以ack的时候，不能跳着确认，只能确认最大的连续收到的包，不然，发送端就以为之前的都收到了。

**超时重传机制**

一种是不回ack，死等3，当发送方发现收不到3的ack超时后，会重传3。一旦接收方收到3后，会ack 回 4——意味着3和4都收到了。

但是，这种方式会有比较严重的问题，那就是因为要死等3，所以会导致4和5即便已经收到了，而发送方也完全不知道发生了什么事，因为没有收到Ack，所以，发送方可能会悲观地认为也丢了，所以有可能也会导致4和5的重传。

对此有两种选择：

一种是仅重传timeout的包。也就是第3份数据。
另一种是重传timeout后所有的数据，也就是第3，4，5这三份数据。
这两种方式有好也有不好。第一种会节省带宽，但是慢，第二种会快一点，但是会浪费带宽，也可能会有无用功。但总体来说都不好。因为都在等timeout，timeout可能会很长（在下篇会说TCP是怎么动态地计算出timeout的）

**快速重传机制**

于是，TCP引入了一种叫Fast Retransmit 的算法，不以时间驱动，而以数据驱动重传。也就是说，如果，包没有连续到达，就ack最后那个可能被丢了的包，如果发送方连续收到3次相同的ack，就重传。Fast Retransmit的好处是不用等timeout了再重传。

比如：如果发送方发出了1，2，3，4，5份数据，第一份先到送了，于是就ack回2，结果2因为某些原因没收到，3到达了，于是还是ack回2，后面的4和5都到了，但是还是ack回2，因为2还是没有收到，于是发送端收到了三个ack=2的确认，知道了2还没有到，于是就马上重转2。然后，接收端收到了2，此时因为3，4，5都收到了，于是ack回6。示意图如下：

![image](http://wx2.sinaimg.cn/large/74b07056ly1fgsz3a9f6ij20ci0833yx.jpg)

Fast Retransmit只解决了一个问题，就是timeout的问题，它依然面临一个艰难的选择，就是，是重传之前的一个还是重传所有的问题。对于上面的示例来说，是重传#2呢还是重传#2，#3，#4，#5呢？因为发送端并不清楚这连续的3个ack(2)是谁传回来的？也许发送端发了20份数据，是#6，#10，#20传来的呢。这样，发送端很有可能要重传从2到20的这堆数据（这就是某些TCP的实际的实现）。可见，这是一把双刃剑。

**SACK 方法**

另外一种更好的方式叫：Selective Acknowledgment (SACK)（参看RFC 2018），这种方式需要在TCP头里加一个SACK的东西，ACK还是Fast Retransmit的ACK，SACK则是汇报收到的数据碎版。参看下图：

![image](http://wx4.sinaimg.cn/large/74b07056ly1fgsz3aoev2j20p00e3mya.jpg)

这样，在发送端就可以根据回传的SACK来知道哪些数据到了，哪些没有到。于是就优化了Fast Retransmit的算法。当然，这个协议需要两边都支持。在 Linux下，可以通过tcp_sack参数打开这个功能（Linux 2.4后默认打开）。

### timeout 设置

我们知道timeout对于数据包的重传是非常重要的。

- 设置超时时间较长，接收端就长时间接收不到丢失的数据包。
- 设置超时时间较短，可能出现接收端ack包没有到达发送端的时候，发送端超时了发送重复的数据包到接收端。

所以timeout根本不是一个固定的值，它是根据当前网络情况动态设定的。关于timeout的算法有

- Karn / Partridge 算法
- Jacobson / Karels 算法

### 滑动窗口

tcp使用滑动窗口进行数据传输的流量控制，tcp头信息中windows属性就是滑动窗口的信息。滑动窗口的大小取决于两个方面。

- 数据接收端缓冲区的大小
- 拥塞控制（之后我们在详细的说）

首先看下数据缓冲区的示意图

![image](http://wx1.sinaimg.cn/large/74b07056ly1fgyg2036fuj20p009ywfc.jpg)

- 接收端LastByteRead指向了TCP缓冲区中读到的位置，NextByteExpected指向的地方是收到的连续包的最后一个位置，LastByteRcved指向的是收到的包的最后一个位置，我们可以看到中间有些数据还没有到达，所以有数据空白区。
- 发送端的LastByteAcked指向了被接收端Ack过的位置（表示成功发送确认），LastByteSent表示发出去了，但还没有收到成功确认的Ack，LastByteWritten指向的是上层应用正在写的地方。

于是：

- 接收端在给发送端回ACK中会汇报自己的AdvertisedWindow = MaxRcvBuffer – LastByteRcvd – 1;
- 而发送方会根据这个窗口来控制发送数据的大小，以保证接收方可以处理。

下图是滑动窗口的示意图

![image](http://wx2.sinaimg.cn/large/74b07056ly1fgyg21opfuj20ic07i750.jpg)

上图中分成了四个部分，分别是：（其中那个黑模型就是滑动窗口）

1. 已收到ack确认的数据。
2. 发还没收到ack的。
3. 在窗口中还没有发出的（接收方还有空间）。
4. 窗口以外的数据（接收方没空间）

下面是个滑动后的示意图（收到36的ack，并发出了46-51的字节）：

![image](http://wx3.sinaimg.cn/large/74b07056ly1fgyg2181xej20ic05uwev.jpg)

下面我们来看一个接受端控制发送端的图示：

![image](http://wx2.sinaimg.cn/large/74b07056ly1fgyg20gvygj20ii0n8wfy.jpg)

**Zero Window**

上图，我们可以看到一个处理缓慢的Server（接收端）是怎么把Client（发送端）的TCP Sliding Window给降成0的。此时，你一定会问，如果Window变成0了，TCP会怎么样？是不是发送端就不发数据了？是的，发送端就不发数据了，你可以想像成“Window Closed”，那你一定还会问，如果发送端不发数据了，接收方一会儿Window size 可用了，怎么通知发送端呢？

解决这个问题，TCP使用了Zero Window Probe技术，缩写为ZWP，也就是说，发送端在窗口变成0后，会发ZWP的包给接收方，让接收方来ack他的Window尺寸，一般这个值会设置成3次，第次大约30-60秒（不同的实现可能会不一样）。如果3次过后还是0的话，有的TCP实现就会发RST把链接断了。  


### 代码

根据上面对tcp的了解，我写了一个简单的tcp server 的代码。使用的网络模型是同步阻塞方式。

```java
public class Server {
	
	private static final int PORT = 2468;
	private static final int BACKLOG = 102; // 设置全链接队列大小
	private static final int INPUTBUFFER = 4194304*4;
	private static final int OUTPUTBUFFER = 4194304*4;
	private static byte[] READBUFFER = new byte[4194304*2];
	
	public static void main(String[] args) throws IOException {
		ServerSocket serverSocket = new ServerSocket(PORT, BACKLOG);
		
		Socket socket = null;
		long startTimeMs = 0l;
		long endTimeMs = 0l;
		
		while (null != (socket = serverSocket.accept())) {// 当调用accept()函数时，链接对象将从全链接队列出队列socket
			BufferedInputStream bis= null;
			BufferedOutputStream bos = null;
			int byteRead = 0;
			try {
				socket.setTcpNoDelay(true); // 是否启用Nagle算法,java默认是关闭的,非完全禁止小的滑动窗口发送
				socket.setReceiveBufferSize(4194304); // 设置接收端缓冲区大小
				bis = new BufferedInputStream(socket.getInputStream(),INPUTBUFFER);
				bos = new BufferedOutputStream(new FileOutputStream(new File("/home/wangliping/testFile")),OUTPUTBUFFER);
				startTimeMs = System.currentTimeMillis();
				while ((byteRead = bis.read(READBUFFER)) != -1) {
					bos.write(READBUFFER,0,byteRead);
					bos.flush();
				}
				endTimeMs = System.currentTimeMillis();
				
				System.out.println(((endTimeMs - startTimeMs) / 1000)+" S");
			}finally {
				bos.close();
				bis.close();
			}
			
		}
	}
	
}
```

- `BACKLOG` 设置了全队列的大小。
- `setReceiveBufferSize` 设置了数据接收缓冲区的大小。缓冲区的最大值 = 2*(`net.core.rmem_max`)
- `setSendBufferSize` 设置了数据发送缓冲区的大小。缓冲区最大值 = 2*(`net.core.wmem_max`)
- `Nagle算法` 开启此算法后将非完全禁止小滑动窗口发送，数据发送方会等待接收缓冲区有足够大的空间，再发送数据。java默认是关闭的。
- 当数据接收缓冲区大小为`net.core.rmem_max`的一半，数据发送缓冲区大小为`net.core.wmem_max`的一半时，服务器接受4GB文件时处理的时间为`136秒`

![image](http://wx2.sinaimg.cn/large/74b07056ly1fh3bxyk4rtj20u20170sk.jpg)

- 当将数据发送缓冲区和数据接收缓冲区都调整至`net.core.rmem_max`和`net.core.wmem_max`的值后，服务器接受4GB文件时处理的时间为`73秒`

![image](http://wx3.sinaimg.cn/large/74b07056ly1fh3bxy6thxj20ue01b0sk.jpg)
