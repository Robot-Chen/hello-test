---
show: step
version: 1.0
enable_checker: true
---
# UDP

## 一、实验说明

#### 1.1 实验内容

本章大概介绍传输层的特点以及从网络层到传输层是如何分发的，然后讲述 UDP 协议，并完成用户协议栈 UDP 协议的通信实验。

#### 1.2 知识点

- 传输层基本功能
- 网络层到传输层的数据报分发
- UDP 协议和代码实现

#### 1.3 实验环境

- Go 1.12.1
- Xfce 终端

#### 1.4 代码获取

```bash
$ wget http://labfile.oss.aliyuncs.com/courses/1300/netstack.zip
$ unzip netstack.zip
```

## 二、传输层简介 

![transport](https://doc.shiyanlou.com/document-uid949121labid10418timestamp1555488741384.png/wm)  
传输层是整个网络体系结构中的关键之一，我们很多编程都是直接和传输层打交道的，我们需要了解以下的概念：

1. 端口的意义 - 上一章已经介绍过了
2. 无连接 UDP 协议及特点 - 本章介绍
3. 面向连接 TCP 协议及特点 - 下章会介绍

传输层向它上面的应用层提供通信服务，传输层主要提供了以下功能：
1. 为相互通信的应用进程提供逻辑通信。
网络层是为主机之间提供通信，而传输层是为应用进程之间提供端到端的逻辑通信。

2. 复用和分用
复用是指发送方不同的应用进程都可以使用同一个传输协议来传送数据，而分用是指接收方的传输层在剥去报文的首部后，
能够把这些数据正确的交付给目的进程。其实复用和分用就是端口来实现的。

3. 报文差错检测
网络层只对 IP 首部进行差错检测，而传输层对整个报文进行差错检测。

4. 提供不可靠和可靠通信
网络层只提供了不可靠通信，而在传输层的 TCP 协议提供了可靠通信。

### 2.1 传输层的初始化

传输层用 transportProtocols 整个映射表来注册存储该协议栈支持的传输层协议，比如 tcp 和 udp 协议，在协议对应的目录下都有一个 protocol.go 文件，里面都有一个 init 函数，表示该协议的初始化。

```go
// 传输层协议的注册储存结构
transportProtocols = make(map[string]TransportProtocolFactory)

// RegisterTransportProtocolFactory 在协议栈中注册一个新的传输协议工厂，
// 以便协议栈可以使用它。此函数在由协议的init函数中使用。
func RegisterTransportProtocolFactory(name string, p TransportProtocolFactory) {
	transportProtocols[name] = p
}

// tcp protocol
func init() {
	stack.RegisterTransportProtocolFactory(ProtocolName, func() stack.TransportProtocol {
		return &protocol{
			sendBufferSize:             SendBufferSizeOption{minBufferSize, DefaultBufferSize, maxBufferSize},
			recvBufferSize:             ReceiveBufferSizeOption{minBufferSize, DefaultBufferSize, maxBufferSize},
			congestionControl:          ccReno,
			availableCongestionControl: []string{ccReno, ccCubic},
		}
	})
}

```
### 2.2 从网络层到传输层的数据分发-分流器

#### 2.2.1 分流器的结构

```go
// 网络层协议号和传输层协议号的组合，当作分流器的key值
type protocolIDs struct {
	network   tcpip.NetworkProtocolNumber
	transport tcpip.TransportProtocolNumber
}

// transportEndpoints 管理给定协议的所有端点。
type transportEndpoints struct {
	mu        sync.RWMutex
	endpoints map[TransportEndpointID]TransportEndpoint
}

// transportDemuxer 解复用针对传输端点的数据包（即，在它们被网络层解析之后）。
// 它执行两级解复用：首先基于网络层协议和传输协议，然后基于端点ID。
type transportDemuxer struct {
	protocol map[protocolIDs]*transportEndpoints
}
```
可以看到分流器是个两级结构，第一级是`protocolIDs`，它是网络层协议号和传输层协议号的组合。第二级是个传输层 ID-`TransportEndpointID`，表示传输层端的四元组：源 IP、源端口、目的 IP、目的端口。分流器执行两级解复用，首先基于网络层协议和传输协议，然后基于端点 ID。

#### 2.2.2 传输层端的注册 

要实现数据的分流，得先在 transportDemuxer 中注册相应的端点，不然当收到数据包时，无法找到相应的端点，
分流器端点的注册发生在协议栈好几个地方  
- udp 连接时会注册一个 udp 端
- udp 绑定某个端口时会注册一个 udp 端
- tcp 建立连接的时候会注册一个 tcp 端
- tcp 监听的时候会注册一个 tcp 端
- tcp 监听后，接收到一个连接时会注册一个 tcp 端

举个例子来说，程序监听 ipv4 协议、tcp 协议、ip 和端口-192.168.1.1:8000，此时会注册一个 tcp 端到分流器，当收到 ipv4 的数据包，且传输层协议为 tcp 时，就会根据 ipv4 协议和 tcp 协议得到 endpoints ，再根据数据包的目的 ip 和端口，就可以在 endpoints 中找到之前注册的 tcp 端，将数据分发到 tcp 端。

### 2.3 数据分流过程

在第三章的网络层中我们分析过当 ipv4 收到数据包时会根据传输层的协议号将数据包分发给相应的协议。
```go
	e.dispatcher.DeliverTransportPacket(r, p, vv)
```
这里的 dispatcher 是个 interface，其真正的实现是 `nic.go` 或者 `stack.go` 里面的 DeliverTransportPacket 来实现的分流，分流器的功能就是根据网络层协议号和传输层协议号的组合，找到这样的多个传输层端，再根据传输层 id，找到传输层 id 对应的传输端，将数据交给这个传输端处理，那么真正决定数据流在协议栈的走向就是五元组，在传输层 id 之上再加一个网络层谢意。

```go
// DeliverTransportPacket 按照传输层协议号，将数据包分发给相应的传输层端
// 比如 protocol=6表示为tcp协议，将会给相应的tcp端分发消息。
func (n *NIC) DeliverTransportPacket(r *Route, protocol tcpip.TransportProtocolNumber, vv buffer.VectorisedView) {
        ...
	// 解析报文得到源端口和目的端口
	srcPort, dstPort, err := transProto.ParsePorts(vv.First())
	if err != nil {
		n.stack.stats.MalformedRcvdPackets.Increment()
		return
	}

    id := TransportEndpointID{dstPort, r.LocalAddress, srcPort, r.RemoteAddress}
   // 调用分流器，根据传输层协议和传输层id分发数据报文
	if n.demux.deliverPacket(r, protocol, vv, id) {
		return
	}
	if n.stack.demux.deliverPacket(r, protocol, vv, id) {
		return
	}

	...
}

```
根据传输层 id 找到传输端，并调用传输层的数据处理函数，是 `transport_demuxer.go` 里名为传输层分流（transportDemuxer）实现的，其代码如下，比如，现在收到一个数据包就是 udp 报文，那么根据 id 找到这样的传输层端 ep， 再调用 ep.HandlePacket(r, id, vv) 函数处理 udp 报文，这里的 HandlePacket 就是 udp 端接收报文处理函数，下面介绍 UDP 源码的时候会详细讲。tcp 协议也是这样，只不过最终调用的 HandlePacket 是 tcp 端的函数。

```go
// 根据传输层的id来找到对应的传输端，再将数据包交给这个传输端处理
func (d *transportDemuxer) deliverPacket(r *Route, protocol tcpip.TransportProtocolNumber, vv buffer.VectorisedView, id TransportEndpointID) bool {
	// 先看看分流器里有没有注册相关协议端，如果没有则返回false
	eps, ok := d.protocol[protocolIDs{r.NetProto, protocol}]
	if !ok {
		return false
	}

	// 从 eps 中找符合 id 的传输端
	eps.mu.RLock()
	ep := d.findEndpointLocked(eps, vv, id)
	eps.mu.RUnlock()

	// Fail if we didn't find one.
	if ep == nil {
		// UDP packet could not be delivered to an unknown destination port.
		if protocol == header.UDPProtocolNumber {
			r.Stats().UDP.UnknownPortErrors.Increment()
		}
		return false
	}

	// Deliver the packet.
	// 找到传输端到，让它来处理数据包
	ep.HandlePacket(r, id, vv)

	return true
}

// 根据传输层id来找到相应的传输层端
func (d *transportDemuxer) findEndpointLocked(eps *transportEndpoints, vv buffer.VectorisedView, id TransportEndpointID) TransportEndpoint {
	// Try to find a match with the id as provided.
	// 从 endpoints 中看有没有匹配到id的传输层端
	if ep := eps.endpoints[id]; ep != nil {
		return ep
	}

	// Try to find a match with the id minus the local address.
	nid := id
	// 如果上面的 endpoints 没有找到，那么去掉本地ip地址，看看有没有相应的传输层端
	// 因为有时候传输层监听的时候没有绑定本地ip，也就是 any address，此时的 LocalAddress
	// 为空。
	nid.LocalAddress = ""
	if ep := eps.endpoints[nid]; ep != nil {
		return ep
	}

	// Try to find a match with the id minus the remote part.
	nid.LocalAddress = id.LocalAddress
	nid.RemoteAddress = ""
	nid.RemotePort = 0
	if ep := eps.endpoints[nid]; ep != nil {
		return ep
	}

	// Try to find a match with only the local port.
	nid.LocalAddress = ""
	return eps.endpoints[nid]
}
```
数据流已经分发到相应的传输层端了，接下来就看传输层端怎么处理传输层数据了，下面我们先看看传输层 udp 协议对数据包的处理。

## 三、UDP 的简介

UDP 是`User Datagram Protocol`的简称，中文名是用户数据报协议。UDP 只在 IP 数据报服务上增加了一点功能，就是复用和分用的功能以及差错检测，UDP 主要的特点是：

1. UDP 是无连接的，即发送数据之前不需要建立连接，发送结束也不需要连接释放，因此减少了开销和发送数据之间的延时。
2. UDP 是不可靠传输，尽最大努力交付，因此不需要维护复杂的连接状态。
3. UDP 的数据报是有消息边界的，发送方发送一个报文，接收方就会完整的收到一个报文。
4. UDP 没有拥塞控制，网络出现阻塞，UDP 是无感知的，也就不会降低发送速度。
5. UDP 支持一对一，一对多，多对一，多对多的通信。

UDP 简单的功能完全体现在了 UDP 协议的首部中，首部只有 8 个字节，由 4 个字段组成。

![udp](https://doc.shiyanlou.com/document-uid949121labid10418timestamp1555488770215.png/wm)  

1. 源端口
源端口号
2. 目的端口
目的端口号
3. 长度
UDP 数据报的长度，包含首部，最小为 8
4. 检验和
UDP 数据报的校验和，如果接收到检验和不正确的情况下，直接丢弃该报文。

### 3.1 校验和的计算

UDP 计算校验和的方法和 IP 数据报首部校验和的方法相似。不同的是：IP 数据报校验和只校验 IP 数据报的首部，但 UDP 的校验和是把首部和数据部分一起都检验。

UDP 的校验和需要计算 UDP 首部加数据荷载部分，但也需要加上 UDP 伪首部。这个伪首部指，源地址、目的地址、UDP 数据长度、协议类型（0x11），协议类型就一个字节，但需要补一个字节的 0x0，构成 12 个字节。伪首部+UDP 首部+数据一起计算校验和。

UDP 检验和的计算方法是：  
按每 16 位求和得出一个 32 位的数；  
如果这个 32 位的数，高 16 位不为 0，则高 16 位加低 16 位再得到一个 32 位的数；  
重复第 2 步直到高 16 位为 0，将低 16 位取反，得到校验和。  


### 3.2 UDP 协议源码实现解析

udp 端的源码实现文件为 `netstack/tcpip/transport/udp` 目录下的文件，上面已经讲了网络层怎么找到传输层端，并将数据分发给对应的传输端，现在看看 UDP 端是怎么处理报文的。首先 UDP 端有接收队列的概念，不像网络层接收到数据包立马发送给传输层，对于协议栈来说，传输层是最后的一站，接下来的数据就需要交给用户层了，但是用户层的行为是不可预知的，不知道用户层何时将数据取走（也就是 UDP Read 过程），那么协议栈就实现一个接收队列，将接收的数据去掉 UDP 头部后保存在这个队列中，用户层需要的时候取走就可以了，但是队列存数据量是有限制的，这个限制叫接收缓存大小，当接收队列中的数据总和超过这个缓存，那么接下来的这些报文将会被直接丢包。
```go
// HandlePacket is called by the stack when new packets arrive to this transport
// endpoint.
// 从网络层接收到UDP数据报时的处理函数
func (e *endpoint) HandlePacket(r *stack.Route, id stack.TransportEndpointID, vv buffer.VectorisedView) {
        ...
	// 去除UDP首部
	vv.TrimFront(header.UDPMinimumSize)

	e.rcvMu.Lock()
	e.stack.Stats().UDP.PacketsReceived.Increment()

	// 如果UDP的接收缓存已经满了，那么丢弃报文。
	if !e.rcvReady || e.rcvClosed || e.rcvBufSize >= e.rcvBufSizeMax {
		e.stack.Stats().UDP.ReceiveBufferErrors.Increment()
		e.rcvMu.Unlock()
		return
	}

	// 接收缓存是否为空
	wasEmpty := e.rcvBufSize == 0

	// Push new packet into receive list and increment the buffer size.
	// 新建一个UDP 数据报结构，插入到接收链表里
	pkt := &udpPacket{
		senderAddress: tcpip.FullAddress{
			NIC:  r.NICID(),
			Addr: id.RemoteAddress,
			Port: hdr.SourcePort(),
		},
	}
	// 复制UDP数据的用户数据
	pkt.data = vv.Clone(pkt.views[:])
	// 插入到接收链表里，并增加已使用的缓存
	e.rcvList.PushBack(pkt)
	e.rcvBufSize += vv.Size()

    ...

	e.rcvMu.Unlock()

	// Notify any waiters that there's data to be read now.
	// 通知程序现在可以读取到数据了
	if wasEmpty {
		e.waiterQueue.Notify(waiter.EventIn)
	}
	log.Printf("recv udp %d bytes", hdr.Length())
}
```
### 3.3 发送报文的处理

UDP 发送报文整体来说也不复杂，用户层最终调用 Write 方法来发送数据，协议栈给数据加上 UDP 头部信息后就发送给网络层，完成了发送的功能。但是这里得注意 UDP 和 ARP 的关系，由于 UDP 是无连接的，当发送一个数据包时，如果目的 IP 在 arp 缓存中没有，那么会先发送 arp 请求来得到目的 IP 对应的 MAC 地址，这点和 ping 程序很类似，有时候我们会发现第一次 ping 的时间比后面的长，有可能就会因为需要先发 arp 请求导致的。
```go
// 用户层最终调用该函数，发送数据包给对端，即使数据写失败，也不会阻塞。
func (e *endpoint) Write(p tcpip.Payload, opts tcpip.WriteOptions) (uintptr, <-chan struct{}, *tcpip.Error) {
	// MSG_MORE is unimplemented. (This also means that MSG_EOR is a no-op.)
	if opts.More {
		return 0, nil, tcpip.ErrInvalidOptionValue
	}

	// 如果报文长度超过65535，将会超过UDP最大的长度表示，这是不允许的。
	if p.Size() > math.MaxUint16 {
		// Payload can't possibly fit in a packet.
		return 0, nil, tcpip.ErrMessageTooLong
	}

	to := opts.To

	e.mu.RLock()
	defer e.mu.RUnlock()

	// If we've shutdown with SHUT_WR we are in an invalid state for sending.
	// 如果设置了关闭写数据，那返回错误
	if e.shutdownFlags&tcpip.ShutdownWrite != 0 {
		return 0, nil, tcpip.ErrClosedForSend
	}

	// Prepare for write.
	// 准备发送数据
	for {
		retry, err := e.prepareForWrite(to)
		if err != nil {
			return 0, nil, err
		}

		if !retry {
			break
		}
	}

	var route *stack.Route
	var dstPort uint16
	if to == nil {
		// 如果没有指定发送的地址，用UDP端 Connect 得到的路由和目的端口
		route = &e.route
		dstPort = e.dstPort

		if route.IsResolutionRequired() {
			// Promote lock to exclusive if using a shared route, given that it may need to
			// change in Route.Resolve() call below.
			// 如果使用共享路由，则将锁定提升为独占路由，因为它可能需要在下面的Route.Resolve（）调用中进行更改。
			e.mu.RUnlock()
			defer e.mu.RLock()

			e.mu.Lock()
			defer e.mu.Unlock()

			// Recheck state after lock was re-acquired.
			// 锁定后重新检查状态。
			if e.state != stateConnected {
				return 0, nil, tcpip.ErrInvalidEndpointState
			}
		}
	} else { // 如果指定了发送的地址
		nicid := to.NIC
		// 如果绑定了网卡，使用该网卡
		if e.bindNICID != 0 {
			// 如果绑定的网卡和参数指定的网卡不同，那么返回错误
			if nicid != 0 && nicid != e.bindNICID {
				return 0, nil, tcpip.ErrNoRoute
			}

			nicid = e.bindNICID
		}

		// 得到目的IP+端口
		toCopy := *to
		to = &toCopy
		netProto, err := e.checkV4Mapped(to, false)
		if err != nil {
			return 0, nil, err
		}

		log.Printf("netProto: 0x%x", netProto)
		// Find the enpoint.
		// 根据目的地址和协议找到相关路由信息
		r, err := e.stack.FindRoute(nicid, e.id.LocalAddress, to.Addr, netProto)
		if err != nil {
			return 0, nil, err
		}
		defer r.Release()

		route = &r
		dstPort = to.Port
	}

	// 如果路由没有下一跳的链路MAC地址，那么触发相应的机制，来填充该路由信息。
	// 比如：IPV4协议，如果没有目的IP对应的MAC信息，从从ARP缓存中查找信息，找到了直接返回，
	// 若没找到，那么发送ARP请求，得到对应的MAC地址。
	if route.IsResolutionRequired() {
		waker := &sleep.Waker{}
		if ch, err := route.Resolve(waker); err != nil {
			if err == tcpip.ErrWouldBlock {
				// Link address needs to be resolved. Resolution was triggered the background.
				// Better luck next time.
				route.RemoveWaker(waker)
				return 0, ch, tcpip.ErrNoLinkAddress
			}
			return 0, nil, err
		}
	}

	// 得到要发送的数据内容
	v, err := p.Get(p.Size())
	if err != nil {
		return 0, nil, err
	}

	ttl := route.DefaultTTL()
	// 如果是多播地址，设置ttl
	if header.IsV4MulticastAddress(route.RemoteAddress) || header.IsV6MulticastAddress(route.RemoteAddress) {
		ttl = e.multicastTTL
	}

	// 增加UDP头部信息，并发送出去
	if err := sendUDP(route, buffer.View(v).ToVectorisedView(), e.id.LocalPort, dstPort, ttl); err != nil {
		return 0, nil, err
	}

	return uintptr(len(v)), nil, nil
}

// 增加UDP头部信息，并发送给给网络层
func sendUDP(r *stack.Route, data buffer.VectorisedView, localPort, remotePort uint16, ttl uint8) *tcpip.Error {
	// Allocate a buffer for the UDP header.
	hdr := buffer.NewPrependable(header.UDPMinimumSize + int(r.MaxHeaderLength()))

	// Initialize the header.
	udp := header.UDP(hdr.Prepend(header.UDPMinimumSize))

	// 得到报文的长度
	length := uint16(hdr.UsedLength() + data.Size())
	// UDP首部的编码
	udp.Encode(&header.UDPFields{
		SrcPort: localPort,
		DstPort: remotePort,
		Length:  length,
	})

	// Only calculate the checksum if offloading isn't supported.
	if r.Capabilities()&stack.CapabilityChecksumOffload == 0 {
		// 检验和的计算
		xsum := r.PseudoHeaderChecksum(ProtocolNumber)
		for _, v := range data.Views() {
			xsum = header.Checksum(v, xsum)
		}
		udp.SetChecksum(^udp.CalculateChecksum(xsum, length))
	}

	// Track count of packets sent.
	r.Stats().UDP.PacketsSent.Increment()

	// 将准备好的UDP首部和数据写给网络层
	log.Printf("send udp %d bytes", hdr.UsedLength()+data.Size())
	return r.WritePacket(hdr, data, ProtocolNumber, ttl)
}
```

### 3.4 UDP 通信的实验

UDP 通信实验分为客户端和服务端，我们将实现用户态协议栈实现 UDP 的服务端，再用 golang 标准库的 UDP 客户端来进行通信。
服务端实现的功能主要是回显，接收到数据将原样返回给客户端。  

#### 3.4.1 服务端实现
服务端代码,创建`src/lab/udp/server/main.go`文件，输入以下代码：

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
	"netstack/tcpip/link/fdbased"
	"netstack/tcpip/link/tuntap"
	"netstack/tcpip/network/arp"
	"netstack/tcpip/network/ipv4"
	"netstack/tcpip/network/ipv6"
	"netstack/tcpip/stack"
	"netstack/tcpip/transport/udp"
	"netstack/waiter"
)

var mac = flag.String("mac", "aa:00:01:01:01:01", "mac address to use in tap device")

func main() {
	flag.Parse()
	if len(flag.Args()) != 4 {
		log.Fatal("Usage: ", os.Args[0], " <tap-device> <local-address/mask> <ip-address> <local-port>")
	}

	log.SetFlags(log.Lshortfile | log.LstdFlags)

	tapName := flag.Arg(0)
	cidrName := flag.Arg(1)
	addrName := flag.Arg(2)
	portName := flag.Arg(3)

	log.Printf("tap: %v, addr: %v, port: %v", tapName, addrName, portName)

	// Parse the mac address.
	maddr, err := net.ParseMAC(*mac)
	if err != nil {
		log.Fatalf("Bad MAC address: %v", *mac)
	}

	parsedAddr := net.ParseIP(addrName)
	if err != nil {
		log.Fatalf("Bad addrress: %v", addrName)
	}

	// Parse the IP address. Support both ipv4 and ipv6.
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

	localPort, err := strconv.Atoi(portName)
	if err != nil {
		log.Fatalf("Unable to convert port %v: %v", portName, err)
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
	tuntap.SetRoute(tapName, cidrName)

	// 抽象的文件接口
	linkID := fdbased.New(&fdbased.Options{
		FD:                 fd,
		MTU:                1500,
		Address:            tcpip.LinkAddress(maddr),
		ResolutionRequired: true,
	})

	// Create the stack with ip and tcp protocols, then add a tun-based
	// NIC and address.
	// 新建相关协议的协议栈
	s := stack.New([]string{ipv4.ProtocolName, ipv6.ProtocolName, arp.ProtocolName},
		[]string{udp.ProtocolName}, stack.Options{})

	// 新建抽象的网卡
	if err := s.CreateNamedNIC(1, "vnic1", linkID); err != nil {
		log.Fatal(err)
	}

	// 在该网卡上添加和注册相应的网络层
	if err := s.AddAddress(1, proto, addr); err != nil {
		log.Fatal(err)
	}

	// 在该协议栈上添加和注册ARP协议
	if err := s.AddAddress(1, arp.ProtocolNumber, arp.ProtocolAddress); err != nil {
		log.Fatal(err)
	}

	// Add default route.
	// 添加默认路由
	s.SetRouteTable([]tcpip.Route{
		{
			Destination: tcpip.Address(strings.Repeat("\x00", len(addr))),
			Mask:        tcpip.AddressMask(strings.Repeat("\x00", len(addr))),
			Gateway:     "",
			NIC:         1,
		},
	})

	var wq waiter.Queue
	// 新建一个UDP端
	ep, e := s.NewEndpoint(udp.ProtocolNumber, proto, &wq)
	if err != nil {
		log.Fatal(e)
	}

	// 绑定本地端口
	if err := ep.Bind(tcpip.FullAddress{1, addr, uint16(localPort)}, nil); err != nil {
		log.Fatal("Bind failed: ", err)
	}

	echo(&wq, ep)
}

func echo(wq *waiter.Queue, ep tcpip.Endpoint) {
	defer ep.Close()

	// Create wait queue entry that notifies a channel.
	waitEntry, notifyCh := waiter.NewChannelEntry(nil)

	wq.EventRegister(&waitEntry, waiter.EventIn)
	defer wq.EventUnregister(&waitEntry)

	var saddr tcpip.FullAddress

	for {
		v, _, err := ep.Read(&saddr)
		if err != nil {
			if err == tcpip.ErrWouldBlock {
				<-notifyCh
				continue
			}

			return
		}

		log.Printf("read and write data: %s", string(v))
		_, _, err = ep.Write(tcpip.SlicePayload(v), tcpip.WriteOptions{To: &saddr})
		if err != nil {
			log.Fatal(err)
		}
	}
}
```
```checker
- name: check server exist
  script: |
    #!/bin/bash
    ls -l /home/shiyanlou/golang/src/lab/udp/server/main.go
  error: 没有找到 /home/shiyanlou/golang/src/lab/udp/server/main.go 文件
```

### 3.4.2 客户端实现

客户端代码，创建 `src/lab/udp/client/main.go` 文件，输入以下代码：
```go
package main

import (
	"flag"
	"log"
	"net"
)

func main() {
	var (
		addr = flag.String("a", "192.168.1.1:9000", "udp dst address")
    )
    
	log.SetFlags(log.Lshortfile | log.LstdFlags)

	udpAddr, err := net.ResolveUDPAddr("udp", *addr)
	if err != nil {
		panic(err)
	}

	// 建立UDP连接（只是填息了目的IP和端口，并未真正的建立连接）
	conn, err := net.DialUDP("udp", nil, udpAddr)
	if err != nil {
		panic(err)
	}

	send := []byte("hello")
	recv := make([]byte, 10)

	conn.Write(send)
	log.Printf("send: %s", string(send))
	rn, _, err := conn.ReadFrom(recv)
	if err != nil {
		panic(err)
	}
	log.Printf("recv: %s", string(recv[:rn]))
}
```
```checker
- name: check client exist
  script: |
    #!/bin/bash
    ls -l /home/shiyanlou/golang/src/lab/udp/client/main.go
  error: 没有找到 /home/shiyanlou/golang/src/lab/udp/client/main.go 文件
```

打开一个终端，进入项目目录，然后执行下面的命令，启动 udp 服务端。   
`go run src/lab/udp/server/main.go tap0 192.168.1.0/24 192.168.1.1 9000`

```
TODO: 本节内容执行 3.4.2 步骤的过程中会报错, 无法继续实验

# netstack/tcpip/link/rawfile
src/netstack/tcpip/link/rawfile/blockingpoll_unsafe_amd64.go:24:6: blockingPoll redeclared in this block
	previous declaration at src/netstack/tcpip/link/rawfile/blockingpoll_unsafe.go:24:66

```
接着再打开另外一个终端，进入项目目录，然后执行下面的命令，启动 udp 客户端。 
`go run src/lab/udp/client/main.go`
```
TODO: 本节内容执行 3.4.2 步骤的过程中会报错, 无法继续实验

# netstack/tcpip/link/rawfile
src/netstack/tcpip/link/rawfile/blockingpoll_unsafe_amd64.go:24:6: blockingPoll redeclared in this block
	previous declaration at src/netstack/tcpip/link/rawfile/blockingpoll_unsafe.go:24:66

```
实验结果：  
服务端的显示  

```txt
2019/02/24 16:07:37 main.go:38: tap: tap0, addr: 192.168.1.1, port: 9000
2019/02/24 16:07:37 ports.go:131: new transport: 17, port: 9000

2019/02/24 16:11:48 arp.go:97: recv arp request
2019/02/24 16:11:48 arp.go:109: send arp reply
2019/02/24 16:11:48 linkaddrcache.go:152: add link cache: 1:10.211.55.23:0-0e:19:fd:61:0b:b9
2019/02/24 16:11:48 ipv4.go:193: recv ipv4 packet 33 bytes, proto: 0x11
2019/02/24 16:11:48 endpoint.go:975: recv udp 13 bytes
2019/02/24 16:11:48 main.go:165: read and write data: hello
2019/02/24 16:11:48 endpoint.go:361: netProto: 0x800
2019/02/24 16:11:48 linkaddrcache.go:205: link addr get linkRes: &arp.protocol{}, addr: 1:10.211.55.23:0
2019/02/24 16:11:48 endpoint.go:590: send udp 13 bytes
2019/02/24 16:11:48 ipv4.go:151: send ipv4 packet 33 bytes, proto: 0x11
```
客户端端的显示  

```txt
2019/02/24 16:11:48 send: hello
2019/02/24 16:11:48 recv: hello
```
从客户端的结果来看，发送 hello 消息给服务端，服务端返回 hello，最后客户端读取到到了 hello ，完成基本的 UDP 通信。

## 四、总结

传输层是整个网络的关键部分，它实现了两个用户进程间端到端的通信，其中 UDP 算是比较简单的传输层协议，它不提供可靠性，也没有拥塞控制，但是理解 UDP 协议也是很有必要的，现在的很多应用都是基于 UDP 的，比如 DNS、DHCP 以及最新的 HTTP3 协议等。接下来我们再详细分析协议栈中可靠的通信协议 TCP，它比 UDP 要复杂的多。

