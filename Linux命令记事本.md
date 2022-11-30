# tcpdump

```
tcpdump tcp -i eth1 -t -s 0 -c 100 and dst port ! 22 and src net 192.168.1.0/24 -w ./target.cap
```

(1)tcp: ip icmp arp rarp 和 tcp、udp、icmp这些选项等都要放到第一个参数的位置，用来过滤数据报的类型
(2)-i eth1 : 只抓经过接口eth1的包, -i any表示抓所有接口的
(3)-t : 不显示时间戳
(4)-s 0 : 抓取数据包时默认抓取长度为68字节。加上-S 0 后可以抓到完整的数据包
(5)-c 100 : 只抓取100个数据包
(6)dst port ! 22 : 不抓取目标端口是22的数据包
(7)src net 192.168.1.0/24 : 数据包的源网络地址为192.168.1.0/24
(8)-w ./target.cap : 保存成cap文件，方便用ethereal(即wireshark)分析

抓取某个host发过来和发给它的数据包:

```
tcpdump -i eth0 'src host 192.168.202.228' or 'dst host 192.168.202.228'
```

# 查看某个服务的日志

```
journalctl -xe -u docker-latest > /tmp/log.out #如docker-latest服务
```

# 查看系统负载

```
vmstat
procs -----------memory---------- ---swap-- -----io---- --system-- -----cpu------
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0      0 496056 889316 4065748    0    0     9    41   55   51  0  0 99  1  0
```

- procs 
  procs 
  r  b
  0  0
  r ：运行和等待cpu时间片的进程数，如果长期大于1，说明cpu不足，需要增加cpu。 
  b ：在等待资源的进程数，比如正在等待I/O、或者内存交换等。 

- memory 
  -----------memory----------
  swpd   free   buff  cache  
  0    496056 889316 4065748
  swpd ：切换到内存交换区的内存数量(k表示)。 
  
       如果swpd的值不为0，或者比较大，比如超过了100m，只要si、so的值长期为0，系统性能还是正常 
  
  free ：当前的空闲页面列表中内存数量(k表示) 
  buff ：作为buffer cache的内存数量，一般对块设备的读写才需要缓冲。 
  cache ：作为page cache的内存数量，一般作为文件系统的cache， 
  
        如果cache较大，说明用到cache的文件较多，如果此时IO中bi比较小，说明文件系统效率比较好。 

- swap 
  ---swap--
  si   so
  0    0 
  si ：由内存进入内存交换区数量。 
  so ：由内存交换区进入内存数量。 

- IO 
  -----io----
  bi    bo 
  9    41
  bi ：从块设备读入数据的总量（读磁盘）（每秒kb）。 
  bo ：块设备写入数据的总量（写磁盘）（每秒kb） 
  这里我们设置的bi+bo参考值为1000，如果超过1000，而且wa值较大应该考虑均衡磁盘负载，可以结合iostat输出来分析。 

- system 显示采集间隔内发生的中断数 
  --system--
  in   cs
  55   51
  in  ：在某一时间间隔中观测到的每秒设备中断数。 
  cs ：每秒产生的上下文切换次数，如当 cs 比磁盘 I/O 和网络信息包速率高得多，都应进行进一步调查。 

- cpu 表示cpu的使用状态 
  -----cpu------
  cs us sy id wa st
  51 0  0  99 1  0
  us ：用户方式下所花费 CPU 时间的百分比。us的值比较高时，说明用户进程消耗的cpu时间多，但是如果长期大于50%，需要考虑优化用户的程序。 
  sy ：内核进程所花费的cpu时间的百分比。这里us + sy的参考值为80%，如果us+sy 大于 80%说明可能存在CPU不足。 
  wa  ：IO等待所占用的CPU时间的百分比。这里wa的参考值为30%，如果wa超过30%，说明IO等待严重， 
  
      这可能是磁盘大量随机访问造成的，也可能磁盘或者磁盘访问控制器的带宽瓶颈造成的(主要是块操作)。 
  
  id ：cpu处在空闲状态的时间百分比 

# 对某个网卡的操作

## 查看某个网卡信息

```
ifconfig em1
```

## 关闭某个网卡

```
ifdown em1
```

# umount不掉

有时候提示目标忙而卸载不了

```
[root@host-206 ~]umount /var/lib/mysql  
umount: /var/lib/mysql: target is busy.
        (In some cases useful info about processes that use
         the device is found by lsof(8) or fuser(1))
```

可以使用如下命令清理:

```
[root@host-206 ~]fuser -k /var/lib/mysql
/var/lib/mysql:       7998c
Killed
```

# 重定向

## 输出重定向

0:表示键盘输入(stdin)
1:表示标准输出(stdout),系统默认是1 
2:表示错误输出(stderr)

在一个目录下创建一个a.txt文件,执行

```
[root@host-96 home]# ls a.txt b.txt
ls: cannot access b.txt: No such file or directory
a.txt
```

**一共有两种输出**，其中`ls: cannot access b.txt: No such file or directory`是错误输出，`a.txt`是标准输出。

```
# ls a.txt b.txt 1>out
ls: 无法访问b.txt: 没有那个文件或目录
# cat out
a.txt
# ls a.txt b.txt >>out
ls: 无法访问b.txt: 没有那个文件或目录
# cat out
a.txt
a.txt
```

在上述命令中，我们将原来的标准输出重定向到了out文件中，所以控制台只剩下了错误提示。并且当执行了追加操作时，out文件的内容非但没有被清空，反而又多了一条a.txt。

同理，我们也可以将错误输出重定向到文件中

```
# ls a.txt b.txt 2>err
a.txt
# cat err
ls: 无法访问b.txt: 没有那个文件或目录
# ls a.txt b.txt >out 2>err
# cat out
a.txt
# cat err
ls: 无法访问b.txt: 没有那个文件或目录
```

## 输入重定向

| 命令                    | 介绍                        |
| --------------------- | ------------------------- |
| command `<filename`   | 以filename文件作为标准输入         |
| command `0<filename`  | 同上                        |
| command `<<delimiter` | 从标准输入中读入，直到遇到delimiter分隔符 |

我们使用<对输入做重定向，如果符号左边没有写值，那么默认就是0。

```
# cat input
aaa
111
# cat >out <input
# cat out
aaa
111
```

out文件里面的内容被替换成了input文件里的内容

```
# cat >out <<end
> 123
> test
> end
# cat out
123
test
```

当我们输入完`cat >out <<end`，然后敲下回车之后，命令并没有结束，此时cat命令像一开始一样，等待你给它输入数据。然后当我们敲入end之后，cat命令就结束了。end之前输入的字符都已经被写入到了out文件中。这就是输入分割符的作用

## 高级用法

### 2>&1

这条命令用到了重定向绑定，采用&可以将两个输出绑定在一起。这条命令的作用是错误输出将和标准输出同用一个文件描述符，

说人话就是错误输出将会和标准输出输出到同一个地方。

linux在执行shell命令之前，就会确定好所有的输入输出位置，并且从左到右依次执行重定向的命令，所以`>/dev/null 2>&1`的作用就是让标准输出重定向到/dev/null中（丢弃标准输出），然后错误输出由于重用了标准输出的描述符，所以错误输出也被定向到了/dev/null中，错误输出同样也被丢弃了。执行了这条命令之后，该条shell命令将不会输出任何信息到控制台，也不会有任何信息输出到文件中。

### 2>&1 >/dev/null

2>&1，将错误输出绑定到标准输出上。由于此时的标准输出是默认值，也就是输出到屏幕，所以错误输出会输出到屏幕。

\>/dev/null，将标准输出1重定向到/dev/null中。

| 命令               | 标准输出 | 错误输出 |
| ---------------- | ---- | ---- |
| \>/dev/null 2>&1 | 丢弃   | 丢弃   |
| 2>&1 >/dev/null  | 丢弃   | 屏幕   |

# history命令显示时间

1、 临时显示

输入 `export HISTTIMEFORMAT='%F %T'`

2、 永久生效

编辑 /root/.bashrc
增加一行 `export HISTTIMEFORMAT='%F %T'`,并执行 `source ~/.bashrc`使之生效

# 生成指定大小的随机字符串文件

```
dd if=/dev/urandom of=randomfile bs=1M count=1
```

- `of=randomfile` 为生成的文件名
- `bs`：缓存大小，用来做创建文件的大小单位，可以是1k，1M，1G，不要超过系统的缓存大小。
- `count`：要创建的文件大小几个单位bs

从中我们可以看出要创建的文件个数为一个，文件大小为：bs*count

# ps

- user（通常缩写为 us），代表用户态 CPU 时间。注意，它不包括下面的 nice 时间，但包括了 guest 时间。

- nice（通常缩写为 ni），代表低优先级用户态 CPU 时间，也就是进程的 nice 值被调整为 1-19 之间时的 CPU 时间。这里注意，nice 可取值范围是 -20 到 19，数值越大，优先级反而越低。

- system（通常缩写为 sys），代表内核态 CPU 时间。

- idle（通常缩写为 id），代表空闲时间。注意，它不包括等待 I/O 的时间（iowait）。

- iowait（通常缩写为 wa），代表等待 I/O 的 CPU 时间。

- irq（通常缩写为 hi），代表处理硬中断的 CPU 时间。

- softirq（通常缩写为 si），代表处理软中断的 CPU 时间。

- steal（通常缩写为 st），代表当系统运行在虚拟机中的时候，被其他虚拟机占用的 CPU 时间。

- guest（通常缩写为 guest），代表通过虚拟化运行其他操作系统的时间，也就是运行虚拟机的 CPU 时间。

- guest_nice（通常缩写为 gnice），代表以低优先级运行虚拟机的时间。

# 查看进程的线程数

```bash
ls -l /proc/1/task/ | wc -l


top命令启动后，按"shift" + "H"
```

```bash
ps aux -T | wc -l
```

# NUMA

> 注意NUMA是一个累计值

`numactl -H` : 显示numa信息

`numastat` ： 显示node使用情况

`numastat`命令接受进程的PID或名称作为参数

```
 numastat -c rsyslogd

Per-node process memory usage (in MBs) for PID 1110 (rsyslogd)
         Node 0 Total
         ------ -----
Huge          0     0
Heap          0     0
Stack         0     0
Private      11    11
-------  ------ -----
Total        11    11
```

`/proc/PID/numa_maps`文件还提供了更详细的信息

可以30s的numa值变化来看miss有没有增长

```
cat /proc/vmstat |grep numa;sleep 30;cat /proc/vmstat |grep numa;
```

# 性能分析

## sar

**sar** 是Linux下系统运行状态统计工具，它将指定的操作系统状态计数器显示到标准输出设备。

```bash
-A 汇总所有的报告
-a 报告文件读写使用情况
-B 报告附加的缓存的使用情况
-b 报告缓存的使用情况
-c 报告系统调用的使用情况
-d 报告磁盘的使用情况
-g 报告串口的使用情况
-h 报告关于buffer使用的统计数据
-m 报告IPC消息队列和信号量的使用情况
-n 报告命名cache的使用情况
-p 报告调页活动的使用情况
-q 报告运行队列和交换队列的平均长度
-R 报告进程的活动情况
-r 报告没有使用的内存页面和硬盘块
-u 报告CPU的利用率
-v 报告进程、i节点、文件和锁表状态
-w 报告系统交换活动状况
-y 报告TTY设备活动状况
```

## top

### 内存模式

打开top之后，先按g，然后输入3，从而进入内存模式，里面有不少字段

- VIRT： 进程申请的虚拟内存大小，比如申请100m，实际只用10m，这个就是100m

- RES：实际进程的物理内存，就是上面例子里面的10m

- SHR：VIRT里有多少其实是共享部分(库文件使用的内存)

- CODE：代码段

- DATA： 数据段

可以根据这些数据来判断内存泄漏情况，比如如果 RES 太高而 SHR 不高，那可能是堆内存泄漏；如果 SHR 很高，那可能是 tmpfs/shm 之类的数据在持续增长，如果 VIRT 很高而 RES 很小，那可能是进程不停地在申请内存，但是却没有对这些内存进行任何的读写操作，即虚拟地址空间存在内存泄漏



# 正则表达式

## 选择含有特定字符串的行

```
^.*(string1|string2|string3).*\n
```

## 选定空白行

```bash
^\s*(?=\r?$)\n
```
