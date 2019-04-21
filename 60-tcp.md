---
show: step
version: 1.0
enable_checker: true
---
# TCP

## 一、实验说明

#### 1.1 实验内容

本章先介绍 TCP 协议的特点，再深入了解 TCP 的协议和详细实现，包括 tcp 状态、可靠传输、流量控制和拥塞控制等。

#### 1.2 知识点
- tcp 协议头部详解
- tcp 的状态解析
- tcp 的可靠性解析
- tcp 的流量控制解析
- tcp 的拥塞控制解析

#### 1.3 实验环境

- Go 1.12.1
- Xfce 终端

#### 1.4 代码获取

```bash
$ wget http://labfile.oss.aliyuncs.com/courses/1300/netstack.zip
$ unzip netstack.zip
```


## 二、传输层 TCP 的实现
#### 2.1 tcp 简介
TCP（Transmission Control Protocol 传输控制协议）是一种面向连接的、可靠的、基于字节流的传输层通信协议，由 IETF 的 RFC 793 定义。  tcp 为不同主机上的应用程序建立一个虚拟的、面向连接的、可靠的逻辑通道，从应用层来看，通过该逻辑通道，运行不同进程的主机好像直接相连一样，发送的数据也是不会丢失且顺序的，实际上这些主机可能相隔千里。

TCP 协议也是我们最常见的协议，在 TCP/IP 协议栈中算是复杂的一个协议，在 HTTP3 之前，HTTP 协议都是基于 TCP 协议的，市场上大部分的应用也是基于 TCP 的，TCP 的广泛应用离不开它的特点。

#### 2.2 tcp 特点
1. tcp 是面向连接的传输协议。
2. tcp 的连接是端到端的。
3. tcp 提供可靠的传输。
4. tcp 的传输以字节流的方式。  
5. tcp 提供全双工的通信。
6. tcp 有拥塞控制。
![steam](https://doc.shiyanlou.com/document-uid949121labid10418timestamp1555573949562.png/wm)

### 2.3 tcp 首部格式

```txt
https://tools.ietf.org/html/rfc793

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|          Source Port          |       Destination Port        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                        Sequence Number                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Acknowledgment Number                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Data |           |U|A|P|R|S|F|                               |
| Offset| Reserved  |R|C|S|S|Y|I|            Window             |
|       |           |G|K|H|T|N|N|                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Checksum            |         Urgent Pointer        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options                    |    Padding    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                             data                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```
1. 源端口和目的端口   
各占 2 个字节，分别 tcp 连接的源端口和目的端口。关于端口的概念之前已经介绍过了。

2. 序号   
占 4 字节，序号范围是[0，2^32 - 1]，共 2^32（即 4294967296）个序号。序号增加到 2^32-1 后，下一个序号就又回到 0。TCP 是面向字节流的，在一个 TCP 连接中传送的字节流中的每一个字节都按顺序编号。整个要传送的字节流的起始序号（ISN）必须在连接建立时设置。`首部中的序号字段值则是指的是本报文段所发送的数据的第一个字节的序号`。例如，一报文段的序号是 301，而接待的数据共有 100 字节。这就表明：本报文段的数据的第一个字节的序号是 301，最后一个字节的序号是 400。显然，下一个报文段（如果还有的话）的数据序号应当从 401 开始，即下一个报文段的序号字段值应为 401。

3. 确认号        
占 4 字节，是期望收到对方下一个报文段的第一个数据字节的序号。例如，B 正确收到了 A 发送过来的一个报文段，其序号字段值是 501，而数据长度是 200 字节（序号 501~700），这表明 B 正确收到了 A 发送的到序号 700 为止的数据。因此，B 期望收到 A 的下一个数据序号是 701，于是 B 在发送给 A 的确认报文段中把确认号置为 701。注意，现在确认号不是 501，也不是 700，而是 701。  
总之：若确认号为 N，则表明：到序号 N-1 为止的所有数据都已正确收到。TCP 除了第一个 SYN 报文之外，所有 TCP 报文都需要携带 ACK 状态位。

4. 数据偏移   
占 4 位，它指出 TCP 报文段的数据起始处距离 TCP 报文段的起始处有多远。这个字段实际上是指出 TCP 报文段的首部长度。由于首部中还有长度不确定的选项字段，因此数据偏移字段是必要的，但应注意，“数据偏移”的单位是 4 个字节，由于 4 位二进制数能表示的最大十进制数字是 15，因此数据偏移的最大值是 60 字节。

5. 保留   
占 6 位，保留为今后使用，但目前应置为 0。

6. 控制报文标志  
`紧急URG（URGent）`   
当 URG=1 时，表明紧急指针字段有效。它告诉系统此报文段中有紧急数据，应尽快发送（相当于高优先级的数据），而不要按原来的排队顺序来传送。例如，已经发送了很长的一个程序要在远地的主机上运行。但后来发现了一些问题，需要取消该程序的运行，因此用户从键盘发出中断命令。如果不使用紧急数据，那么这两个字符将存储在接收 TCP 的缓存末尾。只有在所有的数据被处理完毕后这两个字符才被交付接收方的应用进程。这样做就浪费了很多时间。  
当 URG 置为 1 时，发送应用进程就告诉发送方的 TCP 有紧急数据要传送。于是发送方 TCP 就把紧急数据插入到本报文段数据的最前面，而在紧急数据后面的数据仍然是普通数据。这时要与首部中紧急指针（Urgent Pointer）字段配合使用。  
`确认ACK（ACKnowledgment）`        
仅当 ACK=1 时确认号字段才有效，当 ACK=0 时确认号无效。TCP 规定，在连接建立后所有的传送的报文段都必须把 ACK 置为 1。  
`推送 PSH（PuSH）`        
当两个应用进程进行交互式的通信时，有时在一端的应用进程希望在键入一个命令后立即就能收到对方的响应。在这种情况下，TCP 就可以使用推送（push）操作。这时，发送方 TCP 把 PSH 置为 1，并立即创建一个报文段发送出去。接收方 TCP 收到 PSH=1 的报文段，就尽快地交付接收应用进程。   
`复位RST（ReSeT）`           
当 RST=1 时，表名 TCP 连接中出现了严重错误（如由于主机崩溃或其他原因），必须释放连接，然后再重新建立传输连接。RST 置为 1 用来拒绝一个非法的报文段或拒绝打开一个连接。  
`同步SYN（SYNchronization）`         
在连接建立时用来同步序号。当 SYN=1 而 ACK=0 时，表明这是一个连接请求报文段。对方若同意建立连接，则应在响应的报文段中使 SYN=1 和 ACK=1，因此 SYN 置为 1 就表示这是一个连接请求或连接接受报文。  
`终止FIN（FINis，意思是“完”“终”）`    
用来释放一个连接。当 FIN=1 时，表明此报文段的发送发的数据已发送完毕，并要求释放运输连接。  

7. 窗口   
占 2 字节，窗口值是[0，2^16-1]之间的整数。窗口指的是发送本报文段的一方的接受窗口（而不是自己的发送窗口）。窗口值告诉对方：从本报文段首部中的确认号算起，接收方目前允许对方发送的数据量（以字节为单位）。之所以要有这个限制，是因为接收方的数据缓存空间是有限的。总之，窗口值作为接收方让发送方设置其发送窗口的依据，作为流量控制的依据，后面会详细介绍。  
总之：窗口字段明确指出了现在允许对方发送的数据量。窗口值经常在动态变化。

8. 检验和         
占 2 字节，检验和字段检验的范围包括首部和数据这两部分。和 UDP 用户数据报一样，在计算检验和时，要在 TCP 报文段的前面加上 12 字节的伪首部。伪首部的格式和 UDP 用户数据报的伪首部一样。但应把伪首部第 4 个字段中的 17 改为 6（TCP 的协议号是 6）；把第 5 字段中的 UDP 中的长度改为 TCP 长度。接收方收到此报文段后，仍要加上这个伪首部来计算检验和。若使用 IPv6，则相应的伪首部也要改变。

9. 紧急指针   
占 2 字节，紧急指针仅在 URG=1 时才有意义，它指出本报文段中的紧急数据的字节数（紧急数据结束后就是普通数据) 。因此，在紧急指针指出了紧急数据的末尾在报文段中的位置。当所有紧急数据都处理完时，TCP 就告诉应用程序恢复到正常操作。值得注意的是，即使窗口为 0 时也可以发送紧急数据。

10. 选项         
选项长度可变，最长可达 40 字节。当没有使用“选项”时，TCP 的首部长度是 20 字节。TCP 首部总长度由 TCP 头中的“数据偏移”字段决定，前面说了，最长偏移为 60 字节。那么“tcp 选项”的长度最大为 60-20=40 字节。

### 2.4 tcp 选项
选项的一般结构

```txt
  1byte    1byte        nbytes
+--------+--------+------------------+ 
| Kind   | Length |       Info       |
+--------+--------+------------------+ 
```
TCP 最初只规定了一种选项，即最大报文段长度 MSS（Maximum Segment Szie）。后来又增加了几个选项如窗口扩大选项、时间戳选项等，下面说明常用的选项。

kind=0 是选项表结束选项。  

kind=1 是空操作（nop）选项  
没有特殊含义，一般用于将 TCP 选项的总长度填充为 4 字节的整数倍，为啥需要 4 字节整数倍？因为前面讲了数据偏移字段的单位是 4 个字节。

kind=2 是最大报文段长度选项  
TCP 连接初始化时，通信双方使用该选项来协商最大报文段长度（Max Segment Size，MSS）。TCP 模块通常将 MSS 设置为（MTU-40）字节（减掉的这 40 字节包括 20 字节的 TCP 头部和 20 字节的 IP 头部）。这样携带 TCP 报文段的 IP 数据报的长度就不会超过 MTU（假设 TCP 头部和 IP 头部都不包含选项字段，并且这也是一般情况），从而避免本机发生 IP 分片。对以太网而言，MSS 值是 1460（1500-40）字节。

kind=3 是窗口扩大因子选项   
TCP 连接初始化时，通信双方使用该选项来协商接收通告窗口的扩大因子。在 TCP 的头部中，接收通告窗口大小是用 16 位表示的，故最大为 65535 字节，但实际上 TCP 模块允许的接收通告窗口大小远不止这个数（为了提高 TCP 通信的吞吐量）。窗口扩大因子解决了这个问题。假设 TCP 头部中的接收通告窗口大小是 N，窗口扩大因子（移位数）是 M，那么 TCP 报文段的实际接收通告窗口大小是 N 乘 2M，或者说 N 左移 M 位。注意，M 的取值范围是 0～14。

和 MSS 选项一样，窗口扩大因子选项只能出现在同步报文段中，否则将被忽略。但同步报文段本身不执行窗口扩大操作，即同步报文段头部的接收通告窗口大小就是该 TCP 报文段的实际接收通告窗口大小。当连接建立好之后，每个数据传输方向的窗口扩大因子就固定不变了。关于窗口扩大因子选项的细节，可参考标准文档 RFC 1323。

kind=4 是选择性确认（Selective Acknowledgment，SACK）选项    
TCP 通信时，如果某个 TCP 报文段丢失，则 TCP 模块会重传最后被确认的 TCP 报文段后续的所有报文段，这样原先已经正确传输的 TCP 报文段也可能重复发送，从而降低了 TCP 性能。SACK 技术正是为改善这种情况而产生的，它使 TCP 模块只重新发送丢失的 TCP 报文段，不用发送所有未被确认的 TCP 报文段。选择性确认选项用在连接初始化时，表示是否支持 SACK 技术。

kind=5 是 SACK 实际工作的选项  
该选项的参数告诉发送方本端已经收到并缓存的不连续的数据块，从而让发送端可以据此检查并重发丢失的数据块。每个块边沿（edge of block）参数包含一个 4 字节的序号。其中块左边沿表示不连续块的第一个数据的序号，而块右边沿则表示不连续块的最后一个数据的序号的下一个序号。这样一对参数（块左边沿和块右边沿）之间的数据是没有收到的。因为一个块信息占用 8 字节，所以 TCP 头部选项中实际上最多可以包含 4 个这样的不连续数据块（考虑选项类型和长度占用的 2 字节）。

kind=8 是时间戳选项  
该选项提供了较为准确的计算通信双方之间的回路时间（Round Trip Time，RTT）的方法，从而为 TCP 流量控制提供重要信息。  

### 2.5 tcp 的基本概念
为了更利于读者理解，这里整理和复习一下 tcp 的基本概念。

#### 2.5.1 tcp 段
tcp 段本质上就是 tcp 的首部加上 tcp 负载，tcp 发送的报文都是 tcp 段，正因为 tcp 段的存在，tcp 才能被称为流式的通道。

#### 2.5.2 MSS
上面说 tcp 选项的时候说了，MSS（Maximum Segment Szie），即最大报文段长度，这里说段长度很容易引起歧义，个人觉得应该叫最大负载长度，因为 MSS 的真正定义是 TCP 报文段长度减去 TCP 首部长度。当用户层发送的数据长度超过 mss 的时候，tcp 会把数据进行分段，切成多个 tcp 段，每个 tcp 段的负载不超过 mss。

#### 2.5.3 序号系统
tcp 的头部采用序号和确认号的两个字段，这两个字段指的是字节序号，而不是段序号，这两个概念要区分开来。上面我们说了 tcp 发送数据都是一段一段的，对于每个段的编号就是段序号，而每个段里面带有多个字节，对这些字节的编号就是字节序号，tcp 里只有字节序号，而没有段序号，但有些协议采用的是段序号，比如：
kcp。

#### 2.5.4 ISN
单方向的起始序列号，第一次握手的时候随机生成。

#### 2.5.5 SACK
选择确认选项（Selective Acknowledgements）

## 三、tcp 状态详解

#### 3.1 tcp 的连接管理及连接状态

tcp 把连接作为最基本的抽象，它是一个虚拟的逻辑通道，这个通道很像哑铃，它有两个端点，那这两个端点具体是什么呢？不是主机，不是主机的 IP 地址，也不是应用程序，tcp 的端点叫做套接字（socket），它由 "ip:port" 构成，即 ip 和端口用冒号连接而成。每一条 tcp 连接都唯一的被两个端点所确定，也就是被两个套接字所确定，最终也就是 (ip1:port1 <-> ip2:port2) 所确定。

`tcp连接 = (socket1 <-> sokcet2) = ("ip1:port1" <-> "ip2:port2")`

注意: 这里 socket 不是指我们平常编写网络代码时调用的的 socket 接口。

既然 TCP 是面向连接的，那么每次通信中必定有建立和释放的过程，因为连接建立好以后就可以进行数据的通信，那么对于一个 tcp 连接的生命周期，我们可以分为三段：
建立连接、数据传输和释放连接。连接管理主要是管理建立连接和释放连接，保证这些过程能够正常工作。

### 3.2 tcp 的连接状态机

![state](https://doc.shiyanlou.com/document-uid949121labid10418timestamp1555574004123.png/wm)
上面是 tcp 的状态转换图，特意画成了从上到下按时序交互，方便理解。每个状态的详细含义如下：  

- CLOSED
表示 TCP 连接是“关闭着的”或“未打开的”。

-  LISTEN
表示服务器端的某个 SOCKET 处于监听状态，可以接受客户端的连接。

-  SYN_RCVD
表示服务器接收到了来自客户端请求连接的 SYN 报文。在正常情况下，这个状态是服务器端的 SOCKET 在建立 TCP 连接时的三次握手会话过程中的一个中间状态，很短暂，基本上用 netstat 很难看到这种状态，除非故意写一个监测程序，将三次 TCP 握手过程中最后一个 ACK 报文不予发送。当 TCP 连接处于此状态时，再收到客户端的 ACK 报文，它就会进入到 ESTABLISHED 状态。

-  SYN_SENT
这个状态与 SYN_RCVD 状态相呼应，当客户端 SOCKET 执行 connect()进行连接时，它首先发送 SYN 报文，然后随即进入到 SYN_SENT 状态，并等待服务端的发送三次握手中的第 2 个报文。SYN_SENT 状态表示客户端已发送 SYN 报文。

-  ESTABLISHED
表示 TCP 连接已经成功建立。

-  FIN_WAIT_1

这个状态得好好解释一下，其实 FIN_WAIT_1 和 FIN_WAIT_2 两种状态的真正含义都是表示等待对方的 FIN 报文。而这两种状态的区别是：FIN_WAIT_1 状态实际上是当 SOCKET 在 ESTABLISHED 状态时，它想主动关闭连接，向对方发送了 FIN 报文，此时该 SOCKET 进入到 FIN_WAIT_1 状态。而当对方回应 ACK 报文后，则进入到 FIN_WAIT_2 状态。当然在实际的正常情况下，无论对方处于任何种情况下，都应该马上回应 ACK 报文，所以 FIN_WAIT_1 状态一般是比较难见到的，而 FIN_WAIT_2 状态有时仍可以用 netstat 看到。

-  FIN_WAIT_2
上面已经解释了这种状态的由来，实际上 FIN_WAIT_2 状态下的 SOCKET 表示半连接，即有一方调用 close()主动要求关闭连接。注意：FIN_WAIT_2 是没有超时的（不像 TIME_WAIT 状态），这种状态下如果对方不关闭（不配合完成 4 次挥手过程），那这个 FIN_WAIT_2 状态将一直保持到系统重启，越来越多的 FIN_WAIT_2 状态会导致内核 crash。

-  TIME_WAIT
表示收到了对方的 FIN 报文，并发送出了 ACK 报文。 TIME_WAIT 状态下的 TCP 连接会等待 2*MSL（Max Segment Lifetime，最大分段生存期，指一个 TCP 报文在 Internet 上的最长生存时间。每个具体的 TCP 协议实现都必须选择一个确定的 MSL 值，RFC 1122 建议是 2 分钟，但 BSD 传统实现采用了 30 秒，Linux 可以 cat /proc/sys/net/ipv4/tcp_fin_timeout 看到本机的这个值），然后即可回到 CLOSED 可用状态了。如果 FIN_WAIT_1 状态下，收到了对方同时带 FIN 标志和 ACK 标志的报文时，可以直接进入到 TIME_WAIT 状态，而无须经过 FIN_WAIT_2 状态。（这种情况应该就是四次挥手变成三次挥手的那种情况）

-  CLOSING
这种状态在实际情况中应该很少见，属于一种比较罕见的例外状态。正常情况下，当一方发送 FIN 报文后，按理来说是应该先收到（或同时收到）对方的 ACK 报文，再收到对方的 FIN 报文。但是 CLOSING 状态表示一方发送 FIN 报文后，并没有收到对方的 ACK 报文，反而却也收到了对方的 FIN 报文。什么情况下会出现此种情况呢？那就是当双方几乎在同时 close() 一个 SOCKET 的话，就出现了双方同时发送 FIN 报文的情况，这是就会出现 CLOSING 状态，表示双方都正在关闭 SOCKET 连接。

-  CLOSE_WAIT
表示正在等待关闭。怎么理解呢？当对方 close()一个 SOCKET 后发送 FIN 报文给自己，你的系统毫无疑问地将会回应一个 ACK 报文给对方，此时 TCP 连接则进入到 CLOSE_WAIT 状态。接下来呢，你需要检查自己是否还有数据要发送给对方，如果没有的话，那你也就可以 close()这个 SOCKET 并发送 FIN 报文给对方，即关闭自己到对方这个方向的连接。有数据的话则看程序的策略，继续发送或丢弃。简单地说，当你处于 CLOSE_WAIT 状态下，需要完成的事情是等待你去关闭连接。

-  LAST_ACK
当被动关闭的一方在发送 FIN 报文后，等待对方的 ACK 报文的时候，就处于 LAST_ACK 状态。当收到对方的 ACK 报文后，也就可以进入到 CLOSED 可用状态了。

### 3.3 tcp 连接的建立

![connection](https://doc.shiyanlou.com/document-uid949121labid10418timestamp1555574034117.png/wm)  
上面的图片显示了 tcp 的三次握手，但只是简单的降了三次报文的交互，下面讲讲详细的三次握手。先讲三次握手的正常情况，接着我们再讲异常情况。

- 正常情况（没有丢包）
1. 主机 A 的 TCP 向主机 B 的 TCP 发出连接请求，发送 syn 报文段在发送 syn 之前，设置握手状态为 SynSent，还需要做一些准备工作，包括：随机生成 ISN1、计算 MSS、计算接收窗口扩展因子、是否开启 sack。 根据这些参数生成 syn 报文的选项参数，附在 tcp 选项中，然后发送带着这些选项的 syn 报文。

2. 主机 B 的 TCP 收到连接请求 syn 报文段后，需要回复 syn+ack 报文因为 tcp 的控制报文需要消耗一个字节的序列号，所以回复的 ack 序列号为 ISN1+1，设置接收窗口，设置握手状态为 SynRcvd，并随机生成 ISN2、计算 MSS、计算接收窗口扩展因子、是否开启 sack。根据这些参数生成 syn+ack 报文的选项参数，附在 tcp 选项中，回复给主机 A。

3. 主机 A 的 TCP 收到 syn+ack 报文段后，还要向 B 回复确认和上面一样，tcp 的控制报文需要消耗一个字节的序列号，所以回复的 ack 序列号为 ISN2+1，发送 ack 报文给主机 B。

4. 主机 A 的 TCP 通知上层应用进程，连接已经建立，可以发送数据了，当主机 B 的 TCP 收到主机 A 的确认后，也通知上层应用进程，连接建立。

- 异常情况（有丢包）  
1. 主机 A 发给主机 B 的 SYN 中途丢失，没有到达主机 B  因为在发送 syn 之前，就设置了超时定时器，如果在一定的时间内没收到回复，就会触发重传，所以主机 A 会周期性超时重传，直到收到主机 B 的确认。重传的周期，一开始默认 1s，每重传一次，变为原来的 2 倍，如果重传周期超过 1 分钟，返回错误，不再尝试重连。

2. 主机 B 发给主机 A 的 SYN +ACK 中途丢失，没有到达主机 A  主机 B 会周期性超时重传，直到收到主机 A 的确认，重传的策略和 syn 报文一样，每重传一次，周期变为原来的 2 倍。

3. 主机 A 发给主机 B 的 ACK 中途被丢，没有到达主机 B  主机 A 发完 ACK，单方面认为 TCP 为 Established 状态，而 B 显然认为 TCP 为 Active 状态：  
a. 如果此时双方都没有数据发送，主机 B 会周期性超时重传，直到收到 A 的确认，收到之后主机 B 的 TCP 连接也为 Established 状态，双向可以发包。  
b. 如果此时 A 有数据发送，主机 B 收到主机 A 的 Data + ACK，自然会切换为 established 状态，并接受主机 A 的 Data。  

### 3.4 tcp 建立的源码分析
```go
// Connect connects the endpoint to its peer.
// 和地址为 addr 的远端建立tcp连接
func (e *endpoint) Connect(addr tcpip.FullAddress) *tcpip.Error {
	return e.connect(addr, true, true)
}

// connect将端点连接到其对等端。在正常的非S/R情况下，新连接应该运行主goroutine并执行握手。
// 在恢复先前连接的端点时，将被动地创建两端（因此不会进行新的握手）;对于应用程序尚未接受的堆栈接受连接，
// 它们将在不运行主goroutine的情况下进行恢复。
func (e *endpoint) connect(addr tcpip.FullAddress, handshake bool, run bool) (err *tcpip.Error) {
	e.mu.Lock()
	defer e.mu.Unlock()
	...

	connectingAddr := addr.Addr

	// 检查ipv4是否映射到ipv6
	netProto, err := e.checkV4Mapped(&addr)
	if err != nil {
		return err
	}

        ...
	// Find a route to the desired destination.
	// 根据目标ip查找路由信息
	r, err := e.stack.FindRoute(nicid, e.id.LocalAddress, addr.Addr, netProto)
	if err != nil {
		return err
	}
	defer r.Release()

	origID := e.id

	netProtos := []tcpip.NetworkProtocolNumber{netProto}
	e.id.LocalAddress = r.LocalAddress
	e.id.RemoteAddress = r.RemoteAddress
	e.id.RemotePort = addr.Port

	if e.id.LocalPort != 0 {
		// 记录和检查原端口是否已被使用
		// The endpoint is bound to a port, attempt to register it.
		err := e.stack.RegisterTransportEndpoint(nicid, netProtos, ProtocolNumber, e.id, e)
		if err != nil {
			return err
		}
	} else {
		// The endpoint doesn't have a local port yet, so try to get
		// one. Make sure that it isn't one that will result in the same
		// address/port for both local and remote (otherwise this
		// endpoint would be trying to connect to itself).
		// 端点还没有本地端口，所以尝试获取一个端口。确保它不会导致本地和远程的相同地址/端口（否则此端点将尝试连接到自身）
		// 远端地址和本地地址是否相同
		sameAddr := e.id.LocalAddress == e.id.RemoteAddress
		// 随机分配一个本地端口 p
		if _, err := e.stack.PickEphemeralPort(func(p uint16) (bool, *tcpip.Error) {
			if sameAddr && p == e.id.RemotePort {
				return false, nil
			}
			if !e.stack.IsPortAvailable(netProtos, ProtocolNumber, e.id.LocalAddress, p) {
				return false, nil
			}

			id := e.id
			id.LocalPort = p
			switch e.stack.RegisterTransportEndpoint(nicid, netProtos, ProtocolNumber, id, e) {
			case nil:
				e.id = id
				return true, nil
			case tcpip.ErrPortInUse:
				return false, nil
			default:
				return false, err
			}
		}); err != nil {
			return err
		}
	}

	...

	// 记录该端点的参数
	e.isRegistered = true
	e.state = stateConnecting
	e.route = r.Clone()
	e.boundNICID = nicid
	e.effectiveNetProtos = netProtos
	e.connectingAddress = connectingAddr

	...

	if run {
		e.workerRunning = true
		e.stack.Stats().TCP.ActiveConnectionOpenings.Increment()
		// tcp的主函数
		go e.protocolMainLoop(handshake)
	}

	return tcpip.ErrConnectStarted
}

// tcp三次握手时候使用的对象
type handshake struct {
	ep *endpoint
	// 握手的状态
	state  handshakeState
	active bool
	flags  uint8
	ackNum seqnum.Value

	// iss is the initial send sequence number, as defined in RFC 793.
	// 初始序列号
	iss seqnum.Value

	// rcvWnd is the receive window, as defined in RFC 793.
	// 接收窗口
	rcvWnd seqnum.Size

	// sndWnd is the send window, as defined in RFC 793.
	// 发送窗口
	sndWnd seqnum.Size

	// mss is the maximum segment size received from the peer.
	// 最大报文段大小
	mss uint16

	// sndWndScale is the send window scale, as defined in RFC 1323. A
	// negative value means no scaling is supported by the peer.
	// 发送窗口扩展因子
	sndWndScale int

	// rcvWndScale is the receive window scale, as defined in RFC 1323.
	// 接收窗口扩展因子
	rcvWndScale int
}

func newHandshake(ep *endpoint, rcvWnd seqnum.Size) (handshake, *tcpip.Error) {
	h := handshake{
		ep:          ep,
		active:      true,
		rcvWnd:      rcvWnd,
		rcvWndScale: FindWndScale(rcvWnd), // 接收窗口扩展因子
	}
	if err := h.resetState(); err != nil {
		return handshake{}, err
	}

	return h, nil
}

// protocolMainLoop 是TCP协议的主循环。它在自己的goroutine中运行，负责握手、发送段和处理收到的段。
func (e *endpoint) protocolMainLoop(handshake bool) *tcpip.Error {
	var closeTimer *time.Timer
	var closeWaker sleep.Waker

	...
	// 如果需要三次握手
	if handshake {
		// This is an active connection, so we must initiate the 3-way
		// handshake, and then inform potential waiters about its
		// completion.
		h, err := newHandshake(e, seqnum.Size(e.receiveBufferAvailable()))
		if err == nil {
			// 执行握手
			err = h.execute()
		}
		// 处理握手有错
		if err != nil {
			e.lastErrorMu.Lock()
			e.lastError = err
			e.lastErrorMu.Unlock()

			e.mu.Lock()
			e.state = stateError
			e.hardError = err
			// Lock released below.
			epilogue()

			return err
		}

		// Transfer handshake state to TCP connection. We disable
		// receive window scaling if the peer doesn't support it
		// (indicated by a negative send window scale).
		// 到这里就表示三次握手已经成功了，那么初始化发送器和接收器
		e.snd = newSender(e, h.iss, h.ackNum-1, h.sndWnd, h.mss, h.sndWndScale)

		e.rcvListMu.Lock()
		e.rcv = newReceiver(e, h.ackNum-1, h.rcvWnd, h.effectiveRcvWndScale())
		e.rcvListMu.Unlock()
	}

	...
}

/*
			c				s
			|				|
   sync_sent|------sync---->|sync_rcvd
			|				|
			|				|
 established|<--sync|ack----|
			|				|
			|				|
			|------ack----->|established
*/
// 执行tcp 3次握手，客户端和服务端都是调用该函数来实现三次握手
func (h *handshake) execute() *tcpip.Error {
	if h.ep.route.IsResolutionRequired() {
		if err := h.resolveRoute(); err != nil {
			return err
		}
	}

	// Initialize the resend timer.
	// 初始化重传定时器
	resendWaker := sleep.Waker{}
	// 设置1s超时
	timeOut := time.Duration(time.Second)
	rt := time.AfterFunc(timeOut, func() {
		resendWaker.Assert()
	})
	defer rt.Stop()

	// Set up the wakers.
	s := sleep.Sleeper{}
	s.AddWaker(&resendWaker, wakerForResend)
	s.AddWaker(&h.ep.notificationWaker, wakerForNotification)
	s.AddWaker(&h.ep.newSegmentWaker, wakerForNewSegment)
	defer s.Done()

	var sackEnabled SACKEnabled
	// 是否开启 sack
	if err := h.ep.stack.TransportProtocolOption(ProtocolNumber, &sackEnabled); err != nil {
		// If stack returned an error when checking for SACKEnabled
		// status then just default to switching off SACK negotiation.
		sackEnabled = false
	}

	// Send the initial SYN segment and loop until the handshake is
	// completed.
	// sync报文的选项参数
	synOpts := header.TCPSynOptions{
		WS:            h.rcvWndScale,
		TS:            true,
		TSVal:         h.ep.timestamp(),
		TSEcr:         h.ep.recentTS,
		SACKPermitted: bool(sackEnabled),
	}

	// Execute is also called in a listen context so we want to make sure we
	// only send the TS/SACK option when we received the TS/SACK in the
	// initial SYN.
	// 表示服务端收到了syn报文
	if h.state == handshakeSynRcvd {
		synOpts.TS = h.ep.sendTSOk
		synOpts.SACKPermitted = h.ep.sackPermitted && bool(sackEnabled)
	}
	// 如果是客户端发送 syn 报文，如果是服务端发送 syn+ack 报文
	sendSynTCP(&h.ep.route, h.ep.id, h.flags, h.iss, h.ackNum, h.rcvWnd, synOpts)
	// 判断握手是否结束，没有结束则循环
	for h.state != handshakeCompleted {
		// 获取事件id
		switch index, _ := s.Fetch(true); index {
		case wakerForResend:
			// 如果是客户端当发送 syn 报文，超过一定的时间未收到回包，触发超时重传
			// 如果是服务端当发送 syn+ack 报文，超过一定的时间未收到 ack 回包，触发超时重传
			// 超时时间变为上次的2倍
			timeOut *= 2
			if timeOut > 60*time.Second {
				return tcpip.ErrTimeout
			}
			rt.Reset(timeOut)
			// 重新发送syn报文
			sendSynTCP(&h.ep.route, h.ep.id, h.flags, h.iss, h.ackNum, h.rcvWnd, synOpts)

		...

		case wakerForNewSegment:
			// 处理握手报文
			if err := h.processSegments(); err != nil {
				return err
			}
		}
	}

	return nil
}

// synSentState handles a segment received when the TCP 3-way handshake is in
// the SYN-SENT state.
// synSentState 是客户端或者服务端接收到第一个握手报文的处理
// 正常情况下，如果是客户端，此时应该收到 syn+ack 报文，处理后发送 ack 报文给服务端。
// 如果是服务端，此时接收到syn报文，那么应该回复 syn+ack 报文给客户端，并设置状态为 handshakeSynRcvd。
func (h *handshake) synSentState(s *segment) *tcpip.Error {
	// RFC 793, page 37, states that in the SYN-SENT state, a reset is
	// acceptable if the ack field acknowledges the SYN.
	if s.flagIsSet(flagRst) {
		if s.flagIsSet(flagAck) && s.ackNumber == h.iss+1 {
			return tcpip.ErrConnectionRefused
		}
		return nil
	}

	if !h.checkAck(s) {
		return nil
	}

	// We are in the SYN-SENT state. We only care about segments that have
	// the SYN flag.
	if !s.flagIsSet(flagSyn) {
		return nil
	}

	// Parse the SYN options.
	rcvSynOpts := parseSynSegmentOptions(s)

	// Remember if the Timestamp option was negotiated.
	h.ep.maybeEnableTimestamp(&rcvSynOpts)

	// Remember if the SACKPermitted option was negotiated.
	h.ep.maybeEnableSACKPermitted(&rcvSynOpts)

	// Remember the sequence we'll ack from now on.
	h.ackNum = s.sequenceNumber + 1
	h.flags |= flagAck
	h.mss = rcvSynOpts.MSS
	h.sndWndScale = rcvSynOpts.WS

	// If this is a SYN ACK response, we only need to acknowledge the SYN
	// and the handshake is completed.
	// 客户端接收到了 syn+ack报文
	if s.flagIsSet(flagAck) {
		// 客户端握手完成，发送 ack 报文给服务端
		h.state = handshakeCompleted
		// 最后依次 ack 报文丢了也没关系，因为后面一但发送任何数据包都是带ack的
		h.ep.sendRaw(buffer.VectorisedView{}, flagAck, h.iss+1, h.ackNum, h.rcvWnd>>h.effectiveRcvWndScale())
		return nil
	}

	// A SYN segment was received, but no ACK in it. We acknowledge the SYN
	// but resend our own SYN and wait for it to be acknowledged in the
	// SYN-RCVD state.
	// 服务端收到了 syn 报文，应该回复客户端 syn+ack 报文，且设置状态为 handshakeSynRcvd
	h.state = handshakeSynRcvd
	synOpts := header.TCPSynOptions{
		WS:    h.rcvWndScale,
		TS:    rcvSynOpts.TS,
		TSVal: h.ep.timestamp(),
		TSEcr: h.ep.recentTS,

		// We only send SACKPermitted if the other side indicated it
		// permits SACK. This is not explicitly defined in the RFC but
		// this is the behaviour implemented by Linux.
		SACKPermitted: rcvSynOpts.SACKPermitted,
	}
	// 发送 syn+ack 报文，如果该报文在链路中丢了，没有关系，客户端会重新发送 syn 报文
	sendSynTCP(&s.route, h.ep.id, h.flags, h.iss, h.ackNum, h.rcvWnd, synOpts)

	return nil
}

// synRcvdState handles a segment received when the TCP 3-way handshake is in
// the SYN-RCVD state.
// 正常情况下，会调用该函数来处理第三次 ack 报文
func (h *handshake) synRcvdState(s *segment) *tcpip.Error {
	if s.flagIsSet(flagRst) {
		// RFC 793, page 37, states that in the SYN-RCVD state, a reset
		// is acceptable if the sequence number is in the window.
		if s.sequenceNumber.InWindow(h.ackNum, h.rcvWnd) {
			return tcpip.ErrConnectionRefused
		}
		return nil
	}

	if !h.checkAck(s) {
		return nil
	}

	// 如果是syn报文，且序列号对应不上，那么返回 rst
	if s.flagIsSet(flagSyn) && s.sequenceNumber != h.ackNum-1 {
		// We received two SYN segments with different sequence
		// numbers, so we reset this and restart the whole
		// process, except that we don't reset the timer.
		ack := s.sequenceNumber.Add(s.logicalLen())
		seq := seqnum.Value(0)
		if s.flagIsSet(flagAck) {
			seq = s.ackNumber
		}
		h.ep.sendRaw(buffer.VectorisedView{}, flagRst|flagAck, seq, ack, 0)

		if !h.active {
			return tcpip.ErrInvalidEndpointState
		}

		if err := h.resetState(); err != nil {
			return err
		}
		synOpts := header.TCPSynOptions{
			WS:            h.rcvWndScale,
			TS:            h.ep.sendTSOk,
			TSVal:         h.ep.timestamp(),
			TSEcr:         h.ep.recentTS,
			SACKPermitted: h.ep.sackPermitted,
		}
		sendSynTCP(&s.route, h.ep.id, h.flags, h.iss, h.ackNum, h.rcvWnd, synOpts)
		return nil
	}

	// We have previously received (and acknowledged) the peer's SYN. If the
	// peer acknowledges our SYN, the handshake is completed.
	// 如果是 ack 报文，表明三次握手已经完成
	if s.flagIsSet(flagAck) {

		// If the timestamp option is negotiated and the segment does
		// not carry a timestamp option then the segment must be dropped
		// as per https://tools.ietf.org/html/rfc7323#section-3.2.
		if h.ep.sendTSOk && !s.parsedOptions.TS {
			h.ep.stack.Stats().DroppedPackets.Increment()
			return nil
		}

		// Update timestamp if required. See RFC7323, section-4.3.
		h.ep.updateRecentTimestamp(s.parsedOptions.TSVal, h.ackNum, s.sequenceNumber)
                // 标记三次握手已经完成
		h.state = handshakeCompleted
		return nil
	}

	return nil
}

// 握手的时候处理tcp段
func (h *handshake) handleSegment(s *segment) *tcpip.Error {
	h.sndWnd = s.window
	if !s.flagIsSet(flagSyn) && h.sndWndScale > 0 {
		h.sndWnd <<= uint8(h.sndWndScale)
	}

	switch h.state {
	case handshakeSynRcvd:
		// 正常情况下，服务端接收客户端第三次 ack 报文
		return h.synRcvdState(s)
	case handshakeSynSent:
		// 客户端发送了syn报文后的处理
		return h.synSentState(s)
	}
	return nil
}

// processSegments goes through the segment queue and processes up to
// maxSegmentsPerWake (if they're available).
func (h *handshake) processSegments() *tcpip.Error {
	for i := 0; i < maxSegmentsPerWake; i++ {
		// 从队列中取出一个tcp段
		s := h.ep.segmentQueue.dequeue()
		if s == nil {
			return nil
		}

		// 处理tcp段
		err := h.handleSegment(s)
		s.decRef()
		if err != nil {
			return err
		}

		// We stop processing packets once the handshake is completed,
		// otherwise we may process packets meant to be processed by
		// the main protocol goroutine.
		if h.state == handshakeCompleted {
			break
		}
	}

	// If the queue is not empty, make sure we'll wake up in the next
	// iteration.
	if !h.ep.segmentQueue.empty() {
		h.ep.newSegmentWaker.Assert()
	}

	return nil
}

```
### 3.5 tcp 连接的释放

![close](https://doc.shiyanlou.com/document-uid949121labid10418timestamp1555574060171.png/wm)  

1）数据传输结束后，主机 A 的应用进程调用 Close 函数，先向其 TCP 发出释放连接请求，不再发送数据。TCP 通知对方要释放从主机 A 到主机 B 的连接，将发往主机 B 的 TCP 报文段首部的终止比特 FIN 置为 1，序号 seq1 等于已传送数据的最后一个字节的序号加 1。

2）主机 B 的 TCP 收到释放连接通知后发出确认，其序号为 seq1+1，同时通知应用进程，这样主机 A 到主机 B 的连接就释放了，连接处于半关闭状态。主机 B 不在接受主机 A 发来的数据；但主机 B 还向 A 发送数据，主机 A 若正确接收数据仍需要发送确认。

3）在主机 B 向主机 A 的数据发送结束后，其应用进程应该主动调用 Close 函数，释放 TCP 连接。主机 B 发出的连接释放报文段必须将终止比特置为 1，并使其序号 seq2 等于前面已经传送过的数据的最后一个字节的序号加 1，还必须回复 ACK=seq1+1。

4）主机 A 对主机 B 的连接释放报文段发出确认，将 ACK 置为 1，ACK=seq2+1, seq=seq1+1。这样才把从 B 到 A 的反方向连接释放掉，主机 A 的 TCP 再向其应用进程报告，整个连接已经全部释放。

还有一个要注意的是，fin 包和数据包一样，如果丢失了，会进行重传，实际上可能是是 fin 丢失或 ack 丢失。重传的周期由 rto 决定。
### 3.6 tcp 连接的释放的源码分析
```go
// Shutdown closes the read and/or write end of the endpoint connection to its
// peer.
// Shutdown 表示关闭该连接的读写。
func (e *endpoint) Shutdown(flags tcpip.ShutdownFlags) *tcpip.Error {
	e.mu.Lock()
	defer e.mu.Unlock()
	e.shutdownFlags |= flags

	switch e.state {
	case stateConnected:
		// Close for write.
		// 不能直接关闭读数据包，因为关闭连接的时候四次挥手还需要读取报文。
		if (e.shutdownFlags & tcpip.ShutdownWrite) != 0 {
			if (e.shutdownFlags & tcpip.ShutdownRead) != 0 {
				// We're fully closed, if we have unread data we need to abort
				// the connection with a RST.
				// 完全关闭连接，也就是读写都关闭
				e.rcvListMu.Lock()
				rcvBufUsed := e.rcvBufUsed
				e.rcvListMu.Unlock()

				if rcvBufUsed > 0 {
					// 如果接收队列还有数据，那么通知对端reset
					e.notifyProtocolGoroutine(notifyReset)
					return nil
				}
			}

			e.sndBufMu.Lock()

			if e.sndClosed {
				// Already closed.
				e.sndBufMu.Unlock()
				break
			}

			// Queue fin segment.
			s := newSegmentFromView(&e.route, e.id, nil)
			e.sndQueue.PushBack(s)
			e.sndBufInQueue++

			// Mark endpoint as closed.
			e.sndClosed = true

			e.sndBufMu.Unlock()

			// Tell protocol goroutine to close.
			// 触发调用 handleClose
			e.sndCloseWaker.Assert()
		}

	case stateListen:
		// Tell protocolListenLoop to stop.
		if flags&tcpip.ShutdownRead != 0 {
			e.notifyProtocolGoroutine(notifyClose)
		}

	default:
		return tcpip.ErrNotConnected
	}

	return nil
}

// 关闭连接的处理，最终会调用 sendData 来发送 fin 包
func (e *endpoint) handleClose() *tcpip.Error {
	// Drain the send queue.
	e.handleWrite()

	// Mark send side as closed.
	// 标记发送器关闭
	e.snd.closed = true

	return nil
}


// 发送数据段，最终掉用 sendSegment 来发送，这里只看发送 fin 报文
func (s *sender) sendData() {
        ...
        
	var seg *segment
	end := s.sndUna.Add(s.sndWnd)
	var dataSent bool
	// 便利发送链表，发送数据
	// tcp流量控制：s.outstanding < s.sndCwnd 判断正在发送的数据量不能超过发送窗口，流量控制。
	for seg = s.writeNext; seg != nil && s.outstanding < s.sndCwnd; seg = seg.Next() {
		...

		var segEnd seqnum.Value
		if seg.data.Size() == 0 { // 数据段没有负载，表示要结束连接
			if s.writeList.Back() != seg {
				panic("FIN segments must be the final segment in the write list.")
			}
			// 发送 fin 报文
                        seg.flags = flagAck | flagFin
                        // fin 报文需要确认，且消耗一个字节序列号
			segEnd = seg.sequenceNumber.Add(1)
		} else {
			...
		}

		if !dataSent {
			dataSent = true
			// We are sending data, so we should stop the keepalive timer to
			// ensure that no keepalives are sent while there is pending data.
			s.ep.disableKeepaliveTimer()
		}
		s.sendSegment(seg.data, seg.flags, seg.sequenceNumber)

		// Update sndNxt if we actually sent new data (as opposed to
		// retransmitting some previously sent data).
		// 发送一个数据段后，更新sndNxt
		if s.sndNxt.LessThan(segEnd) {
			s.sndNxt = segEnd
		}
	}

        ...
        
	// Enable the timer if we have pending data and it's not enabled yet.
	// tcp的可靠性：如果重发定时器没有启动 且 snduna 不等于 sndNxt，启动定时器
	if !s.resendTimer.enabled() && s.sndUna != s.sndNxt {
		// 启动定时器，并且设定定时器的间隔为s.rto
		s.resendTimer.enable(s.rto)
        }
        
	...
}

// 从 handleSegments 接收到tcp段，然后进行处理，这里只看对 fin 包的处理。
func (r *receiver) handleRcvdSegment(s *segment) {
	...

	segLen := seqnum.Size(s.data.Size())
	segSeq := s.sequenceNumber

	...

	if !r.consumeSegment(s, segSeq, segLen) {
		// 如果有负载数据或者是 fin 报文，立即回复一个 ack 报文
		if segLen > 0 || s.flagIsSet(flagFin) {
			...

			// Immediately send an ack so that the peer knows it may
			// have to retransmit.
			r.ep.snd.sendAck()
		}
		return
	}

	...
}


// tcp可靠性：consumeSegment 尝试使用r接收tcp段。该数据段可能刚刚收到或可能已经收到，但尚未准备好被消费。
// 如果数据段被消耗则返回true，如果由于缺少段而无法消耗，则返回false。
func (r *receiver) consumeSegment(s *segment, segSeq seqnum.Value, segLen seqnum.Size) bool {
	...

	// 如果收到 fin 报文
	if s.flagIsSet(flagFin) {
		// 控制报文消耗一个字节的序列号，因此这边期望下次收到的序列号加1
		r.rcvNxt++

		// 收到 fin，立即回复 ack
		r.ep.snd.sendAck()

		// 标记接收器关闭
		// 触发上层应用可以读取
		r.closed = true
		r.ep.readyToRead(nil)

		...
	}

	return true
}

```
### 3.7 tcp 连接和断开的实验

这里采用换回网卡作为底层的网卡 IO，实现 tcp 客户端和服务端的连接。
```go
// +build linux

package main

import (
	"flag"
	"log"
	"net"
	"os"
	"strconv"
	"strings"
	"time"

	"netstack/tcpip"
	"netstack/tcpip/link/loopback"
	"netstack/tcpip/network/arp"
	"netstack/tcpip/network/ipv4"
	"netstack/tcpip/stack"
	"netstack/tcpip/transport/tcp"
	"netstack/waiter"
)

var mac = flag.String("mac", "aa:00:01:01:01:01", "mac address to use in tap device")

// cd state; go build
// ./state tap0 192.168.1.1 9000
func main() {
	flag.Parse()
	log.SetFlags(log.Lshortfile | log.LstdFlags)

	if len(os.Args) != 3 {
		log.Fatal("Usage: ", os.Args[0], "<ipv4-address> <port>")
	}

	addrName := os.Args[1]
	portName := os.Args[2]

	addr := tcpip.Address(net.ParseIP(addrName).To4())
	port, err := strconv.Atoi(portName)
	if err != nil {
		log.Fatalf("Unable to convert port %v: %v", portName, err)
	}

	s := newStack(addr, port)

	done := make(chan int, 1)
	go tcpServer(s, addr, port, done)
	<-done

	tcpClient(s, addr, port)
}

func newStack(addr tcpip.Address, port int) *stack.Stack {
	// 创建本地环回网卡
	linkID := loopback.New()

	// 新建相关协议的协议栈
	s := stack.New([]string{ipv4.ProtocolName, arp.ProtocolName},
		[]string{tcp.ProtocolName}, stack.Options{})

	// 新建抽象的网卡
	if err := s.CreateNamedNIC(1, "lo0", linkID); err != nil {
		log.Fatal(err)
	}

	// 在该网卡上添加和注册相应的网络层
	if err := s.AddAddress(1, ipv4.ProtocolNumber, addr); err != nil {
		log.Fatal(err)
	}

	// 在该协议栈上添加和注册ARP协议
	if err := s.AddAddress(1, arp.ProtocolNumber, arp.ProtocolAddress); err != nil {
		log.Fatal(err)
	}

	// 添加默认路由
	s.SetRouteTable([]tcpip.Route{
		{
			Destination: tcpip.Address(strings.Repeat("\x00", len(addr))),
			Mask:        tcpip.AddressMask(strings.Repeat("\x00", len(addr))),
			Gateway:     "",
			NIC:         1,
		},
	})

	return s
}

func tcpServer(s *stack.Stack, addr tcpip.Address, port int, done chan int) {
	var wq waiter.Queue
	// 新建一个TCP端
	ep, e := s.NewEndpoint(tcp.ProtocolNumber, ipv4.ProtocolNumber, &wq)
	if e != nil {
		log.Fatal(e)
	}

	// 绑定本地端口
	if err := ep.Bind(tcpip.FullAddress{0, "", uint16(port)}, nil); err != nil {
		log.Fatal("Bind failed: ", err)
	}

	// 监听tcp
	if err := ep.Listen(10); err != nil {
		log.Fatal("Listen failed: ", err)
	}

	// Wait for connections to appear.
	waitEntry, notifyCh := waiter.NewChannelEntry(nil)
	wq.EventRegister(&waitEntry, waiter.EventIn)
	defer wq.EventUnregister(&waitEntry)

	done <- 1

	for {
		n, _, err := ep.Accept()
		if err != nil {
			if err == tcpip.ErrWouldBlock {
				<-notifyCh
				continue
			}

			log.Fatal("Accept() failed:", err)
		}
		ra, err := n.GetRemoteAddress()
		log.Printf("new conn: %v %v", ra, err)
	}
}

func tcpClient(s *stack.Stack, addr tcpip.Address, port int) {
	remote := tcpip.FullAddress{
		Addr: addr,
		Port: uint16(port),
	}

	var wq waiter.Queue
	// 新建一个TCP端
	ep, e := s.NewEndpoint(tcp.ProtocolNumber, ipv4.ProtocolNumber, &wq)
	if e != nil {
		log.Fatal(e)
	}

	// Issue connect request and wait for it to complete.
	waitEntry, notifyCh := waiter.NewChannelEntry(nil)
	wq.EventRegister(&waitEntry, waiter.EventOut)
	terr := ep.Connect(remote)
	if terr == tcpip.ErrConnectStarted {
		log.Println("Connect is pending...")
		<-notifyCh
		terr = ep.GetSockOpt(tcpip.ErrorOption{})
	}
	wq.EventUnregister(&waitEntry)

	if terr != nil {
		log.Fatal("Unable to connect: ", terr)
	}

	log.Println("Connected")
	time.Sleep(1 * time.Second)

	ep.Close()
	log.Println("tcp disconected")

	time.Sleep(3 * time.Second)
}

```

### 3.8 实验结果

```txt
2019/03/31 19:23:31 ports.go:131: new transport: 6, port: 9000
2019/03/31 19:23:31 main.go:147: Connect is pending...
2019/03/31 19:23:31 connect.go:675: send tcp syn segment to 192.168.1.1:9000, seq: 517094415, ack: 0, rcvWnd: 65535
2019/03/31 19:23:31 ipv4.go:151: send ipv4 packet 60 bytes, proto: 0x6
2019/03/31 19:23:31 ipv4.go:193: recv ipv4 packet 60 bytes, proto: 0x6
2019/03/31 19:23:31 endpoint.go:1395: recv tcp syn segment from 192.168.1.1:26913, seq: 517094415, ack: 0
2019/03/31 19:23:31 connect.go:675: send tcp ack|syn segment to 192.168.1.1:26913, seq: 559240690, ack: 517094416, rcvWnd: 65535
2019/03/31 19:23:31 ipv4.go:151: send ipv4 packet 60 bytes, proto: 0x6
2019/03/31 19:23:31 ipv4.go:193: recv ipv4 packet 60 bytes, proto: 0x6
2019/03/31 19:23:31 endpoint.go:1395: recv tcp ack|syn segment from 192.168.1.1:9000, seq: 559240690, ack: 517094416
2019/03/31 19:23:31 connect.go:675: send tcp ack segment to 192.168.1.1:9000, seq: 517094416, ack: 559240691, rcvWnd: 32768
2019/03/31 19:23:31 ipv4.go:151: send ipv4 packet 52 bytes, proto: 0x6
2019/03/31 19:23:31 ipv4.go:193: recv ipv4 packet 52 bytes, proto: 0x6
2019/03/31 19:23:31 endpoint.go:1395: recv tcp ack segment from 192.168.1.1:26913, seq: 517094416, ack: 559240691
2019/03/31 19:23:31 main.go:157: Connected
2019/03/31 19:23:31 main.go:125: new conn: 1:192.168.1.1:26913 <nil>
2019/03/31 19:23:32 main.go:161: tcp disconected
2019/03/31 19:23:32 connect.go:675: send tcp ack|fin segment to 192.168.1.1:9000, seq: 517094416, ack: 559240691, rcvWnd: 32768
2019/03/31 19:23:32 ipv4.go:151: send ipv4 packet 52 bytes, proto: 0x6
2019/03/31 19:23:32 ipv4.go:193: recv ipv4 packet 52 bytes, proto: 0x6
2019/03/31 19:23:32 endpoint.go:1395: recv tcp ack|fin segment from 192.168.1.1:26913, seq: 517094416, ack: 559240691
2019/03/31 19:23:32 connect.go:675: send tcp ack segment to 192.168.1.1:26913, seq: 559240691, ack: 517094417, rcvWnd: 32768
2019/03/31 19:23:32 ipv4.go:151: send ipv4 packet 52 bytes, proto: 0x6
2019/03/31 19:23:32 ipv4.go:193: recv ipv4 packet 52 bytes, proto: 0x6
2019/03/31 19:23:32 endpoint.go:1395: recv tcp ack segment from 192.168.1.1:9000, seq: 559240691, ack: 517094417
2019/03/31 19:23:35 connect.go:675: send tcp ack|rst segment to 192.168.1.1:9000, seq: 517094417, ack: 559240691, rcvWnd: 0
```

### 3.9 tcp 的可靠性机制

本小节讨论 tcp 可靠性的实现，首先得知道可靠性指的是什么。可靠性指的是网络层能通信的前提下，保证数据包正确且按序到达对端。
比如发送端发送了“12345678”，那么接收端一定能收到“12345678”，不会乱序“12456783”，也不会少或多数据。

实现 TCP 的可靠传输有以下机制：
1. 校验和机制（检测和重传受到损伤的报文段）
2. 确认应答机制（保存失序到达的报文段直至缺失的报文到期，以及检测和丢弃重复的报文段）
3. 超时重传机制（重传丢失的报文段）

正因为 tcp 实现了可靠性，那么基于 tcp 的应用就可以不用担心发送的数据包丢失、乱序、不正确等，减轻了上层开发的负担。

#### 3.9.1 检验和
每个 tcp 段都包含了一个检验和字段，用来检查报文段是否收到损伤。如果某个报文段因检验和无效而被检查出受到损伤，就由终点 TCP 将其丢弃，并被认为是丢失了。TCP 规定每个报文段都必须使用 16 位的检验和。

#### 3.9.2 确认机制
控制报文段不携带数据，但需要消耗一个序号，它也需要被确认，而 ACK 报文段永远不需要确认，ACK 报文段不消耗序号，也不需要被确认。在以前，TCP 只使用一种类型的确认，叫积累确认，目前 TCP 实现还实现了选择确认。

- 累积确认（ACK）
接收方通告它期望接收的下一个字节的序号，并忽略所有失序到达并被保存的报文段。有时这被称为肯定累积确认。在 TCP 首部的 32 位 ACK 字段用于积累确认，而它的值仅在 ACK 标志为 1 时才有效。举个例子来说，这里先不考虑 tcp 的序列号，如果发送方发了数据包 p1，p2，p3，p4；接受方成功收到 p1，p2，p4。那么接收方需要发回一个确认包，序号为 3(3 表示期望下一个收到的包的序号)，那么发送方就知道 p1 到 p2 都发送接收成功，必要时重发 p3。一个确认包确认了累积到某一序号的所有包，而不是对每个序号都发确认包。实际的 tcp 确认的都是序列号，而不是包的序号，但原理是一样的。

累积确认是快速重传的基础，这个后面讲拥塞控制的时候会详细说明。

- 选择确认（SACK）

选择确认 SACK 要报告失序的数据块以及重复的报文段块，是为了更准确的告诉发送方需要重传哪些数据块。SACK 并没有取代 ACK，而是向发送方报告了更多的信息。SACK 是作为 TCP 首部末尾的选项来实现的。  
首先是否要启动 sack，应该在握手的时候告诉对方自己是否开启了 sack，这个是通过 kind=4 是选择性确认（Selective Acknowledgment，SACK）选项来实现的。   
实际传送 sack 信息的是 kind=5 的选项，其格式如下：  

```txt
         +--------+--------+
         | Kind=5 | Length | 
+--------+--------+--------+---------+ 
|          Start of 1st Block        | 
+--------+--------+--------+---------+ 
|           End of 1st Block         | 
+--------+--------+--------+---------+ 
|                                    | 
/            . . . . . .             / 
|                                    | 
+--------+--------+--------+---------+ 
|          Start of nth Block        | 
+--------+--------+--------+---------+ 
|           End of nth Block         | 
+--------+--------+--------+---------+ 
```
sack 的每个块是由两个参数构成的`{ Start, End }` Start 不连续块的第一个数据的序列号。End 不连续块的最后一个数据的序列号之后的序列号。
该选项参数告诉对方已经接收到并缓存的不连续的数据块，`注意都是已经接收的`，发送方可根据此信息检查究竟是哪个块丢失，从而发送相应的数据块。
比如下图：
![sack](https://doc.shiyanlou.com/document-uid949121labid10418timestamp1555574090941.png/wm)

如图所示，tcp 接收方在接收到不连续的 tcp 段，可以看出，序号 1～1000，1501～3000，3501～4500 接收到了，但却少了序号 1001～1500，3001～3500 。
前面说了，sack 报告的是已接收的不连续的块，在这个例子中，sack 块的内容为`{Start:1501, End:3001},{Start:3501, End:4501}`，
注意：**这里的 End 不是接收到数据段最后的序列号，而是最后的序列号加 1**。

### 3.10 产生确认的情况
1. 当接收方收到了按序到达（序号是所期望的）的报文段，那么接收方就累积发送确认报文段。
2. 当具有所期望的序号的报文段到达，而前一个按序到达的报文段还没有被确认，那么接收方就要立即发送 ACK 报文段。
3. 当序号比期望的序号还大的失序报文段到达时，接收方立即发送 ACK 报文段，并宣布下一个期望的报文段序号。这将导致对丢失报文段的快重传。
4. 当一个丢失的报文段到达时，接收方要发送 ACK 报文段，并宣布下一个所期望的序号。
5. 如果到达一个重复的报文段，接收方丢弃该报文段，但是应当立即发送确认，指出下一个期望的报文段。
6. 收到 fin 报文的时候，立即回复确认。 

#### 3.10.1 重传机制
关于重传的基本概念  
RTO 即超时重传时间  
RTT 数据包往返时间  
平均偏差是指单项测定值与平均值的偏差（取绝对值）之和，除以测定次数。 

![retranst](https://doc.shiyanlou.com/document-uid949121labid10418timestamp1555574119125.png/wm)
 
可靠性的核心就是报文段的重传。在一个报文段发送时，它会被保存到一个队列中，直至被确认为止。当重传计时器超时，或者发送方收到该队列中第一个报文段的三个重复的 ACK 时，该报文段被重传。

超时重传的概念很简单，就是一定时间内未收到确认，进行再次发送，但是如何计算重传的时间确实 tcp 最复杂的问题之一，毕竟要适应各种网络情况。TCP 一个连接期间只有一个 RTO 计时器，目前大部分实现都是采用`Jacobaon/Karels 算法`，详细可以看[RFC6298](https://tools.ietf.org/html/rfc6298)，其计算公式如下，  
rto 的计算公式：

```txt
第一次rtt计算： 
SRTT = R
RTTVAR = R/2
RTO = SRTT + max (G, K*RTTVAR)
K = 4

之后：
RTTVAR = (1 - beta) * RTTVAR + beta * |SRTT - R'|
SRTT = (1 - alpha) * SRTT + alpha * R'
RTO = SRTT + max (G, K*RTTVAR)
K = 4
```
SRTT(smoothed round-trip time)平滑 RTT 时间  
RTTVAR(round-trip time variation)RTT 变量，其实就是 rtt 平均偏差  
G 表示系统时钟的粒度，一般很小，us 级别。
beta = 1/4, alpha = 1/8  

发送方 TCP 的计时器时间到，TCP 发送队列中最前面的报文段（即序列号最小的报文段），并重启计时器。

### 3.11 tcp 校验和计算源码
```go
// sendTCP sends a TCP segment with the provided options via the provided
// network endpoint and under the provided identity.
// 发送一个tcp段数据，封装 tcp 首部，并写入网路层
func sendTCP(r *stack.Route, id stack.TransportEndpointID, data buffer.VectorisedView, ttl uint8, flags byte, seq, ack seqnum.Value, rcvWnd seqnum.Size, opts []byte) *tcpip.Error {
	optLen := len(opts)
	// Allocate a buffer for the TCP header.
	hdr := buffer.NewPrependable(header.TCPMinimumSize + int(r.MaxHeaderLength()) + optLen)

	if rcvWnd > 0xffff {
		rcvWnd = 0xffff
	}

	// Initialize the header.
	tcp := header.TCP(hdr.Prepend(header.TCPMinimumSize + optLen))
	tcp.Encode(&header.TCPFields{
		SrcPort:    id.LocalPort,
		DstPort:    id.RemotePort,
		SeqNum:     uint32(seq),
		AckNum:     uint32(ack),
		DataOffset: uint8(header.TCPMinimumSize + optLen),
		Flags:      flags,
		WindowSize: uint16(rcvWnd),
	})
	copy(tcp[header.TCPMinimumSize:], opts)

	// Only calculate the checksum if offloading isn't supported.
	if r.Capabilities()&stack.CapabilityChecksumOffload == 0 {
        length := uint16(hdr.UsedLength() + data.Size())
        // tcp伪首部校验和的计算
		xsum := r.PseudoHeaderChecksum(ProtocolNumber)
		for _, v := range data.Views() {
			xsum = header.Checksum(v, xsum)
		}

		// tcp的可靠性：校验和的计算，用于检测损伤的报文段
		tcp.SetChecksum(^tcp.CalculateChecksum(xsum, length))
	}

	...

	log.Printf("send tcp %s segment to %s, seq: %d, ack: %d, rcvWnd: %d",
		flagString(flags), fmt.Sprintf("%s:%d", id.RemoteAddress, id.RemotePort),
		seq, ack, rcvWnd)

	return r.WritePacket(hdr, data, ProtocolNumber, ttl)
}

// 校验和的计算
func Checksum(buf []byte, initial uint16) uint16 {
	v := uint32(initial)

	l := len(buf)
	if l&1 != 0 {
		l--
		v += uint32(buf[l]) << 8
	}

	for i := 0; i < l; i += 2 {
		v += (uint32(buf[i]) << 8) + uint32(buf[i+1])
	}

	return ChecksumCombine(uint16(v), uint16(v>>16))
}

// ChecksumCombine combines the two uint16 to form their checksum. This is done
// by adding them and the carry.
func ChecksumCombine(a, b uint16) uint16 {
	v := uint32(a) + uint32(b)
	return uint16(v + v>>16)
}

// PseudoHeaderChecksum calculates the pseudo-header checksum for the
// given destination protocol and network address, ignoring the length
// field. Pseudo-headers are needed by transport layers when calculating
// their own checksum.
func PseudoHeaderChecksum(protocol tcpip.TransportProtocolNumber, srcAddr tcpip.Address, dstAddr tcpip.Address) uint16 {
	xsum := Checksum([]byte(srcAddr), 0)
	xsum = Checksum([]byte(dstAddr), xsum)
	return Checksum([]byte{0, uint8(protocol)}, xsum)
}
```
## 四、tcp ack 和 sack 源码 

#### 4.1 ack
```go
// handleSegments pulls segments from the queue and processes them. It returns
// no error if the protocol loop should continue, an error otherwise.
// handleSegments 从队列中取出 tcp 段数据，然后处理它们。
func (e *endpoint) handleSegments() *tcpip.Error {
	checkRequeue := true
	for i := 0; i < maxSegmentsPerWake; i++ {
		s := e.segmentQueue.dequeue()
		if s == nil {
			checkRequeue = false
			break
        }
        
        ...

		if s.flagIsSet(flagRst) {
			// 如果收到 rst 报文
			if e.rcv.acceptable(s.sequenceNumber, 0) {
				// RFC 793, page 37 states that "in all states
				// except SYN-SENT, all reset (RST) segments are
				// validated by checking their SEQ-fields." So
				// we only process it if it's acceptable.
				s.decRef()
				return tcpip.ErrConnectionReset
			}
		} else if s.flagIsSet(flagAck) {
			// 处理正常的报文
			// Patch the window size in the segment according to the
			// send window scale.
			s.window <<= e.snd.sndWndScale

			...

			// RFC 793, page 41 states that "once in the ESTABLISHED
			// state all segments must carry current acknowledgment
			// information."
			// 处理tcp数据段，同时给接收器和发送器
			// 为何要给发送器传接收到的数据段呢？主要是为了滑动窗口的滑动和拥塞控制处理
			e.rcv.handleRcvdSegment(s)
			e.snd.handleRcvdSegment(s)
		}
		s.decRef()
	}

	...

	// Send an ACK for all processed packets if needed.
	// tcp可靠性：累积确认
	// 如果发送的最大ack不等于下一个接收的序列号，发送ack
	if e.rcv.rcvNxt != e.snd.maxSentAck {
		e.snd.sendAck()
	}

	...

	return nil
}
```
### 4.2 sack
```go
const (
	// MaxSACKBlocks is the maximum number of SACK blocks stored
	// at receiver side.
	// MaxSACKBlocks 是接收端存储的最大SACK块数。
	MaxSACKBlocks = 6
)

// UpdateSACKBlocks updates the list of SACK blocks to include the segment
// specified by segStart->segEnd. If the segment happens to be an out of order
// delivery then the first block in the sack.blocks always includes the
// segment identified by segStart->segEnd.
// tcp的可靠性：UpdateSACKBlocks 更新SACK块列表以包含 segStart-segEnd 指定的段，只有没有被消费掉的seg才会被用来更新sack。
// 如果该段恰好是无序传递，那么sack.blocks中的第一个块总是包括由 segStart-segEnd 标识的段。
func UpdateSACKBlocks(sack *SACKInfo, segStart seqnum.Value, segEnd seqnum.Value, rcvNxt seqnum.Value) {
	newSB := header.SACKBlock{Start: segStart, End: segEnd}
	if sack.NumBlocks == 0 {
		sack.Blocks[0] = newSB
		sack.NumBlocks = 1
		return
	}
	var n = 0
	for i := 0; i < sack.NumBlocks; i++ {
		start, end := sack.Blocks[i].Start, sack.Blocks[i].End
		if end.LessThanEq(start) || start.LessThanEq(rcvNxt) {
			// Discard any invalid blocks where end is before start
			// and discard any sack blocks that are before rcvNxt as
			// those have already been acked.
			continue
		}
		if newSB.Start.LessThanEq(end) && start.LessThanEq(newSB.End) {
			// Merge this SACK block into newSB and discard this SACK
			// block.
			if start.LessThan(newSB.Start) {
				newSB.Start = start
			}
			if newSB.End.LessThan(end) {
				newSB.End = end
			}
		} else {
			// Save this block.
			sack.Blocks[n] = sack.Blocks[i]
			n++
		}
	}
	if rcvNxt.LessThan(newSB.Start) {
		// If this was an out of order segment then make sure that the
		// first SACK block is the one that includes the segment.
		//
		// See the first bullet point in
		// https://tools.ietf.org/html/rfc2018#section-4
		if n == MaxSACKBlocks {
			// If the number of SACK blocks is equal to
			// MaxSACKBlocks then discard the last SACK block.
			n--
		}
		for i := n - 1; i >= 0; i-- {
			sack.Blocks[i+1] = sack.Blocks[i]
		}
		sack.Blocks[0] = newSB
		n++
	}
	sack.NumBlocks = n
}

// TrimSACKBlockList updates the sack block list by removing/modifying any block
// where start is < rcvNxt.
// tcp的可靠性：TrimSACKBlockList 通过删除/修改 start为 <rcvNxt 的任何块来更新sack块列表。
func TrimSACKBlockList(sack *SACKInfo, rcvNxt seqnum.Value) {
	n := 0
	for i := 0; i < sack.NumBlocks; i++ {
		if sack.Blocks[i].End.LessThanEq(rcvNxt) {
			continue
		}
		if sack.Blocks[i].Start.LessThan(rcvNxt) {
			// Shrink this SACK block.
			sack.Blocks[i].Start = rcvNxt
		}
		sack.Blocks[n] = sack.Blocks[i]
		n++
	}
	sack.NumBlocks = n
}


// handleRcvdSegment handles TCP segments directed at the connection managed by
// r as they arrive. It is called by the protocol main loop.
// 从 handleSegments 接收到tcp段，然后进行处理消费，所谓的消费就是将负载内容插入到接收队列中
func (r *receiver) handleRcvdSegment(s *segment) {
	// We don't care about receive processing anymore if the receive side
	// is closed.
	if r.closed {
		return
	}

	segLen := seqnum.Size(s.data.Size())
	segSeq := s.sequenceNumber

	// If the sequence number range is outside the acceptable range, just
	// send an ACK. This is according to RFC 793, page 37.
	// tcp可靠性：判断该数据段的序列号是否在接收窗口内，如果不在，立即返回ack给对端。
	if !r.acceptable(segSeq, segLen) {
		r.ep.snd.sendAck()
		return
	}

	// Defer segment processing if it can't be consumed now.
	// tcp可靠性：r.consumeSegment 返回值是个bool类型，如果是true，表示已经消费该数据段，
	// 如果不是，那么进行下面的处理，插入到 pendingRcvdSegments，且进行堆排序。
	if !r.consumeSegment(s, segSeq, segLen) {
		// 如果有负载数据或者是 fin 报文，立即回复一个 ack 报文
		if segLen > 0 || s.flagIsSet(flagFin) {
			// We only store the segment if it's within our buffer
			// size limit.
			// tcp可靠性：对于乱序的tcp段，应该在等待处理段中缓存
			if r.pendingBufUsed < r.pendingBufSize {
				r.pendingBufUsed += s.logicalLen()
				s.incRef()
				// 插入堆中，且进行排序
				heap.Push(&r.pendingRcvdSegments, s)
			}

			// tcp的可靠性：更新 sack 块信息
			UpdateSACKBlocks(&r.ep.sack, segSeq, segSeq.Add(segLen), r.rcvNxt)

			// Immediately send an ack so that the peer knows it may
			// have to retransmit.
			r.ep.snd.sendAck()
		}
		return
	}

	// By consuming the current segment, we may have filled a gap in the
	// sequence number domain that allows pending segments to be consumed
	// now. So try to do it.
	// tcp的可靠性：通过使用当前段，我们可能填补了序列号域中的间隙，该间隙允许现在使用待处理段。
	// 所以试着去消费等待处理段。
	for !r.closed && r.pendingRcvdSegments.Len() > 0 {
		s := r.pendingRcvdSegments[0]
		segLen := seqnum.Size(s.data.Size())
		segSeq := s.sequenceNumber

		// Skip segment altogether if it has already been acknowledged.
		if !segSeq.Add(segLen-1).LessThan(r.rcvNxt) &&
			!r.consumeSegment(s, segSeq, segLen) {
			break
		}

		// 如果该tcp段，已经被正确消费，那么中等待处理段中删除
		heap.Pop(&r.pendingRcvdSegments)
		r.pendingBufUsed -= s.logicalLen()
		s.decRef()
	}
}

```
### 4.3 tcp 设置 rto 和 rto 处理源码
```go
// 根据rtt来更新计算rto
func (s *sender) updateRTO(rtt time.Duration) {
	s.rtt.Lock()
	// 第一次计算
	if !s.srttInited {
		s.rtt.rttvar = rtt / 2
		s.rtt.srtt = rtt
		s.srttInited = true
	} else {
		diff := s.rtt.srtt - rtt
		if diff < 0 {
			diff = -diff
		}
		// Use RFC6298 standard algorithm to update rttvar and srtt when
		// no timestamps are available.
		if !s.ep.sendTSOk {
			s.rtt.rttvar = (3*s.rtt.rttvar + diff) / 4
			s.rtt.srtt = (7*s.rtt.srtt + rtt) / 8
		} else {
			// When we are taking RTT measurements of every ACK then
			// we need to use a modified method as specified in
			// https://tools.ietf.org/html/rfc7323#appendix-G
			if s.outstanding == 0 {
				s.rtt.Unlock()
				return
			}
			// Netstack measures congestion window/inflight all in
			// terms of packets and not bytes. This is similar to
			// how linux also does cwnd and inflight. In practice
			// this approximation works as expected.
			expectedSamples := math.Ceil(float64(s.outstanding) / 2)

			// alpha & beta values are the original values as recommended in
			// https://tools.ietf.org/html/rfc6298#section-2.3.
			const alpha = 0.125
			const beta = 0.25

			alphaPrime := alpha / expectedSamples
			betaPrime := beta / expectedSamples
			rttVar := (1-betaPrime)*s.rtt.rttvar.Seconds() + betaPrime*diff.Seconds()
			srtt := (1-alphaPrime)*s.rtt.srtt.Seconds() + alphaPrime*rtt.Seconds()
			s.rtt.rttvar = time.Duration(rttVar * float64(time.Second))
			s.rtt.srtt = time.Duration(srtt * float64(time.Second))
		}
	}

	// 更新 rto 值
	s.rto = s.rtt.srtt + 4*s.rtt.rttvar
	s.rtt.Unlock()
	if s.rto < minRTO {
		s.rto = minRTO
	}
}

// sendData sends new data segments. It is called when data becomes available or
// when the send window opens up.
// 发送数据段后，启动rto定时器
func (s *sender) sendData() {
	...

	// Enable the timer if we have pending data and it's not enabled yet.
	// tcp的可靠性：如果重传定时器没有启动 且 snduna 不等于 sndNxt，启动定时器
	if !s.resendTimer.enabled() && s.sndUna != s.sndNxt {
		// 启动定时器，并且设定定时器的间隔为s.rto
		s.resendTimer.enable(s.rto)
	}
	...
}

// tcp的可靠性：重传定时器触发的时候调用这个函数，也就是超时重传
func (s *sender) retransmitTimerExpired() bool {
	// Check if the timer actually expired or if it's a spurious wake due
	// to a previously orphaned runtime timer.
	// 检查计时器是否真的到期
	if !s.resendTimer.checkExpiration() {
		return true
	}

	// Give up if we've waited more than a minute since the last resend.
	// 如果rto已经超过了1分钟，直接放弃发送，返回错误
	if s.rto >= 60*time.Second {
		return false
	}

	// Set new timeout. The timer will be restarted by the call to sendData
	// below.
	// 每次超时，rto都变成原来的2倍
	s.rto *= 2

    ...
	
	// tcp可靠性：将下一个段标记为第一个未确认的段，然后再次开始发送。将未完成的数据包数设置为0，以便我们能够重新传输。
	// 当我们收到我们传输的数据时，我们将继续传输（或重新传输）。
	s.outstanding = 0
	s.writeNext = s.writeList.Front()
	s.sendData()

	return true
}
```



## 五、tcp 的流量控制
本节介绍 tcp 的流量控制和源码分析。

### 5.1 为何需要流量控制
任何生产消费者模型都会有速率不对等的问题，对于 tcp 来说，发送方是生产者，它通过网络通道发送数据给接收端，接收方是消费者，如果消费者消费的速率不够快，那么就会导致数据在接收方累积或者丢弃，接收端一但丢弃数据，tcp 发送端就会认为丢包，导致进入拥塞避免阶段。

那如何避免这种速率不对等呢？一般有两种方案：  
- 增大接收方的 buffer（地铁进站排队通道来回迂折）
- 通过反馈机制，告诉发送方我们能接收多少

增大接收方的 buffer，是很常用的一种手段，它能够一定程度缓解速率不对等问题，但是它不能从根本上解决问题，因为计算机的资源是有限的，增大 buffer 意味要增大内存使用量，如果计算机的内存是无限的，那么确实可以解决这个问题。

通过反馈机制，接收方告诉发送方发送的传输速率不能大于应用的数据处理速率，tcp 中就是这么实现的，协议中有 window 字段，表示接收端窗口的大小，单位为 byte。

#### 5.1.1 生活中流控现象
进火车站安检的时候，会有两个关卡，关卡 1 有个门，安检人员时不时打开放人进去，关卡 2 是旅客过安检设备。因为关卡 2 过安检设备比较慢，所以关卡 1 安检人员会看关卡 2 旅客差不多都通过安检设备后，才打开关卡 1 的门，放一波人进去。

### 5.2 tcp 流量控制的实现-滑动窗口
接收端在给发送端回 ACK 中会汇报自己的 AdvertisedWindow，而发送方会根据这个窗口来控制发送数据的大小，以保证接收方可以处理。  
要明确的是滑动窗口分为两个窗口，接收窗口和发送窗口  

#### 5.2.1 接收窗口
接收窗口不仅可以限制发送端发送的速率，还可以提高效率，因为接收窗口的机制，可以允许发送端一次多发送几个片段，而不必等候 ACK，而且可以允许等待一定情况下的乱序，
比如说先缓存提前到的数据，然后去等待需要的数据。

![recv-win]((https://doc.shiyanlou.com/document-uid949121labid10418timestamp1555574147849.png/wm)  
接收的窗口可以分为四段：  
* 数据已经被 tcp 确认，但用户程序还未读取数据内容
* 中间还有些数据没有到达
* 数据已经接收到，但 tcp 未确认
* 通告窗口，也就是接收端在给发送端回 ACK 中会汇报自己的窗口大小

当接收端接收到数据包时，会判断该数据包的序列号是不是在接收窗口內，如果不在窗口內会立即回一个 ack 给发送端，
且丢弃该报文。

滑动： 当用户程序读取接收窗口的内容后，窗口向右滑行

#### 5.2.2 发送窗口
发送窗口的值是由接收窗口和拥塞窗口一起决定的，发送窗口的大小也决定了发送的速率。  
![send-win](https://doc.shiyanlou.com/document-uid949121labid10418timestamp1555574248321.png/wm)  

发送窗口的上限值 = Min [rwnd, cwnd]，cwnd 拥塞窗口   
上图中分成了四个部分，分别是：（其中那个黑模型就是滑动窗口）  
* 已收到 ack 确认的数据
* 已经发送，但还没收到 ack 的数据
* 在窗口中还没有发出的（接收方还有空间）
* 窗口以外的数据（接收方没空间）

滑动： 当发送端收到数据 ack 确认时，窗口向右滑

### 5.3 Zero Window
如果一个处理缓慢的 Server（接收端）是怎么把 Client（发送端）的`TCP Sliding Window`给降成 0 的。此时，你一定会问，如果 Window 变成 0 了，TCP 会怎么样？是不是发送端就不发数据了？是的，发送端就不发数据了，你可以想像成“Window Closed”，那你一定还会问，如果发送端不发数据了，接收方一会儿 Window size 可用了，怎么通知发送端呢？

1. 当接收方的应用程序读取了接收缓冲区中的数据以后，接收方会发送一个 ACK，通过通告窗口字段告诉发送方自己又可以接收数据了，发送方收到这个 ACK 之后，就知道自己可以继续发送数据了。

2. 同时发送端使用了`Zero Window Probe`技术，缩写为 ZWP，当接收方的接收窗口为 0 时，每隔一段时间，发送方会主动发送探测包，迫使对端响应来得知其接收窗口有无打开。

既然接收端会主动通知发送端，为何还需要发送端定时探测？

### 5.4 Silly Window Syndrome

`Silly Window Syndrome`翻译成中文就是“糊涂窗口综合症”。正如你上面看到的一样，如果我们的接收方太忙了，来不及取走 Receive Windows 里的数据，那么，就会导致发送方越来越小。到最后，如果接收方腾出几个字节并告诉发送方现在有几个字节的 window，而我们的发送方会义无反顾地发送这几个字节。

要知道，我们的 TCP+IP 头有 40 个字节，为了几个字节，要达到这么大的开销，这太不经济了。

所以，`Silly Windows Syndrome`这个现像就像是你本来可以坐 200 人的飞机里只做了一两个人。要解决这个问题也不难，就是避免对小的 window size 做出响应，直到有足够大的 window size 再响应，这个思路可以同时实现在 sender 和 receiver 两端。

如果这个问题是由 Receiver 端引起的，那么就会使用`David D Clark’s`方案。在 receiver 端，如果收到的数据导致`window size`小于某个值，可以直接 ack(0)回 sender，这样就把 window 给关闭了，也阻止了 sender 再发数据过来，等到 receiver 端处理了一些数据后`windows size`大于等于了 MSS，或者，`receiver buffer`有一半为空，就可以把 window 打开让 sender 发送数据过来。

如果这个问题是由 Sender 端引起的，那么就会使用著名的`Nagle’s algorithm`。这个算法的思路也是延时处理，他有两个主要的条件：
1. 要等到 `Window Size >= MSS` 或是 `Data Size >= MSS`
2. 收到之前发送数据的 ack 回包，他才会发数据，否则就是在攒数据


### 5.5 发送窗口的维护

```txt
					 +-------> sndWnd <-------+
					 |						  |
---------------------+-------------+----------+--------------------
|      acked		 | * * * * * * | # # # # #|   unable send
---------------------+-------------+----------+--------------------
					 ^             ^
					 |			   |
				   sndUna        sndNxt
***** in flight data
##### able send date
```
发送窗口主要维护这些变量，sndBufSize、sndBufUsed、sndUna、sndNxt 和 sndWnd。sndUna 表示是下一个未确认的序列号，sndNxt 是要发送的下一个段的序列号，sndWnd 是接受端通告的窗口大小。
首先是处理接收方的窗口通告，当收到报文时，一定会带接收窗口和确认号，此时先更新发送器的发送窗口大小为接收窗口大小。
```go
// Write writes data to the endpoint's peer.
// 接收上层的数，通过tcp连接发送到对端
func (e *endpoint) Write(p tcpip.Payload, opts tcpip.WriteOptions) (uintptr, <-chan struct{}, *tcpip.Error) {
	// Linux completely ignores any address passed to sendto(2) for TCP sockets
	// (without the MSG_FASTOPEN flag). Corking is unimplemented, so opts.More
	// and opts.EndOfRecord are also ignored.

	e.mu.RLock()
	defer e.mu.RUnlock()

	// The endpoint cannot be written to if it's not connected.
	// 判断tcp状态，必须已经建立了连接才能发送数据
	if e.state != stateConnected {
		switch e.state {
		case stateError:
			return 0, nil, e.hardError
		default:
			return 0, nil, tcpip.ErrClosedForSend
		}
	}

	// Nothing to do if the buffer is empty.
	// 检查负载的长度，如果为0，直接返回
	if p.Size() == 0 {
		return 0, nil, nil
	}

	e.sndBufMu.Lock()

	// Check if the connection has already been closed for sends.
	if e.sndClosed {
		e.sndBufMu.Unlock()
		return 0, nil, tcpip.ErrClosedForSend
	}

	// Check against the limit.
	// tcp流量控制：未被占用发送缓存还剩多少，如果发送缓存已经被用光了，返回 ErrWouldBlock
	avail := e.sndBufSize - e.sndBufUsed
	if avail <= 0 {
		e.sndBufMu.Unlock()
		return 0, nil, tcpip.ErrWouldBlock
	}

	v, perr := p.Get(avail)
	if perr != nil {
		e.sndBufMu.Unlock()
		return 0, nil, perr
	}

	var err *tcpip.Error
	if p.Size() > avail {
		err = tcpip.ErrWouldBlock
	}
	l := len(v)
	s := newSegmentFromView(&e.route, e.id, v)

	// Add data to the send queue.
	// 插入发送队列
	e.sndBufUsed += l
	e.sndBufInQueue += seqnum.Size(l)
	e.sndQueue.PushBack(s)

	e.sndBufMu.Unlock()

	// 发送数据，最终会调用 sender sendData 来发送数据。
	if e.workMu.TryLock() {
		// Do the work inline.
		e.handleWrite()
		e.workMu.Unlock()
	} else {
		// Let the protocol goroutine do the work.
		e.sndWaker.Assert()
	}
	return uintptr(l), nil, err
}

// 收到tcp段时调用 handleRcvdSegment; 它负责更新与发送相关的状态。
func (s *sender) handleRcvdSegment(seg *segment) {
    ...
    
	// 存放当前窗口大小。
	s.sndWnd = seg.window

    // 获取确认号
	ack := seg.ackNumber
	// 如果ack在最小未确认的seq和下一seg的seq之间
	if (ack - 1).InRange(s.sndUna, s.sndNxt) {
		...
		// Remove all acknowledged data from the write list.
		acked := s.sndUna.Size(ack)
		s.sndUna = ack

		ackLeft := acked
		originalOutstanding := s.outstanding
		for ackLeft > 0 {
			// We use logicalLen here because we can have FIN
			// segments (which are always at the end of list) that
			// have no data, but do consume a sequence number.
			seg := s.writeList.Front()
			datalen := seg.logicalLen()

			if datalen > ackLeft {
				seg.data.TrimFront(int(ackLeft))
				break
			}

			if s.writeNext == seg {
				s.writeNext = seg.Next()
			}
			s.writeList.Remove(seg)
			s.outstanding--
			seg.decRef()
			ackLeft -= datalen
		}

		// Update the send buffer usage and notify potential waiters.
		s.ep.updateSndBufferUsage(int(acked))

		...
	}

	...
}
```

### 5.6 接收窗口的维护
接收窗口主要维护这几个变量，rcvBufSize、rcvBufUsed、rcvNxt 和 rcvAcc，
```go
// tcp流量控制：计算未被占用的接收缓存大小
func (e *endpoint) receiveBufferAvailable() int {
	e.rcvListMu.Lock()
	size := e.rcvBufSize
	used := e.rcvBufUsed
	e.rcvListMu.Unlock()

	// We may use more bytes than the buffer size when the receive buffer
	// shrinks.
	if used >= size {
		return 0
	}

	return size - used
}

func (e *endpoint) receiveBufferSize() int {
	e.rcvListMu.Lock()
	size := e.rcvBufSize
	e.rcvListMu.Unlock()

	return size
}

// zeroReceiveWindow 根据可用缓冲区的数量和接收窗口缩放，检查现在要宣布的接收窗口是否为零。
func (e *endpoint) zeroReceiveWindow(scale uint8) bool {
	if e.rcvBufUsed >= e.rcvBufSize {
		return true
	}

	return ((e.rcvBufSize - e.rcvBufUsed) >> scale) == 0
}

// tcp流量控制：判断 segSeq 在窗口內
func (r *receiver) acceptable(segSeq seqnum.Value, segLen seqnum.Size) bool {
	rcvWnd := r.rcvNxt.Size(r.rcvAcc)
	if rcvWnd == 0 {
		return segLen == 0 && segSeq == r.rcvNxt
	}

	return segSeq.InWindow(r.rcvNxt, rcvWnd) ||
		seqnum.Overlap(r.rcvNxt, rcvWnd, segSeq, segLen)
}

// tcp流量控制：当接收窗口从零增长到非零时，调用 nonZeroWindow;在这种情况下，
// 我们可能需要发送一个 ack，以便向对端表明它可以恢复发送数据。
func (r *receiver) nonZeroWindow() {
	if (r.rcvAcc-r.rcvNxt)>>r.rcvWndScale != 0 {
		// We never got around to announcing a zero window size, so we
		// don't need to immediately announce a nonzero one.
		return
	}

	// Immediately send an ack.
	r.ep.snd.sendAck()
}

// 从tcp的接收队列中读取数据，并从接收队列中删除已读数据
func (e *endpoint) readLocked() (buffer.View, *tcpip.Error) {
	if e.rcvBufUsed == 0 {
		if e.rcvClosed || e.state != stateConnected {
			return buffer.View{}, tcpip.ErrClosedForReceive
		}
		return buffer.View{}, tcpip.ErrWouldBlock
	}

	s := e.rcvList.Front()
	views := s.data.Views()
	v := views[s.viewToDeliver]
	s.viewToDeliver++

	if s.viewToDeliver >= len(views) {
		e.rcvList.Remove(s)
		s.decRef()
	}

	scale := e.rcv.rcvWndScale
	// tcp流量控制：检测接收窗口是否为0
	wasZero := e.zeroReceiveWindow(scale)
	e.rcvBufUsed -= len(v)
	// 检测糊涂窗口，主动发送窗口不为0的通告给对方
	if wasZero && !e.zeroReceiveWindow(scale) {
		e.notifyProtocolGoroutine(notifyNonZeroReceiveWindow)
	}

	return v, nil
}
```

 
## 六、tcp 的拥塞控制
本节介绍 tcp 的拥塞控制，拥塞控制控制是 tcp 协议中最复杂问题之一，主要是如何探测链路已经拥塞？探测到拥塞后如何处理？

### 6.1 为何需要拥塞控制
![Van_Jacobson](https://doc.shiyanlou.com/document-uid949121labid10418timestamp1555574278026.png/wm)     
tcp 拥塞控制提出者  

事实上，早期 TCP 实现是没有拥塞控制的，拥塞控制是网络出现问题后才提出的，在 1986 年，互联网首次出现了一系列“拥堵事故”，从伯克利实验室到加州大学伯克利分校的数据吞吐量从`32 Kbps`降至`40 bps`。然后在伯克利实验室的工作人员 Van_Jacobson（上图就是他）很好奇为何网络会拥堵，并开始调查网络变得如此糟糕，2 年后，他提出了拥塞算法。

我们知道 TCP 通过采样了 RTT 并计算 RTO，但是，如果网络上的延时突然增加超过了 RTO，那么 TCP 对这个事做出的应对只有重传数据，但是，重传会导致网络的负担更重，于是会导致更大的延迟以及更多的丢包，这样就会进入恶性循环被不断地放大。试想一下，如果一个网络内有成千上万的 TCP 连接都这么行事，那么马上就会形成“网络风暴”，TCP 这个协议就会拖垮整个网络。这是一个灾难。

所以，TCP 不能忽略网络上发生的事情，而一个劲的重发数据，对网络造成更大的伤害。于是就提出了拥塞控制，当拥塞发生的时候，要做自我牺牲，降低发送速率。就像交通阻塞一样，每个车都应该把路让出来，而不要再去抢路了。

要注意流量控制和拥塞控制的区别，流量控制只控制两个端的速度，它抑制发送端的速度，以便接收端能接收，但它并不关心中间链路的网络情况。而拥塞控制是关心中间链路的网络情况，防止过多的数据注入到网络中，以便防止中间的链路或路由不过载。

#### 6.1.1 如何知道网络拥塞过
要避免网络拥塞，第一个问题是我们需要知道何时发生了拥堵。根据定义，拥塞意味着中间设备（路由器）过载了，路由器通过丢弃数据报来处理过载的情况。当这些数据报包含 TCP 段时，这些段不会到达其目的地，因此它们不会被确认并最终过期并被重新传输。这意味着当设备发送 TCP 段并且没有接收到它们的确认时，可以假设在大多数情况下，它们由于拥塞而被中间设备丢弃。通过检测发送端超时的数据段，TCP 设备可以推断出中间的链路网络拥塞了。

还有一种情况可以显示的知道网络拥堵了，这需要路由器的支持，这种机制叫显式拥塞通知 ECN，基本原理是路由器在出现拥塞时通知 TCP。当 TCP 段传递时，路由器使用 IP 首部中的 2 位来记录拥塞，当 TCP 段到达后，接收方知道报文段是否在某个位置经历过拥塞。然而，需要了解拥塞发生情况的是发送方，而非接收方。因此，接收方使用下一个 ACK 通知发送方有拥塞发生，然后发送方做出响应，缩小自己的拥塞窗口。

### 6.2 拥塞控制的算法
TCP 通过维护一个拥塞窗口(cwnd 全称 Congestion Window)来进行拥塞控制，拥塞控制的原则是，只要网络中没有出现拥塞，拥塞窗口的值就可以再增大一些，以便把更多的数据包发送出去，但只要网络出现拥塞，拥塞窗口的值就应该减小一些，以减少注入到网络中的数据包数。
拥塞控制主要是四个算法：1）慢启动，2）拥塞避免，3）快速重传，4）快速恢复。
这四个算法不是一天都搞出来的，这个四算法的发展经历了很时间，至今都还在优化中，仅实现这个四个算法的拥塞算法叫 Reno 算法。

#### 6.2.1 慢启动（slow start）
![start](https://doc.shiyanlou.com/document-uid949121labid10418timestamp1555574529514.png/wm)  

慢启动的意思是，加入网络的连接，一点一点地提速，不要一上来就突发流量挤占通道。
慢启动的算法如下：  
1. 连接建好的开始先初始化`cwnd = 1`，表明可以传一个 MSS 大小的数据。
2. 每当收到一个 ACK，`cwnd++`; 呈线性上升
3. 每当过了一个 RTT，`cwnd = cwnd*2`; 呈指数上升
4. 设置一个慢启动阀值 ssthresh（slow start threshold），是慢启动和拥塞避免的一个临界值，当`cwnd >= ssthresh`时，就会进入`拥塞避免阶段`

慢启动应用的的阶段是 tcp 连接刚刚建立或者监测到丢包了。  
慢启动很慢吗？  
注意 cwnd 的单位是 mss，试算一下慢启动的速度。   
rtt=100ms，loss=0%，mss=1460， 1s 后，`cwnd = (2^9) = 512`，按这个 cwnd 发送的速度是`v = cwnd*1460/0.1*8=59,801,600(bps)`  

#### 6.2.2 拥塞避免（congestion avoidance）
当`cwnd >= ssthresh`时，就会进入“拥塞避免算法”。一般来说初始 ssthresh 的值是很大的，当 cwnd 达到这个值时后，算法如下：  
1. 每当收到一个 ACK 时，`cwnd = cwnd + 1/cwnd`
2. 每当过一个 RTT 时，`cwnd = cwnd + 1`

拥塞避免的思想就是转指数增大变为线性增大。这样就可以避免增长过快导致网路拥塞，慢慢的增加调整到网络的最佳值。

#### 6.2.3 快速重传（Fast Retransmit）
![Retransmit](https://doc.shiyanlou.com/document-uid949121labid10418timestamp1555574601517.png/wm) 

快速重传算法，也就是在收到 3 个重复 ACK 时就开启重传，而不用等到 RTO 超时。   
1. 执行`ssthresh = max (cwnd/2, 2)`，然后进入快速恢复算法，ssthresh 至少有 2 个 mss。  

#### 6.2.4 快速恢复（Fast Recovery）
快速重传和快速恢复算法一般同时使用。快速恢复算法是认为，你还有 3 个`Duplicated Acks`回来，说明网络也不那么糟糕，所以没有必要像 RTO 超时那么强烈。注意，正如前面所说，进入快速重传之前，sshthresh 已被更新`ssthresh = max (cwnd/2, 2)`然后，真正的`Fast Recovery`算法如下：  
1. `cwnd = sshthresh + 3`（3 的意思是确认有 3 个数据包被收到了）
2. 重传重复 ACKs 指定的数据包  
3. 如果再收到重复 Acks，那么`cwnd = cwnd + 1`  
如果收到了新的 Ack，那么，`cwnd = sshthresh`，然后就进入了拥塞避免的算法了。  

我们可以看到 RTO 超时后，sshthresh 会变成 cwnd 的一半，当下一个 ACK 到达时，如果当前的`cwnd < ssthresh`，那么 cwnd 又会很快指数增涨，
直到`cwnd >= ssthresh`时 就会成慢慢的线性增涨。可以看到 TCP 是通过这种强烈地震荡，快速而小心得找到流量的平衡点的。

其实现实中实现 tcp 拥塞算法的时候，还必须处理一种情况，那就是处理丢包的情况。在 tcp reno 算法中，丢包的时候：  
1. `ssthresh = max (cwnd/2, 2)`
2. `cwnd = 1`

备注:  
* 1988 年，TCP Tahoe 提出了 1）慢启动，2）拥塞避免，3）快速重传。
* 1990 年，TCP Reno 在 Tahoe 的基础上增加了 4）快速恢复，是现有的众多拥塞控制算法的基础，被认为标准 tcp 拥塞行为。
* 2004 年，TCP BIC（Binary Increase Congestion control），在`Linux 2.6.8`中是默认拥塞控制算法，用的是 Binary Search——二分查找来找拥塞窗口。
* 2008 年，TCP CUBIC 是比 BIC 更温和和系统化的分支版本，其使用三次函数代替二分算法作为其拥塞窗口算法，并且使用函数拐点作为拥塞窗口的设置值。
`Linux 2.6.19`后使用该算法作为默认 TCP 拥塞算法，现在也是。
* 2016 年，TCP BBR 是由 Google 设计，于 2016 年发布的拥塞算法，交替测量一段时间内的带宽极大值和时延极小值，将其乘积作为作为拥塞窗口大小，认为当网络上的数据包总量大于瓶颈链路带宽和时延的乘积时才出现了拥塞。  

### 6.3 CUBIC 算法介绍 (cube+bic)
cubic 算法旨在优化高速高延迟网络（长肥网络）的拥塞控制，它使用使用立方函数代替标准 TCP 的线性窗口增加功能。在拥塞事件之后的拥塞避免期间，CUBIC 改变标准 TCP 的窗口增加功能。但是慢启动和快速恢复和标准 TCP 一样的。假设 W_max 是在最后一次拥塞事件中窗口减少之前的窗口大小。窗口的增长函数如下：  
```txt
W_cubic(t) = C*(t-K)^3 + W_max （1）
```
其中`C`是固定的常数，用于确定高 BDP 网络中窗口增加的积极性，`t`是从当前拥塞避免开始经过的时间，`K`是上述函数在没有进一步丢失事件时将 W 增加到`W_max`所需的时间段。 
使用以下等式计算：  
```txt
K = cubic_root(W_max*(1-beta_cubic)/C) （2）
```
其中`cubic_root`是三次方根，`beta_cubic`是 CUBIC 乘法减少因子，也就是说，当检测到拥塞事件时，窗口根据下面的函数来减少：  
```go
W_cubic(0)=W_max*beta_cubic （3） 
W_cubic(0)至少等于2。
```
一般`C=0.4`， `beta_cubic=0.7`，可以看出它的窗口增长函数仅仅取决于连续的两次拥塞事件的时间间隔值，窗口增长完全独立于网络的时延 RTT。   

![cubic-func](https://doc.shiyanlou.com/document-uid949121labid10418timestamp1555574392514.png/wm)   

在拥塞避免阶段每收到一个 ACK，CUBIC 都会使用公式（1）W(t+RTT)作为拥塞窗口的候选值，假设当前拥塞窗口大小为 cwnd。根据 cwnd 的值，CUBIC 有三种运行模式。  
1. 首先，如果 cwnd 小于（标准）TCP 在上次丢包事件之后 t 时刻到达的窗口大小，那么 CUBIC 处于 TCP 模式，使用下面的公式计算。
```txt
W_est(t) = W_max*beta_cubic + [3*(1-beta_cubic)/(1+beta_cubic)] * (t/RTT) 
cwnd = W_est(t)
```
2. 否则，如果 cwnd 小于 Wmax，那么 CUBIC 在三次函数的凹轮廓区域。
```txt
cwnd_inc = (W_cubic(t+RTT) - cwnd)/cwnd
cwnd = cwnd + cwnd_inc
```
3. 如果 cwnd 大于 Wmax，那么，CUBIC 处于三次函数的凸轮廓区域。
```txt
cwnd_inc = (W_cubic(t+RTT) - cwnd)/cwnd
cwnd = cwnd + cwnd_inc
```

当出现丢包事件时，CUBIC 会记录这时的拥塞窗口大小作为 W_max，接着通过公式（3）确定`sshthresh`的大小，设置`cwnd = 1`，并进行正常的 TCP 快速恢复、重传和慢启动。从快速恢复阶段进入拥塞避免后，再继续前面讲的关于拥塞避免阶段的处理。

为了提高 CUBIC 的收敛速度，在协议中加入了快速收敛机制。当新的流量加入网络时，网络中的现有流量需要放弃其带宽份额，以使新流量有一定的增长空间。
下面详细描述一下快速收敛的过程。在发生丢包前，CUBIC 会记录一个最大窗口值`W_max`，当发生丢包后，在降低窗口前，CUBIC 又会记录当前的窗口值作为新的`W_max`，
为了不至于混淆，可以将之前记录的`W_max`标记位`W_last_max`。当发生丢包时，CUBIC 会比较`W_last_max`的`W_max`大小，如果`W_max`小于`W_last_max`，这表明由于可用带宽的变化，该流所经历的饱和点正在降低。这种情况下，CUBIC 的做法是通过进一步的减小 Wmax 来释放更多的可用带宽。
```txt
if (W_max < W_last_max){ // should we make room for others
    W_last_max = W_max;             // remember the last W_max
    W_max = W_max*(1.0+beta_cubic)/2.0; // further reduce W_max
} else {
    W_last_max = W_max              // remember the last W_max
}
```

![cubic-win-time](https://doc.shiyanlou.com/document-uid949121labid10418timestamp1555574419473.png/wm)  
cubic 算法的优点在于只要没有出现丢包，就不会主动降低自己的发送速度，可以最大程度的利用网络剩余带宽，提高吞吐量，在高带宽、低丢包率的网络中可以发挥较好的性能,
适用于高带宽、低丢包率网络，能够有效利用带宽。

cubic 算法的不足之处是过于激进，在没有出现丢包时会不停地增加拥塞窗口的大小，向网络注入流量，将网络设备的缓冲区填满，出现 Bufferbloat（缓冲区膨胀）。由于缓冲区长期趋于饱和状态，新进入网络的的数据包会在缓冲区里排队，增加无谓的排队时延，缓冲区越大，时延就越高。另外 cubic 算法在高带宽利用率的同时依然在增加拥塞窗口，间接增加了丢包率，造成网络抖动加剧。


### 6.4 之前的拥塞算法有何问题

丢包即拥塞的思想已经沿用了很多年，很多拥塞控制算法也是基于此的，比如当前`Linux kernel`的默认拥塞控制算法`CUBIC`，还有`Reno`和`FAST TCP`等，都是基于这一思想进行的拥塞控制。在技术受限的年代，这一思想（丢包即拥塞）没有错，但是，现在，当网卡的处理能力从 Mbps 升级到 Gpbs，路由内存从 KB 升级到 GB，拥塞和丢包的关系就没那么紧密了。   
在现在高 BDP 网络环境下，丢包即拥塞思想带来的问题包括：当因为瓶颈缓存满而出现丢包时，会引起`bufferbloat`(缓冲区爆满，排队延迟影响网络整体性能）现象，网络延迟高；但是，当瓶颈缓冲很小时，这时出现丢包，网络会误认为是发生了拥塞，从而降低发送窗口，这样就会造成地吞吐量。想要解决上面的问题，那么就需要抛弃基于丢包的拥塞控制思想，换个新的。

![tcp-bbr](https://doc.shiyanlou.com/document-uid949121labid10418timestamp1555574639011.png/wm)   

图中的一些符号解释如下：
* `RTprop` 链路的最小时延（其实是 2 倍时延，因为是一个来回，但不影响讨论），这取决于物理距离；
* `BtlBw` 链路中最慢的那段的带宽，称为瓶颈带宽；  
* `BtlBufSize` 链路中，每个路由器都有自己的缓存，这些缓存的容量之和，称为瓶颈缓存大小；    
* `BDP` 带宽时延积，表示整条物理链路（不含路由器缓存）所能储藏的比特数据之和，`BDP = BtlBw * RTprop`；    
* 蓝线表示 RTprop 常量；  
* 绿线表示 BtlBw 常量；  
* 红线表示瓶颈缓存；

* 上半图  
当链路上正在传输的比特数据未超过整条链路的物理容量（BDP）之前，传输时延的极限就是 RTprop，对应上半图中蓝色的横线当数据塞满了整条链路的物理容量后，路由器开始启用缓存来存储比特数据，这相当于拉长了整个链路，造成传输时延开始变大，偏离了物理极限 RTprop，于是有了`slope = 1/BtlBw`那条绿色斜线，当路由器的缓存填满后（BDP+BtlBufSize），整条链路开始丢数据，`1/BtlBw`斜线消失，对应于上半图中红点虚线。

* 下半图
当链路上正在传输的比特数据未超过整条链路的物理容量（BDP）之前，观察到的数据带宽是逐渐往上涨的，这个带宽的上涨速率和 RTT 成反比，即`slope = 1/RTprop`，对应下半图中蓝色斜线，当数据塞满了整条链路的物理容量后，路由器开始启用缓存来存储比特数据，但不影响 B 端观察到的带宽，这个带宽的极限就是 BtlBw，对应于图中那条绿色的 BtlBw 横线，当路由器的缓存填满后（BDP+BtlBufSize），整条链路开始丢数据，但 B 端观察到的带宽极限还是 BtlBw，对应于下半图中红点虚线。

基于丢包的拥塞控制如上半图中指出的那样，其作用域在`bandwidth limited`，这时缓冲慢慢被填满，最后导致 buffer 溢出，出现丢包。在早前，内存价格较贵时，缓冲大小约等于一个 BDP，现在由于技术的进步，内存价格一直下降，缓冲大小越来越大，都快高出 BDP 一个数量级。这样，时延也由以前的毫秒级升到了秒级，自然也带来了`bufferbloat`问题。

可以看出经典 TCP 拥塞算法的目标都是收敛于一个逻辑滞后的收敛点，不断地增加数据的传输，试图填满整个网络以及网络上的所有缓存，以为这样就会达到比较高的带宽利用率，直到发现丢包，然后迅速降低数据发送量，之后重新向那个错误的收敛点前进，如此反复。这就是锯齿的根源。而 BBR 则在一个不随时间滑动的大概 10 秒的时间窗口中采集最小 RTT，如果有更多的带宽，那么就利用它，如果没有，就退回到之前的状态。

### 6.5 BBR 的实现

前面说了要抛弃基于丢包的拥塞控制思想，换个新的思想。那到底应该怎么做才能达到`高吞吐量`和`低延迟`？。其实很简单，我只要算出链路的 BDP，
然后保持下面两个条件：
1. `Rate balance`：发送达速率等于传输流的瓶颈带宽-保证高吞吐量。
2. `Full pipe`：沿着路径流动的总数据量等于 BDP(BtlBw*Rtprop)-保证低延时。  

第一个条件保证瓶颈带宽被 100%的利用，第二个条件防止瓶颈链路出现饥饿，但又不会溢出。这时候链路的丢包对我来说已经不重要了。    
那要算出 BDP，又该如何操作？因为`BDP = BW_max * RTT_min`，所以需要测量的是`BW_max`和`RTT_min`，也就是 BBR 的 BtlBw 和 Rtprop。
然而`BW_max`和`RTT_min`不能被同时测得。要测量最大带宽，就要把瓶颈链路填满，此时缓冲中有一定量的数据包，延迟较高。
要测量最低延迟，就要保证缓冲为空，网络中数据包越少越好，但此时带宽较低。  
BBR 的解决办法是：交替测量带宽和延迟，用一段时间内的带宽极大值和延迟极小值作为估计值。

### 6.6 即时带宽

这句话讲的太笼统，怎么测试带宽？怎么测量延时？怎么个交替法？在多长时间內测试？实时测试带宽对于 bbr 是如此重要，以至于 bbr 的提出者专门写了一个 rfc 来说明应该怎么测试 BtlBw，BtlBw 是 bbr 一切计算的基准，理解了 BtlBw 的计算，解其他部分都是水到渠成的事。  bbr 即时计算两种速率`Send Rate`和`ACK Rate`取其中较小者。
* Send Rate
```txt
send_rate = data_acked / send_elapsed
send_rate = (C.delivered - P.delivered) /
                (P.sent_time - P.first_sent_time)
```
* ACK Rate
```txt
ack_rate = data_acked / ack_elapsed
ack_rate = (C.delivered - P.delivered) /
                (C.delivered_time - P.delivered_time)
```
`delivery_rate = min(send_rate, ack_rate)`  

因为都是取 data_acked 来算，区别是采样的时间不一样，所以我们只需要取 ack_elapsed 和 send_elapsed 中的较大者就可以了
```txt
delivery_elapsed = max(send_elapsed, ack_elapsed)
delivery_rate = data_acked / delivery_elapsed
```
带宽给人的直观印象是数据包通过网络传输的速度，data_acked 为 T 时间内交付到对端的数据包个数，这里的交付包括对端 ack 确认的数据包，SACK 确认的数据包以及 D-SACK 确认的数据包。T 时间为输出时间间隔，max(send_elapsed, ack_elapsed)，即发送时间间隔与接收时间间隔的最大值。

![bw-calc](https://doc.shiyanlou.com/document-uid949121labid10418timestamp1555574454331.png/wm)

1，tcp 收到 a 数据包的确认，会记录 a 的交付时间 T1，得到 a 数据包发送时间为 Ta，此时交付到对端的数据包总数为 D1。
2，收到 a 的确认后，又可以发送一个新的数据包 b，此时会将 T1、Ta、D1，以及数据包 b 的发送时间 Tb，记录在数据包 b 的信息里。
3，数据包 b 被确认时，交付时间为 T2，交付到对端的数据包个数为 D2，会从数据包 b 的信息中取出 T1，Ta，Tb，D1，计算发送时间间隔`Tb-Ta`，ack 时间间隔`T2-T1`，交付数据量`D2-D1`，带宽 = `(D2-D1)/ max(T2-T1, Ta-Tb)`。
从物理上来看，估算出来的速度不可能比发送还快，所以当`ack rate`比`send rate`还高时，应该过滤掉。

### 6.7 最小时延
BtlBw 我们知道了，还需要知道 Rtprop，往返传播延迟可以在连接的整个生命周期内变化，所以 BBR 使用最近的往返延迟样本不断估计 RTProp。
BBR 不使用基于重发分组的传输时间的 RTT 样本，因为这些是不明确的，因此不可靠。BBR 也不使用
BBR 使用在过去的 RTpropFilterLen 秒内通过连接看到的最近的最小 RTT 样本来估计 BBR.RTprop，RTpropFilterLen=10s。
```txt
onACK:
    packet.rtt = Now() - packet.send_time // 自己计算rtt，不使用tcp自带的srtt

    BBRUpdateRTprop()
        // 判断在RTpropFilterLen时间內rtprop_stamp是否更改，过期的话会进入ProbeRTT阶段，下面会讲。
        BBR.rtprop_expired =
            Now() > BBR.rtprop_stamp + RTpropFilterLen 
    
        if (packet.rtt >= 0 and     // 如果rtt小于之前的采样 或 需要重新探测，那么更新min rtt
            (packet.rtt <= BBR.RTprop or BBR.rtprop_expired))
            BBR.RTprop = packet.rtt
            BBR.rtprop_stamp = Now()
```
#### 6.7.1 pacing rate
为了帮助将数据包到达率与瓶颈链路的离开率相匹配，BBR 需要实现调整数据包的发送速度，BBR 通过`pacing rate`机制实现特性。  
发送主机通过在每个分组被调度用于传输时保持分组间间隔来实现 pacing，计算给定流的分组的下一传输时间`next_send_time`：
```txt
pacing_rate = BBR.pacing_gain * BBR.BtlBw
next_send_time = Now() + packet.size / pacing_rate
```
BBR.pacing_gain 是 BBR 带宽增益系数，用来控制和探测带宽，后面会详细讲，计算出`next_send_time`后，每次发包
都需要先检查当前时间是否超过`next_send_time`，如果没有超过，那么不能允许该数据包发送。

#### 6.7.2 target cwnd
```txt
BBRUpdateTargetCwnd():
    BBR.target_cwnd =  BBR.BtlBw * BBR.RTprop * BBR.cwnd_gain
```
cwnd 控制 BBR 允许在任何时间在网络中飞行的最大数据量，BBR 根据其网络路径模型和状态机来决定来调整 cwnd。默认情况下，BBR 增长其 cwnd 以满足其目标 cwnd，该目标 cwnd 被缩放以适应从其路径模型计算的估计 BDP。但 BBR 选择的 cwnd 旨在明确权衡各种条件下的竞争考虑因素。因此，在损失恢复中，BBR 更保守地基于更近期的传递样本调整其发送行为，并且如果 BBR 需要重新探测路径的当前 BBR.RTprop，则其相应地削减其 cwnd。

BBR 之前--依靠拥塞算法加性增窗，然后用这个窗口计算 Pacing Rate，窗口控制了一切。BBR 以及之后--正好相反，先采集并调整 Pacing Rate，再通过这个 Pacing Rate 和最小 RTT 计算窗口。因此，BBR 的窗口是相对精确”测量“出来的，而不是通过算法”计算“出来的。对于 BBR 而言，窗口的意义仅仅是限制网络中存在的最大数据量，而无法控制数据发送的 Pacing Rate 了

### 6.8 状态转换
```txt
                |
                V
       +---> Startup  ----+
       |        |         |
       |        V         |
       |      Drain   ----+
       |        |         |
       |        V         |
       +---> ProbeBW -----+
       |      ^    |      |
       |      |    |      |
       |      +----+      |
       |                  |
       +---- ProbeRTT <---+
```

1. Startup 当连接建立时，BBR 采用类似标准 TCP 的`slow start`，指数增加发送速率，目的也是尽可能快的占满管道，经过三次发现投递率不再增长，说明管道被填满，开始占用`buffer`，接下来进入排空阶段。

2. Drain 在排空阶段，指数降低发送速率，（相当于是`startup`的逆过程）将多占的`buffer`慢慢排空，直到 inflight <= BDP 时，进入 ProbeBW 阶段。

3. ProbeBW 完成上面两步，进入稳定状态后，进行带宽探测，占据 BBR 绝大部分时间，并根据 8 个阶段的`cycle`设置增益参数，每个阶段持续时间为一个`RTprop`。
 
![probebw](https://doc.shiyanlou.com/document-uid949121labid10418timestamp1555574488116.png/wm)

在`startup`阶段，BBR 已经得到网络带宽的估计值。在带宽探测阶段，BBR 利用一个叫做`cycle gain`的数组控制发送速率，进行带宽的更新。`cycle gain`数组的值为`5/4, 3/4, 1, 1, 1, 1, 1, 1`，其实就是我们上讲的在计算`pacing rate`的`BBR.pacing_gain`，BBR 将`max_BtlBW * cycle gain`的值作为发送速率。故如果数组的值是 1，就保持当前的发送速率，如果是 1.25，就增加发送速率至 1.25 倍 BW ，如果是 0.75，BBR 减小发送速率至 0.75 倍 BW 。

4. ProbeRTT 还有一个阶段是延迟探测阶段，BBR 每过 10 秒，如果估计延迟不变，就进入延迟探测阶段，为了探测最小延迟，BBR 在这段时间内(200ms)发送窗口固定为 4 个包，即几乎不发包，占整个过程 2% 的时间。还有一点需要注意的是 BBR 没有 ssthresh，因为根据计算的 delivery rate 就可以大致推断 sender 的发送速率是否达到网络带宽上限。

### 6.9 应用限速的检测  

#### 6.9.1 其他
丢包的处理  
```txt
    // 保存丢包的前的窗口
    BBR.prior_cwnd = BBRSaveCwnd()
    // 设置窗口为1
    cwnd = 1
```
快速恢复的处理  


检查管道是否充满  

#### 6.9.2 bbr 的问题

在队列缓存比较大的时候，BBR 将等不到 Reno，CUBIC 之类的流出事而降速，深队列将会让努力不排队不丢包的 BBR 耗不起。BBR 维持一个比较高的速度来发送数据，此时来了 CUBIC 流，在它占据缓存到丢包之前，BBR 流的速率是逐渐降低的--这是因为队列被 CUBIC 流所占，BBR 包之间被 CUBIC 包插队，RTT 是逐渐升高的--这是因为 CUBIC 流排队增加了排队延迟。BBR 此时在等待队列被填满后 CUBIC 乘性减窗，但是缓存实在太大了，在队列填满之前，BBR 的缺省 10 个采样周期带宽窗口已经滑过去了几个，窗口内留下的是一个又一个减小的带宽，同样，只要在缺省的 10 秒内，RTT 会一直持续增加，结局就是，BBR 带宽在持续减少。

至此三种拥塞算法已经讲完，tcp 的拥塞算法远不止这些，感兴趣的同学可以去网上查找更多的拥塞算法，现在我们看一下 Reno 和 Cubic 算法的实现源码。
### 6.10 拥塞控制框架
因为拥塞控制，控制的是发送端发送数据的多少，所以拥塞控制的实现全部在发送器上实现。
```go
// tcp拥塞控制：根据算法名，新建拥塞控制算法和初始化
func (s *sender) initCongestionControl(congestionControlName CongestionControlOption) congestionControl {
	switch congestionControlName {
	case ccCubic:
		return newCubicCC(s)
	case ccReno:
		fallthrough
	default:
		return newRenoCC(s)
	}
}
```

接收到 ack 时，拥塞控制的处理
```go
//收到段时调用 handleRcvdSegment; 它负责更新与发送相关的状态。
func (s *sender) handleRcvdSegment(seg *segment) {
	...

	// Count the duplicates and do the fast retransmit if needed.
	// tcp的拥塞控制：检查是否有重复的ack，是否进入快速重传和快速恢复状态
	rtx := s.checkDuplicateAck(seg)

    ...
    
	// Ignore ack if it doesn't acknowledge any new data.
	// 获取确认号
	ack := seg.ackNumber
	// 如果ack在最小未确认的seq和下一seg的seq之间
	if (ack - 1).InRange(s.sndUna, s.sndNxt) {
		s.dupAckCount = 0
		// When an ack is received we must reset the timer. We stop it
		// here and it will be restarted later if needed.
		s.resendTimer.disable()
        ...

		// Remove all acknowledged data from the write list.
		acked := s.sndUna.Size(ack)
		s.sndUna = ack

		ackLeft := acked
		originalOutstanding := s.outstanding
		for ackLeft > 0 {
			// We use logicalLen here because we can have FIN
			// segments (which are always at the end of list) that
			// have no data, but do consume a sequence number.
			seg := s.writeList.Front()
			datalen := seg.logicalLen()

			if datalen > ackLeft {
				seg.data.TrimFront(int(ackLeft))
				break
			}

			if s.writeNext == seg {
				s.writeNext = seg.Next()
			}
			s.writeList.Remove(seg)
			s.outstanding--
			seg.decRef()
			ackLeft -= datalen
		}

	    ...

		// If we are not in fast recovery then update the congestion
		// window based on the number of acknowledged packets.
		if !s.fr.active {
			s.cc.Update(originalOutstanding - s.outstanding)
		}

		// It is possible for s.outstanding to drop below zero if we get
		// a retransmit timeout, reset outstanding to zero but later
		// get an ack that cover previously sent data.
		if s.outstanding < 0 {
			s.outstanding = 0
		}
	}

	// 如果需要快速重传，则从传数据，但是只是重传下一个数据
	if rtx {
		// tcp拥塞控制：快速重传
		s.resendSegment()
	}

	// 现在某些待处理数据已被确认，或者窗口打开，或者由于快速恢复期间出现重复的ack而导致拥塞窗口膨胀，
	// 因此发送更多数据。如果需要，这也将重新启用重传计时器。
	s.sendData()
}
```

```go
// tcp的拥塞控制：发生重传即认为发送丢包，拥塞控制需要对丢包进行相应的处理。
func (s *sender) retransmitTimerExpired() bool {
	// Check if the timer actually expired or if it's a spurious wake due
	// to a previously orphaned runtime timer.
	// 检查计时器是否真的到期
	if !s.resendTimer.checkExpiration() {
		return true
	}

	// Give up if we've waited more than a minute since the last resend.
	// 如果rto已经超过了1分钟，直接放弃发送，返回错误
	if s.rto >= 60*time.Second {
		return false
	}

	// Set new timeout. The timer will be restarted by the call to sendData
	// below.
	// 每次超时，rto都变成原来的2倍
	s.rto *= 2

	// tcp的拥塞控制：
	if s.fr.active {
		// We were attempting fast recovery but were not successful.
		// Leave the state. We don't need to update ssthresh because it
		// has already been updated when entered fast-recovery.
		s.leaveFastRecovery()
	}

	// See: https://tools.ietf.org/html/rfc6582#section-3.2 Step 4.
	// We store the highest sequence number transmitted in cases where
	// we were not in fast recovery.
	s.fr.last = s.sndNxt - 1

	// tcp的拥塞控制：
	s.cc.HandleRTOExpired()

	// Mark the next segment to be sent as the first unacknowledged one and
	// start sending again. Set the number of outstanding packets to 0 so
	// that we'll be able to retransmit.
	//
	// We'll keep on transmitting (or retransmitting) as we get acks for
	// the data we transmit.
	// tcp可靠性：将下一个段标记为第一个未确认的段，然后再次开始发送。将未完成的数据包数设置为0，以便我们能够重新传输。
	// 当我们收到我们传输的数据时，我们将继续传输（或重新传输）。
	s.outstanding = 0
	s.writeNext = s.writeList.Front()
	s.sendData()

	return true
}

// 进入快速恢复状态和相应的处理
func (s *sender) enterFastRecovery() {
	s.fr.active = true
	// Save state to reflect we're now in fast recovery.
	// See : https://tools.ietf.org/html/rfc5681#section-3.2 Step 3.
	// We inflat the cwnd by 3 to account for the 3 packets which triggered
	// the 3 duplicate ACKs and are now not in flight.
	s.sndCwnd = s.sndSsthresh + 3
	s.fr.first = s.sndUna
	s.fr.last = s.sndNxt - 1
	s.fr.maxCwnd = s.sndCwnd + s.outstanding
}

// 退出快速恢复状态和相应的处理
func (s *sender) leaveFastRecovery() {
	s.fr.active = false
	s.fr.first = 0
	s.fr.last = s.sndNxt - 1
	s.fr.maxCwnd = 0
	s.dupAckCount = 0

	// Deflate cwnd. It had been artificially inflated when new dups arrived.
	s.sndCwnd = s.sndSsthresh
	s.cc.PostRecovery()
}

// checkDuplicateAck is called when an ack is received. It manages the state
// related to duplicate acks and determines if a retransmit is needed according
// to the rules in RFC 6582 (NewReno).
// 收到确认时调用 checkDuplicateAck。它管理与重复确认相关的状态，
// 并根据RFC 6582（NewReno）中的规则确定是否需要重新传输
func (s *sender) checkDuplicateAck(seg *segment) (rtx bool) {
	ack := seg.ackNumber
	if s.fr.active {
		// We are in fast recovery mode. Ignore the ack if it's out of
		// range.
		if !ack.InRange(s.sndUna, s.sndNxt+1) {
			return false
		}

		// Leave fast recovery if it acknowledges all the data covered by
		// this fast recovery session.
		if s.fr.last.LessThan(ack) {
			s.leaveFastRecovery()
			return false
		}

		// Don't count this as a duplicate if it is carrying data or
		// updating the window.
		if seg.logicalLen() != 0 || s.sndWnd != seg.window {
			return false
		}

		// Inflate the congestion window if we're getting duplicate acks
		// for the packet we retransmitted.
		if ack == s.fr.first {
			// We received a dup, inflate the congestion window by 1
			// packet if we're not at the max yet.
			if s.sndCwnd < s.fr.maxCwnd {
				s.sndCwnd++
			}
			return false
		}

		// A partial ack was received. Retransmit this packet and
		// remember it so that we don't retransmit it again. We don't
		// inflate the window because we're putting the same packet back
		// onto the wire.
		//
		// N.B. The retransmit timer will be reset by the caller.
		s.fr.first = ack
		s.dupAckCount = 0
		return true
	}

	// We're not in fast recovery yet. A segment is considered a duplicate
	// only if it doesn't carry any data and doesn't update the send window,
	// because if it does, it wasn't sent in response to an out-of-order
	// segment.
	if ack != s.sndUna || seg.logicalLen() != 0 || s.sndWnd != seg.window || ack == s.sndNxt {
		s.dupAckCount = 0
		return false
	}

	s.dupAckCount++
	// Do not enter fast recovery until we reach nDupAckThreshold.
	if s.dupAckCount < nDupAckThreshold {
		return false
	}

	// See: https://tools.ietf.org/html/rfc6582#section-3.2 Step 2
	//
	// We only do the check here, the incrementing of last to the highest
	// sequence number transmitted till now is done when enterFastRecovery
	// is invoked.
	if !s.fr.last.LessThan(seg.ackNumber) {
		s.dupAckCount = 0
		return false
	}

	s.cc.HandleNDupAcks()
	s.enterFastRecovery()
	s.dupAckCount = 0
	return true
}

```

### 6.11 Reno
reno 的核心实现  
```go
type renoState struct {
	s *sender
}

// newRenoCC initializes the state for the NewReno congestion control algorithm.
// 新建 reno 算法对象
func newRenoCC(s *sender) *renoState {
	return &renoState{s: s}
}

// updateSlowStart will update the congestion window as per the slow-start
// algorithm used by NewReno. If after adjusting the congestion window
// we cross the SSthreshold then it will return the number of packets that
// must be consumed in congestion avoidance mode.
// updateSlowStart 将根据NewReno使用的慢启动算法更新拥塞窗口。
// 如果在调整拥塞窗口后我们越过了 SSthreshold ，那么它将返回在拥塞避免模式下必须消耗的数据包的数量。
func (r *renoState) updateSlowStart(packetsAcked int) int {
	// Don't let the congestion window cross into the congestion
	// avoidance range.
	// 在慢启动阶段，每次收到ack，sndCwnd加上已确认的段数
	newcwnd := r.s.sndCwnd + packetsAcked
	// 判断增大过后的拥塞窗口是否超过慢启动阀值 sndSsthresh，
	// 如果超过 sndSsthresh ，将窗口调整为 sndSsthresh
	if newcwnd >= r.s.sndSsthresh {
		newcwnd = r.s.sndSsthresh
		r.s.sndCAAckCount = 0
	}
	// 是否超过 sndSsthresh， packetsAcked>0表示超过
	packetsAcked -= newcwnd - r.s.sndCwnd
	// 更新拥塞窗口
	r.s.sndCwnd = newcwnd
	return packetsAcked
}

// updateCongestionAvoidance will update congestion window in congestion
// avoidance mode as described in RFC5681 section 3.1
// updateCongestionAvoidance 将在拥塞避免模式下更新拥塞窗口，
// 如RFC5681第3.1节所述
func (r *renoState) updateCongestionAvoidance(packetsAcked int) {
	// Consume the packets in congestion avoidance mode.
	// sndCAAckCount 累计收到的tcp段数
	r.s.sndCAAckCount += packetsAcked
	// 如果累计的段数超过当前的拥塞窗口，那么 sndCwnd 加上 sndCAAckCount/sndCwnd 的整数倍
	if r.s.sndCAAckCount >= r.s.sndCwnd {
		r.s.sndCwnd += r.s.sndCAAckCount / r.s.sndCwnd
		r.s.sndCAAckCount = r.s.sndCAAckCount % r.s.sndCwnd
	}
}

// reduceSlowStartThreshold reduces the slow-start threshold per RFC 5681,
// page 6, eq. 4. It is called when we detect congestion in the network.
// 当检测到网络拥塞时，调用 reduceSlowStartThreshold。
// 它将 sndSsthresh 变为 outstanding 的一半。
// sndSsthresh 最小为2，因为至少要比丢包后的拥塞窗口（cwnd=1）来的大，才会进入慢启动阶段。
func (r *renoState) reduceSlowStartThreshold() {
	r.s.sndSsthresh = r.s.outstanding / 2
	if r.s.sndSsthresh < 2 {
		r.s.sndSsthresh = 2
	}
}

// Update updates the congestion state based on the number of packets that
// were acknowledged.
// Update implements congestionControl.Update.
// packetsAcked 表示已确认的tcp段数
func (r *renoState) Update(packetsAcked int) {
	// 当拥塞窗口没有超过慢启动阀值的时候，使用慢启动来增大窗口，
	// 否则进入拥塞避免阶段
	if r.s.sndCwnd < r.s.sndSsthresh {
		packetsAcked = r.updateSlowStart(packetsAcked)
		if packetsAcked == 0 {
			return
		}
	}
	// 进入拥塞避免阶段
	r.updateCongestionAvoidance(packetsAcked)
}

// HandleNDupAcks implements congestionControl.HandleNDupAcks.
// 当收到三个重复ack时，调用 HandleNDupAcks 来处理。
func (r *renoState) HandleNDupAcks() {
	// A retransmit was triggered due to nDupAckThreshold
	// being hit. Reduce our slow start threshold.
	// 减小慢启动阀值
	r.reduceSlowStartThreshold()
}

// HandleRTOExpired implements congestionControl.HandleRTOExpired.
// 当当发生重传包时，调用 HandleRTOExpired 来处理。
func (r *renoState) HandleRTOExpired() {
	// We lost a packet, so reduce ssthresh.
	// 减小慢启动阀值
	r.reduceSlowStartThreshold()

	// Reduce the congestion window to 1, i.e., enter slow-start. Per
	// RFC 5681, page 7, we must use 1 regardless of the value of the
	// initial congestion window.
	// 更新拥塞窗口为1，这样就会重新进入慢启动
	r.s.sndCwnd = 1
}
```


### 6.12 CUBIC
```go
// cubicState 存储与TCP CUBIC拥塞控制算法状态相关的变量。详见: https://tools.ietf.org/html/rfc8312.
type cubicState struct {
	// wLastMax is the previous wMax value.
	// 上次最大的拥塞窗口
	wLastMax float64

	// wMax is the value of the congestion window at the
	// time of last congestion event.
	// 上次拥塞时间时的拥塞窗口大小
	wMax float64

	// t denotes the time when the current congestion avoidance
	// was entered.
	// t 表进入拥塞避免阶段的时间。
	t time.Time

	// numCongestionEvents tracks the number of congestion events since last
	// RTO.
	// numCongestionEvents 跟踪自上次 RTO 以来的拥塞事件数。
	numCongestionEvents int

	// c is the cubic constant as specified in RFC8312. It's fixed at 0.4 as
	// per RFC.
	// 三次方函数的系数
	c float64

	// k is the time period that the above function takes to increase the
	// current window size to W_max if there are no further congestion
	// events and is calculated using the following equation:
	// k是在没有其他拥塞事件时将当前窗口大小增加到W_max所需的时间段，并使用以下公式计算：
	//
	// K = cubic_root(W_max*(1-beta_cubic)/C) (Eq. 2)
	k float64

	// beta is the CUBIC multiplication decrease factor. that is, when a
	// congestion event is detected, CUBIC reduces its cwnd to
	// W_cubic(0)=W_max*beta_cubic.
	// beta 是CUBIC乘法减少因子。也就是说，当检测到拥塞事件时，CUBIC将其cwnd减少到
	beta float64

	// wC is window computed by CUBIC at time t. It's calculated using the
	// formula:
	// wC 是由CUBIC在时间t计算的窗口。它使用公式计算：
	//
	//  W_cubic(t) = C*(t-K)^3 + W_max (Eq. 1)
	wC float64

	// wEst is the window computed by CUBIC at time t+RTT i.e
	// wEs t是CUBIC在时间 t+RTT 计算的窗口，即
	// W_cubic(t+RTT).
	wEst float64

	s *sender
}

// newCubicCC returns a partially initialized cubic state with the constants
// beta and c set and t set to current time.
// newCubicCC 返回部分初始化的 cubic 状态，常量为beta和c，t为当前时间。
func newCubicCC(s *sender) *cubicState {
	return &cubicState{
		t:    time.Now(),
		beta: 0.7,
		c:    0.4,
		s:    s,
	}
}

// enterCongestionAvoidance is used to initialize cubic in cases where we exit
// SlowStart without a real congestion event taking place. This can happen when
// a connection goes back to slow start due to a retransmit and we exceed the
// previously lowered ssThresh without experiencing packet loss.
//
// Refer: https://tools.ietf.org/html/rfc8312#section-4.8
// enterCongestionAvoidance 用于在我们退出 SlowStart 而没有发生真正的拥塞事件的情况下初始化 cubic。
// 当连接由于重新传输而返回慢启动时会发生这种情况，并且我们超过先前降低的ssThresh而不会遇到丢包。
func (c *cubicState) enterCongestionAvoidance() {
	// See: https://tools.ietf.org/html/rfc8312#section-4.7 &
	// https://tools.ietf.org/html/rfc8312#section-4.8
	// 初次进入拥塞避免，只记录当前拥塞避免的时间点，当前的窗口wMax=cwnd
	if c.numCongestionEvents == 0 {
		c.k = 0
		c.t = time.Now()
		c.wLastMax = c.wMax
		c.wMax = float64(c.s.sndCwnd)
	}
}

// updateSlowStart will update the congestion window as per the slow-start
// algorithm used by NewReno. If after adjusting the congestion window we cross
// the ssThresh then it will return the number of packets that must be consumed
// in congestion avoidance mode.
// updateSlowStart 将根据NewReno使用的慢启动算法更新拥塞窗口。
// 如果在调整拥塞窗口之后我们越过ssThresh，那么它将返回在拥塞避免模式下必须消耗的数据包的数量。
func (c *cubicState) updateSlowStart(packetsAcked int) int {
	// Don't let the congestion window cross into the congestion
	// avoidance range.
	newcwnd := c.s.sndCwnd + packetsAcked
	enterCA := false
	if newcwnd >= c.s.sndSsthresh {
		newcwnd = c.s.sndSsthresh
		c.s.sndCAAckCount = 0
		enterCA = true
	}

	packetsAcked -= newcwnd - c.s.sndCwnd
	c.s.sndCwnd = newcwnd
	if enterCA {
		// 进入拥塞避免
		c.enterCongestionAvoidance()
	}
	return packetsAcked
}

// Update updates cubic's internal state variables. It must be called on every
// ACK received.
// Refer: https://tools.ietf.org/html/rfc8312#section-4
// Update 更新 cubic 内部状态变量，每次收到ack都必须调用
func (c *cubicState) Update(packetsAcked int) {
	if c.s.sndCwnd < c.s.sndSsthresh {
		packetsAcked = c.updateSlowStart(packetsAcked)
		if packetsAcked == 0 {
			return
		}
	} else {
		// 拥塞避免阶段
		c.s.rtt.Lock()
		srtt := c.s.rtt.srtt
		c.s.rtt.Unlock()
		c.s.sndCwnd = c.getCwnd(packetsAcked, c.s.sndCwnd, srtt)
	}
}

// cubicCwnd computes the CUBIC congestion window after t seconds from last
// congestion event.
// cubicCwnd 在上次拥塞事件发生t秒后计算CUBIC拥塞窗口。
func (c *cubicState) cubicCwnd(t float64) float64 {
	return c.c*math.Pow(t, 3.0) + c.wMax
}

// getCwnd returns the current congestion window as computed by CUBIC.
// Refer: https://tools.ietf.org/html/rfc8312#section-4
// getCwnd 返回由CUBIC计算的当前拥塞窗口。
func (c *cubicState) getCwnd(packetsAcked, sndCwnd int, srtt time.Duration) int {
	elapsed := time.Since(c.t).Seconds()

	// Compute the window as per Cubic after 'elapsed' time
	// since last congestion event.
	c.wC = c.cubicCwnd(elapsed - c.k)

	// Compute the TCP friendly estimate of the congestion window.
	c.wEst = c.wMax*c.beta + (3.0*((1.0-c.beta)/(1.0+c.beta)))*(elapsed/srtt.Seconds())

	// Make sure in the TCP friendly region CUBIC performs at least
	// as well as Reno.
	if c.wC < c.wEst && float64(sndCwnd) < c.wEst {
		// TCP Friendly region of cubic.
		return int(c.wEst)
	}

	// In Concave/Convex region of CUBIC, calculate what CUBIC window
	// will be after 1 RTT and use that to grow congestion window
	// for every ack.
	tEst := (time.Since(c.t) + srtt).Seconds()
	wtRtt := c.cubicCwnd(tEst - c.k)
	// As per 4.3 for each received ACK cwnd must be incremented
	// by (w_cubic(t+RTT) - cwnd/cwnd.
	cwnd := float64(sndCwnd)
	for i := 0; i < packetsAcked; i++ {
		// Concave/Convex regions of cubic have the same formulas.
		// See: https://tools.ietf.org/html/rfc8312#section-4.3
		cwnd += (wtRtt - cwnd) / cwnd
	}
	return int(cwnd)
}

// HandleNDupAcks implements congestionControl.HandleNDupAcks.
// 收到三次重复ack，调用 HandleNDupAcks
func (c *cubicState) HandleNDupAcks() {
	// See: https://tools.ietf.org/html/rfc8312#section-4.5
	c.numCongestionEvents++
	c.t = time.Now()
	c.wLastMax = c.wMax
	c.wMax = float64(c.s.sndCwnd)

	c.fastConvergence()
	c.reduceSlowStartThreshold()
}

// HandleRTOExpired implements congestionContrl.HandleRTOExpired.
// 发生重传时调用 HandleRTOExpired。
func (c *cubicState) HandleRTOExpired() {
	// See: https://tools.ietf.org/html/rfc8312#section-4.6
	c.t = time.Now()
	c.numCongestionEvents = 0
	c.wLastMax = c.wMax
	c.wMax = float64(c.s.sndCwnd)

	c.fastConvergence()

	// We lost a packet, so reduce ssthresh.
	c.reduceSlowStartThreshold()

	// Reduce the congestion window to 1, i.e., enter slow-start. Per
	// RFC 5681, page 7, we must use 1 regardless of the value of the
	// initial congestion window.
	c.s.sndCwnd = 1
}

// fastConvergence implements the logic for Fast Convergence algorithm as
// described in https://tools.ietf.org/html/rfc8312#section-4.6.
// 快速收敛
func (c *cubicState) fastConvergence() {
	if c.wMax < c.wLastMax {
		c.wLastMax = c.wMax
		c.wMax = c.wMax * (1.0 + c.beta) / 2.0
	} else {
		c.wLastMax = c.wMax
	}
	// Recompute k as wMax may have changed.
	c.k = math.Cbrt(c.wMax * (1 - c.beta) / c.c)
}

// PostRecovery implemements congestionControl.PostRecovery.
// 更新t为当前的时间，当发送方退出快速恢复阶段时，将调用 PostRecovery
func (c *cubicState) PostRecovery() {
	c.t = time.Now()
}

// reduceSlowStartThreshold returns new SsThresh as described in
// https://tools.ietf.org/html/rfc8312#section-4.7.
// 按 cubic 的算法更新慢启动和拥塞避免之间的阈值
func (c *cubicState) reduceSlowStartThreshold() {
	c.s.sndSsthresh = int(math.Max(float64(c.s.sndCwnd)*c.beta, 2.0))
}

```

### 6.13 tcp 消息回显实验
```go
// +build linux

package main

import (
	"flag"
	"log"
	"net"
	"os"
	"strconv"
	"strings"

	"netstack/tcpip"
	"netstack/tcpip/buffer"
	"netstack/tcpip/link/loopback"
	"netstack/tcpip/network/arp"
	"netstack/tcpip/network/ipv4"
	"netstack/tcpip/stack"
	"netstack/tcpip/transport/tcp"
	"netstack/waiter"
)

func main() {
	flag.Parse()
	log.SetFlags(log.Lshortfile | log.LstdFlags)

	if len(os.Args) != 3 {
		log.Fatal("Usage: ", os.Args[0], "<ipv4-address> <port>")
	}

	addrName := os.Args[1]
	portName := os.Args[2]

	addr := tcpip.Address(net.ParseIP(addrName).To4())
	port, err := strconv.Atoi(portName)
	if err != nil {
		log.Fatalf("Unable to convert port %v: %v", portName, err)
	}

	s := newStack(addr, port)

	done := make(chan int, 1)
	go tcpServer(s, addr, port, done)
	<-done

	tcpClient(s, addr, port)
}

func newStack(addr tcpip.Address, port int) *stack.Stack {
	// 创建本地环回网卡
	linkID := loopback.New()

	// 新建相关协议的协议栈
	s := stack.New([]string{ipv4.ProtocolName, arp.ProtocolName},
		[]string{tcp.ProtocolName}, stack.Options{})

	// 新建抽象的网卡
	if err := s.CreateNamedNIC(1, "lo0", linkID); err != nil {
		log.Fatal(err)
	}

	// 在该网卡上添加和注册相应的网络层
	if err := s.AddAddress(1, ipv4.ProtocolNumber, addr); err != nil {
		log.Fatal(err)
	}

	// 在该协议栈上添加和注册ARP协议
	if err := s.AddAddress(1, arp.ProtocolNumber, arp.ProtocolAddress); err != nil {
		log.Fatal(err)
	}

	// 添加默认路由
	s.SetRouteTable([]tcpip.Route{
		{
			Destination: tcpip.Address(strings.Repeat("\x00", len(addr))),
			Mask:        tcpip.AddressMask(strings.Repeat("\x00", len(addr))),
			Gateway:     "",
			NIC:         1,
		},
	})

	return s
}

func tcpServer(s *stack.Stack, addr tcpip.Address, port int, done chan int) {
	var wq waiter.Queue
	// 新建一个TCP端
	ep, e := s.NewEndpoint(tcp.ProtocolNumber, ipv4.ProtocolNumber, &wq)
	if e != nil {
		log.Fatal(e)
	}

	// 绑定本地端口
	if err := ep.Bind(tcpip.FullAddress{0, "", uint16(port)}, nil); err != nil {
		log.Fatal("Bind failed: ", err)
	}

	// 监听tcp
	if err := ep.Listen(10); err != nil {
		log.Fatal("Listen failed: ", err)
	}

	// Wait for connections to appear.
	waitEntry, notifyCh := waiter.NewChannelEntry(nil)
	wq.EventRegister(&waitEntry, waiter.EventIn)
	defer wq.EventUnregister(&waitEntry)
	done <- 1
	for {
		n, wq, err := ep.Accept()
		if err != nil {
			if err == tcpip.ErrWouldBlock {
				<-notifyCh
				continue
			}

			log.Fatal("Accept() failed:", err)
		}
		ra, err := n.GetRemoteAddress()
		log.Printf("new conn: %v %v", ra, err)
		go echo(wq, n)
	}
}

func tcpClient(s *stack.Stack, addr tcpip.Address, port int) {
	remote := tcpip.FullAddress{
		Addr: addr,
		Port: uint16(port),
	}

	var wq waiter.Queue
	// 新建一个TCP端
	ep, e := s.NewEndpoint(tcp.ProtocolNumber, ipv4.ProtocolNumber, &wq)
	if e != nil {
		log.Fatal(e)
	}

	defer ep.Close()

	// Issue connect request and wait for it to complete.
	waitEntry, notifyCh := waiter.NewChannelEntry(nil)
	wq.EventRegister(&waitEntry, waiter.EventOut)
	terr := ep.Connect(remote)
	if terr == tcpip.ErrConnectStarted {
		log.Println("Connect is pending...")
		<-notifyCh
		terr = ep.GetSockOpt(tcpip.ErrorOption{})
	}
	wq.EventUnregister(&waitEntry)

	if terr != nil {
		log.Fatal("Unable to connect: ", terr)
	}

	log.Println("Connected")

	// Start the writer in its own goroutine.
	writerCompletedCh := make(chan struct{})
	go writer(writerCompletedCh, ep)

	// Read data and write to standard output until the peer closes the
	// connection from its side.
	wq.EventRegister(&waitEntry, waiter.EventIn)
	for {
		v, _, err := ep.Read(nil)
		if err != nil {
			if err == tcpip.ErrClosedForReceive {
				break
			}

			if err == tcpip.ErrWouldBlock {
				<-notifyCh
				continue
			}

			log.Fatal("Read() failed:", err)
		}

		log.Printf("tcp client read data: %s", string(v))
	}
	wq.EventUnregister(&waitEntry)

	// The reader has completed. Now wait for the writer as well.
	<-writerCompletedCh

	ep.Close()
}

// writer reads from standard input and writes to the endpoint until standard
// input is closed. It signals that it's done by closing the provided channel.
func writer(ch chan struct{}, ep tcpip.Endpoint) {
	defer func() {
		ep.Shutdown(tcpip.ShutdownWrite)
		close(ch)
	}()

	data := []byte("tcp test")
	v := buffer.View(data)
	for i := 0; i < 2; i++ {
		_, _, err := ep.Write(tcpip.SlicePayload(v), tcpip.WriteOptions{})
		if err != nil {
			log.Println("Write failed:", err)
			return
		}
	}
}

func echo(wq *waiter.Queue, ep tcpip.Endpoint) {
	defer ep.Close()

	// Create wait queue entry that notifies a channel.
	waitEntry, notifyCh := waiter.NewChannelEntry(nil)

	wq.EventRegister(&waitEntry, waiter.EventIn)
	defer wq.EventUnregister(&waitEntry)

	for {
		v, _, err := ep.Read(nil)
		if err != nil {
			if err == tcpip.ErrWouldBlock {
				<-notifyCh
				continue
			}

			return
		}

		log.Printf("tcp server echo data: %s", string(v))
		_, _, err = ep.Write(tcpip.SlicePayload(v), tcpip.WriteOptions{})
		if err != nil {
			log.Fatal(err)
		}
	}
}

```
### 6.14 实验结果
```txt
2019/03/31 19:22:05 ports.go:131: new transport: 6, port: 9000
2019/03/31 19:22:05 main.go:144: Connect is pending...
2019/03/31 19:22:05 connect.go:675: send tcp syn segment to 192.168.1.1:9000, seq: 1155880812, ack: 0, rcvWnd: 65535
2019/03/31 19:22:05 ipv4.go:151: send ipv4 packet 60 bytes, proto: 0x6
2019/03/31 19:22:05 ipv4.go:193: recv ipv4 packet 60 bytes, proto: 0x6
2019/03/31 19:22:05 endpoint.go:1395: recv tcp syn segment from 192.168.1.1:26913, seq: 1155880812, ack: 0
2019/03/31 19:22:05 connect.go:675: send tcp ack|syn segment to 192.168.1.1:26913, seq: 3223858476, ack: 1155880813, rcvWnd: 65535
2019/03/31 19:22:05 ipv4.go:151: send ipv4 packet 60 bytes, proto: 0x6
2019/03/31 19:22:05 ipv4.go:193: recv ipv4 packet 60 bytes, proto: 0x6
2019/03/31 19:22:05 endpoint.go:1395: recv tcp ack|syn segment from 192.168.1.1:9000, seq: 3223858476, ack: 1155880813
2019/03/31 19:22:05 connect.go:675: send tcp ack segment to 192.168.1.1:9000, seq: 1155880813, ack: 3223858477, rcvWnd: 32768
2019/03/31 19:22:05 ipv4.go:151: send ipv4 packet 52 bytes, proto: 0x6
2019/03/31 19:22:05 ipv4.go:193: recv ipv4 packet 52 bytes, proto: 0x6
2019/03/31 19:22:05 endpoint.go:1395: recv tcp ack segment from 192.168.1.1:26913, seq: 1155880813, ack: 3223858477
2019/03/31 19:22:05 main.go:154: Connected
2019/03/31 19:22:05 connect.go:675: send tcp ack|psh segment to 192.168.1.1:9000, seq: 1155880813, ack: 3223858477, rcvWnd: 32768
2019/03/31 19:22:05 ipv4.go:151: send ipv4 packet 60 bytes, proto: 0x6
2019/03/31 19:22:05 ipv4.go:193: recv ipv4 packet 60 bytes, proto: 0x6
2019/03/31 19:22:05 endpoint.go:1395: recv tcp ack|psh segment from 192.168.1.1:26913, seq: 1155880813, ack: 3223858477
2019/03/31 19:22:05 connect.go:675: send tcp ack|psh segment to 192.168.1.1:9000, seq: 1155880821, ack: 3223858477, rcvWnd: 32768
2019/03/31 19:22:05 ipv4.go:151: send ipv4 packet 60 bytes, proto: 0x6
2019/03/31 19:22:05 ipv4.go:193: recv ipv4 packet 60 bytes, proto: 0x6
2019/03/31 19:22:05 endpoint.go:1395: recv tcp ack|psh segment from 192.168.1.1:26913, seq: 1155880821, ack: 3223858477
2019/03/31 19:22:05 connect.go:675: send tcp ack|fin segment to 192.168.1.1:9000, seq: 1155880829, ack: 3223858477, rcvWnd: 32768
2019/03/31 19:22:05 ipv4.go:151: send ipv4 packet 52 bytes, proto: 0x6
2019/03/31 19:22:05 ipv4.go:193: recv ipv4 packet 52 bytes, proto: 0x6
2019/03/31 19:22:05 endpoint.go:1395: recv tcp ack|fin segment from 192.168.1.1:26913, seq: 1155880829, ack: 3223858477
2019/03/31 19:22:05 main.go:119: new conn: 1:192.168.1.1:26913 <nil>
2019/03/31 19:22:05 connect.go:675: send tcp ack segment to 192.168.1.1:26913, seq: 3223858477, ack: 1155880830, rcvWnd: 32767
2019/03/31 19:22:05 ipv4.go:151: send ipv4 packet 52 bytes, proto: 0x6
2019/03/31 19:22:05 ipv4.go:193: recv ipv4 packet 52 bytes, proto: 0x6
2019/03/31 19:22:05 endpoint.go:1395: recv tcp ack segment from 192.168.1.1:9000, seq: 3223858477, ack: 1155880830
2019/03/31 19:22:05 main.go:227: tcp server echo data: tcp test
2019/03/31 19:22:05 connect.go:675: send tcp ack|psh segment to 192.168.1.1:26913, seq: 3223858477, ack: 1155880830, rcvWnd: 32767
2019/03/31 19:22:05 ipv4.go:151: send ipv4 packet 60 bytes, proto: 0x6
2019/03/31 19:22:05 ipv4.go:193: recv ipv4 packet 60 bytes, proto: 0x6
2019/03/31 19:22:05 endpoint.go:1395: recv tcp ack|psh segment from 192.168.1.1:9000, seq: 3223858477, ack: 1155880830
2019/03/31 19:22:05 main.go:227: tcp server echo data: tcp test
2019/03/31 19:22:05 connect.go:675: send tcp ack|psh segment to 192.168.1.1:26913, seq: 3223858485, ack: 1155880830, rcvWnd: 32768
2019/03/31 19:22:05 ipv4.go:151: send ipv4 packet 60 bytes, proto: 0x6
2019/03/31 19:22:05 ipv4.go:193: recv ipv4 packet 60 bytes, proto: 0x6
2019/03/31 19:22:05 endpoint.go:1395: recv tcp ack|psh segment from 192.168.1.1:9000, seq: 3223858485, ack: 1155880830
2019/03/31 19:22:05 connect.go:675: send tcp ack|fin segment to 192.168.1.1:26913, seq: 3223858493, ack: 1155880830, rcvWnd: 32768
2019/03/31 19:22:05 ipv4.go:151: send ipv4 packet 52 bytes, proto: 0x6
2019/03/31 19:22:05 ipv4.go:193: recv ipv4 packet 52 bytes, proto: 0x6
2019/03/31 19:22:05 endpoint.go:1395: recv tcp ack|fin segment from 192.168.1.1:9000, seq: 3223858493, ack: 1155880830
2019/03/31 19:22:05 connect.go:675: send tcp ack segment to 192.168.1.1:9000, seq: 1155880830, ack: 3223858494, rcvWnd: 32767
2019/03/31 19:22:05 ipv4.go:151: send ipv4 packet 52 bytes, proto: 0x6
2019/03/31 19:22:05 ipv4.go:193: recv ipv4 packet 52 bytes, proto: 0x6
2019/03/31 19:22:05 endpoint.go:1395: recv tcp ack segment from 192.168.1.1:26913, seq: 1155880830, ack: 3223858494
2019/03/31 19:22:05 main.go:178: tcp client read data: tcp test
2019/03/31 19:22:05 main.go:178: tcp client read data: tcp test
```

```
TODO: 6.14实验结果是如何运行出来的，麻烦大佬提供一下命令，谢谢
```

## 七、总结

到此为止 tcp 的各个功能的实现已经讲完了，可以看出 tcp 确实比较复杂，新增了很多概念，但是将每个功能拆分开来讲解，也就不会那么难以理解了，前面一共讲了 tcp 的这么几个模块，tcp 的状态、tcp 的可靠性、tcp 的流量控制和 tcp 的拥塞控制。 这里再对比一下与 UDP 的不同点：  
- 面向连接
- 有序数据传输
- 重发丢失的数据包
- 舍弃重复的数据包
- 无差错的数据传输
- 阻塞/流量控制

