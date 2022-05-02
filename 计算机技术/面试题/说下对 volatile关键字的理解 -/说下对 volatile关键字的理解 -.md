# 说下对 volatile关键字的理解 ?

首先，需要了解 [JMM（Java Memory Model）](../../JAVA%20技术栈/JMM（Java%20Memory%20Model）/JMM（Java%20Memory%20Model）.md "JMM（Java Memory Model）")

`volatile` 是 JVM 提供的 **最轻量级的同步机制**。

`volatile` 的中文意思是不稳定的，易变的。

## volatile 变量的特性

`volatile` 变量具有两种特性：

*   保证变量对所有线程的**可见性**。

*   禁止进行指令重排序(**有序性**)

*   **不保证原子性**（也就是说多个线程并发修改某个变量时，依旧会产生多线程问题，但适合使用一个线程写，多个线程读的场合。）

## 实现原理

volatile 是依赖于硬件层面的支持，即需要 CPU 的指定来实现。

对于volatile修饰的变量，在汇编语言层面会多一行指令 `0x01a3de24: lock addl $0x0,(%esp);`。而该`lock`指令通过查[IA-32架构](https://www.intel.cn/content/www/cn/zh/architecture-and-technology/64-ia-32-architectures-software-developer-vol-1-manual.html "IA-32架构")可知主要做两件事 ：

1.  将当前处理器缓存行的数据写回到系统内存中。

2.  该写回内存操作会引起其他CPU里缓存了该内存地址的数据失效。 =》 CPU 的嗅探机制。

详细原理如下：

处理器为了提高处理速度，不直接和内存进行通讯，而是**先将系统内存的数据读到内部缓存（L1,L2 或其他）后再进行操作，但操作完之后不知道何时会写到内存**，如果对**声明了 \*\*\*\*\*\*\*\* 变量**进行写操作，**JVM 就会向处理器发送一条 Lock 前缀的指令**，将这个变量所在缓存行的数据写回到系统内存。

但是就算写回到内存，如果其他处理器缓存的值还是旧的，再执行计算操作就会有问题，所以在多处理器下，**为了保证各个处理器的缓存是一致的，就会实现缓存一致性协议**[缓存一致性协议-MESI](../../标准和协议/缓存一致性协议-MESI/缓存一致性协议-MESI.md "缓存一致性协议-MESI")，每个**处理器通过嗅探在总线上传播的数据来检查自己缓存的值是不是过期了**，当处理器发现自己缓存行对应的内存地址被修改，就会将当前处理器的缓存行设**置成无效状态**，当处理器要对这个数据进行修改操作的时候，会**强制重新从系统内存里把数据读到处理器缓存里**。

`volatile` 语义中的 内存屏障volatile的内存屏障策略非常严格保守，非常悲观且毫无安全感的心态：在每个volatile写操作前插入StoreStore屏障（在《[The JSR-133 Cookbook for Compiler Writers](http://gee.cs.oswego.edu/dl/jmm/cookbook.html "The JSR-133 Cookbook for Compiler Writers")》中，也很明确的指出了这一点），在写操作后插入StoreLoad屏障； 在每个volatile读操作前插入LoadLoad屏障，在读操作后插入LoadStore屏障； 由于内存屏障的作用，避免了volatile变量和其它指令重排序、线程之间实现了通信，使得volatile表现出了锁的特性。 &#x20;

## volatile 的使用场景

*   运算结果并不依赖变量的当前值，或者能够确保只有单一的线程修改变量的值

*   变量不需要与其他的状态变量共同参与不变约束

总结起来，就是“一次写入，到处读取”，某一线程负责更新变量，其他线程只读取变量(不更新变量)，并根据变量的新值执行相应逻辑。例如状态标志位更新，观察者模型变量值发布。

## 参考

*   [https://github.com/dunwu/javacore/blob/master/docs/concurrent/Java并发简介.md](https://github.com/dunwu/javacore/blob/master/docs/concurrent/Java并发简介.md "https://github.com/dunwu/javacore/blob/master/docs/concurrent/Java并发简介.md")

*   [https://www.infoq.cn/article/ftf-java-volatile](https://www.infoq.cn/article/ftf-java-volatile "https://www.infoq.cn/article/ftf-java-volatile")

*   [http://www.cs.umd.edu/\~pugh/java/memoryModel/jsr-133-faq.html#volatile](http://www.cs.umd.edu/\~pugh/java/memoryModel/jsr-133-faq.html#volatile "http://www.cs.umd.edu/\~pugh/java/memoryModel/jsr-133-faq.html#volatile")
