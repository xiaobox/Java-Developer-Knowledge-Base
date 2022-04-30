# JVM最多可以创建多少线程？

![](https://mmbiz.qpic.cn/mmbiz_jpg/YZibCWq4rxDicwpkFs519kxdmhNpQ4eF2o7JibZGG0ibvNIC4QmIicTxD5AHQCjiajTFk1UsIzFEIB8Vib9kdPPJZd0xg/640?wx_fmt=jpeg\&wxfrom=5\&wx_lazy=1\&wx_co=1)

具体计算公式如下： &#x20;

**(MaxProcessMemory - JVMMemory - ReservedOsMemory) / (ThreadStackSize) = Number of threads**

*   MaxProcessMemory : 进程的最大寻址空间

*   JVMMemory : JVM内存

*   ReservedOsMemory : 保留的操作系统内存，如Native heap，JNI之类，一般100多M

*   ThreadStackSize : 线程栈的大小，jvm启动时由Xss指定 默认1M

MaxProcessMemory：如32位的linux默认每个进程最多申请3G的地址空间，64位的操作系统可以支持到46位（64TB）的物理地址空间和47位（128T）的进程虚拟地址空间（linux 64位CPU内存限制）。

JVM内存：由Heap区和Perm区组成。通过-Xms和-Xmx可以指定heap区大小，通过-XX:PermSize和-XX:MaxPermSize指定perm区的大小(默认从32MB 到64MB，和JVM版本有关)。

**总结下影响Java线程数量的因素：**

*   Java虚拟机本身：-Xms，-Xmx，-Xss；

*   系统限制：

    *   /proc/sys/kernel/pid\_max&#x20;

    *   /proc/sys/kernel/thread-max

    *   /max\_user\_process（ulimit -u）

    *   /proc/sys/vm/max\_map\_count

**想增加线程数，在JVM内部可以通过减少最大堆或减少栈容量来实现**
