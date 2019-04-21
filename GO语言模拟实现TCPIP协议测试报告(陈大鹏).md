### 预估学习时长
- TCPIP 和开放系统互连（OSI）模型(15min)
- 链路层的基本实现(1h)
- 网络层的实现(1h)
- 端口(30min)
- 传输层概念及UDP协议(30min)
- TCP原理及实现(至少2h)

### 面向用户
- linux 基础
- go 语言基础
- 对TCP/IP有兴趣
- 研究僧吧，本科生表示最后一篇文章的公式和算法伤不起，本科生多看几片论文其实也行的，没事少看这玩意儿，掉头发

### 整体质量点评
- 优点：能从go语言的角度实例化对TCP/IP的介绍和应用，
- 缺点：能耗时1h以上的文章可以拆成两篇或者三篇，这样看下来不会太累


### 其他小的细节
- 建议统一成TCP/IP 的写法，TCPIP或tcpip看着不舒服
- 本教程中的$GOPATH默认为/home/shiyanlou/golang，也是项目的根目录
- 必须将代码下载到/home/shiyanlou/golang/src
- 所有的go build和go run都报同一个错误
```
TODO: 本节内容执行  步骤的过程中会报错, 无法继续实验

# netstack/tcpip/link/rawfile
src/netstack/tcpip/link/rawfile/blockingpoll_unsafe_amd64.go:24:6: blockingPoll redeclared in this block
	previous declaration at src/netstack/tcpip/link/rawfile/blockingpoll_unsafe.go:24:66

```