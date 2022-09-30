# TCP连接和断开

## 连接

TCP 连接的建⽴是⼀个从 Client 侧调⽤ connect()，到 Server 侧 accept() 成功返回的过程。

Client 调⽤ connect() 后，Linux 内核就开始进⾏三次握⼿

Server ⽆法处理，此时 Client 这⼀侧就会触发超时重传机制。但是也不能⼀直重传下去，重传的次数也是有限制的，这就是 tcp_syn_retries 这个配置项来决定的。

如果 Server 没有响应 Client 的 SYN，除了Server已经不存在了这种情况外，还有可能是因为Server 太忙没有来得及响应，或者是 Server 已经积压了太多的半连接（incomplete）⽽⽆法及时去处理。

半连接，即收到了 SYN 后还没有回复 SYNACK 的连接，Server 每收到⼀个新的 SYN 包，都会创建⼀个半连接，然后把该半连接加⼊到半连接队列（syn queue）中。syn queue 的⻓度就是 tcp_max_syn_backlog 这个配置项来决定的，当系统中积压的半连接个数超过了该值后，新的SYN 包就会被丢弃。对于服务器⽽⾔，可能瞬间会有⾮常多的新建连接

全连接队列（accept queue）的⻓度是由 listen(sockfd,backlog) 这个函数⾥的 backlog 控制的，⽽该 backlog 的最⼤值则是 somaxconn。somaxconn 在 5.4 之前的内核中，默认都是 128（5.4 开始调整为了默认 4096）

当服务器中积压的全连接个数超过该值后，新的全连接就会被丢弃掉。Server 在将新连接丢弃时，有的时候需要发送reset 来通知 Client，这样 Client 就不会再次重试了。不过，默认⾏为是直接丢弃不去通知 Client。⾄于是否需要给Client 发送 reset，是由 tcp_abort_on_overflow 这个配置项来控制的，该值默认为 0，即不发送 reset 给 Client。

## 断开

当应⽤程序调⽤ close() 时，会向对端发送 FIN包，然后会接收 ACK；对端也会调⽤ clsoe() 来发送 FIN，然后本端也会向对端回 ACK，这就是 TCP 的四次挥⼿过程。

⾸先调⽤ close() 的⼀侧是 active close（主动关闭）；⽽接收到对端的 FIN 包后再调⽤close() 来关闭的⼀侧，称之为 passive close（被动关闭）。在四次挥⼿的过程中，有动关闭⽅的 CLOSE_WAIT 状态。除了 CLOSE_WAIT 状态外，其余两个状态都有对应的系统配置项来控制。

FIN_WAIT_2 状态，TCP 进⼊到这个状态后，如果本端迟迟收不到对端的 FIN 包，那就会⼀直处于这个状态，于是就会⼀直消耗系统资源。Linux 为了防⽌这种资源的开销，设置了这个状态的超时时间tcp_fin_timeout，默认为 60s，超过这个时间后就会⾃动销毁该连接。

TIME_WAIT 状态存在的意义是：最后发送的这个 ACK 包可能会被丢弃掉或者有延迟，这样对端就会再次发送 FIN 包



# TCP收发包

TCP 收包和发包的过程也是容易引起问题的地方。

收包是指数据到达网卡再到被应用程序 开始处理的过程；发包则是应用程序调用发包函数到数据包从网卡发出的过程

## 发包

首先，应用程序调用 Socket API（比如 sendmsg）发送网络包。

由于这是一个系统调用，所以会陷入到内核态的套接字层中。套接字层会把数据包放到 Socket 发送缓冲区中。

接下来，网络协议栈从 Socket 发送缓冲区中，取出数据包；再按照 TCP/IP 栈，从上到下逐层处理。比如，传输层和网络层，分别为其增加 TCP 头和 IP 头，执行路由查找确认下一跳的 IP，并按照 MTU 大小进行分片。

分片后的网络包，再送到网络接口层，进行物理地址寻址，以找到下一跳的 MAC 地址。然后添加帧头和帧尾，放到发包队列中。这一切完成后，会有软中断通知驱动程序：发包队列中有新的网络帧需要发送。

最后，驱动程序通过 DMA ，从发包队列中读出网络帧，并通过物理网卡把它发送出去。

### TCP发送缓冲区

应用程序调用 write(2) 或者 send(2) 系列系 统调用开始往外发包时，这些系统调用会把数据包从用户缓冲区拷贝到 TCP 发送缓冲区 （TCP Send Buffer），这个 TCP 发送缓冲区的大小是受限制的，这里也是容易引起问题 的地方

TCP 发送缓冲区的大小默认是受 net.ipv4.tcp_wmem 来控制：

```
net.ipv4.tcp_wmem = 8192 65536 16777216
```

tcp_wmem 中这三个数字的含义分别为 min、default、max。TCP 发送缓冲区的大小会 在 min 和 max 之间动态调整，初始的大小是 default，这个动态调整的过程是由内核自动 来做的，应用程序无法干预。自动调整的目的，是为了在尽可能少的浪费内存的情况下来 满足发包的需要

tcp_wmem 中的 max 不能超过 net.core.wmem_max 这个配置项的值

```
 cat /proc/sys/net/core/wmem_max 
16777216
```

如果超过了， TCP 发送缓冲区最大就是 net.core.wmem_max。通常情况下，我们需要设置 net.core.wmem_max 的值大于等于 net.ipv4.tcp_wmem 的 max

应用程序有的时候会很明确地知道自己发送多大的数据，需要多大的 TCP 发送缓冲区，这 个时候就可以通过 setsockopt(2) 里的 SO_SNDBUF 来设置固定的缓冲区大小。一旦进行了这种设置后，tcp_wmem 就会失效，而且这个缓冲区大小设置的是固定值，内核也不会 对它进行动态调整

tcp_wmem 以及 wmem_max 的大小设置都是针对单个 TCP 连接的，这两个值的单位都 是 Byte（字节）。系统中可能会存在非常多的 TCP 连接，如果 TCP 连接太多，就可能导 致内存耗尽。因此，所有 TCP 连接消耗的总内存也有限制：

```
net.ipv4.tcp_mem = 8388608 12582912 16777216
```

我们通常也会把这个配置项给调大。与前两个选项不同的是，该选项中这些值的单位是 Page（页数），也就是 4K。它也有 3 个值：min、pressure、max。当所有 TCP 连接消 耗的内存总和达到 max 后，也会因达到限制而无法再往外发包

### IP

TCP 层处理完数据包后，就继续往下来到了 IP 层。IP 层这里容易触发问题的地方是 net.ipv4.ip_local_port_range 这个配置选项

```bash
net.ipv4.ip_local_port_range = 1024 65535
```

它是指和其他服务器建立 IP 连接时本地端 口（local port）的范围。如果端口范围太小，可能无法创 建新连接的问题。所以通常情况下，都会扩大默认的端口范围

为了能够对 TCP/IP 数据流进行流控，Linux 内核在 IP 层实现了 qdisc（排队规则）。我们平时用到的 TC 就是基于 qdisc 的流控工具。qdisc 的队列长度是我们用 ifconfig 来看 到的 txqueuelen，我们在生产环境中也遇到过因为 txqueuelen 太小导致数据包被丢弃的情况

```
ip -s -s link ls dev eth0 …

TX: bytes packets errors dropped carrier collsns 
3263284 25060 0 0 0 0
```

如果观察到 dropped 这一项不为 0，那就有可能是 txqueuelen 太小导致的。当遇到这种 情况时，你就需要增大该值了，比如增加 eth0 这个网络接口的 txqueuelen

```
ifconfig eth0 txqueuelen 2000
```

## 收包

当一个网络帧到达网卡后，网卡会通过 DMA 方式，把这个网络包放到收包队列中；然后通过硬中断，告诉中断处理程序已经收到了网络包。

接着，网卡中断处理程序会为网络帧分配内核数据结构（sk_buff），并将其拷贝到 sk_buff 缓冲区中；然后再通过软中断，通知内核收到了新的网络帧。

接下来，内核协议栈从缓冲区中取出网络帧，并通过网络协议栈，从下到上逐层处理这个网络帧。比如，

- 在链路层检查报文的合法性，找出上层协议的类型（比如 IPv4 还是 IPv6），再去掉帧头、帧尾，然后交给网络层。
- 网络层取出 IP 头，判断网络包下一步的走向，比如是交给上层处理还是转发。当网络层确认这个包是要发送到本机后，就会取出上层协议的类型（比如 TCP 还是 UDP），去掉 IP 头，再交给传输层处理。
- 传输层取出 TCP 头或者 UDP 头后，根据 < 源 IP、源端口、目的 IP、目的端口 > 四元组作为标识，找出对应的 Socket，并把数据拷贝到 Socket 的接收缓存中。

最后，应用程序就可以使用 Socket 接口，读取到新接收到的数据了

### 网卡poll

TCP 数据包的接收流程在整体上与发送流程类似，只是方向是相反的。 数据包到达网卡后，就会触发中断（IRQ）来告诉 CPU 读取这个数据包。但是在高性能网 络场景下，数据包的数量会非常大，如果每来一个数据包都要产生一个中断，那 CPU 的处 理效率就会大打折扣，所以就产生了 NAPI（New API）这种机制让 CPU 一次性地去轮询 （poll）多个数据包，以批量处理的方式来提升效率，降低网卡中断带来的性能开销

那在 poll 的过程中，一次可以 poll 多少个呢？这个 poll 的个数可以通过 sysctl 选项来控 制：

```
net.core.netdev_budget = 600
```

该控制选项的默认值是 300，在网络吞吐量较大的场景中，我们可以适当地增大该值，比 如增大到 600。增大该值可以一次性地处理更多的数据包。但是这种调整也是有缺陷的， 因为这会导致 CPU 在这里 poll 的时间增加，如果系统中运行的任务很多的话，其他任务 的调度延迟就会增加。

数据包到达网卡后会触发 CPU 去 poll 数据包，这些 poll 的数据包紧接着就会到达 IP 层去处理，然后再达到 TCP 层

### tcp接收缓冲区

与 TCP 发送缓冲区类似，TCP 接收缓冲区的大小也是受控制的。通常情况下，默认都是使 用 tcp_rmem 来控制缓冲区的大小

```
net.ipv4.tcp_rmem = 8192 87380 16777216
```

不过跟发送缓冲区不同的是，这个动态调整是可以通过控制选项来关闭的，这个 选项是 tcp_moderate_rcvbuf 。通常我们都是打开它，这也是它的默认值：

```
net.ipv4.tcp_moderate_rcvbuf = 1
```

之所以接收缓冲区有选项可以控制自动调节，而发送缓冲区没有，那是因为 TCP 接收缓冲 区会直接影响 TCP 拥塞控制，进而影响到对端的发包，所以使用该控制选项可以更加灵活 地控制对端的发包行为

除了 tcp_moderate_rcvbuf 可以控制 TCP 接收缓冲区的动态调节外，也可以通过 setsockopt() 中的配置选项 SO_RCVBUF 来控制，这与 TCP 发送缓冲区是类似的。如果 应用程序设置了 SO_RCVBUF 这个标记，那么 TCP 接收缓冲区的动态调整就是关闭，即使 tcp_moderate_rcvbuf 为 1，接收缓冲区的大小始终就为设置的 SO_RCVBUF 这个值。

也就是说，只有在 tcp_moderate_rcvbuf 为 1，并且应用程序没有通过 SO_RCVBUF 来 配置缓冲区大小的情况下，TCP 接收缓冲区才会动态调节。

同样地，与 TCP 发送缓冲区类似，SO_RCVBUF 设置的值最大也不能超过 net.core.rmem_max。通常情况下，我们也需要设置 net.core.rmem_max 的值大于等于 net.ipv4.tcp_rmem 的 max

# TCP拥塞控制

拥塞控制就是根据 TCP 的数据传输 状况来灵活地调整拥塞窗口，从而控制发送方发送数据包的行为。换句话说，拥塞窗口的大小可以表示网络传输链路的拥塞情况。TCP 连接 cwnd 的大小可以通过 ss 这个命令来查看：

```
$ ss -nipt 
State Recv-Q Send-Q                           Local Address:Port
ESTAB 0 36                                    172.23.245.7:22
users:(("sshd",pid=19256,fd=3)) 
  cubic wscale:5,7 rto:272 rtt:71.53/1.068 ato:40 mss:1248 rcvmss:1248 advmss
```

TCP 拥塞控制大致分为四个阶段

## 拥塞控制四阶段

### 慢启动

TCP 连接建立好后，发送方就进入慢速启动阶段，然后逐渐地增大发包数量（TCP Segments）。这个阶段每经过一个 RTT（round-trip time），发包数量就会翻倍

初始发送数据包的数量是由 init_cwnd（初始拥塞窗口）来决定的，该值在 Linux 内核中 被设置为 10（TCP_INIT_CWND），这是由 Google 的研究人员总结出的一个经验值

增大 init_cwnd 的值对于提升短连接的网络性能会很有效，特别是数据量在慢启动阶段就 能发送完的短连接，比如针对 http 这种服务，http 的短连接请求数据量一般不大，通常 在慢启动阶段就能传输完，这些都可以通过 tcpdump 来进行观察。

在慢启动阶段，当拥塞窗口（cwnd）增大到一个阈值（ ssthresh，慢启动阈值）后，TCP 拥塞控制就进入了下一个阶段：拥塞避免（Congestion Avoidance）

### 拥塞避免

在这个阶段 cwnd 不再成倍增加，而是一个 RTT 增加 1，即缓慢地增加 cwnd，以防止网 络出现拥塞。网络出现拥塞是难以避免的，由于网络链路的复杂性，甚至会出现乱序（Out of Order）报文

### 快速重传和快速恢复

快速重传和快速恢复是一起工作的，它们是为了应对丢包这种行为而做的优化，在这种情 况下，由于网络并没有出现拥塞，所以拥塞窗口不必恢复到初始值。判断丢包的依据就是 收到 3 个相同的 ack。

## 接收方是如何影响发送方发送数据的

接收方在收到数据包后，会给发送方回一个 ack，然后把自己的 rwnd 大小 写入到 TCP 头部的 win 这个字段，这样发送方就能根据这个字段来知道接收方的 rwnd 了。接下来，发送方在发送下一个 TCP segment 的时候，会先对比发送方的 cwnd 和接 收方的 rwnd，得出这二者之间的较小值，然后控制发送的 TCP segment 个数不能超过这 个较小值。

首先要注意ack 里是否出现 win 为 0 的情况，

其次因为 TCP 头部大小是有限制的，而其中的 win 这个字段只有 16bit，win 能够表示的大小 最大只有 65535（64K），所以如果想要支持更大的接收窗口以满足高性能网络，我们就 需要打开下面这个配置项，系统中也是默认打开了该选项

```
net.ipv4.tcp_window_scaling = 1
```

# 常见问题

## TCP重传率

TCP 重传率是通过解析 /proc/net/snmp 这个文件里的指标计算出来的，包含了下面一些关键指标

- ActiveOpens：主动打开的TCP连接数量

- PassiveOpens：被动打开的TCP连接数量

- InSegs： 收到的TCP报文数量

- OutSegs：发出的TCP报文数量

- EstabResets： TCP连接处于Establishd时发生的Reset

- AttemptFails：连接失败的数量

- CurrEstab： 当前状态为Establishd的TCP连接数

- RetransSegs： 重传的报文数量

TCP 重传率的计算公式如下

```
retrans = (RetransSegs－last RetransSegs) ／ (OutSegs－last OutSegs) * 100
```

单位时间内 TCP 重传包的数量除以 TCP 总的发包数量，就是 TCP 重传率

> 每发出去一个 TCP 包（包括重传包）， OutSegs 会相应地加 1；每发出去一个重传包，RetransSegs 会相应地加 1

发送端在发送一个 TCP 数据包后，会把该数据包放在发 送端的发送队列里，也叫重传队列。此时，OutSegs 会相应地加 1，队列长度也为 1。如 果可以收到接收端对这个数据包的 ACK，该数据包就会在发送队列中被删掉，然后队列长 度变为 0；如果收不到这个数据包的 ACK，就会触发重传机制

发送端在发送数据包的时候，会启动一个超时重传定时器（RTO），如果超过了这个时间，发送端还没有收到 ACK，就会重传该数据包，然后 OutSegs 加 1，同时 RetransSegs 也会加 1
