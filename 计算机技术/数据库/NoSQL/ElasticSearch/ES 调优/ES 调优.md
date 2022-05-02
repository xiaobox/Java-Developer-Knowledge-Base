# ES 调优

参考 ：[https://juejin.im/post/6844903762025250829](https://juejin.im/post/6844903762025250829 "https://juejin.im/post/6844903762025250829")

## swap（交换区）是性能终结者

这应该显而易见了，但仍然需要明确的写出来：把内存换成硬盘将毁掉服务器的性能，想象一下：涉及内存的操作是需要快速执行的。如果介质从内存变为了硬盘，一个10微秒的操作变成需要10毫秒。而且这种延迟发生在所有本该只花费10微秒的操作上，就不难理解为什么交换区对于性能来说是噩梦。
最好的选择是禁用掉操作系统的交换区。可以用以下命令：

```bash
sudo swapoff -a
```

建议把 `index.merge.scheduler.max_thread_count`: 1 索引 merge 最大线程数设置为 1 个，该参数可以有效调节写入的性能。因为在存储介质上并发写，由于寻址的原因，写入性能不会提升，只会降低。

```纯文本
https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-merge.html

Merge schedulingedit
The merge scheduler (ConcurrentMergeScheduler) controls the execution of merge operations when they are needed. Merges run in separate threads, and when the maximum number of threads is reached, further merges will wait until a merge thread becomes available.

The merge scheduler supports the following dynamic setting:

index.merge.scheduler.max_thread_count
The maximum number of threads on a single shard that may be merging at once. Defaults to Math.max(1, Math.min(4, <<node.processors, node.processors>> / 2)) which works well for a good solid-state-disk (SSD). If your index is on spinning platter drives instead, decrease this to 1.
```
