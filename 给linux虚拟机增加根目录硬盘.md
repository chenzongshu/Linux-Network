# 目的

VMVare或者云ECS中, 发现根目录磁盘太小, 需要增加根目录磁盘大小.

# 申请磁盘

这个步骤省略, 可在VMVare软件更改磁盘大小或申请云硬盘.

# 准备分区

没有处理的磁盘, 现在Linux里面是看不到的.

先查看磁盘分区

```
[root@centos dev]# fdisk -l

磁盘 /dev/sda：42.9 GB, 42949672960 字节，83886080 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节
磁盘标签类型：dos
磁盘标识符：0x000b340a

   设备 Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048     2099199     1048576   83  Linux
/dev/sda2         2099200    41943039    19921920   8e  Linux LVM

磁盘 /dev/mapper/centos-root：18.2 GB, 18249416704 字节，35643392 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节


磁盘 /dev/mapper/centos-swap：2147 MB, 2147483648 字节，4194304 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节

```

上面看到有2个分区`sda1`, `sda2`, 这个时候`sda`已经扩为40G了, 我们来创建新的`sda3`

```
[root@centos dev]# fdisk /dev/sda
欢迎使用 fdisk (util-linux 2.23.2)。

更改将停留在内存中，直到您决定将更改写入磁盘。
使用写入命令前请三思。


命令(输入 m 获取帮助)：m
命令操作
   a   toggle a bootable flag
   b   edit bsd disklabel
   c   toggle the dos compatibility flag
   d   delete a partition
   g   create a new empty GPT partition table
   G   create an IRIX (SGI) partition table
   l   list known partition types
   m   print this menu
   n   add a new partition
   o   create a new empty DOS partition table
   p   print the partition table
   q   quit without saving changes
   s   create a new empty Sun disklabel
   t   change a partition\'s system id
   u   change display/entry units
   v   verify the partition table
   w   write table to disk and exit
   x   extra functionality (experts only)

命令(输入 m 获取帮助)：n  // 命令n用于添加新分区
Partition type:
   p   primary (2 primary, 0 extended, 2 free)
   e   extended
Select (default p): p    // 选择创建主分区
分区号 (3,4，默认 3)：   //fdisk会让你选择主分区的编号,前面有1,2了
起始 扇区 (41943040-83886079，默认为 41943040)：
将使用默认值 41943040
Last 扇区, +扇区 or +size{K,M,G} (41943040-83886079，默认为 83886079)：
将使用默认值 83886079
分区 3 已设置为 Linux 类型，大小设为 20 GiB

命令(输入 m 获取帮助)：w // w 保存所有并退出，分区划分完毕
The partition table has been altered!

Calling ioctl() to re-read partition table.
```

我们的新建分区/dev/sda3，却不是LVM的。所以，接下来使用fdisk将其改成LVM的

```
[root@centos dev]# fdisk /dev/sda
欢迎使用 fdisk (util-linux 2.23.2)。

更改将停留在内存中，直到您决定将更改写入磁盘。
使用写入命令前请三思。


命令(输入 m 获取帮助)：m
命令操作
   a   toggle a bootable flag
   b   edit bsd disklabel
   c   toggle the dos compatibility flag
   d   delete a partition
   g   create a new empty GPT partition table
   G   create an IRIX (SGI) partition table
   l   list known partition types
   m   print this menu
   n   add a new partition
   o   create a new empty DOS partition table
   p   print the partition table
   q   quit without saving changes
   s   create a new empty Sun disklabel
   t   change a partition's system id
   u   change display/entry units
   v   verify the partition table
   w   write table to disk and exit
   x   extra functionality (experts only)

命令(输入 m 获取帮助)：t //改变分区系统id
分区号 (1-3，默认 3)：3  //指定分区号
Hex 代码(输入 L 列出所有代码)：8e  //指定要改成的id号，8e代表LVM
已将分区“Linux”的类型更改为“Linux LVM” 

命令(输入 m 获取帮助)：w
The partition table has been altered!
```

重启系统`reboot`后，登陆系统。（一定要重启系统，否则无法扩充新分区）

# 新分区挂到根目录

这个时候 `fdisk -l`可以看到新分区了, 然后格式化新分区

```
mkfs -t ext3 /dev/sda3   //在硬盘分区“/dev/sda3”上创建“ext3”文件系统
```

这个时候我们可以使用`pvcreate`扩充新分区了

pvcreate指令用于将物理硬盘分区初始化为物理卷，以便被LVM使用。要创建物理卷必须首先对硬盘进行分区，并且将硬盘分区的类型设置为“8e”后，才能使用pvcreat指令将分区初始化为物理卷.

```
[root@centos1 ~]# pvcreate /dev/sda3
WARNING: ext3 signature detected on /dev/sda3 at offset 1080. Wipe it? [y/n]: y
  Wiping ext3 signature on /dev/sda3.
  Physical volume "/dev/sda3" successfully created.
```

然后我们使用`vgextend`扩充当前的lvm组,具体lvm组名可以使用下面命令查看

```
[root@centos1 ~]# vgs
  VG     #PV #LV #SN Attr   VSize   VFree
  centos   1   2   0 wz--n- <19.00g    0
  
// 对比df 命令和lvs命令
[root@centos1 ~]# df -h
文件系统                 容量  已用  可用 已用% 挂载点
/dev/mapper/centos-root   17G  5.4G   12G   32% /
devtmpfs                 895M     0  895M    0% /dev

[root@centos1 ~]# lvs
  LV   VG     Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root centos -wi-ao---- <17.00g
  swap centos -wi-ao----   2.00g
```

知道当前有1个lvm组centos, 下面2个LV. 然后使用`vgextend + lvm组名` 

```
[root@centos1 ~]# vgextend centos /dev/sda3
  Volume group "centos" successfully extended
```

这个时候可以看到加入的磁盘容量, 前面部分省略, 注意Free PE, 就是我们新增的容量

```
[root@centos1 ~]# vgdisplay
.....
  Alloc PE / Size       4863 / <19.00 GiB
  Free  PE / Size       5119 / <20.00 GiB
  VG UUID               oU868Z-fAk7-kSaN-wAud-AC3U-WDBh-N75dZS
```

然后扩充空间

```
[root@centos1 centos]# lvextend -L+19.8G /dev/centos/root /dev/sda3
  Rounding size to boundary between physical extents: 19.80 GiB.
  Size of logical volume centos/root changed from <17.00 GiB (4351 extents) to <36.80 GiB (9420 extents).
  Logical volume centos/root successfully resized.
```

最后, 增大"ext2/ext3"文件系统大小

```
[root@centos1 centos]# xfs_growfs /dev/centos/root
meta-data=/dev/mapper/centos-root isize=512    agcount=4, agsize=1113856 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0 spinodes=0
data     =                       bsize=4096   blocks=4455424, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal               bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 4455424 to 9646080
```

操作完最后一步, 才能看到根目录已经扩展为40G了

```
[root@centos1 centos]# df -h
文件系统                 容量  已用  可用 已用% 挂载点
/dev/mapper/centos-root   37G  5.4G   32G   15% /
devtmpfs                 895M     0  895M    0% /dev
```
