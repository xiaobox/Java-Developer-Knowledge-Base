# JVM 调优之 glibc 引发的内存泄露

## 回顾

上篇文章中我们将 G1 换成 CMS 并调整了 JVM 参数，由于 GC 选择和参数设置的更加合理，所以内存的增长非常缓慢了。

但这并没有从根本解决问题，通过观察发现，最高的时候一天 RSS 会增长 100M 左右，而且整体的趋势仍然是向上增长的，并没有一丝的回落迹象。

## 问题分析

虽然增长缓慢，哪怕每天只有 1M，离 OOM 也只是时间问题。这就使我们不得不再次仔细分析为什么 RSS 会一直增长。

### 堆内存分析

通过监控发现，堆内存呈周期性的增长和回收，与我们的 JVM 参数设置一致，而且通过 dump 文件也没有发现明显的业务代码问题。

### 堆外内存

回顾一下我们的 options 参数设置 :

```text
-Xms2048m -Xmx2048m
-XX:+HeapDumpOnOutOfMemoryError
-XX:+CrashOnOutOfMemoryError
-XX:NativeMemoryTracking=detail
-XX:+UseConcMarkSweepGC 
-XX:MetaspaceSize=256M 
-XX:MaxMetaspaceSize=256M
-XX:ReservedCodeCacheSize=128m 
-XX:InitialCodeCacheSize=128m
-Xss512k
-XX:+AlwaysPreTouch。
```

metaspace 和 codecache 是限制死了，再来是 Buffer Pools 中的 Direct Buffers

![](https://tva1.sinaimg.cn/large/008i3skNly1gxtcpydxbyj30mp085mxh.jpg)

可以看到是一条横线，也没有什么波动。

但是 RSS，确实就是一直在增长，期间也利用 Native Memory Tracking 追踪过 JVM 内部内存的使用情况，具体是这样做的

由于我们开启了 NMT `-XX:NativeMemoryTracking=detail`

先设置一个基线：

```bash
jcmd 1 VM.native_memory baseline
```

然后过一段时间执行：

```bash
jcmd 1 VM.native_memory summary.diff
```

对比地看一下统计信息。下图只做示例，具体数字不做参考，因为是我临时执行出来的，数字不对。

![](https://tva1.sinaimg.cn/large/008i3skNly1gxtcxwk1l8j30je09zwf9.jpg)

真实环境中，增长最多的就是 class 中的 **malloc**

malloc ? 这是申请内存的函数啊，为什么要申请这么多呢？难道没有释放？ 于是想到用 pmap 命令看一下内存映射情况。

> Pmap 提供了进程的内存映射，pmap 命令用于显示一个或多个进程的内存状态。其报告进程的地址空间和内存状态信息

执行了以下命令：

```bash
pmap -x 1 | sort -n -k3
```

发现了一些端倪：

![](https://tva1.sinaimg.cn/large/008i3skNly1gxtd3cu0o9j30u60s2te7.jpg)

有一些 64M 左右的内存分配，且越来越多。

### glibc

搞不懂了，于是 google 了一下。发现是有这一类问题由于涉及许多底层基础知识，这里就大概解析一下，有兴趣的读者可以查询更多资料了解：

目前大部分服务端程序使用 glibc 提供的 malloc/free 系列函数来进行内存的分配。

> Linux 中 malloc 的早期版本是由 Doug Lea 实现的，它有一个严重问题是内存分配只有一个分配区（arena），每次分配内存都要对分配区加锁，分配完释放锁，导致多线程下并发申请释放内存锁的竞争激烈。arena 单词的字面意思是「舞台；竞技场」
> 于是修修补补又一个版本，你不是多线程锁竞争厉害吗，那我多开几个 arena，锁竞争的情况自然会好转。
> Wolfram Gloger 在 Doug Lea 的基础上改进使得 glibc 的 malloc 可以支持多线程，这就是 ptmalloc2。在只有一个分配区的基础上，增加了非主分配区 (non main arena)，主分配区只有一个，非主分配可以有很多个
> 当调用 malloc 分配内存的时候，会先查看当前线程私有变量中是否已经存在一个分配区 arena。如果存在，则尝试会对这个 arena 加锁如果加锁成功，则会使用这个分配区分配内存
> 如果加锁失败，说明有其它线程正在使用，则遍历 arena 列表寻找没有加锁的 arena 区域，如果找到则用这个 arena 区域分配内存。
> 主分配区可以使用 brk 和 mmap 两种方式申请虚拟内存，非主分配区只能 mmap。glibc 每次申请的虚拟内存区块大小是 64MB，glibc 再根据应用需要切割为小块零售。

这就是 linux 进程内存分布中典型的 64M 问题，那有多少个这样的区域呢？在 64 位系统下，这个值等于 8 \* number of cores，**如果是 4 核，则最多有 32 个 64M 大小的内存区域**。

glibc 从 2.11 开始对每个线程引入内存池，而我们使用的版本是 2.17，可以通过下面的命令查询版本号

```bash
# 查看 glibc 版本
ldd --version  
```

## 问题解决

通过服务器上一个参数 MALLOC\_ARENA\_MAX 可以控制最大的 arena 数量

```纯文本
export MALLOC_ARENA_MAX=1
```

由于我们使用的是 docker 容器，于是是在 docker 的启动参数上添加的。

容器重启后发现果然没有了 64M 的内存分配。

but RSS 依然还在增长，虽然这次的增长好像更慢了。于是再次 google 。(**事后在其他环境拉长时间观察，其实是有效的，短期内虽然有增长，但后面还会有回落**)

查询到可能是因为 glibc 的内存分配策略导致的碎片化内存回收问题，导致看起来像是内存泄露。那有没有更好一点的对碎片化内存的 malloc 库呢？业界常见的有 google 家的 tcmalloc 和 facebook 家的 jemalloc。

### tcmalloc

安装

```bash
yum install gperftools-libs.x86_64 
```

使用 LD\_PRELOAD 挂载

```bash
export LD_PRELOAD="/usr/lib64/libtcmalloc.so.4.4.5"
```

注意 java 应用要重启，经过我的测试使用 tcmalloc RSS 内存依然在涨，对我无效。

### jemalloc

安装

```bash
yum install epel-release  -y
yum install jemalloc -y
```

使用 LD\_PRELOAD 挂载

```bash
export LD_PRELOAD="/usr/lib64/libjemalloc.so.1"
```

使用 jemalloc 后，RSS 内存呈周期性波动，波动范围约 2 个百分点以内，基本控制住了。

### jemalloc 原理

与 tcmalloc 类似，每个线程同样在<32KB 的时候无锁使用线程本地 cache。

Jemalloc 在 64bits 系统上使用下面的 size-class 分类：

*   Small: \[8], \[16, 32, 48, …, 128], \[192, 256, 320, …, 512], \[768, 1024, 1280, …, 3840]

*   Large: \[4 KiB, 8 KiB, 12 KiB, …, 4072 KiB]

*   Huge: \[4 MiB, 8 MiB, 12 MiB, …]

small/large 对象查找 metadata 需要常量时间， huge 对象通过全局红黑树在对数时间内查找。

虚拟内存被逻辑上分割成 chunks（默认是 4MB，1024 个 4k 页），应用线程通过 round-robin 算法在第一次 malloc 的时候分配 arena， 每个 arena 都是相互独立的，维护自己的 chunks， chunk 切割 pages 到 small/large 对象。free() 的内存总是返回到所属的 arena 中，而不管是哪个线程调用 free()。

![](https://tva1.sinaimg.cn/large/008i3skNly1gxteyio7m2j30ao0ckaam.jpg)

上图可以看到每个 arena 管理的 arena chunk 结构， 开始的 header 主要是维护了一个 page map（1024 个页面关联的对象状态）， header 下方就是它的页面空间。 Small 对象被分到一起， metadata 信息存放在起始位置。 large chunk 相互独立，它的 metadata 信息存放在 chunk header map 中。

通过 arena 分配的时候需要对 arena bin（每个 small size-class 一个，细粒度）加锁，或 arena 本身加锁。并且线程 cache 对象也会通过垃圾回收指数退让算法返回到 arena 中。

![](https://tva1.sinaimg.cn/large/008i3skNly1gxteyo909zj30ft0ergmz.jpg)

### jemalloc tcmalloc 对比

![](https://tva1.sinaimg.cn/large/008i3skNly1gxteuqwnajj30k00eraak.jpg)

上图是服务器吞吐量分别用 6 个 malloc 实现的对比数据，可以看到 tcmalloc 和 jemalloc 最好 (facebook 在 2011 年的测试结果，tcmalloc 这里版本较旧）。

总结来看：在多线程环境使用 tcmalloc 和 jemalloc 效果非常明显。当线程数量固定，不会频繁创建退出的时候， 可以使用 jemalloc；反之使用 tcmalloc 可能是更好的选择。

## 总结

如果你观察到内存有这许多这种 64m 的分配，可能就踩到这个坑了，那么可以修改 MALLOC\_ARENA\_MAX ，然后耐心观察，不行的话，尝试使用 jemalloc 或 tcmalloc

## 参考

*   [http://www.cnhalo.net/2016/06/13/memory-optimize/](http://www.cnhalo.net/2016/06/13/memory-optimize/ "http://www.cnhalo.net/2016/06/13/memory-optimize/")

*   [https://tech.meituan.com/2019/01/03/spring-boot-native-memory-leak.html](https://tech.meituan.com/2019/01/03/spring-boot-native-memory-leak.html "https://tech.meituan.com/2019/01/03/spring-boot-native-memory-leak.html")

*   [https://www.heapdump.cn/article/1709425](https://www.heapdump.cn/article/1709425 "https://www.heapdump.cn/article/1709425")

*   [https://engineering.fb.com/2011/01/03/core-data/scalable-memory-allocation-using-jemalloc/](https://engineering.fb.com/2011/01/03/core-data/scalable-memory-allocation-using-jemalloc/ "https://engineering.fb.com/2011/01/03/core-data/scalable-memory-allocation-using-jemalloc/")
