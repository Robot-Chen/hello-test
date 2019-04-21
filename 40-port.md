---
show: step
version: 1.0
enable_checker: true
---
# Port

## 一、实验说明

#### 1.1 实验内容

本章主要介绍端口的概念，了解为何需要端口以及常见的端口类型。

#### 1.2 知识点

- 端口的概念
- 端口的类型
- 协议栈端口管理的实现

#### 1.3 实验环境

- Go 1.12.1
- Xfce 终端

#### 1.4 代码获取

```bash
$ wget http://labfile.oss.aliyuncs.com/courses/1300/netstack.zip
$ unzip netstack.zip
```

## 二、端口的概念

在互联网上，各主机间通过 TCP/IP 协议发送和接收数据包，各个数据包根据其目的主机的 ip 地址来进行互联网络中的路由选择,把数据包顺利的传送到目的主机。大多数操作系统都支持多程序（进程）同时运行，那么目的主机应该把接收到的数据包传送给众多同时运行的进程中的哪一个呢？显然这个问题有待解决，运行在计算机中的进程是用进程标识符来标志的。一开始我们可能会想到根据进程标识符来区分数据包给哪个进程，但是因为在因特网上使用的计算机的操作系统种类很多，而不同的操作系统又使用不同格式的进程标识符，因此发送方非常可能无法识别其他机器上的进程。为了使运行不同操作系统的计算机的应用进程能够互相通信，就必须用统一的方法对 TCP/IP 体系的应用进程进行标志，因此 TCP/IP 体系的传输层端口被提了出来。

![port](https://doc.shiyanlou.com/document-uid949121labid10418timestamp1555484076771.png/wm)

TCP/IP 协议在运输层使用协议端口号(protocol port number)，或通常简称为端口(port)，端口统一用一个 16 位端口号进行标志。端口号只具有本地意义，即端口号只是为了标志本计算机应用层中的各进程。在因特网中不同计算机的相同端口号是没有联系的。虽然通信的终点是应用进程，但我们可以把端口想象是通信的终点，因为我们只要把要传送的报文交到目的主机的某一个合适的目的端口，剩下的工作（即最后交付目的进程）就由 TCP 来完成。

如果把 IP 地址比作一栋楼房，端口号就是这栋楼房里各个房子的房间号。数据包来到主机这栋大楼，会查看是个房间号，再把数据发给相应的房间。端口号只有整数，范围是从 0 到 65535（2^16-1），其中 0 一般作为保留端口，表示让系统自动分配端口。

### 2.1 端口类型

最常见的是 TCP 端口和 UDP 端口。由于 TCP 和 UDP 两个协议是独立的，因此各自的端口号也相互独立，比如 TCP 有 235 端口，UDP 也可以有 235 端口，两者并不冲突。

TCP 和 UDP 协议首部的前四个字节都是用来表示端口的，分别表示源端口和目的端口，各占 2 个字节，详细的 TCP、UDP 协议头部会在下面的文章中讲到。

![tcp port](https://doc.shiyanlou.com/document-uid949121labid10418timestamp1555484120164.png/wm)

1. 周知端口（Well Known Ports）  
周知端口是众所周知的端口号，范围从 0 到 1023，其中 80 端口分配给 WWW 服务，21 端口分配给 FTP 服务等。我们在 IE 的地址栏里输入一个网址的时候是不必指定端口号的，因为在默认情况下 WWW 服务的端口是"80"。网络服务是可以使用其他端口号的，如果不是默认的端口号则应该在 地址栏上指定端口号，方法是在地址后面加上冒号":"，再加上端口号。比如使用"8080"作为 WWW 服务的端口，则需要在地址栏里输入"网址:8080"。但是有些系统协议使用固定的端口号，它是不能被改变的，比如 139 端口专门用于 NetBIOS 与 TCP/IP 之间的通信，不能手动改变。
2. 注册端口（Registered Ports）   
端口 1024 到 49151，分配给用户进程或应用程序。这些进程主要是用户选择安装的一些应用程序，而不是已经分配好了公认端口的常用程序。这些端口在没有被服务器资源占用的时候，可以用用户端动态选用为源端口。
3. 动态端口（Dynamic Ports）  
动态端口的范围是从 49152 到 65535。之所以称为动态端口，是因为它 一般不固定分配某种服务，而是动态分配。比如本地想和远端建立 TCP 连接，如果没有指定本地源端口，系统就会给你自动分配一个未占用的源端口，这个端口值就是动态的，当你断开再次建立连接的时候，很有可能你的源端口和上次得到的端口不一样。

一些常见的端口号及其用途如下：   
TCP21 端口：FTP 文件传输服务  
TCP22 端口：SSH 安全外壳协议  
TCP23 端口：TELNET 终端仿真服务  
TCP25 端口：SMTP 简单邮件传输服务  
UDP53 端口：DNS 域名解析服务  
UDP67 端口：DHCP 的服务端端口  
UDP68 端口：DHCP 的客户端端口  
TCP80 端口：HTTP 超文本传输服务  
TCP110 端口：POP3“邮局协议版本 3”使用的端口  
TCP443 端口：HTTPS 加密的超文本传输服务  

### 2.2 用 go 实现端口的管理

端口的分配是由协议栈全局管理的。由一个端口管理的对象 PortManager 来进行管理。
```go
// 端口的唯一标识: 网络层协议-传输层协议-端口号
type portDescriptor struct {
	network   tcpip.NetworkProtocolNumber
	transport tcpip.TransportProtocolNumber
	port      uint16
}

// 管理端口的对象，由它来保留和释放端口
type PortManager struct {
	mu sync.RWMutex
	// 用一个map接口来保存端口被占用
	allocatedPorts map[portDescriptor]bindAddresses
}

// bindAddresses is a set of IP addresses.
type bindAddresses map[tcpip.Address]struct{}

```

可以看到 PortManager 主要就是一个字段 allocatedPorts ，它是一个 map 结构，键是 portDescriptor ，用来表示唯一端口的标识，值也是一个 map 结构 bindAddresses 表示这个 portDescriptor 绑定的低脂。

```go
// isAvailable checks whether an IP address is available to bind to.
func (b bindAddresses) isAvailable(addr tcpip.Address) bool {
	if addr == anyIPAddress {
		return len(b) == 0
	}

	// If all addresses for this portDescriptor are already bound, no
	// address is available.
	if _, ok := b[anyIPAddress]; ok {
		return false
	}

	if _, ok := b[addr]; ok {
		return false
	}
	return true
}

// NewPortManager 新建一个端口管理器
func NewPortManager() *PortManager {
	return &PortManager{allocatedPorts: make(map[portDescriptor]bindAddresses)}
}

// PickEphemeralPort 从端口管理器中随机分配一个端口，并调用testPort来检测是否可用。
func (s *PortManager) PickEphemeralPort(testPort func(p uint16) (bool, *tcpip.Error)) (port uint16, err *tcpip.Error) {
	count := uint16(math.MaxUint16 - FirstEphemeral + 1)
	offset := uint16(rand.Int31n(int32(count)))

	for i := uint16(0); i < count; i++ {
		port = FirstEphemeral + (offset+i)%count
		ok, err := testPort(port)
		if err != nil {
			return 0, err
		}

		if ok {
			return port, nil
		}
	}

	return 0, tcpip.ErrNoPortAvailable
}

// IsPortAvailable tests if the given port is available on all given protocols.
func (s *PortManager) IsPortAvailable(networks []tcpip.NetworkProtocolNumber, transport tcpip.TransportProtocolNumber,
	addr tcpip.Address, port uint16) bool {
	s.mu.Lock()
	defer s.mu.Unlock()
	return s.isPortAvailableLocked(networks, transport, addr, port)
}

// isPortAvailableLocked 根据参数判断该端口号是否已经被占用了
func (s *PortManager) isPortAvailableLocked(networks []tcpip.NetworkProtocolNumber, transport tcpip.TransportProtocolNumber,
	addr tcpip.Address, port uint16) bool {
	for _, network := range networks {
		desc := portDescriptor{network, transport, port}
		if addrs, ok := s.allocatedPorts[desc]; ok {
			if !addrs.isAvailable(addr) {
				return false
			}
		}
	}
	return true
}

// ReservePort 将端口和IP地址绑定在一起，这样别的程序就无法使用已经被绑定的端口。
// 如果传入的端口不为0，那么会尝试绑定该端口，若该端口没有被占用，那么绑定成功。
// 如果传人的端口等于0，那么就是告诉协议栈自己分配端口，端口管理器就会随机返回一个端口。
func (s *PortManager) ReservePort(networks []tcpip.NetworkProtocolNumber, transport tcpip.TransportProtocolNumber,
	addr tcpip.Address, port uint16) (reservedPort uint16, err *tcpip.Error) {
	s.mu.Lock()
	defer s.mu.Unlock()

	// If a port is specified, just try to reserve it for all network
	// protocols.
	if port != 0 {
		if !s.reserveSpecificPort(networks, transport, addr, port) {
			return 0, tcpip.ErrPortInUse
		}
		reservedPort = port
		log.Printf("new transport: %d, port: %d", transport, reservedPort)
		return
	}

	// 随机分配一个未占用的端口
	reservedPort, err = s.PickEphemeralPort(func(p uint16) (bool, *tcpip.Error) {
		return s.reserveSpecificPort(networks, transport, addr, p), nil
	})
	log.Printf("new transport: %d, port: %d", transport, reservedPort)
	return
}

// reserveSpecificPort 尝试根据协议号和IP地址绑定一个端口
func (s *PortManager) reserveSpecificPort(networks []tcpip.NetworkProtocolNumber, transport tcpip.TransportProtocolNumber,
	addr tcpip.Address, port uint16) bool {
	if !s.isPortAvailableLocked(networks, transport, addr, port) {
		return false
	}

	// Reserve port on all network protocols.
	// 根据给定的网络层协议号（IPV4或IPV6），绑定端口
	for _, network := range networks {
		desc := portDescriptor{network, transport, port}
		m, ok := s.allocatedPorts[desc]
		if !ok {
			m = make(bindAddresses)
			s.allocatedPorts[desc] = m
		}
		// 注册该地址被绑定了
		m[addr] = struct{}{}
	}

	return true
}

// 释放绑定的端口，以便别的程序复用。
func (s *PortManager) ReleasePort(networks []tcpip.NetworkProtocolNumber, transport tcpip.TransportProtocolNumber,
	addr tcpip.Address, port uint16) {
	s.mu.Lock()
	defer s.mu.Unlock()

	// 删除绑定关系
	for _, network := range networks {
		desc := portDescriptor{network, transport, port}
		if m, ok := s.allocatedPorts[desc]; ok {
			log.Printf("delete transport: %d, port: %d", transport, port)
			delete(m, addr)
			if len(m) == 0 {
				delete(s.allocatedPorts, desc)
			}
		}
	}
}
```

可以看到端口的分配是很简单的，就是对 allocatedPorts 的查询与插入，来表示端口的占用情况。注释已经写的很明白了，这里就不再赘述了，来看看下面的实验。

### 2.3 端口分配和回收的实验

在`src/lab/port`目录下创建文件 `main.go`，输入以下代码：

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
	"netstack/tcpip/transport/tcp"
	"netstack/tcpip/transport/udp"
	"netstack/waiter"
)

var mac = flag.String("mac", "01:01:01:01:01:01", "mac address to use in tap device")

func main() {
	flag.Parse()
	if len(flag.Args()) != 3 {
		log.Fatal("Usage: ", os.Args[0], " <tap-device> <listen-address> port")
	}

	log.SetFlags(log.Lshortfile | log.LstdFlags)
	tapName := flag.Arg(0)
	listeAddr := flag.Arg(1)
	portName := flag.Arg(2)

	log.Printf("tap: %v, listeAddr: %v, portName: %v", tapName, listeAddr, portName)

	// Parse the mac address.
	maddr, err := net.ParseMAC(*mac)
	if err != nil {
		log.Fatalf("Bad MAC address: %v", *mac)
	}

	parsedAddr := net.ParseIP(listeAddr)

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
	// 设置tap网卡IP地址
	tuntap.AddIP(tapName, listeAddr)

	// 抽象网卡的文件接口
	linkID := fdbased.New(&fdbased.Options{
		FD:      fd,
		MTU:     1500,
		Address: tcpip.LinkAddress(maddr),
	})

	// 新建相关协议的协议栈
	s := stack.New([]string{ipv4.ProtocolName, arp.ProtocolName},
		[]string{tcp.ProtocolName, udp.ProtocolName}, stack.Options{})

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

	// 同时监听tcp和udp localPort端口
	tcpEp := tcpListen(s, proto, localPort)
	// tcpListen(s, proto, localPort) // port is in use
	udpEp := udpListen(s, proto, localPort)
	// 关闭监听服务，此时会释放端口
	tcpEp.Close()
	udpEp.Close()
}

func tcpListen(s *stack.Stack, proto tcpip.NetworkProtocolNumber, localPort int) tcpip.Endpoint {
	var wq waiter.Queue
	// 新建一个tcp端
	ep, err := s.NewEndpoint(tcp.ProtocolNumber, proto, &wq)
	if err != nil {
		log.Fatal(err)
	}

	// 绑定IP和端口，这里的IP地址为空，表示绑定任何IP
	// 此时就会调用端口管理器
	if err := ep.Bind(tcpip.FullAddress{0, "", uint16(localPort)}, nil); err != nil {
		log.Fatal("Bind failed: ", err)
	}

	// 开始监听
	if err := ep.Listen(10); err != nil {
		log.Fatal("Listen failed: ", err)
	}

	return ep
}

func udpListen(s *stack.Stack, proto tcpip.NetworkProtocolNumber, localPort int) tcpip.Endpoint {
	var wq waiter.Queue
	// 新建一个udp端
	ep, err := s.NewEndpoint(udp.ProtocolNumber, proto, &wq)
	if err != nil {
		log.Fatal(err)
	}

	// 绑定IP和端口，这里的IP地址为空，表示绑定任何IP
	// 此时就会调用端口管理器
	if err := ep.Bind(tcpip.FullAddress{0, "", uint16(localPort)}, nil); err != nil {
		log.Fatal("Bind failed: ", err)
	}

	// 注意UDP是无连接的，它不需要Listen
	return ep
}
```
```checker
- name: check port exist
  script: |
    #!/bin/bash
    ls -l /home/shiyanlou/golang/src/lab/port/main.go
  error: 没有找到 /home/shiyanlou/golang/src/lab/port/main.go 文件
```

进入项目目录，然后执行下面的命令。    
`go run src/lab/port/main.go tap0 192.168.1.1 9000`

```
TODO: 本节内容执行 2.3 步骤的过程中会报错, 无法继续实验

# netstack/tcpip/link/rawfile
src/netstack/tcpip/link/rawfile/blockingpoll_unsafe_amd64.go:24:6: blockingPoll redeclared in this block
	previous declaration at src/netstack/tcpip/link/rawfile/blockingpoll_unsafe.go:24:66

```


实验结果：
```txt
2019/01/27 16:02:08 main.go:38: tap: tap0, listeAddr: 192.168.1.1, portName: 9000
2019/01/27 16:02:08 ports.go:131: new transport: 6, port: 9000
2019/01/27 16:02:08 ports.go:131: new transport: 17, port: 9000
2019/01/27 16:02:08 ports.go:178: delete transport: 6, port: 9000
2019/01/27 16:02:08 ports.go:178: delete transport: 17, port: 9000
```
该实验同时监听 tcp 和 udp 的 9000 端口，之前说过这样是可以的，但如果你监听 tcp 的 9000 端口两次，那是会报错的，同学们可以自己试试。

## 三、总结

端口在 tcpip 协议栈中算是比较简单的概念，提出端口的本质需求是希望能将数据包准确的发给某台主机上的进程，实现进程与进程之间的通信。
协议栈全局管理端口，一个端口被分配以后，不允许给其他进程使用，但是要注意的是端口是`网络层协议地址+传输层协议号+端口号`来区分的，比如：  

1. `ipv4的tcp 80`端口和`ipv4的udp 80`端口不会冲突。
2. 如果你主机有两个 ip 地址 ip1 和 ip2，那么你同时监听`ip1:80`和`ip2:80`不会冲突。
3. `ipv4的tcp 80`端口和`ipv6的tcp 80`端口不会冲突。

 
