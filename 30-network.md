---
show: step
version: 1.0
enable_checker: true
---
# NetWork

## 一、实验说明

#### 1.1 实验内容

本章介绍网络层的实现，网络层又称网际层、ip 层，它是 tcpip 架构中核心的实现，全球计算机的互联很大部分归功于网络层，核心网络（路由器）都跑在网络层，为网络提供路由交换的功能，将数据包分发到相应的主机。

#### 1.2 知识点
- 网络层的基本功能
- ipv4 协议和代码实现
- 路由和路由表
- icmp 协议和代码实现

#### 1.3 实验环境

- Go 1.12.1
- Xfce 终端

#### 1.4 代码获取

```bash
$ wget http://labfile.oss.aliyuncs.com/courses/1300/netstack.zip
$ unzip netstack.zip
```

## 二、网络层基本实现

本章介绍网络层的实现，网络层又称网际层、ip 层，它是 tcpip 架构中核心的实现，全球计算机的互联很大部分归功于网络层，核心网络（路由器）都跑在网络层，为网络提供路由交换的功能，将数据包分发到相应的主机。虽然网络层在路由器上的实现比较复杂，因为要实现各种路由协议，但主机协议栈中的网络层并不复杂，因为它没有实现各种路由协议，路由表也很简单。下面介绍网络层提供的服务和实现网络层的 ip 协议-ipv4。

#### 2.1 网络层提供的服务

在计算机网络领域，曾经为网络层应该提供怎样的服务（面向连接还是无连接）引起了长时间的争论。最终因特网采用的设计思路是：`网络层向上提供简单灵活的、无连接的、尽最大努力交付的数据报服务`。所谓的数据报服务具有以下几个特点：  
1. 无需建立连接
2. 不保证可靠性
3. 每个分组都有终点的完整地址
4. 每个分组独立选择路由进行转发
5. 可靠通信应该有上层负责  
网络层的目的是实现两个主机之间的数据透明传送，具体功能包括寻址和路由选择等。它提供的服务使传输层不需要了解网络中的数据传输和交换技术。对网络层而言使用一种逻辑地址来唯一标识互联网上的设备，网络层依靠逻辑地址进行相互通信（类似于数据链路层的 MAC 地址），逻辑地址编址方案现主要有两种，IPv4 和 IPv6，我们主要讲协议栈对 IPv4 协议的处理。一般我们说 IP 地址，指的是 ipv4 地址。

#### 2.2 网络层和链路层的功能区别

之前讲过链路层也可以实现主机到主机的数据透明传输，那为何还需要网络层实现主机到主机的数据传输？  因为链路层的数据交换是在同个局域网实现的，链路层的交换也就是二层交换，它依赖二层广播 ARP 报文，来学习 MAC 地址和端口的对应关系。当交换机从某个端口收到一个数据包，它会先读取包中的源 MAC 地址，再去读取包中的目的 MAC 地址，并在地址表中查找对应的端口，如表中有和目的 MAC 地址对应的端口，就把数据包直接复制到这个端口上。链路层其最基本的服务是将源自网络层来的数据可靠地传输到相邻节点的目标机网络层。  而网络层的数据交换是不限于局域网的，网络层连接着因特网中各局域网、广域网的设备，是互联网络的枢纽。网络层的数据交换（路由交换）是根据目的 IP，查找路由表找到下一跳的 IP 地址，再根据这个下一跳 IP 地址，查找转发表，将数据包转发给相应的端口。简单的说链路层的寻址关心 MAC 地址而不管数据包中的 IP 地址，而网络层的寻址关心IP 地址，而不关心 MAC 地址，链路层和网络层的结合实现了世界上两台主机的数据互相传输。

## 三、ipv4 简介

IPv4，是互联网协议（Internet Protocol，IP）的第四版，也是第一个被广泛使用，构成现今互联网技术的基础的协议。IPv4 是一种无连接的协议，操作在使用分组交换的链路层（如以太网）上。此协议会尽最大努力交付数据包，意即它不保证任何数据包均能送达目的地，也不保证所有数据包均按照正确的顺序无重复地到达。这些方面是由上层的传输协议（如传输控制协议）处理的。

### 3.1 ip 报文

![ip_header](https://doc.shiyanlou.com/document-uid949121labid10418timestamp1555407794966.png/wm)  
* 版本（Version）   
版本字段占 4bit，通信双方使用的版本必须一致。对于 IPv4，字段的值是 4。

* 首部长度（Internet Header Length， IHL）
占 4bit，首部长度说明首部有多少 32 位字（4 字节）。由于 IPv4 首部可能包含数目不定的选项，这个字段也用来确定数据的偏移量。这个字段的最小值是 5（二进制 0101），相当于 5*4=20 字节（RFC 791），最大十进制值是 15。

* 区分服务（Differentiated Services，DS）
占 8bit，最初被定义为服务类型字段，实际上并未使用，但 1998 年被 IETF 重定义为区分服务 RFC 2474。只有在使用区分服务时，这个字段才起作用，在一般的情况  下都不使用这个字段。例如需要实时数据流的技术会应用这个字段，一个例子是 VoIP。

* 显式拥塞通告（ Explicit Congestion Notification，ECN） 
在 RFC 3168 中定义，允许在不丢弃报文的同时通知对方网络拥塞的发生。ECN 是一种可选的功能，仅当两端都支持并希望使用，且底层网络支持时才被使用。

* 全长（Total Length）  
这个 16 位字段定义了报文总长，包含首部和数据，单位为字节。这个字段的最小值是 20（20 字节首部+0 字节数据），最大值是 216-1=65,535。IP 规定所有主机都必须支持最小 576 字节的报文，这是假定上层数据长度 512 字节，加上最长 IP 首部 60 字节，加上 4 字节富裕量，得出 576 字节，但大多数现代主机支持更大的报文。当下层的数据链路协议的最大传输单元（MTU）字段的值小于 IP 报文长度时间，报文就必须被分片，详细见下个标题。

* 标识符（Identification）  
占 16 位，这个字段主要被用来唯一地标识一个报文的所有分片，因为分片不一定按序到达，所以在重组时需要知道分片所属的报文。每产生一个数据报，计数器加 1，并赋值给此字段。一些实验性的工作建议将此字段用于其它目的，例如增加报文跟踪信息以协助探测伪造的源地址。

* 标志 （Flags）  
这个 3 位字段用于控制和识别分片，它们是：   
位 0：保留，必须为 0；  位 1：禁止分片（Don’t Fragment，DF），当 DF=0 时才允许分片；  位 2：更多分片（More Fragment，MF），MF=1 代表后面还有分片，MF=0 代表已经是最后一个分片。  如果 DF 标志被设置为 1，但路由要求必须分片报文，此报文会被丢弃。这个标志可被用于发往没有能力组装分片的主机。当一个报文被分片，除了最后一片外的所有分片都设置 MF 为 1。最后一个片段具有非零片段偏移字段，将其与未分片数据包区分开，未分片的偏移字段为 0。

* 分片偏移 （Fragment Offset）  
这个 13 位字段指明了每个分片相对于原始报文开头的偏移量，以 8 字节作单位。

* 存活时间（Time To Live，TTL）  
这个 8 位字段避免报文在互联网中永远存在（例如陷入路由环路）。存活时间以秒为单位，但小于一秒的时间均向上取整到一秒。在现实中，这实际上成了一个跳数计数器：报文经过的每个路由器都将此字段减 1，当此字段等于 0 时，报文不再向下一跳传送并被丢弃，最大值是 255。常规地，一份 ICMP 报文被发回报文发送端说明其发送的报文已被丢弃。这也是 traceroute 的核心原理。

* 协议 （Protocol）  
占 8bit，这个字段定义了该报文数据区使用的协议。IANA 维护着一份协议列表（最初由 RFC 790 定义），详细参见 IP 协议号列表。

* 首部检验和 （Header Checksum）  
这个 16 位检验和字段只对首部查错，不包括数据部分。在每一跳，路由器都要重新计算出的首部检验和并与此字段进行比对，如果不一致，此报文将会被丢弃。重新计算的必要性是因为每一跳的一些首部字段（如 TTL、Flag、Offset 等）都有可能发生变化，不检查数据部分是为了减少工作量。数据区的错误留待上层协议处理——用户数据报协议（UDP）和传输控制协议（TCP）都有检验和字段。此处的检验计算方法不使用 CRC。

* 源地址  
一个 IPv4 地址由四个字节共 32 位构成，此字段的值是将每个字节转为二进制并拼在一起所得到的 32 位值。例如，10.9.8.7 是 00001010000010010000100000000111。但请注意，因为 NAT 的存在，这个地址并不总是报文的真实发送端，因此发往此地址的报文会被送往 NAT 设备，并由它被翻译为真实的地址。

* 目的地址  
与源地址格式相同，但指出报文的接收端。

* 选项  
附加的首部字段可能跟在目的地址之后，但这并不被经常使用，从 1 到 40 个字节不等。请注意首部长度字段必须包括足够的 32 位字来放下所有的选项（包括任何必须的填充以使首部长度能够被 32 位整除）。当选项列表的结尾不是首部的结尾时，EOL（选项列表结束，0x00）选项被插入列表末尾。下表列出了可能。  

|字段|	长度（位）|	描述|
|:--:|:--:|:--:|
|备份|	1|	当此选项需要被备份到所有分片中时，设为 1。|
|类|	2|	常规的选项类别，0 为“控制”，2 为“查错和措施”，1 和 3 保留。|
|数字|	5|	指明一个选项。|
|长度|	8|	指明整个选项的长度，对于简单的选项此字段可能不存在。|
|数据|	可变|	选项相关数据，对于简单的选项此字段可能不存在。|

注：如果首部长度大于 5，那么选项字段必然存在并必须被考虑。  
注：备份、类和数字经常被一并称呼为“类型”。  
* 数据
数据字段不是首部的一部分，因此并不被包含在首部检验和中。数据的格式在协议首部字段中被指明，并可以是任意的传输层协议。
一些常见协议的协议字段值被列在下面

|协议字段值|协议名|	缩写|
|:--:|:--:|:--:|
|1|	互联网控制消息协议|	ICMP|
|2|	互联网组管理协议|	IGMP|
|6|	传输控制协议|	TCP|
|17|	用户数据报协议|	UDP|
|41|	IPv6 封装|	ENCAP|
|89|	开放式最短路径优先|	OSPF|
|132|	流控制传输协议|	SCTP|

### 3.2 ipv4 地址

IPv4 使用 32 位（4 字节）地址，因此地址空间中只有 4,294,967,296（232）个地址。不过，一些地址是为特殊用途所保留的，如专用网络（约 1800 万 个地址）和多播地址（约 2.7 亿个地址），这减少了可在互联网上路由的地址数量。随着地址不断被分配给最终用户，IPv4 地址枯竭问题也在随之产生。基于分类网络、无类别域间路由和网络地址转换的地址结构重构显著地减少了地址枯竭的速度。但在 2011 年 2 月 3 日，在最后 5 个地址块被分配给 5 个区域互联网注册管理机构之后，IANA 的主要地址池已经用尽。

IPv4 地址可被写作任何表示一个 32 位整数值的形式，但为了方便人类阅读和分析，它通常被写作点分十进制的形式，即四个字节被分开用十进制写出，中间用点分隔，如`192.168.1.1`。ip 地址的编址方法一共经历过三个阶段：  

1. 分类的 IP 地址  

分类 ip 地址就是把 ip 地址划分成若干个固定类，每类地址由两部分组成，网络号和主机号。
```txt
IP地址 = (网络号，主机号)
```
IP 地址分为 5 类：A 类、B 类、C 类、D 类和 E 类。各类的地址分配方案如图所示。在 IP 地址中，全 0 代表网络，全 1 代表广播。  
![addr](https://doc.shiyanlou.com/document-uid949121labid10418timestamp1555407831412.png/wm)  
A 类网络地址占有 1 个字节（8 位），定义最高位为 0 来标识此类网络，余下 7 位为真正的网络地址。后面 3 个字节（24）为主机地址。A 类网络地址第一个字节的十进制值为：`001~127`.通常用于大型网络。  
B 类网络地址占 2 个字节，使用最高两位为“10”来标识此类地址，其余 14 位为真正的网络地址，主机地址占后面的 2 个字节（16 位）。B 类网络地址第一个字节的十进制值为：`128~191`.通常用于中型网络。  
C 类网络地址占 3 个字节，它是最通用的 Internet 地址。使用最高三为为“110”来标识此类地址。其余 21 位为真正的网络地址。主机地址占最后 1 个字节。C 类网络地址第一个字节的十进制值为：`192~223`。通常用于小型网络。  
D 类地址是相当新的。它的识别头是 1110，用于组播，例如用于路由器修改。D 类网络地址第一个字节的十进制值为：`224~239`。  
E 类地址为实验保留，其识别头是 1111。E 类网络地址第一个字节的十进制值为：`240~255`。  
但要注意得是，上面得这些地址分类已成为了历史，现在用的都是无分类 IP 地址进行路由选择。

2. 子网的划分

由于上面固定分类的 IP 地址有不少的缺陷，比如，IP 地址空间的利用率很低、固定就意味着不够灵活、使路由表太大而影响性能，为了解决上述的问题，在 IP 地址概念中，又增加了一个“子网字段”，这样的话，一个 IP 地址可以用下面的方式表示，
```txt
IP地址 = (网络号，子网号，主机号)
```
3. 无分类编址（CIDR）

为了提高 ip 地址资源的利用率，提出了变长子网掩码（VLSM），而在 VLSM 的研究基础上又提出了“无分类编址”方法，也叫无分类域间路由选择-CIDR。 CIDR 最主要有两个以下特点：  
- 消除传统的 A，B，C 地址和划分子网的概念，更有效的分配 IPv4 的地址空间，CIDR 使 IP 地址又回到无分类的两级编码。记法：IP 地址：：={<<网络前缀>，<<主机号>}。CIDR 还使用“斜线记法”即在 IP 地址后面加上“/”然后写网络前缀所占的位数。
- CIDR 把网络前缀都相同的连续 IP 地址组成一个“CIDR 地址块”，即强化路由聚合（构成超网）。
其表示方法

```txt
IP地址 = (网络前缀，主机号)
```
CIDR 还使用“斜线记法”，在 IP 地址后面加个“/”，紧跟着网络前缀所占的位数。例如：192.168.1.0/24，这种表示方式其实我们在上一章就用了，也是我们最常用的编址方式。

### 3.3 网络层在协议栈的初始化

协议栈中网络层协议是需要注册和初始化的，这样协议栈收到相应的报文才能进行处理，代码如下

```go
// 网络层协议的存储结构
networkProtocols = make(map[string]NetworkProtocolFactory)

// 以便协议栈可以使用它。此函数在由协议的init函数中使用。
func RegisterNetworkProtocolFactory(name string, p NetworkProtocolFactory) {
	networkProtocols[name] = p
}

// 初始化的时候初始化buckets，用于hash计算
// 并在网络层协议中注册ipv4协议
func init() {
	ids = make([]uint32, buckets)

	// Randomly initialize hashIV and the ids.
	r := hash.RandN32(1 + buckets)
	for i := range ids {
		ids[i] = r[i]
	}
	hashIV = r[buckets]

	stack.RegisterNetworkProtocolFactory(ProtocolName, func() stack.NetworkProtocol {
		return &protocol{}
	})
}
```

### 3.4 用 go 实现 ip 数据报文的处理

ip 首部信息

```go
// IPv4Fields contains the fields of an IPv4 packet. It is used to describe the
// fields of a packet that needs to be encoded.
// 表示IPv4头部信息的结构体
type IPv4Fields struct {
	// IHL is the "internet header length" field of an IPv4 packet.
	// 头部长度
	IHL uint8

	// TOS is the "type of service" field of an IPv4 packet.
	// 服务区分的表示
	TOS uint8

	// TotalLength is the "total length" field of an IPv4 packet.
	// 数据报文总长
	TotalLength uint16

	// ID is the "identification" field of an IPv4 packet.
	// 标识符
	ID uint16

	// Flags is the "flags" field of an IPv4 packet.
	// 标签
	Flags uint8

	// FragmentOffset is the "fragment offset" field of an IPv4 packet.
	// 分片偏移
	FragmentOffset uint16

	// TTL is the "time to live" field of an IPv4 packet.
	// 存活时间
	TTL uint8

	// Protocol is the "protocol" field of an IPv4 packet.
	// 表示的传输层协议
	Protocol uint8

	// Checksum is the "checksum" field of an IPv4 packet.
	// 首部校验和
	Checksum uint16

	// SrcAddr is the "source ip address" of an IPv4 packet.
	// 源IP地址
	SrcAddr tcpip.Address

	// DstAddr is the "destination ip address" of an IPv4 packet.
	// 目的IP地址
	DstAddr tcpip.Address
}
```
详细可以看`src\netstack\tcpip\header\ipv4.go`， 其他代码都是从二进制中读取 tcp 头部信息。

### 3.5 ip 报文的接收处理

当从网卡接收到 IPv4 报文，通过 DeliverNetworkPacket 分发到 ipv4.HandlePacket，然后进行下面的处理，首先检查了报文是否有效，接着根据分片标志中的 MF（More Fragment）位 MF 位为 1 表示当前数据报还有更多的分片，为 0 表示当前分片是该数据报最后一个分片。）判断是否是最后一个分片报文，如果是，则根据分片偏移量计算各个分片报文在原始数据报中的位置，进行重组。如果不是最后一个分片，则需等待所有分片到达后再完成重组。然后再判断是不是 ICMP 协议，如果是，则进入 ICMP 协议的处理，否则根据不同协议分发给上层协议处理。比如如果协议为 TCP，那么就会调用 tcp.HandlePacket 来进行处理。 

```go
// DeliverNetworkPacket 找到适当的网络协议端点并将数据包交给它进一步处理。
// 当NIC从物理接口接收数据包时，将调用此函数。比如protocol是arp协议号， 那么会找到arp.HandlePacket来处理数据报。
// protocol是ipv4协议号， 那么会找到ipv4.HandlePacket来处理数据报。
func (n *NIC) DeliverNetworkPacket(linkEP LinkEndpoint, remoteLinkAddr, localLinkAddr tcpip.LinkAddress,
	protocol tcpip.NetworkProtocolNumber, vv buffer.VectorisedView) {
	netProto, ok := n.stack.networkProtocols[protocol]
	...

	src, dst := netProto.ParseAddresses(vv.First())

	// 根据网络协议和数据包的目的地址，找到网络端
	// 然后将数据包分发给网络层
	if ref := n.getRef(protocol, dst); ref != nil {
		r := makeRoute(protocol, dst, src, linkEP.LinkAddress(), ref)
		r.RemoteLinkAddress = remoteLinkAddr
		ref.ep.HandlePacket(&r, vv)
		ref.decRef()
		return
	}

	...
}

// ip端表示
type endpoint struct {
	// 网卡id
	nicid tcpip.NICID
	// 表示该endpoint的id，也是ip地址
	id stack.NetworkEndpointID
	// 链路端的表示
	linkEP stack.LinkEndpoint
	// 报文分发器
	dispatcher stack.TransportDispatcher
	// ping请求报文接收队列
	echoRequests chan echoRequest
	// ip报文分片处理器
	fragmentation *fragmentation.Fragmentation
}

// HandlePacket is called by the link layer when new ipv4 packets arrive for
// this endpoint.
// 收到ip包的处理
func (e *endpoint) HandlePacket(r *stack.Route, vv buffer.VectorisedView) {
	// 得到ip报文
	h := header.IPv4(vv.First())
	// 检查报文是否有效
	if !h.IsValid(vv.Size()) {
		return
	}

	hlen := int(h.HeaderLength())
	tlen := int(h.TotalLength())
	vv.TrimFront(hlen)
	vv.CapLength(tlen - hlen)

	// 检查ip报文是否有更多的分片
	more := (h.Flags() & header.IPv4FlagMoreFragments) != 0
	// 是否需要ip重组
	if more || h.FragmentOffset() != 0 {
		// The packet is a fragment, let's try to reassemble it.
		last := h.FragmentOffset() + uint16(vv.Size()) - 1
		var ready bool
		// ip分片重组
		vv, ready = e.fragmentation.Process(hash.IPv4FragmentHash(h), h.FragmentOffset(), last, more, vv)
		if !ready {
			return
		}
	}
	// 得到传输层的协议
	p := h.TransportProtocol()
	// 如果时ICMP协议，则进入ICMP处理函数
	if p == header.ICMPv4ProtocolNumber {
		e.handleICMP(r, vv)
		return
	}
	r.Stats().IP.PacketsDelivered.Increment()
	// 根据协议分发到不通处理函数，比如协议时TCP，最终会进入tcp.HandlePacket
	e.dispatcher.DeliverTransportPacket(r, p, vv)
}
```

### 3.6 IP 报文的发送处理

当从传输层接发送报文时，会到 IP 层发送处理程序，处理的步骤主要是封装 IP 的头部信息，写入网卡，也就是链路层。
```go
// WritePacket writes a packet to the given destination address and protocol.
// 将传输层的数据封装加上IP头，并调用网卡的写入接口，写入IP报文
func (e *endpoint) WritePacket(r *stack.Route, hdr buffer.Prependable, payload buffer.VectorisedView,
	protocol tcpip.TransportProtocolNumber, ttl uint8) *tcpip.Error {
	// 预留ip报文的空间
	ip := header.IPv4(hdr.Prepend(header.IPv4MinimumSize))
	length := uint16(hdr.UsedLength() + payload.Size())
	id := uint32(0)
	// 如果报文长度大于68
	if length > header.IPv4MaximumHeaderSize+8 {
		// Packets of 68 bytes or less are required by RFC 791 to not be
		// fragmented, so we only assign ids to larger packets.
		id = atomic.AddUint32(&ids[hashRoute(r, protocol)%buckets], 1)
	}
	// ip首部编码
	ip.Encode(&header.IPv4Fields{
		IHL:         header.IPv4MinimumSize,
		TotalLength: length,
		ID:          uint16(id),
		TTL:         ttl,
		Protocol:    uint8(protocol),
		SrcAddr:     r.LocalAddress,
		DstAddr:     r.RemoteAddress,
	})
	// 计算校验和和设置校验和
	ip.SetChecksum(^ip.CalculateChecksum())
	r.Stats().IP.PacketsSent.Increment()

	// 写入网卡接口
	return e.linkEP.WritePacket(r, hdr, payload, ProtocolNumber)
}
```

## 四、陆游和路由表

![route-table](https://doc.shiyanlou.com/document-uid949121labid10418timestamp1555407867800.png/wm)  
图片来自http://www.mathcs.emory.edu/~cheung/Courses/558/Syllabus/13-Routing/intro.html

路由是数据通信网络中最基本的要素。路由信息就是指导报文发送的路径信息，路由的过程就是报文转发的过程。路由器中的路由表决定了数据下一跳的地址，而路由表目前是由分布在各地的路由协商得出的，这个协议过程被称为路由协议。路由协议创建了路由表，描述了网络拓扑结构，路由协议与路由器协同工作，执行路由选择和数据包转发功能。

路由协议作为 TCP/IP 协议族中重要成员之一，其选路过程实现的好坏会影响整个 Internet 网络的效率。按应用范围的不同，路由协议可分为两类：在一个 AS（Autonomous System，自治系统，指一个互连网络，就是把整个 Internet 划分为许多较小的网络单位，这些小的网络有权自主地决定在本系统中应采用何种路由协议）内的路由协议称为内部网关协议（interior gateway protocol），AS 之间的路由协议称为外部网关协议（exterior gateway protocol）。这里网关是路由器的旧称。正在使用的内部网关路由协议有以下几种：RIP-1，RIP-2，IGRP，EIGRP，IS-IS 和 OSPF。其中前 3 种路由协议采用的是距离向量算法，IS-IS 和 OSPF 采用的是链路状态算法,EIGRP 是结合了链路状态和距离矢量型路由选择协议的 Cisco 私有路由协议。对于小型网络，采用基于距离向量算法的路由协议易于配置和管理，且应用较为广泛，但在面对大型网络时，不但其固有的环路问题变得更难解决，所占用的带宽也迅速增长，以至于网络无法承受。因此对于大型网络，采用链路状态算法的 IS-IS 和 OSPF 较为有效，并且得到了广泛的应用。IS-IS 与 OSPF 在质量和性能上的差别并不大，但 OSPF 更适用于 IP，较 IS-IS 更具有活力。IETF 始终在致力于 OSPF 的改进工作，其修改节奏要比 IS-IS 快得多。这使得 OSPF 正在成为应用广泛的一种路由协议。不论是传统的路由器设计，还是即将成为标准的 MPLS（多协议标记交换），均将 OSPF 视为必不可少的路由协议。外部网关协议最初采用的是 EGP。EGP 是为一个简单的树形拓扑结构设计的，随着越来越多的用户和网络加入 Internet，给 EGP 带来了很多的局限性。为了摆脱 EGP 的局限性，IETF 边界网关协议工作组制定了标准的边界网关协议--BGP

常见的路由协议有 RIP、IGRP（Cisco 私有协议）、EIGRP（Cisco 私有协议）、OSPF、IS-IS、BGP 等。

### 4.1 查看本机路由实验

我们可以用`ip route`命令来查看本机路由，`ip route show`  
```shell
default via 10.211.55.1 dev enp0s5 onlink
10.211.55.0/24 dev enp0s5  proto kernel  scope link  src 10.211.55.14
10.211.55.0/24 dev enp0s6  proto kernel  scope link  src 10.211.55.16
169.254.0.0/16 dev enp0s5  scope link  metric 1000
```

### 4.2 协议栈中的路由系统

协议栈中的路由系统和真实的路由器还是有区别的，它是贯穿整个协议栈的，代表二层和三层的寻址。且我们也不会去实现路由协议，
仅仅实现路由表的配置和根据路由表来转发数据。协议栈里的路由结构需要知道二层和三层的地址，这样的就可以准确的知道一个数据报该发往哪里。

协议栈的路由结构体
```go
// Route represents a route through the networking stack to a given destination.
// 贯穿整个协议栈的路由，也就是在链路层和网络层都可以路由
// 如果目标地址是链路层地址，那么在链路层路由，
// 如果目标地址是网络层地址，那么在网络层路由。
type Route struct {
	// 远端网络层地址，ipv4或者ipv6地址
	RemoteAddress tcpip.Address

	// RemoteLinkAddress is the link-layer (MAC) address of the
	// final destination of the route.
	// 远端网卡MAC地址
	RemoteLinkAddress tcpip.LinkAddress

	// LocalAddress is the local address where the route starts.
	// 本地网络层地址，ipv4或者ipv6地址
	LocalAddress tcpip.Address

	// LocalLinkAddress is the link-layer (MAC) address of the
	// where the route starts.
	// 本地网卡MAC地址
	LocalLinkAddress tcpip.LinkAddress

	// NextHop is the next node in the path to the destination.
	// 下一跳网络层地址
	NextHop tcpip.Address

	// NetProto is the network-layer protocol.
	// 网络层协议号
	NetProto tcpip.NetworkProtocolNumber

	// ref a reference to the network endpoint through which the route
	// starts.
	// 相关的网络层终端
	ref *referencedNetworkEndpoint
}
```
当发送或接收 ip 数据报时，协议栈会根据网卡和 ip 地址信息填充上面的结构体，这样处理数据报的时候就可以根据该信息进行路由。
```go
// 根据参数新建一个路由，并关联一个网络层端
func makeRoute(netProto tcpip.NetworkProtocolNumber, localAddr, remoteAddr tcpip.Address,
	localLinkAddr tcpip.LinkAddress, ref *referencedNetworkEndpoint) Route {
	return Route{
		NetProto:         netProto,
		LocalAddress:     localAddr,
		LocalLinkAddress: localLinkAddr,
		RemoteAddress:    remoteAddr,
		ref:              ref,
	}
}

// Resolve 如有必要，解决尝试解析链接地址的问题。如果地址解析需要阻塞，则返回ErrWouldBlock，
// 例如等待ARP回复。地址解析完成（成功与否）时通知Waker。
// 如果需要地址解析，则返回ErrNoLinkAddress和通知通道，以阻止顶级调用者。
// 地址解析完成后，通道关闭（不管成功与否）。
func (r *Route) Resolve(waker *sleep.Waker) (<-chan struct{}, *tcpip.Error) {
	if !r.IsResolutionRequired() {
		// Nothing to do if there is no cache (which does the resolution on cache miss) or
		// link address is already known.
		return nil, nil
	}

	nextAddr := r.NextHop
	if nextAddr == "" {
		// Local link address is already known.
		if r.RemoteAddress == r.LocalAddress {
			r.RemoteLinkAddress = r.LocalLinkAddress
			return nil, nil
		}
		nextAddr = r.RemoteAddress
	}
	// 调用地址解析协议来解析IP地址
	linkAddr, ch, err := r.ref.linkCache.GetLinkAddress(r.ref.nic.ID(), nextAddr, r.LocalAddress, r.NetProto, waker)
	if err != nil {
		return ch, err
	}
	r.RemoteLinkAddress = linkAddr
	return nil, nil
}


// 路由查找实现，比如当tcp建立连接时，会用该函数得到路由信息
func (s *Stack) FindRoute(id tcpip.NICID, localAddr, remoteAddr tcpip.Address, netProto tcpip.NetworkProtocolNumber) (Route, *tcpip.Error) {
	s.mu.RLock()
	defer s.mu.RUnlock()

	// 遍历路由表
	for i := range s.routeTable {
		if (id != 0 && id != s.routeTable[i].NIC) || (len(remoteAddr) != 0 && !s.routeTable[i].Match(remoteAddr)) {
			continue
		}

		nic := s.nics[s.routeTable[i].NIC]
		if nic == nil {
			continue
		}

		var ref *referencedNetworkEndpoint
		if len(localAddr) != 0 {
			ref = nic.findEndpoint(netProto, localAddr, CanBePrimaryEndpoint)
		} else {
			ref = nic.primaryEndpoint(netProto)
		}
		if ref == nil {
			continue
		}

		if len(remoteAddr) == 0 {
			// If no remote address was provided, then the route
			// provided will refer to the link local address.
			remoteAddr = ref.ep.ID().LocalAddress
		}

		r := makeRoute(netProto, ref.ep.ID().LocalAddress, remoteAddr, nic.linkEP.LinkAddress(), ref)
		r.NextHop = s.routeTable[i].Gateway
		return r, nil
	}

	return Route{}, tcpip.ErrNoRoute
}

// WritePacket writes the packet through the given route.
func (r *Route) WritePacket(hdr buffer.Prependable, payload buffer.VectorisedView,
	protocol tcpip.TransportProtocolNumber, ttl uint8) *tcpip.Error {
	err := r.ref.ep.WritePacket(r, hdr, payload, protocol, ttl)
	if err == tcpip.ErrNoRoute {
		r.Stats().IP.OutgoingPacketErrors.Increment()
	}
	return err
}
```

上面的代码片段，看起来不是很容易理解，我们追踪数据包在整个协议中的流向来理解协议栈的路由，先看看协议栈接收的流程。

```txt
链路层接收数据包 -> 解析以太网帧，得到源和目的mac地址 -> 调用DeliverNetworkPacket分发数据包 -> 
根据协议和目的ip地址找到网络层端 -> 新建一个协议栈路由并填充，此时路由填充的信息有网络层协议、源mac地址、目的mac地址、源ip地址和目的ip地址 -> 
网络层收到数据包的路由信息和数据包内容，调用DeliverTransportPacket分发数据包  -> 
根据协议栈路由信息里的ip地址和解析传输层包得到端口信息，新建一个传输层id来找到传输层端 -> 将数据包分发给传输层端
```

发送流程

```txt
首先上层数据想发送数据包定会提供目的ip地址和网络层协议 根据网络层协议和目的地址从协议栈的路由表中匹配目的ip地址，也得到匹配的网卡和网络层端 ->
此时可以获取到网卡的mac地址（本地mac地址）和该网络层端的ip地址（本地ip地址），新建一个协议栈路由并填充，此时路由填充的信息有网络层协议、源mac地址、源ip地址和目的ip地址 ->
此时还差目的mac地址，会优先从arp缓存中根据目的ip来查找目的mac地址，如果查找不到则调用Resolve来解析目的ip的mac地址，填充路由信息的目的mac地址 ->
调用路由的WritePacket来发送路由信息和数据包内容，最终会调用网络层的WritePackef来发送数据 -> 网络层根据路由信息中的ip地址来填充ip首部信息，然后写入链路层端 ->
链路层端根据路由信息填充链路层首部，写入网卡。
```
简单来说协议栈路由记录了数据包的链路层地址、网络层协议和网络层地址，用来打通二三层和传输层，让上层的数据包明确的知道这个数据是从哪来的和发送数据时知道该往哪个网卡发送数据。

## 五、icmp 的简介

ICMP 的全称是 Internet Control Message Protocol 。与 IP 协议一样同属 TCP/IP 模型中的网络层，并且 ICMP 数据包是包裹在 IP 数据包中的。他的作用是报告一些网络传输过程中的错误与做一些同步工作。ICMP 数据包有许多类型。每一个数据包只有前 4 个字节是相同域的，剩余的字段有不同的数据包类型的不同而不同。ICMP 数据包的格式如下  

```txt
https://tools.ietf.org/html/rfc792

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     Type      |     Code      |          Checksum             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                   不同的Type和Code有不同的内容                    |         
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

从技术角度来说，ICMP 就是一个“错误侦测与回报机制”，
其目的就是让我们能够检测网路的连线状况﹐也能确保连线的准确性﹐其功能主要有：
* 侦测远端主机是否存在。
* 建立及维护路由信息。
* 重导数据传送路径（ICMP 重定向）。
* 数据流量控制。  

ICMP 在沟通之中，主要是透过不同的类别(Type)与代码(Code) 让机器来识别不同的连线状况。  

### 5.1 完整类型列表

|TYPE|	CODE|	Description	|
|----|------|-------------|
|0|	0|	Echo Reply——回显应答（Ping 应答）	　|
|3|	0|	Network Unreachable——网络不可达	　	|
|3|	1|	Host Unreachable——主机不可达	　	|
|3|	2|	Protocol Unreachable——协议不可达	　	|
|3|	3|	Port Unreachable——端口不可达	　	|
|3|	4|	Fragmentation needed but no frag. bit set——需要进行分片但设置不分片标志	　	|
|3|	5|	Source routing failed——源站选路失败	　	|
|3|	6|	Destination network unknown——目的网络未知	　	|
|3|	7|	Destination host unknown——目的主机未知	　	|
|3|	8|	Source host isolated (obsolete)——源主机被隔离（作废不用）	　	|
|3|	9|	Destination network administratively prohibited——目的网络被强制禁止	　	|
|3|	10|	Destination host administratively prohibited——目的主机被强制禁止	　	|
|3|	11|	Network unreachable for TOS——由于服务类型 TOS，网络不可达	　	|
|3|	12|	Host unreachable for TOS——由于服务类型 TOS，主机不可达	　	|
|3|	13|	Communication administratively prohibited by filtering——由于过滤，通信被强制禁止	　	|
|3|	14|	Host precedence violation——主机越权	　	|
|3|	15|	Precedence cutoff in effect——优先中止生效	　	|
|4|	0|	Source quench——源端被关闭（基本流控制）	　	　|
|5|	0|	Redirect for network——对网络重定向	　	　|
|5|	1|	Redirect for host——对主机重定向	　	　|
|5|	2|	Redirect for TOS and network——对服务类型和网络重定向	　	　|
|5|	3|	Redirect for TOS and host——对服务类型和主机重定向	　	　|
|8|	0|	Echo request——回显请求（Ping 请求）		　|
|9|	0|	Router advertisement——路由器通告	　	　|
|10|	0|	Route solicitation——路由器请求	　	　|
|11|	0|	TTL equals 0 during transit——传输期间生存时间为 0	　	|
|11|	1|	TTL equals 0 during reassembly——在数据报组装期间生存时间为 0	　	|
|12|	0|	IP header bad (catchall error)——坏的 IP 首部（包括各种差错）	　	|
|12|	1|	Required options missing——缺少必需的选项	　	|
|13|	0|	Timestamp request (obsolete)——时间戳请求（作废不用）		　|
|14|	　|	Timestamp reply (obsolete)——时间戳应答（作废不用）		　|
|15|	0|	Information request (obsolete)——信息请求（作废不用）		　|
|16|	0|	Information reply (obsolete)——信息应答（作废不用）		　|
|17|	0|	Address mask request——地址掩码请求		　|
|18|	0|	Address mask |reply——地址掩码应答|

ICMP 是个非常有用的协议，尤其是当我们要对网路连接状况进行判断的时候。

### 5.2 icmp 报文的处理

```go
// 处理ICMP报文
func (e *endpoint) handleICMP(r *stack.Route, vv buffer.VectorisedView) {
	v := vv.First()
	if len(v) < header.ICMPv4MinimumSize {
		return
	}
	h := header.ICMPv4(v)

	// 更具icmp的类型来进行相应的处理
	switch h.Type() {
	case header.ICMPv4Echo: // icmp echo请求
		if len(v) < header.ICMPv4EchoMinimumSize {
			return
		}
		vv.TrimFront(header.ICMPv4MinimumSize)
		log.Printf("icmp echo")
		req := echoRequest{r: r.Clone(), v: vv.ToView()}
		select {
		case e.echoRequests <- req: // 发送给echoReplier处理
		default:
			req.r.Release()
		}

	case header.ICMPv4EchoReply: // icmp echo响应
		if len(v) < header.ICMPv4EchoMinimumSize {
			return
		}
		e.dispatcher.DeliverTransportPacket(r, header.ICMPv4ProtocolNumber, vv)

	case header.ICMPv4DstUnreachable: // 目标不可达
		if len(v) < header.ICMPv4DstUnreachableMinimumSize {
			return
		}
		vv.TrimFront(header.ICMPv4DstUnreachableMinimumSize)
		switch h.Code() {
		case header.ICMPv4PortUnreachable: // 端口不可达
			e.handleControl(stack.ControlPortUnreachable, 0, vv)

		case header.ICMPv4FragmentationNeeded: // 需要进行分片但设置不分片标志
			mtu := uint32(binary.BigEndian.Uint16(v[header.ICMPv4DstUnreachableMinimumSize-2:]))
			e.handleControl(stack.ControlPacketTooBig, calculateMTU(mtu), vv)
		}
	}
	// TODO: Handle other ICMP types.
}
```
```go
// 处理icmp echo请求的goroutine
func (e *endpoint) echoReplier() {
	for req := range e.echoRequests {
		sendPing4(&req.r, 0, req.v)
		req.r.Release()
	}
}

// 根据icmp echo请求，封装icmp echo响应报文，并传给ip层处理
func sendPing4(r *stack.Route, code byte, data buffer.View) *tcpip.Error {
	hdr := buffer.NewPrependable(header.ICMPv4EchoMinimumSize + int(r.MaxHeaderLength()))

	icmpv4 := header.ICMPv4(hdr.Prepend(header.ICMPv4EchoMinimumSize))
	icmpv4.SetType(header.ICMPv4EchoReply)
	icmpv4.SetCode(code)
	copy(icmpv4[header.ICMPv4MinimumSize:], data)
	data = data[header.ICMPv4EchoMinimumSize-header.ICMPv4MinimumSize:]
	icmpv4.SetChecksum(^header.Checksum(icmpv4, header.Checksum(data, 0)))

	log.Printf("icmp reply")
	// 传给ip层处理
	return r.WritePacket(hdr, data.ToVectorisedView(), header.ICMPv4ProtocolNumber, r.DefaultTTL())
}
```
这里协议栈实现对 icmp 相对简单，并没有处理所有的类型情况，但是可以明确的是，icmp 处理是属于协议栈的一部分，并不需要其他程序。当 IP 层接收到一个 icmp 报文时，会根据 icmp 的类型分别处理，比如为`Echo request`时，协议栈会返回`Echo Reply`报文。

### 5.3 ping 的实验

实验代码，代码路径`src/lab/ping/main.go`  
```go
// +build linux

package main

import (
	"flag"
	"log"
	"net"
	"os"
	"strings"

	"netstack/tcpip"
	"netstack/tcpip/link/fdbased"
	"netstack/tcpip/link/tuntap"
	"netstack/tcpip/network/arp"
	"netstack/tcpip/network/ipv4"
	"netstack/tcpip/network/ipv6"
	"netstack/tcpip/stack"
)

func main() {
	flag.Parse()
	if len(flag.Args()) != 2 {
		log.Fatal("Usage: ", os.Args[0], " <tap-device> <local-address/mask>")
	}

	log.SetFlags(log.Lshortfile | log.LstdFlags)
	tapName := flag.Arg(0)
	cidrName := flag.Arg(1)

	log.Printf("tap: %v, cidrName: %v", tapName, cidrName)

	parsedAddr, cidr, err := net.ParseCIDR(cidrName)
	if err != nil {
		log.Fatalf("Bad cidr: %v", cidrName)
	}

	// 解析地址ip地址，ipv4或者ipv6地址都支持
	var addr tcpip.Address
	var proto tcpip.NetworkProtocolNumber
	if parsedAddr.To4() != nil {
		addr = tcpip.Address(parsedAddr.To4())
		proto = ipv4.ProtocolNumber
	} else if parsedAddr.To16() != nil {
		addr = tcpip.Address(parsedAddr.To16())
		proto = ipv6.ProtocolNumber
	} else {
		log.Fatalf("Unknown IP type: %v", parsedAddr)
	}

	// 虚拟网卡配置
	conf := &tuntap.Config{
		Name: tapName,
		Mode: tuntap.TAP,
	}

	var fd int
	// 新建虚拟网卡
	fd, err = tuntap.NewNetDev(conf)
	if err != nil {
		log.Fatal(err)
	}

	// 启动tap网卡
	tuntap.SetLinkUp(tapName)
	// 设置路由
	tuntap.SetRoute(tapName, cidr.String())

	// 获取网卡mac地址
	mac, err := tuntap.GetHardwareAddr(tapName)
	if err != nil {
		panic(err)
	}

	// 抽象网卡的文件接口
	linkID := fdbased.New(&fdbased.Options{
		FD:      fd,
		MTU:     1500,
		Address: tcpip.LinkAddress(mac),
	})

	// 新建相关协议的协议栈
	s := stack.New([]string{ipv4.ProtocolName, arp.ProtocolName},
		[]string{}, stack.Options{})

	// 新建抽象的网卡
	if err := s.CreateNamedNIC(1, "vnic1", linkID); err != nil {
		log.Fatal(err)
	}

	// 在该协议栈上添加和注册相应的网络层
	if err := s.AddAddress(1, proto, addr); err != nil {
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

	select {}
}

```
```checker
- name: check ping exist
  script: |
    #!/bin/bash
    ls -l /home/shiyanlou/golang/src/lab/ping/main.go
  error: 没有找到 /home/shiyanlou/golang/src/lab/ping/main.go 文件
```

进入项目根目录，然后编译运行程序。 
```shell
export GOPATH=`pwd`
cd src/lab/ping
go build
sudo ./ping tap0 192.168.1.1/24
```

```
TODO: 本节内容执行 5.3 步骤的过程中会报错, 无法继续实验

# netstack/tcpip/link/rawfile
../../../netstack/tcpip/link/rawfile/blockingpoll_unsafe_amd64.go:24:6: blockingPoll redeclared in this block
	
previous declaration at ../../../netstack/tcpip/link/rawfile/blockingpoll_unsafe.go:24:66

```
然后在另一个终端用 ping 程序测试协议栈是否对 icmp 有响应。  
测试结果:  
```shell
arp.go:97: recv arp request
arp.go:109: send arp reply
icmp.go:75: icmp echo
icmp.go:131: icmp reply
icmp.go:75: icmp echo
icmp.go:131: icmp reply
icmp.go:75: icmp echo
icmp.go:131: icmp reply
icmp.go:75: icmp echo
icmp.go:131: icmp reply
```

```shell
ping 192.168.1.1 -c 4
PING 192.168.1.1 (192.168.1.1) 56(84) bytes of data.
64 bytes from 192.168.1.1: icmp_seq=1 ttl=255 time=0.453 ms
64 bytes from 192.168.1.1: icmp_seq=2 ttl=255 time=0.267 ms
64 bytes from 192.168.1.1: icmp_seq=3 ttl=255 time=0.283 ms
64 bytes from 192.168.1.1: icmp_seq=4 ttl=255 time=0.391 ms

--- 192.168.1.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3001ms
rtt min/avg/max/mdev = 0.267/0.348/0.453/0.079 ms
```

可以看到协议栈打印接收到了 icmp echo 报文，然后回应了 icmp reply 报文。且 ping 命令也显示正常，表示协议栈实现了网络层的基本功能，ip 和 icmp 报文的处理。

## 六、总结

IP 层最重要的目的是让两个主机之间通信，无论他们相隔多远。IP 协议理论上允许的最大 IP 数据报为 65535 字节（16 位来表示包总长）。但是因为协议栈网络层下面的数据链路层一般允许的帧长远远小于这个值，例如以太网的 MTU 通常在 1500 字节左右。所以较大的 IP 数据包会被分片传递给数据链路层发送，分片的 IP 数据报可能会以不同的路径传输到接收主机，接收主机通过一系列的重组，将其还原为一个完整的 IP 数据报，再提交给上层协议处理。IP 分片会带来一定的问题，分片和重组会消耗发送方、接收方一定的 CPU 等资源，如果存在大量的分片报文的话，可能会造成较为严重的资源消耗；分片丢包导致的重传问题；分片攻击；
