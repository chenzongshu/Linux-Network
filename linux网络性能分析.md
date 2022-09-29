# 网络性能

## 性能指标

通常用带宽、吞吐量、延时、PPS（Packet Per Second）等指标衡量网络的性能。

- **带宽**，表示链路的最大传输速率，单位通常为 b/s （比特 / 秒）。
- **吞吐量**，表示单位时间内成功传输的数据量，单位通常为 b/s（比特 / 秒）或者 B/s（字节 / 秒）。吞吐量受带宽限制，而吞吐量 / 带宽，也就是该网络的使用率。
- **延时**，表示从网络请求发出后，一直到收到远端响应，所需要的时间延迟。在不同场景中，这一指标可能会有不同含义。比如，它可以表示，建立连接需要的时间（比如 TCP 握手延时），或一个数据包往返所需的时间（比如 RTT）。
- **PPS**，是 Packet Per Second（包 / 秒）的缩写，表示以网络包为单位的传输速率。PPS 通常用来评估网络的转发能力，比如硬件交换机，通常可以达到线性转发（即 PPS 可以达到或者接近理论最大值）。而基于 Linux 服务器的转发，则容易受网络包大小的影响。

除了这些指标，**网络的可用性**（网络能否正常通信）、**并发连接数**（TCP 连接数量）、**丢包率**（丢包百分比）、**重传率**（重新传输的网络包比例）等也是常用的性能指标。

## 套接字信息

可以用 netstat 或者 ss ，来查看套接字、网络栈、网络接口以及路由表的信息。

> ss比 netstat 提供了更好的性能（速度更快）。

比如，你可以执行下面的命令，查询套接字信息：

```bash
# head -n 3 表示只显示前面 3 行
# -l 表示只显示监听套接字
# -n 表示显示数字地址和端口 (而不是名字)
# -p 表示显示进程信息
$ netstat -nlp | head -n 3
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      840/systemd-resolve 
# -l 表示只显示监听套接字
# -t 表示只显示 TCP 套接字
# -n 表示显示数字地址和端口 (而不是名字)
# -p 表示显示进程信息
$ ss -ltnp | head -n 3
State    Recv-Q    Send-Q        Local Address:Port        Peer Address:Port
LISTEN   0         128           127.0.0.53%lo:53               0.0.0.0:*        users:(("systemd-resolve",pid=840,fd=13))
LISTEN   0         128                 0.0.0.0:22               0.0.0.0:*        users:(("sshd",pid=1459,fd=3))
```

netstat 和 ss 的输出也是类似的，都展示了套接字的状态、接收队列、发送队列、本地地址、远端地址、进程 PID 和进程名称等。

其中，接收队列（Recv-Q）和发送队列（Send-Q）需要你特别关注，它们通常应该是 0。当你发现它们不是 0 时，说明有网络包的堆积发生。当然还要注意，在不同套接字状态下，它们的含义不同。

当套接字处于连接状态（Established）时，

- Recv-Q 表示套接字缓冲还没有被应用程序取走的字节数（即接收队列长度）。
- 而 Send-Q 表示还没有被远端主机确认的字节数（即发送队列长度）。

当套接字处于监听状态（Listening）时，

- Recv-Q 表示 syn backlog 的当前值。
- 而 Send-Q 表示最大的 syn backlog 值。

## 协议栈统计信息

使用 netstat 或 ss ，也可以查看协议栈的信息：

```bash
$ netstat -s
...
Tcp:
    3244906 active connection openings
    23143 passive connection openings
    115732 failed connection attempts
    2964 connection resets received
    1 connections established
    13025010 segments received
    17606946 segments sent out
    44438 segments retransmitted
    42 bad segments received
    5315 resets sent
    InCsumErrors: 42
... 
$ ss -s
Total: 186 (kernel 1446)
TCP:   4 (estab 1, closed 0, orphaned 0, synrecv 0, timewait 0/0), ports 0 
Transport Total     IP        IPv6
*      1446      -         -
RAW      2         1         1
UDP      2         2         0
TCP      4         3         1
...
```

这些协议栈的统计信息都很直观。ss 只显示已经连接、关闭、孤儿套接字等简要统计，而 netstat 则提供的是更详细的网络协议栈信息。

## 网络吞吐和PPS

sar在前面的 CPU、内存和 I/O 模块中也经常会用到

给 sar 增加 -n 参数就可以查看网络的统计信息，比如网络接口（DEV）、网络接口错误（EDEV）、TCP、UDP、ICMP 等等。执行下面的命令，你就可以得到网络接口统计信息：

```bash
# 数字 1 表示每隔 1 秒输出一组数据
$ sar -n DEV 1
Linux 4.15.0-1035-azure (ubuntu)     01/06/19     _x86_64_    (2 CPU) 
13:21:40        IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
13:21:41         eth0     18.00     20.00      5.79      4.25      0.00      0.00      0.00      0.00
13:21:41      docker0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
13:21:41           lo      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
```

- rxpck/s 和 txpck/s 分别是接收和发送的 PPS，单位为包 / 秒。
- rxkB/s 和 txkB/s 分别是接收和发送的吞吐量，单位是 KB/ 秒。
- rxcmp/s 和 txcmp/s 分别是接收和发送的压缩数据包数，单位是包 / 秒。
- %ifutil 是网络接口的使用率，即半双工模式下为 (rxkB/s+txkB/s)/Bandwidth，而全双工模式下为 max(rxkB/s, txkB/s)/Bandwidth。

其中，Bandwidth 可以用 ethtool 来查询，它的单位通常是 Gb/s 或者 Mb/s，不过注意这里小写字母 b ，表示比特而不是字节。我们通常提到的千兆网卡、万兆网卡等，单位也都是比特。如下你可以看到，我的 eth0 网卡就是一个千兆网卡：

```bash
$ ethtool eth0 | grep Speed
    Speed: 1000Mb/s
```

## 连通性和延时

通常用ping来测试

## 测试工具

### TCP/UDP性能

 iperf 或者 netperf

在目标机器上启动 iperf 服务端：

```ruby
# -s 表示启动服务端，-i 表示汇报间隔，-p 表示监听端口
$ iperf3 -s -i 1 -p 10000
```

接着，在另一台机器上运行 iperf 客户端，运行测试：

```shell
# -c 表示启动客户端，192.168.0.30 为目标服务器的 IP
# -b 表示目标带宽 (单位是 bits/s)
# -t 表示测试时间
# -P 表示并发数，-p 表示目标服务器监听端口
$ iperf3 -c 192.168.0.30 -b 1G -t 15 -P 2 -p 10000
```

稍等一会儿（15 秒）测试结束后，回到目标服务器，查看 iperf 的报告：

```css
[ ID] Interval           Transfer     Bandwidth
...
[SUM]   0.00-15.04  sec  0.00 Bytes  0.00 bits/sec                  sender
[SUM]   0.00-15.04  sec  1.51 GBytes   860 Mbits/sec                  receiver
```

最后的 SUM 行就是测试的汇总结果，包括测试时间、数据传输量以及带宽等。按照发送和接收，这一部分又分为了 sender 和 receiver 两行。

从测试结果你可以看到，这台机器 TCP 接收的带宽（吞吐量）为 860 Mb/s

### HTTP性能

测试 HTTP 的性能，有大量的工具可以使用，比如 ab、webbench 等，都是常用的 HTTP 压力测试工具。其中，ab 是 Apache 自带的 HTTP 压测工具，主要测试 HTTP 服务的每秒请求数、请求延迟、吞吐量以及请求延迟的分布情况等。

运行 ab 命令

```bash
# -c 表示并发请求数为 1000，-n 表示总的请求数为 10000
$ ab -c 1000 -n 10000 http://192.168.0.30/
...
Server Software:        nginx/1.15.8
Server Hostname:        192.168.0.30
Server Port:            80 
... 
Requests per second:    1078.54 [#/sec] (mean)
Time per request:       927.183 [ms] (mean)
Time per request:       0.927 [ms] (mean, across all concurrent requests)
Transfer rate:          890.00 [Kbytes/sec] received 
Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0   27 152.1      1    1038
Processing:     9  207 843.0     22    9242
Waiting:        8  207 843.0     22    9242
Total:         15  233 857.7     23    9268 
Percentage of the requests served within a certain time (ms)
  50%     23
  66%     24
  75%     24
  80%     26
  90%    274
  95%   1195
  98%   2335
  99%   4663
 100%   9268 (longest request)
```

可以看到，ab 的测试结果分为三个部分，分别是请求汇总、连接时间汇总还有请求延迟汇总。以上面的结果为例，我们具体来看。

在请求汇总部分，你可以看到：

- Requests per second 为 1074；
- 每个请求的延迟（Time per request）分为两行，第一行的 927 ms 表示平均延迟，包括了线程运行的调度时间和网络请求响应时间，而下一行的 0.927ms ，则表示实际请求的响应时间；
- Transfer rate 表示吞吐量（BPS）为 890 KB/s。

连接时间汇总部分，则是分别展示了建立连接、请求、等待以及汇总等的各类时间，包括最小、最大、平均以及中值处理时间。

最后的请求延迟汇总部分，则给出了不同时间段内处理请求的百分比，比如， 90% 的请求，都可以在 274ms 内完成。

### 应用负载性能

TCP、HTTP 等的性能数据不能表示应用程序的实际性能，可以用 wrk、TCPCopy、Jmeter 或者 LoadRunner 等模拟用户的请求负载

### DNS 性能

可用 nslookup 和 dig命令来分析到DNS服务器的解析时间

nslookup加一个 time 命令，输出解析所用时间。如果一切正常，你可能会看到如下输出：

```yaml
/# time nslookup baidu.com
Server:        8.8.8.8
Address:    8.8.8.8#53 
Non-authoritative answer:
Name:    baidu.com
Address: 39.106.233.176 
real    0m10.349s
user    0m0.004s
sys    0m0.0
```

可以看到

```python
# +trace 表示开启跟踪查询
# +nodnssec 表示禁止 DNS 安全扩展
$ dig +trace +nodnssec baidu.com 
; <<>> DiG 9.11.3-1ubuntu1.3-Ubuntu <<>> +trace +nodnssec baidu.com
;; global options: +cmd
.            322086    IN    NS    m.root-servers.net.
.            322086    IN    NS    a.root-servers.net.
.            322086    IN    NS    i.root-servers.net.
.            322086    IN    NS    d.root-servers.net.
.            322086    IN    NS    g.root-servers.net.
.            322086    IN    NS    l.root-servers.net.
.            322086    IN    NS    c.root-servers.net.
.            322086    IN    NS    b.root-servers.net.
.            322086    IN    NS    h.root-servers.net.
.            322086    IN    NS    e.root-servers.net.
.            322086    IN    NS    k.root-servers.net.
.            322086    IN    NS    j.root-servers.net.
.            322086    IN    NS    f.root-servers.net.
;; Received 239 bytes from 114.114.114.114#53(114.114.114.114) in 1340 ms 
org.            172800    IN    NS    a0.org.afilias-nst.info.
org.            172800    IN    NS    a2.org.afilias-nst.info.
org.            172800    IN    NS    b0.org.afilias-nst.org.
org.            172800    IN    NS    b2.org.afilias-nst.org.
org.            172800    IN    NS    c0.org.afilias-nst.info.
org.            172800    IN    NS    d0.org.afilias-nst.org.
;; Received 448 bytes from 198.97.190.53#53(h.root-servers.net) in 708 ms 
geekbang.org.        86400    IN    NS    dns9.hichina.com.
geekbang.org.        86400    IN    NS    dns10.hichina.com.
;; Received 96 bytes from 199.19.54.1#53(b0.org.afilias-nst.org) in 1833 ms 
time.geekbang.org.    600    IN    A    39.106.233.176
;; Received 62 bytes from 140.205.41.16#53(dns10.hichina.com) in 4 ms
```

dig trace 的输出，主要包括四部分。

- 第一部分，是从 114.114.114.114 查到的一些根域名服务器（.）的 NS 记录。
- 第二部分，是从 NS 记录结果中选一个（h.root-servers.net），并查询顶级域名 org. 的 NS 记录。
- 第三部分，是从 org. 的 NS 记录中选择一个（b0.org.afilias-nst.org），并查询二级域名 geekbang.org. 的 NS 服务器。
- 最后一部分，就是从 geekbang.org. 的 NS 服务器（dns10.hichina.com）查询最终主机 time.geekbang.org. 的 A 记录。

如果和DNS Server解析时间太长，可以用ping等工具测试和DNS Server的网络情况，然后再分析是否是网络问题，或者更换快一点的DNS服务器

# tcpdump

基本公式是

```bash
tcpdump [选项] [过滤表达式]
```

| 选项     | 事例                   | 说明                              |
|:------:| -------------------- | ------------------------------- |
| -i     | tcpdump -i eth0      | 指定网络接口，默认是0号接口（如eth0），any表示所有接口 |
| -nn    | tcpdump -nn          | 不解析IP地址和端口号名称                   |
| -c     | tcpdump -c5          | 限制抓取网络包个数                       |
| -A     | tcpdump -A           | 以ASCII格式显示网络包内存（不指定时只显示头部信息）    |
| -w     | tcpdump -w file.pcap | 保存到文件中，文件名通常用.pcap为后缀           |
| houb-e | tcpdump -e           | 输出链路层的头部信息                      |

表达式

| 表达式                                  | 事例                                       | 说明        |
| ------------------------------------ | ---------------------------------------- | --------- |
| host, src host, dst host             | tcpdump -nn host 192.221.12.200          | 主机过滤      |
| net, src net, dst net                | tcpdump -nn 192.221.0.0                  | 网络过滤      |
| port, port range, src port, dst port | tcpdump -nn dst port 80                  | 端口过滤      |
| ip, ip6, arp, tcp, udp, icmp         | tcpdump -nn tcp                          | 协议过滤      |
| and, or, not                         | tcpdump -nn udp or icmp                  | 逻辑表达式     |
| tcp[tcpflags]                        | tcpdump -nn "tcp[tcpflags] & tcp-syn!=0" | 特定状态的TCP包 |

tcpdump的输出格式

```bash
时间戳 协议 源地址. 源端口 > 目的地址. 目的端口 网络包详细信息
```
