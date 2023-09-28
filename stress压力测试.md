stress,顾名思义是一款压力测试工具。你可以用它来对系统CPU，内存，以及磁盘IO生成负载。

## 安装stress

几乎所有主流的linux发行版的软件仓库中都收录有stress，可以直接使用包管理起来安装

```
sudo pacman -S stress --noconfirm
```

## 使用stress

直接运行 `stress` 就会列出关于 stress 的简单说明

```
stress
`stress' imposes certain types of compute stress on your system

Usage: stress [OPTION [ARG]] ...
 -?, --help         show this help statement
     --version      show version statement
 -v, --verbose      be verbose
 -q, --quiet        be quiet
 -n, --dry-run      show what would have been done
 -t, --timeout N    timeout after N seconds
     --backoff N    wait factor of N microseconds before work starts
 -c, --cpu N        spawn N workers spinning on sqrt()
 -i, --io N         spawn N workers spinning on sync()
 -m, --vm N         spawn N workers spinning on malloc()/free()
     --vm-bytes B   malloc B bytes per vm worker (default is 256MB)
     --vm-stride B  touch a byte every B bytes (default is 4096)
     --vm-hang N    sleep N secs before free (default none, 0 is inf)
     --vm-keep      redirty memory instead of freeing and reallocating
 -d, --hdd N        spawn N workers spinning on write()/unlink()
     --hdd-bytes B  write B bytes per hdd worker (default is 1GB)

Example: stress --cpu 8 --io 4 --vm 2 --vm-bytes 128M --timeout 10s

Note: Numbers may be suffixed with s,m,h,d,y (time) or B,K,M,G (size).
```

### 对CPU进行压力测试

使用 `stress -c N` 会让stress生成N个工作进程进行开方运算，以此对CPU产生负载。

比如你的CPU有四个核，那么可以运行

```
stress -c 4
```

这是查看stress进程信息

```
ps -elf |grep stress |grep -v grep
0 S lujun99+ 17738  1501  0  80   0 -  1996 -      23:06 pts/0    00:00:00 stress -c 4
1 R lujun99+ 17739 17738 95  80   0 -  1996 -      23:06 pts/0    00:01:17 stress -c 4
1 R lujun99+ 17740 17738 93  80   0 -  1996 -      23:06 pts/0    00:01:16 stress -c 4
1 R lujun99+ 17741 17738 94  80   0 -  1996 -      23:06 pts/0    00:01:17 stress -c 4
1 R lujun99+ 17742 17738 94  80   0 -  1996 -      23:06 pts/0    00:01:17 stress -c 4
```

你会发现一共有5个stress进程，其中有4个进程是 `17738` 进程派生出来的工作进程。而且每个工作进程占用的CPU利用率都接近100%

### 对内存进行压力测试

类似的，使用 `stress -m N` 会让stress生成N个工作进程来占用内存。每个进程默认占用256M内存，但可以通过 `--vm-bytes` 来进行设置。 例如

```
stress -m 3 --vm-bytes 300M
```

会生成3个进程，每个进程占用300M内存

```
ps -elf |grep stress |grep -v grep
0 S lujun99+ 18700  1501  0  80   0 -  1996 -      23:26 pts/0    00:00:00 stress -m 3 --vm-bytes 300M
1 R lujun99+ 18701 18700 99  80   0 - 78797 -      23:26 pts/0    00:02:10 stress -m 3 --vm-bytes 300M
1 R lujun99+ 18702 18700 99  80   0 - 78797 -      23:26 pts/0    00:02:10 stress -m 3 --vm-bytes 300M
1 R lujun99+ 18703 18700 99  80   0 - 78797 -      23:26 pts/0    00:02:09 stress -m 3 --vm-bytes 300M
```

而且你会发现，虽然只是对内存进行压力测试，但实际上CPU也是很繁忙的，占有率也接近100%

### 对磁盘进行压力测试

对磁盘压力测试有两个参数：

`stress -i N` 会产生N个进程，每个进程反复调用sync()将内存上的内容写到硬盘上.

而 `stress -d N` 会产生N个进程，每个进程往当前目录中写入固定大小的临时文件，然后执行unlink操作删除该临时文件。 临时文件的大小默认为1G，但可以通过 `--hdd-bytes` 设置临时文件的大小。比如

```
stress -i 2 -d 4 --hdd-bytes 512M
```

你会发现压力测试时，当前目录所在可用空间少了2G,如下所示：

```
[lujun9972@T430S ~]$ df -h .
\文件系统        容量  已用  可用 已用% 挂载点
/dev/sdb1       466G  255G  211G   55% /home
[lujun9972@T430S ~]$ stress -i 2 -d 4 --hdd-bytes 512M &
[1] 20101
[lujun9972@T430S ~]$ stress: info: [20101] dispatching hogs: 0 cpu, 2 io, 0 vm, 4 hdd
[lujun9972@T430S ~]$ df -h .
文件系统        容量  已用  可用 已用% 挂载点
/dev/sdb1       466G  257G  209G   56% /home
```

### 同时对多项指标进行压力测试

stress支持同时对多个指标进行压力测试，只需要把上面的参数组合起来就行

```
stress -c 4 -m 2 -d 1
```

这个时候你再看stress进程

```
ps -elf |grep stress |grep -v grep
0 S lujun99+ 19048  1501  0  80   0 -  1996 -      23:36 pts/0    00:00:00 stress -c 4 -m 2 -d 1
1 R lujun99+ 19049 19048 56  80   0 -  1996 -      23:36 pts/0    00:00:25 stress -c 4 -m 2 -d 1
1 R lujun99+ 19050 19048 55  80   0 - 67533 -      23:36 pts/0    00:00:25 stress -c 4 -m 2 -d 1
1 D lujun99+ 19051 19048 28  80   0 -  2221 -      23:36 pts/0    00:00:12 stress -c 4 -m 2 -d 1
1 R lujun99+ 19052 19048 58  80   0 -  1996 -      23:36 pts/0    00:00:26 stress -c 4 -m 2 -d 1
1 R lujun99+ 19053 19048 56  80   0 - 67533 -      23:36 pts/0    00:00:25 stress -c 4 -m 2 -d 1
1 R lujun99+ 19054 19048 57  80   0 -  1996 -      23:36 pts/0    00:00:25 stress -c 4 -m 2 -d 1
1 R lujun99+ 19055 19048 58  80   0 -  1996 -      23:36 pts/0    00:00:26 stress -c 4 -m 2 -d 1
```

你会发现工作进程一共有7个，也就是说每个进程只负责一项测试。

### 设置超时时间

通过 `-t TIMEOUT` 可以让stress只运行一段时间后自动退出。这一般在写脚本的时候会用到。

比如我想要运行上面的测试，但是10秒后自动退出,那么

```
stress -c 4 -m 2 -d 1 -t 10s
stress: info: [19302] dispatching hogs: 4 cpu, 0 io, 2 vm, 1 hdd
stress: info: [19302] successful run completed in 11s
```