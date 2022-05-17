### 最大报文长度
* TCP 可从缓存中取出并放入报文段的数据数量受限于**最大报文长度**（MMS）
* MMS受限于最大链路层帧长度（最大传输单元MTU），MTU一般为1500字节（以太网和PPP链路层协议），TCP/IP首部为40字节，MMS为1460字节
### TCP报文段结构
![tcpstructure](/files/pic/tcpstructure.png)
* 首部长度（偏移）4比特：因为有可变部分，在此指定首部长度
* flags 6比特：（CWR ECE） URG ACK PSH RST SYN FIN 
* 可变部分：发送方与接收方协商最大报文长度，或在高速网络环境下用作窗口调节因子时使用。
* 紧急指针：尾指针，使用时，TCP必须同通知接收端的上层实体（与URG同用），而PSH位置位时，所有数据都提交上层。（一般不使用这三个字段）

### 序号与确认号
* 报文段的序号是最后一个字节的序号，累计确认（确认最后一个字节的序号）
* 初始序号随机选择，可以减少新连接将那些仍在网络中存在的就连接的报文段误认为有效报告文段的有效性
* 乱序报文段的处理为规定，由用户决定
* 确认号和数据报文段可以在一起发送，称为捎带
### 超时
每次重传后，超时时间翻倍
成功收到ACK的情况，记录一个SampleRTT
跟据多次SampleRTT，计算EstimatedRTT = (1-α) * EstimatedRTT + α * SampleRTT
α = 0.125
还会记录DevRTT = (1 - β) * DevETT + β * | SampleRTT - EstimatedRTT|
β = 0.25
TimeoutInterval = EstimatedRTT + 4 * DevRTT
### 快速重传
接收方收到乱序报文段将回复一个冗余确认
发送方收到三个冗余确认，将直接重传
### 流量控制
明确两个概念
* 流量控制是为了消除发送方使接收方缓存溢出的可能性
* 拥塞控制是为了适应IP网络的拥塞

接收窗口用rwnd表示，
rwnd = RcvBuffer - [LastByteRcvd - LastByteRead]
LastByteSent - LastByteAcked <= rwnd

**当接收方的窗口为0时，而且接收方没有要发送给发送方的报文段，发送方就会一直以为窗口为0，那么发送方将不再发送数据，这个问题如何解决？**
发送方收到窗口为0的报文段时，发送一个只有一个字节数据的报文段，这些报文段将被接收方接收确认，当接收方缓存清空后，在确认报文包含一个非0的rwnd值


### 连接管理
**握手**
![tcpconnect](pic/tcpconnect.png)
**注意几点：**
* 为了某些安全原因，序号应该随机开始
* 最后一次ACK报文已经可以携带数据(处于ESTABLISHED状态)

**socket接口与握手的关系**
![tcpsocket](pic/tcpsocket.png)

**挥手**
![tcpdisconnect](pic/tcpdisconnect.png)

**泛洪攻击**
攻击者只发送SYN报文，不发送第三个ACK报文，导致半连接队列溢出
如何解决呢？
采用SYN cookie方式，半连接不分配资源，将端口号和ip用私钥加密，通过synack回复给客户端，如果客户端回复的cookie验证正确（此报文的ip地址和端口加密后与cookie一致），那么为其分配资源。

### 错误情况
1. 客户端要连接的端口不接受连接，对方会返回一个RST标志报文段，如果时UDP，则返回一个特殊的ICMP数据报

