---
show: step
version: 1.0
enable_checker: true
---
# Link

## 一、实验说明

#### 1.1 实验内容

本节主要介绍链路层的基本实现，主要讲以太网网卡、虚拟网卡和 arp 协议。

#### 1.2 知识点

- 以太网的基本参数
- linux 的 tun/tap 虚拟网卡介绍
- 实现 tap 网卡的数据处理，包括读取与写入
- 协议栈网卡 IO 和管理
- 以太网数据帧协议
- arp 协议的处理和实现

#### 1.3 实验环境

- Go 1.12.1
- Xfce 终端

#### 1.4 代码获取

```bash
$ wget http://labfile.oss.aliyuncs.com/courses/1300/netstack.zip
$ unzip netstack.zip
```

## 二、链路层的介绍和基本实现

本节主要介绍链路层的基本实现，主要讲以太网网卡、虚拟网卡和 arp 协议。

#### 2.1 链路层的目的

数据链路层属于计算机网络的底层，使用的信道主要有点对点信道和广播信道两种类型。   
在 TCP/IP 协议族中，数据链路层主要有以下几个目的：  

1. 接收和发送链路层数据，提供 io 的能力。
2. 为 IP 模块发送和接收数据  
3. 为 ARP 模块发送 ARP 请求和接收 ARP 应答  
4. 为 RARP 模块发送 RARP 请求和接收 RARP 应答  

TCP/IP 支持多种不同的链路层协议，这取决于网络所使用的硬件。 
数据链路层的协议数据单元—`帧`：将 IP 层（网络层）的数据报添加首部和尾部封装成帧。  
数据链路层协议有许多种，都会解决三个基本问题，封装成帧，透明传输，差错检测。  

#### 2.2 以太网介绍

我们这章讲的是链路层，为何要讲以太网，那是因为以太网实在应用太广了，以至于我们在现实生活中看到的链路层协议的数据封装都是以太网协议封装的，所以要实现链路层数据的处理，我们必须要了解以太网。

以太网（Ethernet）是一种计算机局域网技术。IEEE 组织的 IEEE 802.3 标准制定了以太网的技术标准，它规定了包括物理层的连线、电子信号和介质访问层协议的内容。以太网是目前应用最普遍的局域网技术，取代了其他局域网标准如令牌环、FDDI 和 ARCNET。以太网协议，是当今现有局域网采用的最通用的通信协议标准，故可认为以太网就是局域网。

#### 2.3 链路层的寻址

通信当然得知道发送者的地址和接收者的地址，这是最基础的。以太网规定，所有连入网络的设备，都必须具有“网卡”接口。然后数据包是从一块网卡，传输到另一块网卡的。网卡的地址，就是数据包的发送地址和接收地址，叫做`MAC`地址，也叫物理地址，这是最底层的地址。每块网卡出厂的时候，都有一个全世界独一无二的 MAC 地址，长度是 48 个二进制位，通常用 12 个十六进制数表示。有了这个地址，我们可以定位网卡和数据包的路径了。

#### 2.4 MTU（最大传输单元）

MTU 表示在链路层最大的传输单元，也就是链路层每一帧数据的数据内容最大长度，单位为字节，MTU 是协议栈实现一个很重要的参数，请大家务必理解该参数。一般网卡默认 MTU 是 1500，当你往网卡写入的内容超过 1518bytes，就会报错，后面我们可以写代码试试。

### 三、linux 上的网卡

![datalink](https://doc.shiyanlou.com/document-uid949121labid10418timestamp1555399038307.png/wm) 

上面的图片是 linux 上链路层的实现，链路层的实现可以分为三层，真实的以太网卡，网卡驱动，网卡逻辑抽象。
真实的网卡我们不关心，因为那是硬件工程，我们只需要知道，它能接收和发送网络数据给网卡驱动就好了。
网卡驱动我们也不关心，一般驱动都是网卡生产商就写好了，我们只需知道，它能接收协议栈的数据发送给网卡，
接收网卡的数据发送给协议栈。网卡逻辑抽象表示，这个是我们关心的，我需要对真实的网卡进行抽象，
在系统中表示，也需要对抽象的网卡进行管理。

注意：后面系统中网卡的逻辑抽象我们都描述为网卡。

比如在 linux 上，当你敲下 ifconfig 命令:  

```txt
eth0      Link encap:Ethernet  HWaddr 00:16:3e:08:a1:7a
          inet addr:172.18.153.158  Bcast:172.18.159.255  Mask:255.255.240.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:285941546 errors:0 dropped:0 overruns:0 frame:0
          TX packets:281609568 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:142994767953 (142.9 GB)  TX bytes:44791940275 (44.7 GB)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:363350690 errors:0 dropped:0 overruns:0 frame:0
          TX packets:363350690 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1
          RX bytes:28099158493 (28.0 GB)  TX bytes:28099158493 (28.0 GB)
```

你会看到两个网卡，一个 eth0 以太网网卡，一个 lo 本地回环网卡。还可以看到两个网卡的信息，当我们要表示一个网卡的时候，需要具备几个属性：

1. 网卡的名字、类型和 MAC 地址  
`eth0      Link encap:Ethernet  HWaddr 00:16:3e:08:a1:7a`     
`eth0`是网卡名，方便表示一个网卡，网卡名在同个系统里不能重复。      
`Link encap:Ethernet` 表示该网卡类型为以太网网卡。  
`HWaddr 00:16:3e:08:a1:7a` 表示 MAC 地址 00:16:3e:08:a1:7a，是链路层寻址的地址。  

2. 网卡的 IP 地址及掩码  
`inet addr:172.18.153.158  Bcast:172.18.159.255  Mask:255.255.240.0`   
`inet addr:172.18.153.158` 表示该网卡的 ipv4 地址是 172.18.153.158。  
`Bcast:172.18.159.255` 表示该网卡 ip 层的广播地址。  
`255.255.240.0` 该网卡的子网掩码。

3. 网卡的状态和 MTU  
`UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1`    
`UP BROADCAST RUNNING MULTICAST` 都是表示网卡的状态，UP（代表网卡开启状态）BROADCAST (支持广播) RUNNING（代表网卡的网线被接上）MULTICAST（支持组播）。  
`MTU:1500` 最大传输单元为 1500 字节。  
`Metric:1` 接口度量值为 1，接口度量值表示在这个路径上发送一个分组的成本。  


### 3.1 linux 的虚拟网卡介绍

实现协议栈，我们需要一个网卡，因为这样我们才能接收和发送网络数据，但是一般情况下，我们电脑的操作系统已经帮我们管理好网卡了，我们想实现自由的控制网卡是不太方便的，还好 linux 系统还有另一个功能-`虚拟网卡`，它是操作系统虚拟出来的一个网卡，我们协议栈的实现都是基于虚拟网卡，用虚拟网卡的好处是：

1. 对于用户来说虚拟网卡和真实网卡几乎没有差别，而且我们控制或更改虚拟网卡大部分情况下不会影响到真实的网卡，也就不会影响到用户的网络。
2. 虚拟网卡的数据可以直接从用户态直接读取和写入，这样我们就可以直接在用户态编写协议栈。

#### 3.1.1 Linux 中虚拟网络设备

TUN/TAP 设备、VETH 设备、Bridge 设备、Bond 设备、VLAN 设备、MACVTAP 设备，下面我们只讲 tun/tap 设备，对其他虚拟设备感兴趣的同学可以去网上自行搜索。  
TAP/TUN 设备是一种让用户态和内核之前进行数据交换的虚拟设备，TAP 工作在二层，TUN 工作在三层，TAP/TUN 网卡的两头分别是内核网络协议栈和用户层,其作用是将协议栈中的部分数据包转发给用户空间的应用程序，给用户空间的程序一个处理数据包的机会。当我们想在 linux 中创建一个 TAP 设备时，其实很容易，像普通文件一样打开字符设备`/dev/net/tun`可以得到一个文件描述符，接着用系统调用 ioctl 将文件描述符和 kernel 的 tap 驱动绑定在一起，那么之后对该文件描述符的读写就是对虚拟网卡 TAP 的读写。详细的实现可以看 (tuntap)[https://www.kernel.org/doc/Documentation/networking/tuntap.txt] 所以最终我们实现的协议栈和 TAP 虚拟网卡的关系，如下图：  

 +----------------------+
 |  userland netstack   |
 +----------------------+
 |         tap          |
 +----------------------+
 |       kernel         |
 +----------------------+


### 3.2 tap 网卡实验

在 linux 中创建虚拟网卡，我们可以用 linux 自带的 ip 命令来实现，关于 ip 命令的更多用法请看 man ip。  

创建 tap 网卡  
``` shell
# 创建一个tap模式的虚拟网卡
sudo ip tuntap add mode tap <device-name>
# 开启该网卡
sudo ip link set <device-name> up
# 设置该网卡的ip及掩码
sudo ip addr add <ipv4-address>/<mask-length> dev <device-name>
```
删除网卡可以用  

```bash
# 删除虚拟网卡
sudo ip tuntap del mode tap <device-name>
```

我们创建一个为名 tap0，ip 及掩码为 192.168.1.1/24 的虚拟网卡  
执行 ifconfig 看看，会看到一个 tap0 的网卡  

```txt
tap0      Link encap:Ethernet  HWaddr 22:e2:f2:93:ff:bf
          inet addr:192.168.1.1  Bcast:0.0.0.0  Mask:255.255.255.0
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```
看起来和真实的网卡没有任何区别，接下来我们自己用 golang 来实现创建网卡。

### 3.3 用 golang 创建许你网卡

golang 创建 tuntap 网卡的库实现，在`$GOPATH/src/netstack/tcpip/link/tuntap`目录下创建源文件 `tuntap.go`，输入以下代码：

```go
// +build linux

package tuntap

import (
	"errors"
	"fmt"
	"os/exec"
	"syscall"
	"unsafe"
)

const (
	TUN = 1
	TAP = 2
)

var (
	ErrDeviceMode = errors.New("unsupport device mode")
)

type rawSockaddr struct {
	Family uint16
	Data   [14]byte
}

// 虚拟网卡设置的配置
type Config struct {
	Name string // 网卡名
	Mode int    // 网卡模式，TUN or TAP
}

// NewNetDev根据配置返回虚拟网卡的文件描述符
func NewNetDev(c *Config) (fd int, err error) {
	switch c.Mode {
	case TUN:
		fd, err = newTun(c.Name)
	case TAP:
		fd, err = newTAP(c.Name)
	default:
		err = ErrDeviceMode
		return
	}
	if err != nil {
		return
	}
	return
}

// SetLinkUp 让系统启动该网卡
func SetLinkUp(name string) (err error) {
	// ip link set <device-name> up
	out, cmdErr := exec.Command("ip", "link", "set", name, "up").CombinedOutput()
	if cmdErr != nil {
		err = fmt.Errorf("%v:%v", cmdErr, string(out))
		return
	}
	return
}

// SetRoute 通过ip命令添加路由
func SetRoute(name, cidr string) (err error) {
	// ip route add 192.168.1.0/24 dev tap0
	out, cmdErr := exec.Command("ip", "route", "add", cidr, "dev", name).CombinedOutput()
	if cmdErr != nil {
		err = fmt.Errorf("%v:%v", cmdErr, string(out))
		return
	}
	return
}

// AddIP 通过ip命令添加IP地址
func AddIP(name, ip string) (err error) {
	// ip addr add 192.168.1.1 dev tap0
	out, cmdErr := exec.Command("ip", "addr", "add", ip, "dev", name).CombinedOutput()
	if cmdErr != nil {
		err = fmt.Errorf("%v:%v", cmdErr, string(out))
		return
	}
	return
}

func GetHardwareAddr(name string) (string, error) {
	fd, err := syscall.Socket(syscall.AF_UNIX, syscall.SOCK_DGRAM, 0)
	if err != nil {
		return "", err
	}

	defer syscall.Close(fd)

	var ifreq struct {
		name [16]byte
		addr rawSockaddr
		_    [8]byte
	}

	copy(ifreq.name[:], name)
	_, _, errno := syscall.Syscall(syscall.SYS_IOCTL, uintptr(fd), syscall.SIOCGIFHWADDR, uintptr(unsafe.Pointer(&ifreq)))
	if errno != 0 {
		return "", errno
	}

	mac := ifreq.addr.Data[:6]
	return string(mac[:]), nil
}

// newTun新建一个tun模式的虚拟网卡，然后返回该网卡的文件描述符
// IFF_NO_PI表示不需要包信息
func newTun(name string) (int, error) {
	return open(name, syscall.IFF_TUN|syscall.IFF_NO_PI)
}

// newTAP新建一个tap模式的虚拟网卡，然后返回该网卡的文件描述符
func newTAP(name string) (int, error) {
	return open(name, syscall.IFF_TAP|syscall.IFF_NO_PI)
}

// 先打开一个字符串设备，通过系统调用将虚拟网卡和字符串设备fd绑定在一起
func open(name string, flags uint16) (int, error) {
	// 打开tuntap的字符设备，得到字符设备的文件描述符
	fd, err := syscall.Open("/dev/net/tun", syscall.O_RDWR, 0)
	if err != nil {
		return -1, err
	}

	var ifr struct {
		name  [16]byte
		flags uint16
		_     [22]byte
	}

	copy(ifr.name[:], name)
	ifr.flags = flags
	// 通过ioctl系统调用，将fd和虚拟网卡驱动绑定在一起
	_, _, errno := syscall.Syscall(syscall.SYS_IOCTL, uintptr(fd), syscall.TUNSETIFF, uintptr(unsafe.Pointer(&ifr)))
	if errno != 0 {
		syscall.Close(fd)
		return -1, errno
	}
	return fd, nil
}
```

```checker
- name: check tuntap exist
  script: |
    #!/bin/bash
    ls -l /home/shiyanlou/golang/src/netstack/tcpip/link/tuntap/tuntap.go
  error: 没有找到 /home/shiyanlou/golang/src/netstack/tcpip/link/tuntap/tuntap.go 文件
```

根据这个库，写一个从网卡读取数据的程序，并打印读取到的字节数。代码存放在`src/lab/link/tap1/main.go`。

```go
package main

import (
	"log"
	"netstack/tcpip/link/rawfile"//该包可通过1.4节提供的方式获取
	"netstack/tcpip/link/tuntap"
)

func main() {
	tapName := "tap0"
	c := &tuntap.Config{tapName, tuntap.TAP}
	fd, err := tuntap.NewNetDev(c)
	if err != nil {
		panic(err)
	}

	// 启动tap网卡
	tuntap.SetLinkUp(tapName)
	// 添加ip地址
	tuntap.AddIP(tapName, "192.168.1.1")

	buf := make([]byte, 1<<16)
	for {
		rn, err := rawfile.BlockingRead(fd, buf)
		if err != nil {
			log.Println(err)
			continue
		}
		log.Printf("read %d bytes", rn)
	}
}
```

```checker
- name: check tap1 exist
  script: |
    #!/bin/bash
    ls -l /home/shiyanlou/golang/src/lab/link/tap1/main.go
  error: 没有找到 /home/shiyanlou/golang/src/lab/link/tap1/main.go 文件
```

然后进入目录 `src/lab/link/tap1` 编译代码。

```bash
cd $GOPATH/src/lab/link/tap1
go build
```
会生成一个叫`tap1`的可执行文件，我们执行它

```
TODO: 本节内容执行 3.3 步骤的过程中会报错, 无法继续实验

# netstack/tcpip/link/rawfile
../../../netstack/tcpip/link/rawfile/blockingpoll_unsafe_amd64.go:24:6: blockingPoll redeclared in this block
	
previous declaration at ../../../netstack/tcpip/link/rawfile/blockingpoll_unsafe.go:24:66

```

```bash
sudo ./tap1
```
稍等一会再打开另一个终端，利用 tcpdump 抓取经过 tap0 网卡的数据，如果执行 tap1，立马就抓包，可能会抓到一些 ipv6 的组播包，我们这里先忽略。

```shell
sudo tcpdump -i tap0 -n 
```
再打开另一个终端，我们试 ping 一下 192.168.1.1

```shell
ping 192.168.1.1
```

但是 tcpdump 抓取数据的终端和我们自己写的打印网卡数据的终端中没有任何 icmp 数据，这是为何？这是因为当给一个网卡添加 ip 地址的时候，系统会将相应的路由添加到“本地路由表”，正因为这样，即使看起来 192.168.1.1 是 tap0 网卡的地址，但实际上我们 ping 的数据并没有走到 tap0 网卡，而是在 lo 网卡上，我们可以试试在终端抓去 lo 网卡数据

```shell
sudo tcpdump -i lo -n 
```
再 ping 一下 192.168.1.1 tcpdump 的输出

```shell
listening on lo, link-type EN10MB (Ethernet), capture size 262144 bytes
22:40:18.028585 IP 192.168.1.1 > 192.168.1.1: ICMP echo request, id 29728, seq 1, length 64
22:40:18.028599 IP 192.168.1.1 > 192.168.1.1: ICMP echo reply, id 29728, seq 1, length 64
22:40:19.029912 IP 192.168.1.1 > 192.168.1.1: ICMP echo request, id 29728, seq 2, length 64
22:40:19.029925 IP 192.168.1.1 > 192.168.1.1: ICMP echo reply, id 29728, seq 2, length 64
```

查看本地路由的信息，通过`ip route show table local`命令。

```bash
broadcast 10.211.55.0 dev enp0s5  proto kernel  scope link  src 10.211.55.14
broadcast 10.211.55.0 dev enp0s6  proto kernel  scope link  src 10.211.55.16
local 10.211.55.14 dev enp0s5  proto kernel  scope host  src 10.211.55.14
local 10.211.55.16 dev enp0s6  proto kernel  scope host  src 10.211.55.16
broadcast 10.211.55.255 dev enp0s5  proto kernel  scope link  src 10.211.55.14
broadcast 10.211.55.255 dev enp0s6  proto kernel  scope link  src 10.211.55.16
broadcast 127.0.0.0 dev lo  proto kernel  scope link  src 127.0.0.1
local 127.0.0.0/8 dev lo  proto kernel  scope host  src 127.0.0.1
local 127.0.0.1 dev lo  proto kernel  scope host  src 127.0.0.1
broadcast 127.255.255.255 dev lo  proto kernel  scope link  src 127.0.0.1
broadcast 192.168.1.0 dev tap0  proto kernel  scope link  src 192.168.1.1
local 192.168.1.1 dev tap0  proto kernel  scope host  src 192.168.1.1
broadcast 192.168.1.255 dev tap0  proto kernel  scope link  src 192.168.1.1
```

可以看到倒数第二行，表示了 192.168.1.1 这个地址，在 local 路由表里。同时路由表也显示，只有 192.168.1.1 这个地址在路由表里，该网段的其他地址不在本地路由，那么应该会进入 tap0 网卡，比如我们试试 192.168.1.2 这个地址，ping 一下

```shell
PING 192.168.1.2 (192.168.1.2) 56(84) bytes of data.
From 192.168.1.1 icmp_seq=1 Destination Host Unreachable
From 192.168.1.1 icmp_seq=2 Destination Host Unreachable
```
然后 tcpdump 在 tap0 网卡上的输出
```shell
listening on tap0, link-type EN10MB (Ethernet), capture size 262144 bytes
22:55:58.322022 ARP, Request who-has 192.168.1.2 tell 192.168.1.1, length 28
22:55:59.320824 ARP, Request who-has 192.168.1.2 tell 192.168.1.1, length 28
```

说明 tap0 网卡收到了 arp 请求，至于我们使用 ping 之后为何接收到的是 arp 请求报文而不是 icmp 报文，这是因为系统不知道 192.168.1.2 的 MAC 地址，后面会详细说明。

在上面的程序中，我们也可以看到上面的程序有打印：

```bash
2018/11/11 23:54:10 read 42 bytes
2018/11/11 23:54:11 read 42 bytes
2018/11/11 23:54:12 read 42 bytes
2018/11/11 23:54:13 read 42 bytes
```
其实在链路层通信，是可以不需要 ip 地址的，我们可以手动配置路由，将数据导入虚拟网卡，现在更改我们的程序，代码存放在`src/lab/link/tap2/main.go`

```go
package main

import (
	"log"
	"netstack/tcpip/link/rawfile"
	"netstack/tcpip/link/tuntap"
)

func main() {
	tapName := "tap0"
	c := &tuntap.Config{tapName, tuntap.TAP}
	fd, err := tuntap.NewNetDev(c)
	if err != nil {
		panic(err)
	}

	// 启动tap网卡
	tuntap.SetLinkUp(tapName)
	// 设置路由
	tuntap.SetRoute(tapName, "192.168.1.0/24")

	buf := make([]byte, 1<<16)
	for {
		rn, err := rawfile.BlockingRead(fd, buf)
		if err != nil {
			log.Println(err)
			continue
		}
		log.Printf("read %d bytes", rn)
	}
}
```

```checker
- name: check tap2 exist
  script: |
    #!/bin/bash
    ls -l /home/shiyanlou/golang/src/lab/link/tap2/main.go
  error: 没有找到 /home/shiyanlou/golang/src/lab/link/tap2/main.go 文件
```

进入目录`src/lab/link/tap2`，然后编译代码。

```shel
cd GOPATH/src/lab/link/tap2
go build
```
会生成一个叫`tap2`的可执行文件，我们执行它

```
TODO: 本节内容执行 3.3 步骤的过程中会报错, 无法继续实验

# netstack/tcpip/link/rawfile
../../../netstack/tcpip/link/rawfile/blockingpoll_unsafe_amd64.go:24:6: blockingPoll redeclared in this block
	
previous declaration at ../../../netstack/tcpip/link/rawfile/blockingpoll_unsafe.go:24:66

```

```shell
sudo ./tap2
```

稍等一会再打开另一个终端，利用 tcpdump 抓取经过 tap0 网卡的数据。

```shell
sudo tcpdump -i tap0 -n 
```
再打开另一个终端，我们试 ping 一下 192.168.1.1
```shell
ping 192.168.1.1
```
结果：
```txt
2019/04/10 11:12:57 read 42 bytes
2019/04/10 11:12:58 read 42 bytes
2019/04/10 11:12:59 read 42 bytes
2019/04/10 11:13:16 read 42 bytes
2019/04/10 11:13:17 read 42 bytes
2019/04/10 11:13:18 read 42 bytes
```
这时候你 ping 192.168.1.0/24 网段的任何一个地址都是进入 tap0 网卡，这样我们就可以实验和处理 tap0 网上上的数据了。目前我们只看到了网卡有读取到数据，而且抓包显示我们现在接收到的数据都是 arp 请求，后面会实现对 arp 报文的处理，接下来我们开始处理网卡的数据并封装链路层，实现网卡的 io。

```checker
- name: check random_output exist
  script: |
    #!/bin/bash
    ls -l /home/shiyanlou/golang2048_game/random_output.go
  error: 没有找到 /home/shiyanlou/golang2048_game/random_output.go 文件
```


## 四、协议栈中的网卡 IO 和网卡管理

数据在链路层传输都是一帧一帧传输的，就像发送邮件一样，将信放入信封中，接着把信封邮寄出去，这样可以把一段信息和另一段信息区分开来，下面先介绍数据帧格式。 

### 4.1 链路层数据帧

历史上以太网帧格式有好几种，这里只介绍`Ethernet V2`帧格式，如下图：  
![datalink-frame](https://doc.shiyanlou.com/document-uid949121labid10418timestamp1555401300841.png/wm) 

目的 MAC 地址：目的设备的 MAC 物理地址。  
源 MAC 地址：发送设备的 MAC 物理地址。  
类型：表示后面所跟数据包的协议类型，例如 Type 为`0x8000`时为 IPv4 协议包，Type 为`0x8060`时，后面为 ARP 协议包。  
数据：表示该帧的数据内容，长度为 46～1500 字节，包含网络层、传输层和应用层的数据。  

关于协议类型详细可以看 (https://en.wikipedia.org/wiki/EtherType)[https://en.wikipedia.org/wiki/EtherType] 。

### 4.2 以太网的数据处理

既然前面我们已经知道了链路层数据帧格式，也知道了链路层协议头的详细信息，那么现在就根据这些信息来处理以太网数据。我们把处理头部数据的代码都放在 header 包中:`src/netstack/tcpip/header/eth.go` 

```go
package header

import (
	"encoding/binary"

	"netstack/tcpip"
)

// 以太网帧头部信息的偏移量
const (
	dstMAC  = 0
	srcMAC  = 6
	ethType = 12
)

// EthernetFields表示链路层以太网帧的头部
type EthernetFields struct {
	// 源地址
	SrcAddr tcpip.LinkAddress

	// 目的地址
	DstAddr tcpip.LinkAddress

	// 协议类型
	Type tcpip.NetworkProtocolNumber
}

// Ethernet以太网数据包的封装
type Ethernet []byte

const (
	// EthernetMinimumSize以太网帧最小的长度
	EthernetMinimumSize = 14

	// EthernetAddressSize以太网地址的长度
	EthernetAddressSize = 6
)

// SourceAddress从帧头部中得到源地址
func (b Ethernet) SourceAddress() tcpip.LinkAddress {
	return tcpip.LinkAddress(b[srcMAC:][:EthernetAddressSize])
}

// DestinationAddress从帧头部中得到目的地址
func (b Ethernet) DestinationAddress() tcpip.LinkAddress {
	return tcpip.LinkAddress(b[dstMAC:][:EthernetAddressSize])
}

// Type从帧头部中得到协议类型
func (b Ethernet) Type() tcpip.NetworkProtocolNumber {
	return tcpip.NetworkProtocolNumber(binary.BigEndian.Uint16(b[ethType:]))
}

// Encode根据传入的帧头部信息编码成Ethernet二进制形式，注意Ethernet应先分配好内存
func (b Ethernet) Encode(e *EthernetFields) {
	binary.BigEndian.PutUint16(b[ethType:], uint16(e.Type))
	copy(b[srcMAC:][:EthernetAddressSize], e.SrcAddr)
	copy(b[dstMAC:][:EthernetAddressSize], e.DstAddr)
}
```

```checker
- name: check eth exist
  script: |
    #!/bin/bash
    ls -l /home/shiyanlou/golang/src/netstack/tcpip/header/eth.go
  error: 没有找到 /home/shiyanlou/golang/src/netstack/tcpip/header/eth.go 文件
```

### 4.3 网卡的 IO 实现

所谓 io 就是数据的输入输出，对于网卡来说就是接收或发送数据，接收意味着对以太网帧解封装和提交给网络层，发送意味着对上层数据的封装和写入网卡。协议栈定义了链路层的接口如下：

```go
// LinkEndpoint是由数据链路层协议（例如，以太网，环回，原始）实现的接口，并由网络层协议用于通过实施者的数据链路端点发送数据包。
type LinkEndpoint interface {
	// MTU is the maximum transmission unit for this endpoint. This is
	// usually dictated by the backing physical network; when such a
	// physical network doesn't exist, the limit is generally 64k, which
	// includes the maximum size of an IP packet.
	// MTU是此端点的最大传输单位。这通常由支持物理网络决定;
	// 当这种物理网络不存在时，限制通常为64k，其中包括IP数据包的最大大小。
	MTU() uint32

	// Capabilities returns the set of capabilities supported by the
	// endpoint.
	// Capabilities返回链路层端点支持的功能集。
	Capabilities() LinkEndpointCapabilities

	// MaxHeaderLength returns the maximum size the data link (and
	// lower level layers combined) headers can have. Higher levels use this
	// information to reserve space in the front of the packets they're
	// building.
	// MaxHeaderLength 返回数据链接（和较低级别的图层组合）标头可以具有的最大大小。
	// 较高级别使用此信息来保留它们正在构建的数据包前面预留空间。
	MaxHeaderLength() uint16

	// LinkAddress returns the link address (typically a MAC) of the
	// link endpoint.
	// 本地链路层地址
	LinkAddress() tcpip.LinkAddress

	// WritePacket writes a packet with the given protocol through the given
	// route.
	//
	// To participate in transparent bridging, a LinkEndpoint implementation
	// should call eth.Encode with header.EthernetFields.SrcAddr set to
	// r.LocalLinkAddress if it is provided.
	// WritePacket通过给定的路由写入具有给定协议的数据包。
	// 要参与透明桥接，LinkEndpoint实现应调用eth.Encode，并将header.EthernetFields.SrcAddr设置为r.LocalLinkAddress（如果已提供）。
	WritePacket(r *Route, hdr buffer.Prependable, payload buffer.VectorisedView, protocol tcpip.NetworkProtocolNumber) *tcpip.Error

	// Attach attaches the data link layer endpoint to the network-layer
	// dispatcher of the stack.
	// Attach 将数据链路层端点附加到协议栈的网络层调度程序。
	Attach(dispatcher NetworkDispatcher)

	// IsAttached returns whether a NetworkDispatcher is attached to the
	// endpoint.
	// 是否已经添加了网络层调度器
	IsAttached() bool
}
```

下面实现一个实例来实现上面的链路层接口

```go
// 负责底层网卡的io读写以及数据分发
type endpoint struct {
	// fd is the file descriptor used to send and receive packets.
	// 发送和接收数据的文件描述符
	fd int

	// mtu (maximum transmission unit) is the maximum size of a packet.
	// 单个帧的最大长度
	mtu uint32

	// hdrSize specifies the link-layer header size. If set to 0, no header
	// is added/removed; otherwise an ethernet header is used.
	// 以太网头部长度
	hdrSize int

	// addr is the address of the endpoint.
	// 网卡地址
	addr tcpip.LinkAddress

	// caps holds the endpoint capabilities.
	// 网卡的能力
	caps stack.LinkEndpointCapabilities

	// closed is a function to be called when the FD's peer (if any) closes
	// its end of the communication pipe.
	closed func(*tcpip.Error)

	iovecs     []syscall.Iovec
	views      []buffer.View
	dispatcher stack.NetworkDispatcher

	// handleLocal indicates whether packets destined to itself should be
	// handled by the netstack internally (true) or be forwarded to the FD
	// endpoint (false).
	// handleLocal指示发往自身的数据包是由内部netstack处理（true）还是转发到FD端点（false）。
	// Resend packets back to netstack if destined to itself
	// Add option to redirect packet back to netstack if it's destined to itself.
	// This fixes the problem where connecting to the local NIC address would
	// not work, e.g.:
	// echo bar | nc -l -p 8080 &
	// echo foo | nc 192.168.0.2 8080
	handleLocal bool
}

// Options specify the details about the fd-based endpoint to be created.
// 创建fdbase端的一些选项参数
type Options struct {
	FD                 int
	MTU                uint32
	ClosedFunc         func(*tcpip.Error)
	Address            tcpip.LinkAddress
	ResolutionRequired bool
	SaveRestore        bool
	ChecksumOffload    bool
	DisconnectOk       bool
	HandleLocal        bool
}

// New creates a new fd-based endpoint.
//
// Makes fd non-blocking, but does not take ownership of fd, which must remain
// open for the lifetime of the returned endpoint.
// 根据选项参数创建一个链路层的endpoint，并返回该endpoint的id
func New(opts *Options) tcpip.LinkEndpointID {
	syscall.SetNonblock(opts.FD, true)

	caps := stack.LinkEndpointCapabilities(0)
	if opts.ResolutionRequired {
		caps |= stack.CapabilityResolutionRequired
	}
	if opts.ChecksumOffload {
		caps |= stack.CapabilityChecksumOffload
	}
	if opts.SaveRestore {
		caps |= stack.CapabilitySaveRestore
	}
	if opts.DisconnectOk {
		caps |= stack.CapabilityDisconnectOk
	}

	e := &endpoint{
		fd:          opts.FD,
		mtu:         opts.MTU,
		caps:        caps,
		closed:      opts.ClosedFunc,
		addr:        opts.Address,
		hdrSize:     header.EthernetMinimumSize,
		views:       make([]buffer.View, len(BufConfig)),
		iovecs:      make([]syscall.Iovec, len(BufConfig)),
		handleLocal: opts.HandleLocal,
	}
	// 全局注册链路层设备
	return stack.RegisterLinkEndpoint(e)
}

// Attach launches the goroutine that reads packets from the file descriptor and
// dispatches them via the provided dispatcher.
// Attach 启动从文件描述符中读取数据包的goroutine，并通过提供的分发函数来分发数据报。
func (e *endpoint) Attach(dispatcher stack.NetworkDispatcher) {
	e.dispatcher = dispatcher
	// Link endpoints are not savable. When transportation endpoints are
	// saved, they stop sending outgoing packets and all incoming packets
	// are rejected.
	// 链接端点不可靠。保存传输端点后，它们将停止发送传出数据包，并拒绝所有传入数据包。
	go e.dispatchLoop()
}

// IsAttached implements stack.LinkEndpoint.IsAttached.
// 判断是否Attach了
func (e *endpoint) IsAttached() bool {
	return e.dispatcher != nil
}

// MTU implements stack.LinkEndpoint.MTU. It returns the value initialized
// during construction.
// 返回当前MTU值
func (e *endpoint) MTU() uint32 {
	return e.mtu
}

// Capabilities implements stack.LinkEndpoint.Capabilities.
// 返回当前网卡的能力
func (e *endpoint) Capabilities() stack.LinkEndpointCapabilities {
	return e.caps
}

// MaxHeaderLength returns the maximum size of the link-layer header.
// 返回当前以太网头部信息长度
func (e *endpoint) MaxHeaderLength() uint16 {
	return uint16(e.hdrSize)
}

// LinkAddress returns the link address of this endpoint.
// 返回当前MAC地址
func (e *endpoint) LinkAddress() tcpip.LinkAddress {
	return e.addr
}

// WritePacket writes outbound packets to the file descriptor. If it is not
// currently writable, the packet is dropped.
// 将上层的报文经过链路层封装，写入网卡中，如果写入失败则丢弃该报文
func (e *endpoint) WritePacket(r *stack.Route, hdr buffer.Prependable, payload buffer.VectorisedView,
	protocol tcpip.NetworkProtocolNumber) *tcpip.Error {
	// 如果目标地址就是设备本身自己，那么将报文重新返回给协议栈
	if e.handleLocal && r.LocalAddress != "" && r.LocalAddress == r.RemoteAddress {
		views := make([]buffer.View, 1, 1+len(payload.Views()))
		views[0] = hdr.View()
		views = append(views, payload.Views()...)
		vv := buffer.NewVectorisedView(len(views[0])+payload.Size(), views)
		e.dispatcher.DeliverNetworkPacket(e, r.RemoteLinkAddress, r.LocalLinkAddress, protocol, vv)
		return nil
	}

	// 封装增加以太网头部
	eth := header.Ethernet(hdr.Prepend(header.EthernetMinimumSize))
	ethHdr := &header.EthernetFields{
		DstAddr: r.RemoteLinkAddress,
		Type:    protocol,
	}

	// Preserve the src address if it's set in the route.
	// 如果路由信息中有配置源MAC地址，那么使用该地址，
	// 如果没有，则使用本网卡的地址
	if r.LocalLinkAddress != "" {
		ethHdr.SrcAddr = r.LocalLinkAddress
	} else {
		ethHdr.SrcAddr = e.addr
	}
	eth.Encode(ethHdr)

	// 写入网卡中
	// log.Printf("write to nic %d bytes", hdr.UsedLength()+payload.Size())
	if payload.Size() == 0 {
		return rawfile.NonBlockingWrite(e.fd, hdr.View())
	}

	return rawfile.NonBlockingWrite2(e.fd, hdr.View(), payload.ToView())
}

func (e *endpoint) capViews(n int, buffers []int) int {
	c := 0
	for i, s := range buffers {
		c += s
		if c >= n {
			e.views[i].CapLength(s - (c - n))
			return i + 1
		}
	}
	return len(buffers)
}

// 按bufConfig的长度分配内存大小
// 注意e.views和e.iovecs共用相同的内存块
func (e *endpoint) allocateViews(bufConfig []int) {
	for i, v := range e.views {
		if v != nil {
			break
		}
		b := buffer.NewView(bufConfig[i])
		e.views[i] = b
		e.iovecs[i] = syscall.Iovec{
			Base: &b[0],
			Len:  uint64(len(b)),
		}
	}
}

// dispatch reads one packet from the file descriptor and dispatches it.
// 从网卡中读取一个数据报，
func (e *endpoint) dispatch() (bool, *tcpip.Error) {
	// 读取数据缓存的分配
	e.allocateViews(BufConfig)

	// 从网卡中读取数据
	n, err := rawfile.BlockingReadv(e.fd, e.iovecs)
	if err != nil {
		return false, err
	}

	// 如果比头部长度还小，直接丢弃
	if n <= e.hdrSize {
		return false, nil
	}

	var (
		p                             tcpip.NetworkProtocolNumber
		remoteLinkAddr, localLinkAddr tcpip.LinkAddress
	)
	// 获取以太网头部信息
	eth := header.Ethernet(e.views[0])
	p = eth.Type()
	remoteLinkAddr = eth.SourceAddress()
	localLinkAddr = eth.DestinationAddress()

	used := e.capViews(n, BufConfig)
	vv := buffer.NewVectorisedView(n, e.views[:used])
	// 将数据内容删除以太网头部信息，也就是将数据指针指向网络层的第一个字节
	vv.TrimFront(e.hdrSize)

	// 调用nic.DeliverNetworkPacket来分发网络层数据
	e.dispatcher.DeliverNetworkPacket(e, remoteLinkAddr, localLinkAddr, p, vv)

	// Prepare e.views for another packet: release used views.
	for i := 0; i < used; i++ {
		e.views[i] = nil
	}

	return true, nil
}

// dispatchLoop reads packets from the file descriptor in a loop and dispatches
// them to the network stack.
// 循环的从fd读取数据，然后将数据包分发给协议栈
func (e *endpoint) dispatchLoop() *tcpip.Error {
	for {
		cont, err := e.dispatch()
		if err != nil || !cont {
			if e.closed != nil {
				e.closed(err)
			}
			return err
		}
	}
}

```

fdbase 实现了链路层的接口，也就具备了网卡 IO 的能力，实际上真实的操作系统的协议栈并不是像上面一样的实现，而是比这个复杂的多，但是目的都是为了实现网卡的 IO，现实中操作系统的实现是网卡驱动和中断机制来实现的网卡 IO，感兴趣的同学可以自行查找资料学习，这里不做更深入的讲述。  
下面来看看对这个链路层端的管理实现。

### 4.4 协议栈中的网卡 NIC

协议栈实现了网卡的对象，以及操作逻辑，包括新建网卡、添加 ip 地址、网络端的管理和查找、子网的添加和删除和分发网络包和传输层的包。下面介绍这些功能的实现。
协议栈中的网卡对象如下：

```go
// 代表一个网卡对象
type NIC struct {
	stack *Stack
	// 每个网卡的唯一标识号
	id tcpip.NICID
	// 网卡名，可有可无
	name string
	// 链路层端
	linkEP LinkEndpoint

	// 传输层的解复用
	demux *transportDemuxer

	mu          sync.RWMutex
	spoofing    bool
	promiscuous bool
	primary     map[tcpip.NetworkProtocolNumber]*ilist.List
	// 网络层端的记录
	endpoints map[NetworkEndpointID]*referencedNetworkEndpoint
	// 子网的记录
	subnets []tcpip.Subnet
}
```

### 4.5 新建网卡

当新建好 tap 网卡并注册到协议栈中，就可以新建 NIC 来管理它了。

```go
// CreateNIC creates a NIC with the provided id and link-layer endpoint.
// 根据nic id和linkEP id来创建和注册一个网卡对象
func (s *Stack) CreateNIC(id tcpip.NICID, linkEP tcpip.LinkEndpointID) *tcpip.Error {
	return s.createNIC(id, "", linkEP, true)
}

// 根据参数新建一个NIC
func newNIC(stack *Stack, id tcpip.NICID, name string, ep LinkEndpoint) *NIC {
	return &NIC{
		stack:     stack,
		id:        id,
		name:      name,
		linkEP:    ep,
		demux:     newTransportDemuxer(stack),
		primary:   make(map[tcpip.NetworkProtocolNumber]*ilist.List),
		endpoints: make(map[NetworkEndpointID]*referencedNetworkEndpoint),
	}
}
```

### 4.6 绑定 ip 地址和网络端的管理和查找

只有绑定了 ip 地址才能实现网络层的功能，因为网络层一定要有 ip 地址才能通信。

```go
// 在NIC上添加addr地址，注册和初始化网络层协议
// 相当于给网卡添加ip地址
func (n *NIC) addAddressLocked(protocol tcpip.NetworkProtocolNumber, addr tcpip.Address,
	peb PrimaryEndpointBehavior, replace bool) (*referencedNetworkEndpoint, *tcpip.Error) {
	// 查看是否支持该协议，若不支持则返回错误
	netProto, ok := n.stack.networkProtocols[protocol]
	if !ok {
		return nil, tcpip.ErrUnknownProtocol
	}

	// Create the new network endpoint.
	// 比如netProto为ipv4，会调用ipv4.NewEndpoint，新建一个网络层端
	ep, err := netProto.NewEndpoint(n.id, addr, n.stack, n, n.linkEP)
	if err != nil {
		return nil, err
	}

	// 获取网络层端的id，其实就是ip地址
	id := *ep.ID()
	if ref, ok := n.endpoints[id]; ok {
		// 不是替换，且该id否存在，返回错误
		if !replace {
			return nil, tcpip.ErrDuplicateAddress
		}

		n.removeEndpointLocked(ref)
	}

	ref := &referencedNetworkEndpoint{
		refs:           1,
		ep:             ep,
		nic:            n,
		protocol:       protocol,
		holdsInsertRef: true,
	}

	// Set up cache if link address resolution exists for this protocol.
	if n.linkEP.Capabilities()&CapabilityResolutionRequired != 0 {
		if _, ok := n.stack.linkAddrResolvers[protocol]; ok {
			ref.linkCache = n.stack
		}
	}

	// 注册该网络端
	n.endpoints[id] = ref

	l, ok := n.primary[protocol]
	if !ok {
		l = &ilist.List{}
		n.primary[protocol] = l
	}

	switch peb {
	case CanBePrimaryEndpoint:
		l.PushBack(ref)
	case FirstPrimaryEndpoint:
		l.PushFront(ref)
	}

	return ref, nil
}

// findEndpoint finds the endpoint, if any, with the given address.
// 根据address参数查找对应的网络层端
func (n *NIC) findEndpoint(protocol tcpip.NetworkProtocolNumber, address tcpip.Address,
	peb PrimaryEndpointBehavior) *referencedNetworkEndpoint {
	id := NetworkEndpointID{address}

	n.mu.RLock()
	ref := n.endpoints[id]
	if ref != nil && !ref.tryIncRef() {
		ref = nil
	}
	spoofing := n.spoofing
	n.mu.RUnlock()

	if ref != nil || !spoofing {
		return ref
	}

	// Try again with the lock in exclusive mode. If we still can't get the
	// endpoint, create a new "temporary" endpoint. It will only exist while
	// there's a route through it.
	n.mu.Lock()
	ref = n.endpoints[id]
	if ref == nil || !ref.tryIncRef() {
		ref, _ = n.addAddressLocked(protocol, address, peb, true)
		if ref != nil {
			ref.holdsInsertRef = false
		}
	}
	n.mu.Unlock()
	return ref
}
```

### 4.7 子网的管理

子网管理实现：

```go
// AddSubnet adds a new subnet to n, so that it starts accepting packets
// targeted at the given address and network protocol.
// AddSubnet向n添加一个新子网，以便它开始接受针对给定地址和网络协议的数据包。
func (n *NIC) AddSubnet(protocol tcpip.NetworkProtocolNumber, subnet tcpip.Subnet) {
	n.mu.Lock()
	n.subnets = append(n.subnets, subnet)
	n.mu.Unlock()
}

// RemoveSubnet removes the given subnet from n.
// 从n中删除一个子网
func (n *NIC) RemoveSubnet(subnet tcpip.Subnet) {
	n.mu.Lock()

	// Use the same underlying array.
	tmp := n.subnets[:0]
	for _, sub := range n.subnets {
		if sub != subnet {
			tmp = append(tmp, sub)
		}
	}
	n.subnets = tmp

	n.mu.Unlock()
}

// ContainsSubnet reports whether this NIC contains the given subnet.
// 判断 subnet 这个子网是否在该网卡下
func (n *NIC) ContainsSubnet(subnet tcpip.Subnet) bool {
	for _, s := range n.Subnets() {
		if s == subnet {
			return true
		}
	}
	return false
}

// Subnets returns the Subnets associated with this NIC.
// 获取该网卡的所有子网
func (n *NIC) Subnets() []tcpip.Subnet {
	n.mu.RLock()
	defer n.mu.RUnlock()
	sns := make([]tcpip.Subnet, 0, len(n.subnets)+len(n.endpoints))
	for nid := range n.endpoints {
		sn, err := tcpip.NewSubnet(nid.LocalAddress, tcpip.AddressMask(strings.Repeat("\xff", len(nid.LocalAddress))))
		if err != nil {
			// This should never happen as the mask has been carefully crafted to
			// match the address.
			panic("Invalid endpoint subnet: " + err.Error())
		}
		sns = append(sns, sn)
	}
	return append(sns, n.subnets...)
}
```
## 五、网络层数据包和传输层数据包的分发

当 NIC 从物理接口接收数据包时，将调用函数 `DeliverNetworkPacket`，用来分发网络层数据包。比如 protocol 是 arp 协议号，那么会找到`arp.HandlePacket`来处理数据报。简单来说就是根据网络层协议和目的地址来找到相应的网络层端，将网络层数据发给它，当前实现的网络层协议有 arp、ipv4 和 ipv6。

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

```
当网络层收到一个数据包时，会根据`传输层协议号、ip地址和端口`来找到相应的传输层端，将传输层数据发送给它。
```go
// DeliverTransportPacket 按照传输层协议号和传输层ID，将数据包分发给相应的传输层端
// 比如 protocol=6表示为tcp协议，将会给相应的tcp端分发消息。
func (n *NIC) DeliverTransportPacket(r *Route, protocol tcpip.TransportProtocolNumber, vv buffer.VectorisedView) {
	// 先查找协议栈是否注册了该传输层协议
	state, ok := n.stack.transportProtocols[protocol]
	if !ok {
		n.stack.stats.UnknownProtocolRcvdPackets.Increment()
		return
	}

	transProto := state.proto
	// 如果报文长度比该协议最小报文长度还小，那么丢弃它
	if len(vv.First()) < transProto.MinimumPacketSize() {
		n.stack.stats.MalformedRcvdPackets.Increment()
		return
	}

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
网卡 IO 和管理都有了，就需要实现主机的链路层寻址了，主机的链路层寻址是通过 arp 表来实现的，接下来我们就了解一下什么 arp 和 arp 报文的处理。

### 5.1 arp 协议介绍

在以太网协议中规定，同一局域网中的一台主机要和另一台主机进行直接通信，必须要知道目标主机的 MAC 地址。而在 TCP/IP 协议中，网络层和传输层只关心目标主机的 IP 地址。这就导致在以太网中使用 IP 协议时，数据链路层的以太网协议接到上层 IP 协议提供的数据中，只包含目的主机的 IP 地址。于是需要一种方法，根据目的主机的 IP 地址，获得其 MAC 地址。这就是 ARP 协议要做的事情。所谓地址解析（address resolution）就是主机在发送帧前将目标 IP 地址转换成目标 MAC 地址的过程。

当发送主机和目的主机不在同一个局域网中时，即便知道目的主机的 MAC 地址，两者也不能直接通信，必须经过路由转发才可以。所以此时，发送主机通过 ARP 协议获得的将不是目的主机的真实 MAC 地址，而是一台可以通往局域网外的路由器的 MAC 地址。于是此后发送主机发往目的主机的所有帧，都将发往该路由器，通过它向外发送。这种情况称为委托 ARP 或 ARP 代理（ARP Proxy）。

还有一种免费 ARP（gratuitous ARP），它是指主机发送 ARP 查询（广播）自己的 IP 地址，当 ARP 功能被开启或者是端口初始配置完成，主机向网络发送免费 ARP 来查询自己的 IP 地址确认地址唯一可用。用来确定网络中是否有其他主机使用了 IP 地址，如果有应答则产生错误消息。免费 ARP 也可以做更新 ARP 缓存用，网络中的其他主机收到该广播则在缓存中更新条目，收到该广播的主机无论是否存在与 IP 地址相关的条目都会强制更新，如果存在旧条目则会将 MAC 更新为广播包中的 MAC。

arp 的请求响应  
![arp](https://doc.shiyanlou.com/document-uid949121labid10418timestamp1555403246679.png/wm) 
主机 A 发送 arp 请求，问目标 ip`192.168.0.1`的 mac 地址是什么？注意这里的 arp 请求是链路层广播，在相同局域网內的所有其他主机都会收到该请求，但是只有主机 ip 为`192.168.0.1`的才会应答，也就是图中的主机 B，它返回 arp 应答，告诉主机 A，ip`192.168.0.1`的主机 mac 地址是`02:f2:02:f2:02:f2`这样当主机 A 往主机 B 发送数据包时，就知道以太网头部目的 mac 地址是`02:f2:02:f2:02:f2`了。

#### 5.1.1 报文格式

![arp_packet](https://doc.shiyanlou.com/document-uid949121labid10418timestamp1555403285368.png/wm)

* 硬件类型(hard type)
硬件类型用来指代需要什么样的物理地址，如果硬件类型为 1，表示以太网地址

* 协议类型
协议类型则是需要映射的协议地址类型，如果协议类型是 0x0800，表示 ipv4 协议。

* 硬件地址长度
表示硬件地址的长度，单位字节，一般都是以太网地址的长度为 6 字节。

* 协议地址长度：  
表示协议地址的长度，单位字节，一般都是 ipv4 地址的长度为 4 字节。

* 操作码 
这些值用于区分具体操作类型，因为字段都相同，所以必须指明操作码，不然连请求还是应答都分不清。  
1=>ARP 请求, 2=>ARP 应答，3=>RARP 请求，4=>RARP 应答。

* 源硬件地址
源物理地址，如`02:f2:02:f2:02:f2`

* 源协议地址
源协议地址，如`192.168.0.1`

* 目标硬件地址
 目标物理地址，如`03:f2:03:f2:03:f2`

* 目标协议地址。
目标协议地址，如`192.168.0.2`  

注意到这里有两个字段是与链路层头部信息重复的。我们在发送 ARP 请求时，只有目标硬件地址是空着的，因为我们请求的就是它的值，当对应机器收到后，就会把自己的硬件地址写到这个字段，并把操作码改成 2，再发回去。于是就知道彼此的硬件地址，开始真正的通讯。

#### 5.1.2 ARP 高速缓存

知道了 ARP 发送的原理后，我们不禁疑惑，如果每次发之前都要发送 ARP 请求硬件地址会不会太慢，但是实际上 ARP 的运行是非常高效的。那是因为每一个主机上都有一个 ARP 高速缓存，我们可以在命令行键入`arp -a`获取本机 ARP 高速缓存的所有内容。

## 六、用 go 实现 arp 数据的处理

#### 6.1 arp

报文的解析和封装 `src/netstack/tcpip/header/arp.go`
```go

package header

import "netstack/tcpip"

const (
	// ARPProtocolNumber是ARP协议号，为0x0806
	ARPProtocolNumber tcpip.NetworkProtocolNumber = 0x0806

	// ARPSize是ARP报文在IPV4网络下的长度
	ARPSize = 2 + 2 + 1 + 1 + 2 + 2*6 + 2*4
)

// ARPOp表示ARP的操作码
type ARPOp uint16

// RFC 826定义的操作码.
const (
	// arp请求
	ARPRequest ARPOp = 1
	// arp应答
	ARPReply ARPOp = 2
)

// ARP报文的封装
type ARP []byte

// 从报文中得到硬件类型
func (a ARP) hardwareAddressSpace() uint16 { return uint16(a[0])<<8 | uint16(a[1]) }

// 从报文中得到协议类型
func (a ARP) protocolAddressSpace() uint16 { return uint16(a[2])<<8 | uint16(a[3]) }

// 从报文中得到硬件地址的长度
func (a ARP) hardwareAddressSize() int { return int(a[4]) }

// 从报文中得到协议的地址长度
func (a ARP) protocolAddressSize() int { return int(a[5]) }

// Op从报文中得到arp操作码.
func (a ARP) Op() ARPOp { return ARPOp(a[6])<<8 | ARPOp(a[7]) }

// SetOp设置arp操作码.
func (a ARP) SetOp(op ARPOp) {
	a[6] = uint8(op >> 8)
	a[7] = uint8(op)
}

// SetIPv4OverEthernet设置IPV4网络在以太网中arp报文的硬件和协议信息.
func (a ARP) SetIPv4OverEthernet() {
	a[0], a[1] = 0, 1       // htypeEthernet
	a[2], a[3] = 0x08, 0x00 // IPv4ProtocolNumber
	a[4] = 6                // macSize
	a[5] = uint8(IPv4AddressSize)
}

// HardwareAddressSender从报文中得到arp发送方的硬件地址
func (a ARP) HardwareAddressSender() []byte {
	const s = 8
	return a[s : s+6]
}

// ProtocolAddressSender从报文中得到arp发送方的协议地址，为ipv4地址
func (a ARP) ProtocolAddressSender() []byte {
	const s = 8 + 6
	return a[s : s+4]
}

// HardwareAddressTarget从报文中得到arp目的方的硬件地址
func (a ARP) HardwareAddressTarget() []byte {
	const s = 8 + 6 + 4
	return a[s : s+6]
}

// ProtocolAddressTarget从报文中得到arp目的方的协议地址，为ipv4地址
func (a ARP) ProtocolAddressTarget() []byte {
	const s = 8 + 6 + 4 + 6
	return a[s : s+4]
}

// IsValid检查arp报文是否有效
func (a ARP) IsValid() bool {
	// 比arp报文的长度小，返回无效
	if len(a) < ARPSize {
		return false
	}
	const htypeEthernet = 1
	const macSize = 6
	// 是否以太网、ipv4、硬件和协议长度都对
	return a.hardwareAddressSpace() == htypeEthernet &&
		a.protocolAddressSpace() == uint16(IPv4ProtocolNumber) &&
		a.hardwareAddressSize() == macSize &&
		a.protocolAddressSize() == IPv4AddressSize
}
```

```checker
- name: check arp exist
  script: |
    #!/bin/bash
    ls -l /home/shiyanlou/golang/src/netstack/tcpip/header/arp.go
  error: 没有找到 /home/shiyanlou/golang/src/netstack/tcpip/header/arp.go 文件
```

之前我们看到网卡收到 arp 请求，但网卡没有任何应答，现在我们来实现网卡的 arp 应答。由于我们的网卡没有设置 ip，按道理 arp 请求问目标 ip 的 mac 是什么，我们无法进行响应，所以我们需要假设一个网卡 ip 地址和网卡 mac 地址。整个架构很像我们在 tap 网卡上又抽象了一个网卡，而 tap 更像网线的一端一样。

 +----------------------+
 |       netstack       |
 +----------------------+
 |        vnic          |
 +----------------------+
 |         tap          |
 +----------------------+

### 6.2 arp 请求响应实现

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

// 表示arp端
type endpoint struct {
	nicid         tcpip.NICID
	addr          tcpip.Address
	linkEP        stack.LinkEndpoint
	linkAddrCache stack.LinkAddressCache
}

// arp数据包的处理，包括arp请求和响应
func (e *endpoint) HandlePacket(r *stack.Route, vv buffer.VectorisedView) {
	v := vv.First()
	h := header.ARP(v)
	if !h.IsValid() {
		return
	}

	// 判断操作码类型
	switch h.Op() {
	case header.ARPRequest:
		// 如果是arp请求
		log.Printf("recv arp request")
		localAddr := tcpip.Address(h.ProtocolAddressTarget())
		if e.linkAddrCache.CheckLocalAddress(e.nicid, header.IPv4ProtocolNumber, localAddr) == 0 {
			return // we have no useful answer, ignore the request
		}
		hdr := buffer.NewPrependable(int(e.linkEP.MaxHeaderLength()) + header.ARPSize)
		pkt := header.ARP(hdr.Prepend(header.ARPSize))
		pkt.SetIPv4OverEthernet()
		pkt.SetOp(header.ARPReply)
		copy(pkt.HardwareAddressSender(), r.LocalLinkAddress[:])
		copy(pkt.ProtocolAddressSender(), h.ProtocolAddressTarget())
		copy(pkt.ProtocolAddressTarget(), h.ProtocolAddressSender())
		log.Printf("send arp reply")
		e.linkEP.WritePacket(r, hdr, buffer.VectorisedView{}, ProtocolNumber)
		// 注意这里的 fallthrough 表示需要继续执行下面分支的代码
		// 当收到 arp 请求需要添加到链路地址缓存中
		fallthrough // also fill the cache from requests
	case header.ARPReply:
		// 这里记录ip和mac对应关系，也就是arp表
		addr := tcpip.Address(h.ProtocolAddressSender())
		linkAddr := tcpip.LinkAddress(h.HardwareAddressSender())
		e.linkAddrCache.AddLinkAddress(e.nicid, addr, linkAddr)
	}
}
```

可以看到 arp 报文的处理是很简单的，就是判断报文的操作码，然后进行相应的处理。首先是网卡收到 arp 数据包以后，通过 DeliverNetworkPacket 将数据包分发给 HandlePacket 函数，接着判断报文操作码，如果是 arp 请求，那么回复 arp 响应，并在 arp 缓存表中记录`源ip`和`源mac`对应关系。下面做个简单的 arp 实验。

### 6.3 arp 请求响应实验

在`src/lab/link/arp`目录下创建`main.go`

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
	if len(flag.Args()) < 2 {
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

	// 获取mac地址
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

	select {}
}

```
```checker
- name: check arp1 exist
  script: |
    #!/bin/bash
    ls -l /home/shiyanlou/golang/src/lab/link/arp/arp.go
  error: 没有找到 /home/shiyanlou/golang/src/lab/link/arp/arp.go 文件
```

在项目根目录执行

```shell
cd src/lab/link/arp/
go build
sudo ./arp tap0 192.168.1.1/24
```

打开一个终端，对 tap0 网卡抓包

```shell
tcpdump -i tap0 -n
```

然后打开另一个终端，我们 ping 一下 192.168.1.1 这个地址，那么系统会先发送 arp 请求来获取 192.168.1.1 的 mac 地址。

```shell
ping -c 1 192.168.1.1
```

抓包结果

```txt
ARP, Request who-has 192.168.1.1 tell 192.168.56.101, length 28
ARP, Reply 192.168.1.1 is-at 76:80:57:45:c3:8b, length 28
IP 192.168.56.101 > 192.168.1.1: ICMP echo request, id 19833, seq 1, length 64
IP 192.168.1.1 > 192.168.56.101: ICMP echo reply, id 19833, seq 1, length 64
```

程序打印结果

```txt
main.go:30: tap: tap0, cidrName: 192.168.1.1/24
arp.go:100: recv arp request
arp.go:112: send arp reply
linkaddrcache.go:152: add link cache: 1:192.168.56.101:0-76:80:57:45:c3:8b
icmp.go:75: icmp echo
icmp.go:131: icmp reply
ipv4.go:151: send ipv4 packet 84 bytes, proto: 0x1
```

通过抓包可以看到 arp 程序接收到了 arp 请求和回复了 arp 响应，告诉系统`192.168.1.1 is-at 76:80:57:45:c3:8b`，可以通过下面的命令来查看系统 arp 表中 ip 地址为 192.168.1.1 的 mac 地址。

```shell
arp 192.168.1.1
```

会得到结果

```shell
Address                  HWtype  HWaddress           Flags Mask            Iface
192.168.1.1              ether   76:80:57:45:c3:8b   C                     tap0
```

到此为止系统就学习到了 tap0 网卡的 ip 地址和 mac 对映关系，以后发送给 ip 为 `192.168.1.1` 的数据都会填充 `76:80:57:45:c3:8b` 目的 mac 地址，从而实现二层的通信。

## 七、总结

链路层主要负责管理网卡和处理网卡数据，包括新建网卡对象绑定真实网卡，更改网卡参数，接收网卡数据，去除以太网头部后分发给上层，接收上层数据，封装以太网头部写入网卡。需要注意的是主机与主机之前的二层通信，也需要主机有 ip 地址，因为主机需要通过 arp 表来进行二层寻址，而 arp 表记录的是 ip 与 mac 地址的映射关系，所以主机的 ip 地址是必须的。经过上面的实验我们已经知道，只要配好路由，我们在系统发送的数据就都可以进入到 tap 网卡，然后程序就可以读取到网卡数据，进行处理，实现对 arp 报文的处理，那如果我们继续处理 ip 报文、tcp 报文就可以实现整个协议栈了。接下来我们处理网络层 ipv4 的报文。

 
