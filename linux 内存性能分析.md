# page

应用程序都是跟虚拟地址打交道，不会直接跟物理地址打交道。而虚拟地址最终都要转换为物理地址，由于 Linux 都是使用 Page（页）来进行管理的，所以这个过程叫 Paging（分页）。

## page cache

### 什么是page cache

为了提升对文件的读写效率，Linux 内核会以页大小（4KB）为单位，将文件划分为多数据块。当用户对文件中的某个数据块进行读写操作时，内核首先会申请一个内存页（称为 `页缓存`）与文件中的数据块进行绑定

page cache是内核管理的内存，他属于内核不属于用户

```bash
  ┌────┐                ┌───────────────────┐                                       
  │User│                │    Application    │──Mmeory-Mapped I/O───┐                
  └────┘                └───────────────────┘                      │                
                                  │                                │                
                                  │                              mmap               
                                  │                                │                
──────────────────────────────syscall──────────────────────────────┼───────────────▶
                                  │                                │                
                                  │                                ▼                
 ┌──────┐                 ┌───────▼──────┐  buffered      ┌────────────────┐        
 │Kernel│                 │     VFS      │────I/O────────▶│   page cache   │        
 └──────┘                 └──────────────┘                └────────────────┘        
                                  │                                                 
                                  │                                                 
                                  │                                                 
                                  ▼                                                 
                      ┌──────────────────────┐                                      
                      │                      │                                      
                      │   Block I/O Layer    │                                      
                      │ ┌──────────────────┐ │                                      
                      │ │  I/O Scheduler   │ │                                      
                      └─┴──────────────────┴─┘                                      
                                  │                                                 
                             Direct I/O                                             
                                  ▼                                                 
                         ┌─────────────────┐                                        
                         │  Device Driver  │                                        
                         └─────────────────┘                                        
                                  │                                                 
                                  │                                                 
                                  ▼                                                 
                       ┌─────────────────────┐                                      
                       │    HDD,SSD,NVME     │                                      
                       └─────────────────────┘                                      
```

标准 I/O 和内存映射会先把数据写⼊到 Page Cache，这样做会通过减少I/O 次数来提升读写效率。

这就是Page Cache 存在的意义：减少 I/O，提升应⽤的 I/O 速度。

### page cache产生的方式

Page Cache 的产⽣有两种不同的⽅式：

- Buffered I/O（标准 I/O）；

- Memory-Mapped I/O（存储映射 I/O）。

⼆者都能产⽣ Page Cache，但是⼆者的还是有些差异的：

标准 I/O 是写的 write ⽤户缓冲区 (Userpace Page 对应的内存)，然后再将⽤户缓冲区⾥的数据拷⻉到内核缓冲区 (Pagecache Page 对应的内存)；如果是读的 read话则是先从内核缓冲区拷⻉到⽤户缓冲区，再从⽤户缓冲区读数据，也就是 buffer 和⽂件内容不存在任何映射关系。
对于存储映射 I/O ⽽⾔，则是直接将 Pagecache Page 给映射到⽤户地址空间，⽤户直接写 Pagecache Page 中内容。

所以使⽤内存映射 I/O ⽐标准 I/O ⽅式性能要好⼀些

## dirty page

大部分文件页，都可以直接回收，以后有需要时，再从磁盘重新读取就可以了。而那些被应用程序修改过，并且暂时还没写入磁盘的数据（也就是脏页 dirty page），就得先写入磁盘，然后才能进行内存释放。

这些脏页，一般可以通过两种方式写入磁盘。

- 可以在应用程序中，通过系统调用 fsync ，把脏页同步到磁盘中；
- 也可以交给系统，由内核线程 pdflush 负责这些脏页的刷新。

## page管理引起的问题

- 直接内存回收引起的 load 飙高；

- 系统中脏页积压过多引起的 load 飙高；

- 系统 NUMA 策略配置不当引起的 load 飙高

## 分析数据和工具

可以用sar -B 来分析分⻚信息 (Paging statistics)， 以及 sar -r来分析内存使⽤情况统计

也可以观察/proc/vmstat

| 问题           | /proc/vmstat指标     | 指标解释                             |
| ------------ | ------------------ | -------------------------------- |
| page cache回收 | pgscan_kswapd      | kswapd后台扫描的Page个数                |
|              | pgsteal_kswapd     | kswapd后台回收的Page个数                |
|              | pgscan_direct      | 进程直接扫描的Page个数                    |
|              | pgsteal_direct     | 进程直接回收的Page个数                    |
| 碎片整理         | compact_stail      | 直接碎片整理的次数                        |
|              | compact_fail       | 直接碎片整理失败的次数                      |
|              | compact_success    | 直接碎片整理成功的次数                      |
| 脏页回写         | nr_dirty           | 脏页个数                             |
| drop cache   | drop_pagecache     | 执行drop_cache来drop page cache的次数  |
|              | drop_slab          | 执行drop_cache来dropslab的次数         |
|              | pginodesteal       | 直接回收inode过程中回收page cache个数       |
|              | kswapd_inodesteal  | kswapd后台回收inode过程中回收page cache个数 |
| I/O          | pgpgin             | 从磁盘读文件读了多少page到内存                |
|              | pgpgout            | 内存多少page被写回磁盘                    |
| SWAP I/O     | pswpin             | 从swap分区读文件读了多少page到内存            |
|              | pswpout            | 内存多少page被写回到swap分区               |
| workingset   | workingset_refault | Page被释放后短时间内再次从磁盘被读入到内存          |
|              | workingset_restore | Page被回收前又被检测为活跃Page从而避免被回收       |

# /proc/meminfo

/proc文件系统是一种内核和内核模块用来向进程发送信息的机制。这个伪文件系统可以和内核内部的数据结构进行交互，获取实时的进程信息。

**/proc虚拟文件系统实质是以文件系统的形式访问内核数据的接口**

```
$cat /proc/meminfo
MemTotal:        8052444 kB
MemFree:         2754588 kB
MemAvailable:    3934252 kB
Buffers:          137128 kB
Cached:          1948128 kB
SwapCached:            0 kB
Active:          3650920 kB
Inactive:        1343420 kB
Active(anon):    2913304 kB
Inactive(anon):   727808 kB
Active(file):     737616 kB
Inactive(file):   615612 kB
Unevictable:         196 kB
Mlocked:             196 kB
SwapTotal:       8265724 kB
SwapFree:        8265724 kB
Dirty:               104 kB
Writeback:             0 kB
AnonPages:       2909332 kB
Mapped:           815524 kB
Shmem:            732032 kB
Slab:             153096 kB
SReclaimable:      99684 kB
SUnreclaim:        53412 kB
KernelStack:       14288 kB
PageTables:        62192 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:    12291944 kB
Committed_AS:   11398920 kB
VmallocTotal:   34359738367 kB
VmallocUsed:           0 kB
VmallocChunk:          0 kB
HardwareCorrupted:     0 kB
AnonHugePages:   1380352 kB
CmaTotal:              0 kB
CmaFree:               0 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
DirectMap4k:      201472 kB
DirectMap2M:     5967872 kB
DirectMap1G:     3145728 kB
```

- MemTotal: 所有内存(RAM)大小,减去一些预留空间和内核的大小。
- MemFree: 完全没有用到的物理内存，lowFree+highFree
- MemAvailable: 在不使用交换空间的情况下，启动一个新的应用最大可用内存的大小，计算方式：MemFree+Active(file)+Inactive(file)-(watermark+min(watermark,Active(file)+Inactive(file)/2))
- Buffers: 块设备所占用的缓存页，包括：直接读写块设备以及文件系统元数据(metadata)，比如superblock使用的缓存页。
- Cached: 表示普通文件数据所占用的缓存页。
- SwapCached: swap cache中包含的是被确定要swapping换页，但是尚未写入物理交换区的匿名内存页。那些匿名内存页，比如用户进程malloc申请的内存页是没有关联任何文件的，如果发生swapping换页，这类内存会被写入到交换区。
- Active: active包含active anon和active file
- Inactive: inactive包含inactive anon和inactive file
- Active(anon): anonymous pages（匿名页），用户进程的内存页分为两种：与文件关联的内存页(比如程序文件,数据文件对应的内存页)和与内存无关的内存页（比如进程的堆栈，用malloc申请的内存），前者称为file pages或mapped pages,后者称为匿名页。
- Inactive(anon): 见上
- Active(file): 见上
- Inactive(file): 见上
- SwapTotal: 可用的swap空间的总的大小(swap分区在物理内存不够的情况下，把硬盘空间的一部分释放出来，以供当前程序使用)
- SwapFree: 当前剩余的swap的大小
- Dirty: 需要写入磁盘的内存去的大小
- Writeback: 正在被写回的内存区的大小
- AnonPages: 未映射页的内存的大小
- Mapped: 设备和文件等映射的大小
- Slab: 内核数据结构slab的大小
- SReclaimable: 可回收的slab的大小
- SUnreclaim: 不可回收的slab的大小
- PageTables: 管理内存页页面的大小
- NFS_Unstable: 不稳定页表的大小
- VmallocTotal: Vmalloc内存区的大小
- VmallocUsed: 已用Vmalloc内存区的大小
- VmallocChunk: vmalloc区可用的连续最大快的大小# 

# TOP

```bash
# 按下 M 切换到内存排序
$ top
...
KiB Mem :  8169348 total,  6871440 free,   267096 used,  1030812 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  7607492 avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
  430 root      19  -1  122360  35588  23748 S   0.0  0.4   0:32.17 systemd-journal
 1075 root      20   0  771860  22744  11368 S   0.0  0.3   0:38.89 snapd
 1048 root      20   0  170904  17292   9488 S   0.0  0.2   0:00.24 networkd-dispat
    1 root      20   0   78020   9156   6644 S   0.0  0.1   0:22.92 systemd
12376 azure     20   0   76632   7456   6420 S   0.0  0.1   0:00.01 systemd
12374 root      20   0  107984   7312   6304 S   0.0  0.1   0:00.00 sshd
...
```

- VIRT 是进程虚拟内存的大小，只要是进程申请过的内存，即便还没有真正分配物理内存，也会计算在内。
- RES 是常驻内存的大小，也就是进程实际使用的物理内存大小，但不包括 Swap 和共享内存。
- SHR 是共享内存的大小，比如与其他进程共同使用的共享内存、加载的动态链接库以及程序的代码段等。
- %MEM 是进程使用物理内存占系统总内存的百分比。

注意两点：

1. 虚拟内存通常并不会全部分配物理内存。从上面的输出，你可以发现每个进程的虚拟内存都比常驻内存大得多。

2. 共享内存 SHR 并不一定是共享的，比方说，程序的代码段、非共享的动态链接库，也都算在 SHR 里。当然，SHR 也包括了进程间真正共享的内存。所以在计算多个进程的内存使用时，不要把所有进程的 SHR 直接相加得出结果。

# /proc/{pid}/smaps

`/proc/{pid}/smaps` 展示了一个进程里面的内存消耗， 分为每个VMA（虚拟内存区域）

```bash
00601000-00602000 rw-p 00001000 fd:20 83894544  /jdk1.8.0_202/bin/java
Size:                  4 kB
KernelPageSize:        4 kB
MMUPageSize:           4 kB
Rss:                   4 kB
Pss:                   4 kB
Shared_Clean:          0 kB
Shared_Dirty:          0 kB
Private_Clean:         0 kB
Private_Dirty:         4 kB
Referenced:            4 kB
Anonymous:             4 kB
LazyFree:              0 kB
AnonHugePages:         0 kB
ShmemPmdMapped:        0 kB
Shared_Hugetlb:        0 kB
Private_Hugetlb:       0 kB
Swap:                  0 kB
SwapPss:               0 kB
Locked:                0 kB
THPeligible:    0
ProtectionKey:         0
VmFlags: rd wr mr mw me dw ac sd 
```

- size： 虚拟内存总空间大小

- Rss：**是实际分配的内存**，这部分物理内存已经分配，不需要缺页中断就可以使用的

- Pss：**是平摊计算后的实际物理使用内存**(有些内存会和其他进程共享，例如mmap进来的)。实际上包含下面private_clean+private_dirty，和按比例均分的shared_clean、shared_dirty



# pmap

```bash
pmap [-x/-d] {pid}


Address           Kbytes Mode  Offset           Device    Mapping
0000000000400000       4 r-x-- 0000000000000000 0fd:00020 java
0000000000600000       4 r---- 0000000000000000 0fd:00020 java
0000000000601000       4 rw--- 0000000000001000 0fd:00020 java
00000000011cf000     132 rw--- 0000000000000000 000:00000   [ anon ]
00000004e0800000 12614528 rw--- 0000000000000000 000:00000   [ anon ]
```



- Address:  start address of map  映像起始地址  

- Kbytes:  size of map in kilobytes  映像大小  

- RSS:  resident set size in kilobytes  驻留集大小  

- Dirty:  dirty pages (both shared and private) in kilobytes  脏页大小  

- Mode:  permissions on map 映像权限: r=read, w=write, x=execute, s=shared, p=private (copy on write)    

- Mapping:  file backing the map , or '[ anon ]' for allocated memory, or '[ stack ]' for the program stack.  映像支持文件,[anon]为已分配内存 [stack]为程序堆栈  

- Offset:  offset into the file  文件偏移  

- Device:  device name (major:minor)  设备名

# 内存Buffer和Cache

Buffer 和 Cache 的设计目的，是为了提升系统的 I/O 性能。它们利用内存，充当起慢速磁盘与快速 CPU 之间的桥梁，可以加速 I/O 的访问速度。

从字面上来说，Buffer 是缓冲区，而 Cache 是缓存，两者都是数据在内存中的临时存储。

> - Buffers 是内核缓冲区用到的内存，对应的是 /proc/meminfo 中的 Buffers 值。
> - Cache 是内核页缓存和 Slab 用到的内存，对应的是 /proc/meminfo 中的 Cached 与 SReclaimable 之和。

**Buffer 是对磁盘数据的缓存，而 Cache 是文件数据的缓存，它们既会用在读请求中，也会用在写请求中**。

## 磁盘和文件区别

磁盘是一个存储设备（确切地说是块设备），可以被划分为不同的磁盘分区。而在磁盘或者磁盘分区上，还可以再创建文件系统，并挂载到系统的某个目录中。这样，系统就可以通过这个挂载目录，来读写文件。

换句话说，磁盘是存储数据的块设备，也是文件系统的载体。所以，文件系统确实还是要通过磁盘，来保证数据的持久化存储。

你在很多地方都会看到这句话， Linux 中一切皆文件。换句话说，你可以通过相同的文件接口，来访问磁盘和文件（比如 open、read、write、close 等）。

- 我们通常说的“文件”，其实是指普通文件。
- 而磁盘或者分区，则是指块设备文件。

在读写普通文件时，I/O 请求会首先经过文件系统，然后由文件系统负责，来与磁盘进行交互。而在读写块设备文件时，会跳过文件系统，直接与磁盘交互，也就是所谓的“裸 I/O”。

这两种读写方式使用的缓存自然不同。文件系统管理的缓存，其实就是 Cache 的一部分。而裸磁盘的缓存，用的正是 Buffer。

# 内存瓶颈分析

具体的分析思路主要有这几步。

1. 先用 free 和 top，查看系统整体的内存使用情况。
2. 再用 vmstat 和 pidstat，查看一段时间的趋势，从而判断出内存问题的类型。
3. 最后进行详细分析，比如内存分配分析、缓存 / 缓冲区分析、具体进程的内存使用分析等。

> 内存泄漏分析工具 memleak

# Linux文件IO

Linux 的两种文件 I/O 模式了：Direct I/O 和 Buffered I/O。

- `Direct I/O 模式`，用户进程如果要写磁盘文件，就会通过 Linux 内核的文件系统层 (filesystem) -> 块设备层 (block layer) -> 磁盘驱动 -> 磁盘硬件，这样一路下去写入磁 盘。

- `Buffered I/O 模式`，用户进程只是把文件数据写到内存中（Page Cache） 就返回了，而 Linux 内核自己有线程会把内存中的数据再写入到磁盘中。在 Linux 里，由 于考虑到性能问题，绝大多数的应用都会使用 Buffered I/O 模式。

当使用 Buffered I/O 的应用程序从虚拟机迁移到容器，这时我们就会发现多了 Memory Cgroup 的限制之后，write() 写相同大小的数据块花费的时间，延时波动会比较大

对于 Buffer I/O，用户的数据是先写入到 Page Cache 里的。而这些写入了数据的内存页面，在它们没 有被写入到磁盘文件之前，就被叫作 dirty pages。

Linux 内核会有专门的内核线程（每个磁盘设备对应的 kworker/flush 线程）把 dirty pages 写入到磁盘中
