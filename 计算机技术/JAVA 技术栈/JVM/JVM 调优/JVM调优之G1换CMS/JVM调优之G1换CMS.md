# JVM调优之G1换CMS

# 线上 JVM 调优

## 问题发现

发现某应用的内存在缓慢的持续增长，系统告警内存使用系统已超过 80%，且正在持续增长。

具体来说是 RES 在增长。

你可以通过以下命令，在目标主机上查看内存情况：

```bash
ps -p 1 -o pcpu,rss,size,vsize
```

> RSS 是常驻内存集（Resident Set Size），表示该进程分配的内存大小。

> SIZE: 进程使用的地址空间，如果进程映射了 100M 的内存，进程的地址空间将报告为 100M 内存。事实上，这个大小不是一个程序实际使用的内存数。

> VSZ 表示进程分配的虚拟内存。包括进程可以访问的所有内存，包括进入交换分区的内容，以及共享库占用的内存。

## 问题分析

从表象分析，怀疑可能是内存泄露，且因为内存是缓慢增长，没有快速持续增长，所以倾向于怀疑不是堆内存泄露。

从监控数据来看，堆和非堆的内存占比都不是很高。于是进入容器内部查看 java 进程的内存情况。

我们使用的 JDK 版本是 AdoptOpenJDK 11.0.8 ，不是 oracle 的官方 JDK, 是 openJDK 的社区版本。

按理说，从 java9 以后默认的 GC 就是 G1 了，然而当我查看生产的 jdk 时，即是这样的：

```bash
java -XX:+PrintCommandLineFlags
```

```txt
-XX:InitialHeapSize=41943040 -XX:MaxHeapSize=671088640 -XX:+PrintCommandLineFlags -XX:ReservedCodeCacheSize=251658240 -XX:+SegmentedCodeCache -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseSerialGC 
```

**居然使用的是 SerialGC**

难以置信，于是我想到查看一下 java 进程中的 GC 参数：

```bash
jhsdb jmap --heap --pid 1
```

```txt
Attaching to process ID 1, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 11.0.8+10

using thread-local object allocation.
Mark Sweep Compact GC
```

得到了印证。Mark Sweep Compact GC，标记-清理-压缩算法。Serial GC (-XX:+UseSerialGC) 在老年代使用的算法。(也有可能是收集器退化导致的)

## 解决

说实话，我之前一直认为我们的 jdk 这个版本默认就是 G1 垃圾回收器，不配置也可以。但事实打脸了，于是我看了一下内存分配，给了 2G，那么其实就有问题了。

问题是到底应该用哪个 GC 的问题，并不是说 JDK9 以上无脑选择 G1 就是对的

![](https://tva1.sinaimg.cn/large/008i3skNly1gxnwzaefe3j30vk05qtbm.jpg)

![](https://tva1.sinaimg.cn/large/008i3skNly1gxnwzfq5auj30vc06ejuj.jpg)

以上出自《深入理解 Java 虚拟机》 第三版 作者：周志明

**根据我们的实际情况，内存在 2G 左右，我的选择更倾向于用 CMS。**

最后调整完的 jvm options 参数如下：

```纯文本
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
-XX:+AlwaysPreTouch
```

在解释其中的一些重要的参数之前，先分析一下 java 应用的内存组成：

```纯文本
Total memory = Heap + Code Cache + Metaspace + Symbol tables +
               Other JVM structures + Thread stacks +
               Direct buffers + Mapped files +
               Native Libraries + Malloc overhead + ...

```

可以看到，大体上分为堆和非堆两部分。

![](https://tva1.sinaimg.cn/large/008i3skNly1gxnxkori4oj30kx0h3jsc.jpg)

也可以通过命令查看具体java进程的内存情况：

```bash
jcmd 1 VM.native_memory
```

显示结果类似：

```txt
Native Memory Tracking:

Total: reserved=1847158KB, committed=1561194KB
-                 Java Heap (reserved=1048576KB, committed=1048576KB)
                            (mmap: reserved=1048576KB, committed=1048576KB) 
 
-                     Class (reserved=405345KB, committed=170849KB)
                            (classes #28273)
                            (  instance classes #26587, array classes #1686)
                            (malloc=8033KB #97026) 
                            (mmap: reserved=397312KB, committed=162816KB) 
                            (  Metadata:   )
                            (    reserved=143360KB, committed=142336KB)
                            (    used=138699KB)
                            (    free=3638KB)
                            (    waste=0KB =0.00%)
                            (  Class space:)
                            (    reserved=253952KB, committed=20480KB)
                            (    used=18340KB)
                            (    free=2140KB)
                            (    waste=0KB =0.00%)
 
-                    Thread (reserved=68673KB, committed=17205KB)
                            (thread #126)
                            (stack: reserved=68072KB, committed=16604KB)
                            (malloc=454KB #758) 
                            (arena=147KB #251)
 
-                      Code (reserved=136634KB, committed=136634KB)
                            (malloc=4538KB #15693) 
                            (mmap: reserved=132096KB, committed=132096KB) 
 
-                        GC (reserved=7511KB, committed=7511KB)
                            (malloc=3575KB #5347) 
                            (mmap: reserved=3936KB, committed=3936KB) 
 
-                  Compiler (reserved=1027KB, committed=1027KB)
                            (malloc=894KB #1383) 
                            (arena=133KB #5)
 
-                  Internal (reserved=18773KB, committed=18773KB)
                            (malloc=18741KB #7974) 
                            (mmap: reserved=32KB, committed=32KB) 
 
-                     Other (reserved=66199KB, committed=66199KB)
                            (malloc=66199KB #85) 
 
-                    Symbol (reserved=32192KB, committed=32192KB)
                            (malloc=28412KB #365132) 
                            (arena=3780KB #1)
 
-    Native Memory Tracking (reserved=8657KB, committed=8657KB)
                            (malloc=544KB #7707) 
                            (tracking overhead=8112KB)
 
-               Arena Chunk (reserved=2036KB, committed=2036KB)
                            (malloc=2036KB) 
 
-                   Logging (reserved=4KB, committed=4KB)
                            (malloc=4KB #191) 
 
-                 Arguments (reserved=18KB, committed=18KB)
                            (malloc=18KB #495) 
 
-                    Module (reserved=2706KB, committed=2706KB)
                            (malloc=2706KB #10716) 
 
-              Synchronizer (reserved=738KB, committed=738KB)
                            (malloc=738KB #6245) 
 
-                 Safepoint (reserved=8KB, committed=8KB)
                            (mmap: reserved=8KB, committed=8KB) 
 
-                   Unknown (reserved=48060KB, committed=48060KB)
                            (mmap: reserved=48060KB, committed=48060KB) 
```

### -XX:NativeMemoryTracking=detail

之所以有上面的显示结果，是因为我加了这个参数

### -XX:MetaspaceSize  深入理解堆外内存 Metaspace

具体到我的配置，为什么要设置 `-XX:MetaspaceSize=256M  -XX:MaxMetaspaceSize=256M` ，如下图所示：

![](https://tva1.sinaimg.cn/large/008i3skNly1gxnx9ha1rlj30j50200sq.jpg)

*   如果开启了-XX:+UseCompressedOops 及-XX:+UseCompressedClassesPointers（默认是开启），则 UseCompressedOops 会使用 32-bit 的 offset 来代表 java object 的引用，而 UseCompressedClassPointers 则使用 32-bit 的 offset 来代表 64-bit 进程中的 class pointer；可以使用 CompressedClassSpaceSize 来设置这块的空间大小

*   如果开启了指针压缩，则 CompressedClassSpace 分配在 MaxMetaspaceSize 里头，即 MaxMetaspaceSize=Compressed Class Space Size + Metaspace area (excluding the Compressed Class Space) Size

所以我们固定 metaspace 中的 compressed class space 大小，因为默认是 `1G`

### -Xss512k

堆栈大小由-Xss 控制。默认值为每个线程 1M ，调整成 512K.

### -XX:+AlwaysPreTouch

先要简单的了解一下，虽然通过 JVM 的参数-Xmx 和-Xms 可以设置 JVM 的堆大小，但是此时操作系统分配的只是虚拟内存，只有 JVM 真正要使用该内存时，才会被分配物理内存。

对象首先会先分配在年轻代，因为之前分配的只是虚拟内存，所以每次新建对象都需要操作系统来先分配物理内存，分配对象速度自然就降低了，只有等第一次新生代 GC 后，该被分配的内存空间都已经分配了，之后分配对象的速度才会加快。

那么老年代也是同理，老年代的空间何时真正使用，自然是对象需要晋升到老年代时，所以新生代 GC 的时候，对象要从新生代晋升到老年代，操作系统也需要为老年代先分配物理内存，这样就间接影响了新生代 GC 的效率。

而使用【-XX:+AlwaysPreTouch】参数能够达到的效果就是，**在服务启动的时候真实的分配物理内存给 JVM，而不再是虚拟内存，效果是可以加快代码运行效率，缺点也是有的，毕竟把分配物理内存的事提前放到 JVM 进程启动时做了，自然就会影响 JVM 进程的启动时间，导致启动时间降低几个数量级。**

### -XX:ReservedCodeCacheSize

*   \-XX:ReservedCodeCacheSize=128m

*   \-XX:InitialCodeCacheSize=128m

Code Cache 就是所谓的代码缓存，由于 JVM 虚拟机的内存默认是有大小限制的，因此代码缓存区域肯定也是有一定大小限制，默认为 240M，我的参数设置的数值是根据系统运行监控数据得出的结论，如果你要设置请根据实际情况设置，不要乱写。

另一方面，如果 code cacahe 满了不去管它不行，可以配置清理

`-XX:+UseCodeCacheFlushing`

> Code Cache 空间 如果满了，通过在启动参数上增加：-XX:+UseCodeCacheFlushing 来启用。打开这个选项，在 JIT 被关闭之前，也就是 CodeCache 装满之前，会在 JIT 关闭前做一次清理，删除一些 CodeCache 的代码；如果清理后还是没有空间，那么 JIT 依然会关闭。`这个选项默认是关闭的`

## 结局

当我将 G1 换成 CMS 后，内存的布局和大小都根据我的设置重置了。内存增长情况呈现趋势增长，但特别缓慢，且有周期性的节奏。

我也 dump 过内存快照，也曾怀疑过像 arthas 内部利用 JNI 导致的内存泄露，但还没有发现直接的证据，目前来看，还是先观察，在观察一定的周期之后根据情况继续分析。
