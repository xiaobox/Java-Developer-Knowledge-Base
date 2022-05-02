# free 命令

Free 命令相对于top 提供了更简洁的查看系统内存使用情况：

```bash
bash-3.00$ free
             total       used       free     shared    buffers     cached
Mem:       1572988    1509260      63728          0      62800     277888
-/+ buffers/cache:    1168572     404416
Swap:      2096472      16628    2079844
```

有时我们需要持续的观察内存的状况，此时可以使用 -s 选项并指定间隔的秒数：

```bash
$ free -h -s 3

```

*   Mem：表示物理内存统计。

    *   total 内存总数&#x20;

    *   used 已经使用的内存数

    *   free 空闲的内存数

    *   shared 当前已经废弃不用，总是0 &#x20;

    *   buffers Buffer Cache表示系统已经分配，但是没有被使用的buffer大小。 &#x20;

    *   cached Page Cache系统已经分配，但是没有被使用的cache大小。

*   \-/+ buffers/cached：表示物理内存的缓存统计 ,**-buffers/cache反映的是被程序实实在在吃掉的内存，而+buffers/cache反映的是可以挪用的内存总数**。

    *   \-buffers/cache 的内存数:等于第1行的 used - buffers - cached

    *   \+buffers/cache 的内存数:等于第1行的 free + buffers + cached

*   Swap：表示硬盘上交换分区的使用情况。只有mem被当前进程实际占用完,即没有了buffers和cache时，才会使用到swap。

### free 与 available 的区别

`free` 是真正尚未被使用的物理内存数量。`available` 是应用程序认为可用内存数量，`available = free + buffer + cache` (注：只是大概的计算方法)

### buffer 和 cache 的区别

cached 列表示当前的[页缓存（page cache）](../../../Linux%20内存管理/页缓存（page%20cache）/页缓存（page%20cache）.md "页缓存（page cache）")占用量，buffers 列表示当前的块缓存（buffer cache）占用量。用一句话来解释：Page Cache 用于缓存文件的页数据，buffer cache 用于缓存块设备（如磁盘）的块数据。页是逻辑上的概念，因此 Page Cache 是与文件系统同级的；块是物理上的概念，因此 buffer cache 是与块设备驱动程序同级的。

## 参考

*   [https://www.cnblogs.com/amilywilly/p/7953398.html](https://www.cnblogs.com/amilywilly/p/7953398.html "https://www.cnblogs.com/amilywilly/p/7953398.html")
