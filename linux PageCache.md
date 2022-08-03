# Linux文件IO

Linux 的两种文件 I/O 模式了：Direct I/O 和 Buffered I/O。

- `Direct I/O 模式`，用户进程如果要写磁盘文件，就会通过 Linux 内核的文件系统层 (filesystem) -> 块设备层 (block layer) -> 磁盘驱动 -> 磁盘硬件，这样一路下去写入磁 盘。

-  `Buffered I/O 模式`，用户进程只是把文件数据写到内存中（Page Cache） 就返回了，而 Linux 内核自己有线程会把内存中的数据再写入到磁盘中。在 Linux 里，由 于考虑到性能问题，绝大多数的应用都会使用 Buffered I/O 模式。

当使用 Buffered I/O 的应用程序从虚拟机迁移到容器，这时我们就会发现多了 Memory Cgroup 的限制之后，write() 写相同大小的数据块花费的时间，延时波动会比较大

对于 Buffer I/O，用户的数据是先写入到 Page Cache 里的。而这些写入了数据的内存页面，在它们没 有被写入到磁盘文件之前，就被叫作 dirty pages。

Linux 内核会有专门的内核线程（每个磁盘设备对应的 kworker/flush 线程）把 dirty pages 写入到磁盘中
