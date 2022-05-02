# IO

当程序调用各类文件操作函数后，用户数据（User Data）到达磁盘（Disk）的流程如图所示:

图中描述了 Linux 下文件操作函数的层级关系和内存缓存层的存在位置。中间的黑色实线是用户态和内核态的分界线

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h181gryrjzj20u00pw769.jpg)

从上往下分析这张图：

**1.** 首先是 C 语言 stdio 库定义的相关文件操作函数，这些都是用户态实现的跨平台封装函数。stdio 中实现的文件操作函数有自己的 stdio buffer，这是在用户态实现的缓存。此处使用缓存的原因很简单 — 系统调用总是昂贵的。如果用户代码以较小的 size 不断的读或写文件的话，stdio 库将多次的读或者写操作通过 buffer 进行聚合是可以提高程序运行效率的。stdio 库同时也支持 fflush 函数来主动的刷新 buffer，主动的调用底层的系统调用立即更新 buffer 里的数据。特别地，setbuf 函数可以对 stdio 库的用户态 buffer 进行设置，甚至取消 buffer 的使用。

**2.** **系统调用的 read/write 和真实的磁盘读写之间也存在一层 buffer**，这里用术语 Kernel buffer cache 来指代这一层缓存。在 Linux 下，文件的缓存习惯性的称之为[页缓存（page cache）](<../Linux 内存管理/页缓存（page cache）/页缓存（page cache）.md> "页缓存（page cache）")，而更低一级的设备的缓存称之为 Buffer Cache。这两个概念很容易混淆，这里简单的介绍下概念上的区别：Page Cache 用于缓存文件的内容，和文件系统比较相关。文件的内容需要映射到实际的物理磁盘，这种映射关系由文件系统来完成；Buffer Cache 用于缓存存储设备块（比如磁盘扇区）的数据，而不关心是否有文件系统的存在（文件系统的元数据缓存在 Buffer Cache 中）。

## 参考

*   [https://mp.weixin.qq.com/s/AnEwjQDDV66Ob3-8EUWMKA](https://mp.weixin.qq.com/s/AnEwjQDDV66Ob3-8EUWMKA "https://mp.weixin.qq.com/s/AnEwjQDDV66Ob3-8EUWMKA")


[Unix 网络 IO 模型](Unix%20网络%20IO%20模型/Unix%20网络%20IO%20模型.md "Unix 网络 IO 模型")
