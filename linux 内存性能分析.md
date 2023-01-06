# 基础概念

## 匿名页

与匿名页相对应的是文件页，文件页我们应该很好理解，就是映射文件的页，如：通过mmap映射文件到虚拟内存然后读文件数据,进程的代码数据段等，这些页有后备缓存也就是块设备上的文件，而匿名页就是没有关联到文件的页，如：进程的堆、栈等

## 零页

系统初始化过程中分配了一页的内存，这段内存全部被填充０。不如定义了一个全局变量，大小为一页，页对齐到bss段，所有**这段数据内核初始化的时候会被清零**，所有称之为０页

# page

应用程序都是跟虚拟地址打交道，不会直接跟物理地址打交道。而虚拟地址最终都要转换为物理地址，由于 Linux 都是使用 Page（页）来进行管理的，所以这个过程叫 Paging（分页）。

# linux 内存管理

- kswapd： 内核进程进行后台内存回收

- kcompactd： 内核进程进行后台内存整理

- khugepaged：透明巨页的回收、整理、折叠

# 大页内存

### 什么是大页内存

系统进程是通过虚拟地址访问内存，但是CPU必须把它转换成物理内存地址才能真正访问内存。为了提高这个转换效率，CPU会缓存最近的虚拟内存地址和物理内存地址的映射关系，并保存在一个由CPU维护的映射表中。为了尽量提高内存的访问速度，需要在映射表中保存尽量多的映射关系。

Linux内存管理采用“分页机制”，内存页面默认大小为4KB。但是当运行内存需求量较大时，默认4KB大小的页面会导致较多的TLB miss和缺页中断，从而大大影响应用程序性能。

所以hugepage是在Linux2.6内核被引入的，主要提供4k的page和比较大的page的选择，一般是2M和1G的大页

### 大页内存分类

大页内存有2种： 普通大页内存和透明大页内存

普通大页对应用性能无影响

#### 普通大页

简单来说就是需要系统初始化的时候提前分配，一般的应用进程无法访问该空间，需要mmap 系统调用或者 shmat 和 shmget 系统调用适配才可以使用大页

优缺点如下：

```YAML
优点:
1.无需交换：不存在页面由于内存空间不足而换入换出的问题。2.减轻 TLB Cache 的压力：相同的内存大小，管理的虚拟地址数量变少，降低了 CPU Cache 可缓存的地址映射压力。3.降低 Page Table 负载：内存页的数量会减少，从而需要更少的 Page table，节约了页表所占用的内存数量。4.消除 Page table 查找负载:：因为页面不需要替换，所以不需要页表查找。5.提高内存的整体性能：页面变大后，处理的页面较少，因此可以明显避免访问页表时可能出现的瓶颈。

缺点：
1.需要合理设置，避免内存浪费：在操作系统启动期间被动态分配并被保留，共享内存不会被置换，在使用 HugePage 的内存不能被其他的进程使用，所以要合理设置该值，避免造成内存浪费。2.静态设置无法自适应：如果增加 HugePage 或添加物理内存或者是当前服务器增加了新的实例发生变化，需要重新设置所需的 HugePage 大小。
```

#### 透明大页

官方文档： [Transparent Hugepage Support ‒ The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/admin-guide/mm/transhuge.html?spm=a2c4g.11186623.0.0.372e790aRWIpL7)

普通大页可带来性能提升，但存在配置和管理上的困难，很难手动管理，而且通常需要对代码进行重大的更改才能有效使用，不能通用适配应用的接入使用

所以在标准大页的基础上，通过一个抽象层，能够自动创建、管理和使用传统大页，用来提高内存管理的性能，解决大页手动管理的问题。

简单来说，透明大页THP可以使内核为用户进程自动分配大页（Huge Pages），而无需像HugeTLB一样预先保留一定数量的大页。此功能在一定程度上使用户的应用程序性能得以提升，然而在实际生产中，如果该选项设置不当，反而可能造成应用程序的性能波动。

```YAML
优点：
相对标准大页，操作系统动态管理大页分配，支持大页分配 ，大页->普通页换出拆分，根据具体应用使用情况，自适应调整分配。使用上对应用透明，且开源社区持续测试、优化和适配大部分应用，同时支持多种大页申请方式配置，满足多场景的分配需求

缺点：
THP 在运行时动态分配内存，可能会带来运行时内存分配的 CPU 开销和上涨问题。THP 同步申请模式下，分配失败时触发申请时，导致应用的一些随机卡顿现象。
```

**khugepaged**： khugepaged 是透明巨页的守护进程，它的主要功能是定时唤醒，根据配置尝试将4k 的普通page转成2M等巨页，减少TLB压力，提高内存使用效率。
khugepaged的处理过程中需要各种内存锁，会在一定程度上影响系统性能。

透明大页的参数位于

```bash
/sys/kernel/mm/transparent_hugepage/
├── defrag      #碎片整理
├── enabled     #系统全局开启透明大页THP功能
├── khugepaged
│   ├── alloc_sleep_millisecs
│   ├── defrag
│   ├── full_scans
│   ├── max_ptes_none
│   ├── pages_collapsed
│   ├── pages_to_scan
│   └── scan_sleep_millisecs
└── use_zero_page
```

khugepaged主要参数配置：

- **defrag**： khugepaged的功能开关。1为打开，0为关闭。

> 由于该操作会在内存路径中加锁，并且khugepaged内核守护进程可能会在错误的时间启动扫描和转换大页，因此存在影响应用性能的可能性。

- **alloc_sleep_millisecs**: 重试间隔。当透明大页THP分配失败时，khugepaged内核守护进程进行下一次大页分配前需要等待的时间。避免短时间内连续发生大页分配失败。默认值为`60000`，单位为毫秒

- **scan_sleep_millisecs**： 唤醒间隔。khugepaged内核守护进程每次唤醒的时间间隔。默认值为`10000`，单位为毫秒，即默认每10秒唤醒一次

- **pages_to_scan**： 扫描页数。khugepaged内核守护进程每次唤醒后扫描的页数。为4096个页

- pages_collapsed： 折叠的页数，这个是一个累加值，可以看到进度

- max_ptes_shared：

- max_ptes_none：指定有多少额外的小页面（即尚未映射的）可以在折叠一组小页到大页中被分配，较高的值会导致程序使用额外的内存。数值越低，获得的thp性能越低。

## 透明大页内存影响

为了尽可能快的响应用户的内存申请需求并保证系统在内存资源紧张时运行，Linux 定义了三条水位线 （high，low，min），当剩余物理内存低于 low 高于 min 水位线时，在用户申请内存时通过 kswapd 内核线程异步回收内存，直到水位线恢复到 high 以上，若异步回收的速度跟不上线程内存申请的速度时，将触发同步的直接内存回收，也就是所有申请内存的线程都同步的参与内存回收，一起将水位线抬上去后再获得内存。这时，若需要回收的页面是干净的，则同步引起的阻塞时间就比较短，反之则很大（比如几十、几百ms 甚至 s 级，取决于后端设备速度）。

除水位线外，当申请大的连续内存时，若剩余物理内存充足，但碎片化比较严重时，内核在做内存规整的时候，也有可能触发直接内存回收（取决于碎片化指数）。因此内存直接回收和内存规整是进程申请内存路径上的可能遇到的主要延迟

透明大页，可以动态地将系统默认内存页块 4KB，交换为 huge pages，在这个过程中，对于操作系统的内存的各种分配活动，都需要各种内存锁，直接影响程序的内存访问性能，这个过程对应用是透明的，在应用层面不可控制，对于专门优化 4KB page 优化的程序来说，可能造成随机的性能下降问题。

结合对应的参数：

```YAML
- 如果透明大页THP的碎片整理开关设置为always，内存紧张时会和普通4 KB页一样，出现内存的直接回收或内存的直接整理，这两个操作均是同步等待的操作，会造成系统性能下降。- 如果khugepaged碎片整理的开关设置为1，在khugepaged内核守护进程进行内存合并操作时，会在内存路径中加锁。如果khugepaged碎片整理在错误的时间被触发，会对内存敏感型应用造成性能影响。- 如果保持开启透明大页THP，同时关闭上述两个碎片整理的开关，则内存分配过程相较于4 KB页可能会更快地消耗完空闲页资源，然后系统开始进入内存回收和内存整理的过程，反而更早的出现系统性能下降。
```

## 透明大页内存查看和监控

### 查看

透明大页THP的使用情况主要分为两个层面，分别如下：

- 系统级别
  在系统中执行下列示例命令，查看透明大页THP的使用情况。

```YAML
cat /proc/meminfo | grep AnonHugePages
```

系统显示的示例如下。

AnonHugePages: 614400 kB

> **说明**：如果系统返回非零值，则说明系统中使用了一定数量的透明大页THP。

- 进程级别
  在系统中执行下列示例命令，查看某个进程使用的透明大页THP。

```YAML
cat /proc/[$PID]/smaps | grep AnonHugePages
```

> **说明**：[$PID]指进程的PID。

系统显示的示例如下。

```YAML
AnonHugePages:         0 kBAnonHugePages:         0 kBAnonHugePages:         0 kBAnonHugePages:         0 kBAnonHugePages:         0 kBAnonHugePages:         0 kBAnonHugePages:         0 kBAnonHugePages:         0 kBAnonHugePages:         0 kBAnonHugePages:         0 kB
```

### 监控透明大页

1. 监控khugepaged进程

2. 上面一节有提到smaps，但是读取smaps文件时昂贵的，且经常会产生开销。

3. 内存碎片和回收指数
- 运行 sar -B 观察 pgscand/s，其含义为每秒发生的直接内存回收次数，当在一段时间内持续大于 0 时，则应继续执行后续步骤进行排查；

- 运行 `cat /sys/lernel/debug/extfrag/extfrag_index`观察内存碎片指数，重点关注 order >= 3 的碎片指数，当接近 1.000 时，表示碎片化严重，当接近 0 时表示内存不足；

- 运行 `cat /proc/buddyinfo, cat /proc/pagetypeinfo`查看内存碎片情况， 指标含义参考 （ [https://man7.org/linux/man-pages/man5/proc.5.html](https://man7.org/linux/man-pages/man5/proc.5.html) ），同样关注 order >= 3 的剩余页面数量，
4. 在**/proc/vmstat**中有许多计数器可以用于监视系统提供大页面
- **thp_fault_alloc** : 每当处理缺页异常时，一个大页面被成功分配，thp_fault_alloc就会增加。这适用于第一次出现缺页异常和COW错误。

- **thp_collapse_alloc**：当它发现一个范围的页面坍缩成一个大页，并有成功分配一个新的巨大页来存储数据，thp_collapse_alloc会被khugepaged增加。

- **thp_fault_fallback**: 如果缺页异常失败的分配一个大页，则thp_fault_fallback被增加，而回退使用小页面。

- **thp_collapse_alloc_failed**: 当它发现一个范围的页面应该被坍缩成一个大页， 但是分配大页失败，thp_collapse_alloc_failed会被khugepaged增加。

- **thp_file_alloc**: 在文件大页成功分配时递增。

- **thp_file_mapped**: 每映射到一个文件大页到用户地址空间，thp_file_mapped就增加一次。

- **thp_split_page**：在每次将一个巨大的页面分裂为普通页时递增。发生这种情况的原因有很多，但都很常见原因是一个巨大的页面是旧的，正在被回收。这个操作意味着分裂页面映射的所有PMD。

- **thp_split_page_failed**：如果内核无法分裂大页，则增加thp_split_page_failed计数。如果页面被人pin住了，就会发生这种情况。

- **thp_deferred_split_page**：当大页被放到分裂队列时，thp_deferred_split_page计数被增加。当一个巨大的页面部分被unmap且分裂它将释放一些内存就会发生这种情况。分裂队列上的页将在内存压力下分裂。

- **thp_split_pmd**: 每当pmd分裂成pte表时，thp_split_pmd就会递增。例如，当应用程序调用mprotect()或unmap()在大页面的一部分。它不会分割大页面，只是页表条目。

- **thp_zero_page_alloc**: thp_zero_page_alloc在每出现一个巨型零页被成功地分配时递增。它包括分配，放弃了与其他分配的竞争。注意，这不算每次巨型零页的映射，只有它的分配。

# page cache

## 什么是page cache

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

## page cache产生的方式

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
