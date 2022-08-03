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
- VmallocChunk: vmalloc区可用的连续最大快的大小
