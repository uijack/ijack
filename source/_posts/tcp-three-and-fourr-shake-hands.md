title: "TCP的3次握手和4次挥手机制"
date: 2016-01-07 19:32:26
description: "主要讲一下TCP(Transmission Control Protocol)传输控制协议3次握手和4次挥手的过程以及相关扩展知识"
tags:
- "TCP"
---

## 看前须知

- 6种位码:

```plain
SYN: Synchronize同步建立连接的标识位.初始为1
ACK: Acknowledgement确认
PSH: push传送
FIN: finish结束
RST: Reset复位,表明TCP连接中出现严重差错(注意崩溃或其他原因),必须释放链接,然后再重新建立运输连接.
URG: Urgent紧急:传输优先级高的数据.

附带:

seq: Sequence number(序号): TCP连接中传送的字节流中的每一个字节都按顺序编号.又叫"报文段序号".
ack: Acknowledge number(确认号):期望收到对方下一个报文段的第一个数据字节的序号
```

`TCP`在网络OSI的七层协议模型中的第4层(Transport, 传输层).定义传输数据的协议端口号,以及控制和差错校验,协议有:TCP,UDP,数据包一旦离开网卡即进入网络传输层.

`IP`在第3层(Network,网络层).进行逻辑地址寻址,实现不同网络之间的路径选择.协议有:ICMP,IGMP,`IP(IPV4,IPV6)`,ARP,RARP.

`数据链路层`在第2层(Link).建立逻辑连接,进行硬件地址寻址,差错校验.将`比特`组合进而合成`帧`,用MAC地址访问介质.错误发现但不纠正.

`物理层`在网络第1层(Physical Layer).建立,维护,断开物理连接.

- 窗口

> 建立连接时,各端分配一块缓冲区用来存储接收的数据,并将缓冲区的尺寸发送给另一端.接收方发送的`确认信息`中包含了自己剩余的缓冲区尺寸.`剩余缓冲区空间的数量叫做窗口`.

- TCP流量控制

所谓流量控制(flow control)就是让发送方的发送速率不要太快,要让接收方来得及接收.否则就会造成`数据丢失`.
TCP的窗口单位是字节,不是报文段.数据报文段序号的初始值设为1.

- 滑动窗口

利用滑动窗口实现`TCP流量控制`.发送方的发送窗口不能超过接收方给出的接收窗口数值.
如我的接收窗口rwnd = 400字节.(receive window).

## 三次握手(建立连接)

![tcp三次握手图](/img/tcp-three-and-four-hand.png)

![tcp首部](/img/tcp-three-and-four-hand-2.png)

注: 主动发起连接建立的应用进程叫做`客户端`.而被动等待连接建立的应用进程叫做`服务器`.
`ACK`表示首部中的确认位ACK,小写的`ack`表示确认字段(确认号)的值.

TCP协议提供可靠的连接服务,采用三次握手连接一个连接.

`TCB(Transimission Control Block)`存储了每一个连接中的重要信息.TCP连接表,发送和接受缓存指针,重传队列指针,当前的发送和接收序号等.

服务器进程先创建传输控制块,准备接受客户端进程的连接请求,然后服务器进程就处于`LISTEN(收听)`状态,等待客户的连接请求.如有,即作出响应.

所谓三次握手就是TCP建立一次连接的过程.

- 第一次握手

> 客户端进程先创建`传输控制块TCB`,然后向服务器发出连接请求`报文`.这时TCP报文首部中的`同部位SYN=1`,同时选择一个`初始序号seq=x`.这时客户端进程进入`SYN-SENT(同步已发送)`状态.

TCP规定,SYN报文段(即SYN=1的报文段)不能携带数据,但要消耗掉一个序号.

- 第二次握手

> 服务器收到连接请求报文后,如同意建立连接,则向客服端发送确认.在`确认报文段`中把`SYN`位和`ACK`位都置1(即 SYN=1, ACK=1),确认号`ack = x+1`, 同时也为自己选择一个`初始序号seq = y`.这时服务器进程进入`SYN-RECV(同步收到)`状态.

注意:这个报文段也不能携带数据,但同样要消耗一个序号.

- 第三次握手

> 客户端进程收到服务器的确认后,还要向服务器给出确认.确认报文段`ACK=1`,确认号`ack = y+1`,而自己的序号`seq = x+1`.这时TCP连接已经建立,客户端进入`ESTABLISHED(已建立连接)`状态.当服务器收到客户端的确认后,也进入ESTABLISHED状态.

TCP标准规定,ACK报文段可以携带数据,但`如果不携带数据则不消耗序号`.在这种情况下,`下一个数据报文段`的序号仍是`seq = x+1`.

- 为什么客户端要发送一次确认报文呢?

主要是为了防止已失效的连接请求报文段突然又传送到了服务器,因而产生错误.所谓`已失效的连接请求报文段`是这样产生的.考虑正常情况,客户端发出连接请求,服务器因连接请求报文丢失而未收到确认报文,于是客户端在重传一次连接请求.后来收到了确认,建立了连接.数据传输完毕,就释放了连接.客户端共发送`两个连接请求报文段`.其中第一个丢失,第二个到了服务器.没有`已经失效的连接请求报文段`.

现假定出现一种异常情况,即客户端发出的第一个连接请求报文段并没有丢失,而是在`某些网络节点长时间滞留了`.以致延误到连接释放以后的某个时间才到达服务器.本来这是一个早已失效的报文段.但服务器收到此失效的连接请求报文段后,就`误认为`是客户端`又`发出一次新的连接请求,于是向客户端发送确认报文段,同意建立连接.假定不采用三次握手,那么只要服务器发出确认,新的连接就建立了.由于现在客服端并没有发出建立连接的请求.因此`不理睬`服务器的确认.也不会向服务器发送数据.但服务器却以为新的运输连接已经建立了,并一直等待客服端发来数据.服务器的许多资源就被`白白浪费`了.
采用`三次握手`的办法可以防止上述现象.因为服务器由于收不到确认.就知道客户端并没有要求建立连接.

## 四次挥手(关闭连接)

![tcp三次握手图](/img/tcp-three-and-four-hand-3.png)

由于TCP连接是全双工的,因此每个方向都必须单独进行关闭.数据传输结束后,通信双方都可以释放连接,现在客户端和服务器都处于也进入ESTABLISHED状态.以下假设客户端先发起释放连接.

所谓四次挥手就是关闭TCP连接的过程.

- 第一次挥手

> 客户端应用进程先向其TCP发出`连接释放`报文段,并停止再发送数据,主动关闭TCP连接,此时客户端把`连接释放报文段首部`的`FIN位置1(即FIN=1)`,其序号`seq = u`,u等于前面客户端已经传过的数据的最后一个字节的序号加1.这时客户端进入`FIN-WAIT-1(终止等待1)`状态,等待服务器确认.

TCP规定,FIN报文段即使不携带数据,它也消耗一个序号.

- 第二次挥手

> 服务器收连接释放报文段后即发出确认,报文段的`确认位ACK=1`,确认号`ack = u+1`,而这个报文段自己的序号`seq = v`, v等于前面服务器已传过的数据的最后一个字节的序号加1. 然后服务器就进入`CLOSE-WAIT(关闭等待)状态.客户端收到来自服务器的确认后,就进入`FIN-WAIT-2(终止等待2)`状态,等待服务器发出的连接释放报文段.

TCP服务器进程这时应通知高层应用进程,因而客户端到服务器方向的连接就释放了.这时候TCP连接处于`半封闭(half-close)`状态,即客户端已经没有数据要发送了,但服务器若发送数据,客户端仍要接收,也就是说.服务器到客户端这个方向的连接并未关闭,

- 第三次挥手

> 若服务器已经没有要向客户端发送的数据,其应用进程就通知TCP释放连接,这时服务器发出的连接释放报文段必须使`FIN=1`,确认位`ACK=1`,现假定服务器的序号`seq = w`(在半关闭状态服务端可能有发送了一些数据).服务端还`必须重复`上次已发送过的`确认号ack = u+1`.这时服务器就进入`LAST-ACK`(最后确认)状态,等待客服端的确认.

- 第四次挥手

> 客户端收到服务器的连接释放报文段后,必须对此发出确认,在确认报文段中把ACK置1(ACK=1),确认号`ack = w+1`,而自己的序号`seq = u+1`(根据TCP标准,前面发送过的FIN报文段要消耗一个序号).然后进入到`TIME-WAIT`(时间等待)状态.

请注意现在的TCP连接还没释放掉.必须经过`时间等待计时器`设置的时间2MSL后,客户端才进入到`CLOSE`状态.时间`MSL`叫做`最长报文段寿命(Maximun Segment Lifetime)`.RFC 793建议一个MSL设为2分钟.从客服端进入到TME-WAIT状态后,要经过4分钟才能进入CLOSED状态.当客服端撤销相应的`传输控制块TCB`后,就结束了这次TCP释放.

- 为什么客户端在TIME-WAIT状态必须等待2MSL的时间呢?两个理由:

1. 为了保证客户端发送的最后一个ACK报文段能够到达服务器,这个ACK报文有可能丢失,因而使处于在LAST-ACK状态的服务器收不到已发送的FIN+ACK报文段确认.服务器会超时重传这个FIN+ACK报文段.而客户端就能在2MSL时间内收到这个重传的FIN+ACK报文.接着客户端重传一次确认,重新启动2MSL计时器.最后客户端和服务器都正常进入到CLOSED状态,如果客户端在TIME-WAIT状态下不能等待一段时间,而在发送完ACK报文段后就立即释放连接,就无法收到服务器重传的FIN+ACK报文段,也不会再发送再次确认报文段.服务器就`无法按照正常步骤进入CLOSED状态`.

2. 防止上一节提到的`已失效的连接请求报文段`出现在本连接中.客户端在发送完最后一个ACK报文段后,在经过时间2MSL,就可以使本连接持续的时间内所产生的`所有报文段都从网络中消失`.这样可以使下一个新的连接中不会出现这种旧的连接请求报文段.服务器只要收到客服端发出的确认,就进入CLOSED状态.服务器在撤销相应的传输控制块TCB后,就结束了这次TCP连接.我们注意到`服务器结束TCP连接的时间要比客服端早一些`.

- 保活计时器(keepalive timer)

假设客户端已主动与服务器建立连了TCP连接,后来出故障.服务器就不可能在收到来自客户端发来的数据.因此应该`有措施使服务器不要白白等待下去`.这时就要使用`保活计时器`.服务器每次收到一次客户端的数据,就重置保护计时器,时间设通常为2小时.两小时没有收到客户端的数据,服务器就发送一个`探测报文段`,以后每隔75分钟发送一次,若连续发送`10个探测报文段`后仍无客户的响应,服务器就认为客户端出了故障,接着就关闭了连接.

![TCP有限状态机](/img/tcp-three-and-four-hand-4.png)

注:`粗实线箭头`表示对客户进程的正常变迁;`虚线箭头`表示对服务器进程的正常变迁;另一种`细箭头`表示异常变迁.