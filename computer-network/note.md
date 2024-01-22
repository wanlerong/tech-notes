# 计算机网络

## 参考文档：

主要：
- [tcp 协议标准](https://datatracker.ietf.org/doc/html/rfc9293)
- 书籍：计算机网络-自顶向下方法

次要：
- [详解](https://www.xiaolincoding.com/network/3_tcp/tcp_interview.html#tcp-%E8%BF%9E%E6%8E%A5%E5%BB%BA%E7%AB%8B)

## packet 交换机的时延（路由器的）

WX20240121-223720

处理时延：
检查分组首部（如checksum）和决定将该分组导向何处所需要的时间是处理时延的一部分。微秒级

排队时延：
在队列中，当分组在链路上等待传输时，它经受排队时延。一个特定分组的排队时延长度将取决于先期到达的正在排队等待向链路传输的分组数量。

传输时延：
这是将所有分组的比特推向链路(即传输，或者说发射)所需要的时间。对于100Mbps的以太网链路，速率R = 100Mbps

传播时延：
光纤就是光速

## 分层

WX20240121-224826

## DNS
基于 UDP

WX20240121-233310

## CDN

用内容分发网（Content Distribution Network, CDN）

深入。是通过在遍及全球的接入ISP中部署服务器集群来深入到ISP的接入网中。

邀请做客。通过在少量（例如10个）关键位置建造大集群来邀请到ISP做客。


大多数CDN利用DNS来截获和重定向请求；

WX20240121-234008

## OSI 七层
应用层
表示层：负责数据的格式，比如字符编码与转换，以及数据加密
会话层：负责建立、维持和终止会话。
传输层
网络层
数据链路层
物理层

## SSL协议
工作在表示层。
如果一个应用程序要使用SSL的服务，它需要在该应用程序的客户端和服务器端包括SSL代码
ssl 将数据加密后，再往传输层传递。

## 运输层
网络层提供了主机之间的逻辑通信，而运输层为运行在不同主机上的进程之间提供了逻辑通信。

## socket
套接字，是传输层面向应用层提供的一套接口。

## UDP
不提供不必要服务的轻量级运输协议，它仅提供最小服务。
为调用它的应用程序提供了一种不可靠、无连接的服务。
可能是乱序的，也没有拥塞控制机制

关于发送什么数据以及何时发送的应用层控制更为精细。因为没有拥塞控制，想发就发。
无需建立连接
分组首部开销小。每个TCP报文段都有 20 字节的首部开销，而UDP仅有8字节
的开销。

WX20240122-003438

发送方的UDP对报文段中的所有16比特字的和进行反码运算, 求和时遇到的任何溢出都被回卷。
接收方也做求和，如果数据是正确的，那么和 checksum 相加后应该是 1111111111111111，16个1。



client.py
```
from socket import *
serverName = 'hostname'
serverPort = 12000
clientSocket = socket(AF_INET, SOCK_DGRAM)
message = raw_input('Input lowercase sentence:')
clientSocket.sendto(message.encode(),(serverName, serverPort))
modifiedMessage, serverAddress = clientsocket.recvfrom(2048)
print(modifiedMessage.decode())
clientSocket.close()
```

AF_INET: 网络层协议是 IPv4
SOCK_DGRAM: 是 UDP


server.py
```
from socket import *
serverPort = 12000
serverSocket = socket(AF_INET, SOCK_DGRAM)
serversocket.bind(('',serverPort))
print("The server is ready to receive")
while True:
	message, clientAddress = serverSocket.recvfrom(2048)
	modifiedMessage = message.decode().upper()
	serverSocket.sendto(modifiedMessage.encode(), clientAddress)
```


### 回退N步，GBN 协议
WX20240122-004217

如果出现超时，发送方重传所有已发送但还未被确认过的分组。

在GBN协议中，接收方丢弃所有失序分组。

累积确认：如果分组 K 已接收并交付，则所有序号比 K 小的分组也已经交付。

### 选择重传

选择重传（SR）协议通过让发送方仅重传那些它怀疑在接收方出错（即丢失或受损）的分组而避免了不必要的重传。

接收方就需要缓存失序的分组。

## TCP

与UDP不同，TCP是一个面向连接的协议。需要通过握手建立连接, client 要先调用 connect 方法，才能 send。udp 可以直接 send。

当创建该TCP连接时，我们将其与 客户套接字地址IP地址和端口号 和服务器套接字地址 IP地址和端口号关联起来。

发送消息时，不需要再指定对方的地址。

TCP是在不可靠的 IP）端到端网络层之上实现的可靠数据传输协议。（即路由器可能断电等等）

client.py
```
from socket import *
serverName = servernamer
serverPort = 12000
clientSocket = socket(AF_INET, SOCK_STREAM)
clientSocket.connect((serverName, serverPort))
sentence = raw_input('Input lowercase sentence:')
clientSocket.send(sentence.encode())
modifiedSentence = clientSocket.recv(1024)
print('From Server:', modifiedSentence.decode())
clientSocket.close()
```

server.py
```
from socket import *
serverPort = 12000
serverSocket = socket(AF_INET, SOCK_STREAM)
serverSocket.bind(('',serverPort))
serverSocket.listen(1)
print('The server is ready to receive')
while True:
	connectionSocket, addr = serverSocket.accept()
	sentence = connectionSocket.recv(1024).decode()
	capitalizedSentence = sentence.upper()
	connectionSocket.send(capitalizedSentence.encode())
	connectionSocket.close()
```

需要为连接开辟，发送缓存，和接收缓存。


### TCP 报文的结构是什么？

![结构](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20231219-162141.png)

[字段解释](https://datatracker.ietf.org/doc/html/rfc9293#name-header-format)


- 32比特的序号字段 (sequence number field) 和 32比特的确认号字段(acknowledgment number field) 
- 16比特的接收窗口字段(receive window field),该字段用于流量控制。我们很快就会看到，该字段用于指示接收方愿意接受的字节数量。

- 4比特的首部长度字段(header length field),该字段指示了以32比特的字为单位的TCP首部长度。由于TCP选项字段的原因，TCP首部的长度是可变的。(通常,选项字段为空，所以TCP首部的典型长度是20字节。)

- 可选与变长的选项字段(options field),该字段用于发送方与接收方协商最大报文段长度(MSS)等。

- 6比特的标志字段(flag field)。ACK比特用于指示确认字段中的值是有效的，即该报文段包括一个对已被成功接收报文段的确认。RST SYN和FIN比特用于连接建立和拆除，我们将在本节后面讨论该问题。在明确拥塞通告中使用了 CWR和ECE比特
	ACK: Acknowledgment field is significant
	RST: Reset the connection
	SYN: Synchronize sequence numbers
	FIN: No more data from sender.


设置该 MSS （Maximum Segment Size 最大报文长度） 要保证一个TCP报文段（当封装在一个IP数据报中）加上TCP/IP首部长度（通常40字节）将适合单个链路层帧。以太网和PPP链路层协议都具有1500字节的MTU,因此MSS的典型值为 1460 字节。


### TCP为什么要三次握手，四次挥手？

[详解](https://www.xiaolincoding.com/network/3_tcp/tcp_interview.html#tcp-%E8%BF%9E%E6%8E%A5%E5%BB%BA%E7%AB%8B)

三次握手才可以阻止重复历史连接的初始化（主要原因）
三次握手才可以同步双方的初始序列号
三次握手才可以避免资源浪费

### 为什么挥手需要四次？
再来回顾下四次挥手双方发 FIN 包的过程，就能理解为什么需要四次了。

关闭连接时，客户端向服务端发送 FIN 时，仅仅表示客户端不再发送数据了但是还能接收数据。
服务端收到客户端的 FIN 报文时，先回一个 ACK 应答报文，而服务端可能还有数据需要处理和发送，等服务端不再发送数据时，才发送 FIN 报文给客户端来表示同意现在关闭连接。
从上面过程可知，服务端通常需要等待完成数据的发送和处理，所以服务端的 ACK 和 FIN 一般都会分开发送，因此是需要四次挥手。

### 超时/重传机制

需要估计往返时间，通过采样：
报文段的样本RTT （表示为SampleRTT）就是从某报文段被发出（即交给IP）到对该报文段的确认被收到之间的时间量。

指数加权移动平均:
EstimatedRTT = 0.875 * EstimatedRTT + 0.125 * SampleRTT

DevRTT 用于估算 SampleRTT 与 EstimatedRTT 之间差值

最终如果时间大于 TimeoutInterval, 就会触发超时重传
TimeoutInterval = EstinMrtedRTT + 4 * DevRTT

TCP也使用流水线，使得发送方在任意时刻都可以有多个已发出但还未被确认的报文段存在。

每次TCP重传时都会将下一次的超时间隔设为先前值的两倍。

GBN不仅会重传分组几，还会重传所有后继的分组n + 1, n+2,…，N 在另一方面
TCP将重传至多一个报文段，即报文段 n。此外，如果对 n+1 的确认报文在报文段 n 超时之前到达
TCP甚至不会重传报文段 n

## 冗余ACK
在TCP的快速重传机制下，收到对一个特定报文段的 3个冗余 ACK 就可作为对后面报文段的一个隐式 NAK, 从而在超时之前触发对该报文段的重传，流水线可显著地增加一个会话的呑吐量


## 流量控制

TCP通过让发送方维护一个称为接收窗口(receive window)的变量来提供流量控制。
通俗地说，接收窗口用于给发送方一个指示一 该接收方还有多少可用的缓存空间。

## 拥塞控制

一个丢失的报文段表意味着拥塞，因此当丢失报文段时应当降低TCP发送方的速率。

给定 ACK 指示源到目的地路径无拥塞，而丢包事件指示路径拥塞，TCP调节其传输速率的策略是增加其速率以响应到达的ACK, 除非岀现
丢包事件，此时才减小传输速率。

TCP拥塞控制算法

1. 慢启动
因此，在慢启动 slow-start 状态，cwnd（拥塞窗口）的值以 1 个 MSS 开始并且每当传输的报文段首次被确认就增加 1 个 MSS
收到第一个 ACK，发送窗口加 1 个MSS，发送 2 个MSS，收到两个 ACk，窗口再加 2，窗口等于 4 ，再发送 4 个 MSS
这一过程每过一个 RTT（往返时延）, 发送速率就翻番。


首先，如果存在一个由超时指示的丢包事件（即拥塞）。
TCP 发送方将 cwnd 设置为 1。并重新开始慢启动过程会 ssthresh（慢启动阈值） 置为拥塞窗口值的一半，
当下一轮慢启动的 cwnd 的值等于 ssthresh 时，结束慢启动并且 TCP 转移到拥塞避免模式。


最后一种结束慢启动的方式是, 如果检测到3个冗余ACK（没超时，但是接受方没收到）
这时TCP执行一种快速重传（参见3. 5. 4节）并进入快速恢复状态

2. 拥塞避免模式

一旦进入拥塞避免状态，cwnd 的值大约是上次遇到拥塞时的值的一半，即距离拥塞可能并不遥远！因此，TCP无法每过一个RTT再将cwnd的值翻番，而是采用了一种较为
保守的方法，每个RTT只将cwnd的值增加一个MSS ［RFC 5681］。
每收到一个ack，就将 cwnd 增加一个 MSS/cwnd 个字节，当前窗口全部 ack 后，才会增加一个 MSS。

出现超时后，还是会重新开始慢启动，且 ssthresh 置为 cwnd 的一半

3. 快速恢复 （为了快速重传）
如果检测到3个冗余ACK，就会进入快速恢复

对收到的每个冗余的 ACK, cwnd 的值增加一个 MSS。
最终，当对丢失报文段的一个ACK到达时，TCP 在降低 cwnd 后进入拥塞避免状态。


### 输入URL会发生什么
### http 请求的调用过程
### http 会话机制如何保证长连接
### 同一个会话里面所有的请求都是共享登录态的还是不共享
### 访问一个页面时候会夹带很多个请求，那这些请求是共用一个连接还是每个请求都有一个连接


