# 百度 UidGenerator 源码解析

## 雪花算法

> 雪花算法（Snowflake）是一种生成分布式全局唯一 ID 的算法，生成的 ID 称为 Snowflake IDs 或 snowflakes。这种算法由 Twitter 创建，并用于推文的 ID。Discord 和 Instagram 等其他公司采用了修改后的版本。一个 Snowflake ID 有 64 位。前 41 位是时间戳，表示了自选定的时期以来的毫秒数。 接下来的 10 位代表计算机 ID，防止冲突。 其余 12 位代表每台机器上生成 ID 的序列号，这允许在同一毫秒内创建多个 Snowflake ID。SnowflakeID 基于时间生成，故可以按时间排序。此外，一个 ID 的生成时间可以由其自身推断出来，反之亦然。该特性可以用于按时间筛选 ID，以及与之联系的对象。

![](https://tva1.sinaimg.cn/large/008i3skNly1gw2w2zd1uzj312c0g20uj.jpg)

*   **第 1 位**

    该位不用主要是为了保持 ID 的自增特性，若使用了最高位，int64 会表示为负数。在 Java 中由于 long 类型的最高位是符号位，正数是 0，负数是 1，一般生成的 ID 为正整数，所以最高位为 0

*   **41 位时间戳**

    毫秒级的时间，一般实现上不会存储当前的时间戳，而是时间戳的差值（当前时间减去固定的开始时间），这样可以使产生的 ID 从更小值开始；

    41 bit 可以表示的数字多达 2^41 - 1，也就是可以标识 2 ^ 41 - 1 个毫秒值，换算成年就是表示 69 年的时间。

    (1L << 41) / (1000L 60 60 24 365) = (2199023255552 / 31536000000) ≈ 69.73 年。

*   **10 位工作机器 ID**

    Twitter 实现中使用前 5 位作为数据中心标识，后 5 位作为机器标识，可以部署 1024 （2^10）个节点。意思就是最多代表 2 ^ 5 个机房（32 个机房），每个机房里可以代表 2 ^ 5 个机器（32 台机器）。具体的分区可以根据自己的需要定义。比如拿出 4 位标识业务号，其他 6 位作为机器号。

*   **12 位序列号**

    支持同一毫秒内同一个节点可以生成 4096 （2^12）个 ID，也就是同一毫秒内同一台机器所生成的最大 ID 数量为 4096。

    简单来说，你的某个服务假设要生成一个全局唯一 id，那么就可以发送一个请求给部署了 SnowFlake 算法的系统，由这个 SnowFlake 算法系统来生成唯一 id。这个 SnowFlake 算法系统首先肯定是知道自己所在的机器号，（这里姑且讲 10bit 全部作为工作机器 ID）接着 SnowFlake 算法系统接收到这个请求之后，首先就会用二进制位运算的方式生成一个 64 bit 的 long 型 id，64 个 bit 中的第一个 bit 是无意义的。接着用当前时间戳（单位到毫秒）占用 41 个 bit，然后接着 10 个 bit 设置机器 id。最后再判断一下，当前这台机房的这台机器上这一毫秒内，这是第几个请求，给这次生成 id 的请求累加一个序号，作为最后的 12 个 bit。

**优点：**

*   理论上 Snowflake 方案的 QPS 约为 409.6w/s（1000 \* 2^12），这种分配方式可以保证在任何一个 IDC 的任何一台机器在任意毫秒内生成的 ID 都是不同的。

**缺点**

*   强依赖机器时钟，如果机器上时钟回拨，会导致发号重复或者服务会处于不可用状态。

## UidGenerator

> UidGenerator 是 Java 实现的，基于 Snowflake 算法的唯一 ID 生成器。UidGenerator 以组件形式工作在应用项目中，支持自定义 workerId 位数和初始化策略，从而适用于 docker 等虚拟化环境下实例自动重启、漂移等场景。 在实现上，UidGenerator 通过借用未来时间来解决 sequence 天然存在的并发限制；采用 RingBuffer 来缓存已生成的 UID, 并行化 UID 的生产和消费，同时对 CacheLine 补齐，避免了由 RingBuffer 带来的硬件级「伪共享」问题。最终单机 QPS 可达 600 万。

![](https://tva1.sinaimg.cn/large/008i3skNly1gw2xruqyzxj30og03kt90.jpg)

UidGenerator 的实现跟 SnowFlake 原始算法不太一样，不过**以下参数均可通过 Spring 进行自定义**：

*   sign(1bit)&#x20;

    固定 1bit 符号标识，即生成的 UID 为正数。

*   delta seconds (28 bits)

    当前时间，相对于时间基点"2016-05-20"的增量值，单位：秒，最多可支持约 8.7 年

*   worker id (22 bits)

    机器 id，最多可支持约 420w 次机器启动。内置实现为在启动时由数据库分配，默认分配策略为用后即弃，后续可提供复用策略。

*   sequence (13 bits)

    每秒下的并发序列，13 bits 可支持每秒 8192 个并发。

**RingBuffer** 环形数组，数组每个元素成为一个 slot。RingBuffer 容量，默认为 Snowflake 算法中 sequence 最大值，且为 2^N。可通过 boostPower 配置进行扩容，以提高 RingBuffer 读写吞吐量。

> 所谓环形缓冲，它是一种拥有读、写两个指针的数据复用结构，在计算机科学中有非常广泛的应用。举个具体例子，譬如一台计算机通过键盘输入，并通过 CPU 读取“HELLO WIKIPEDIA”这个长 14 字节的单词，通常需要一个至少 14 字节以上的缓冲区才行。但如果是环形缓冲结构，读取和写入就应当一起进行，在读取指针之前的位置均可以重复使用，理想情况下，只要读取指针不落后于写入指针一整圈，这个缓冲区就可以持续工作下去，能容纳无限多个新字符。否则，就必须阻塞写入操作去等待读取清空缓冲区。
>
> ![](http://icyfenix.cn/assets/img/Circular_Buffer_Animation.c3d3d834.gif)

**Tail 指针、Cursor 指针用于环形数组上读写 slot：**

*   Tail 指针

表示 Producer 生产的最大序号（此序号从 0 开始，持续递增）。Tail 不能超过 Cursor，即生产者不能覆盖未消费的 slot。当 Tail 已赶上 curosr，此时可通过 rejectedPutBufferHandler 指定 PutRejectPolicy

*   Cursor 指针

表示 Consumer 消费到的最小序号（序号序列与 Producer 序列相同）。Cursor 不能超过 Tail，即不能消费未生产的 slot。当 Cursor 已赶上 tail，此时可通过 rejectedTakeBufferHandler 指定 TakeRejectPolicy

CachedUidGenerator 采用了双 RingBuffer，Uid-RingBuffer 用于存储 Uid、Flag-RingBuffer 用于存储 Uid 状态 (**是否可填充、是否可消费**)

![](https://tva1.sinaimg.cn/large/008i3skNly1gw32tcajmqj30ko0e8js1.jpg)

由于数组元素在内存中是连续分配的，可最大程度利用 CPU cache 以提升性能。但同时会带来「伪共享」FalseSharing 问题，为此在 Tail、Cursor 指针、Flag-RingBuffer 中采用了 CacheLine 补齐方式。

![](https://tva1.sinaimg.cn/large/008i3skNly1gw32x0t4t3j310u0lamzl.jpg)

关于更多伪共享的知识，可以参考：[https://www.cnblogs.com/cyfonly/p/5800758.html](https://www.cnblogs.com/cyfonly/p/5800758.html， "https://www.cnblogs.com/cyfonly/p/5800758.html") [缓存一致性协议-MESI](../../../标准和协议/缓存一致性协议-MESI/缓存一致性协议-MESI.md "缓存一致性协议-MESI")

总结来说，伪共享会导致性能问题，解决了能提升性能，就算不解决也不会出现数据不一致等严重的问题。

**RingBuffer 填充时机**

*   初始化预填充

RingBuffer 初始化时，预先填充满整个 RingBuffer.

*   即时填充

Take 消费时，即时检查剩余可用 slot 量 (tail - cursor)，如小于设定阈值，则补全空闲 slots。阈值可通过 paddingFactor 来进行配置

*   周期填充

通过 Schedule 线程，定时补全空闲 slots。可通过 scheduleInterval 配置，以应用定时填充功能，并指定 Schedule 时间间隔

## 源码分析

### 概况

整个项目共 2386 行 java 代码 &#x20;

![](https://tva1.sinaimg.cn/large/008i3skNly1gw37caf8ohj30fw07laap.jpg)

代码内部 class 的依赖结构是这样的：

![](https://tva1.sinaimg.cn/large/008i3skNly1gw37eyy7obj31320lj77g.jpg)

可见 `RingBuffer` 是个核心类。

### 目录结构

```bash
com
  └── baidu
      └── fsg
          └── uid
              ├── BitsAllocator.java                  - Bit 分配器 (C)
              ├── UidGenerator.java                   - UID 生成的接口 (I)
              ├── buffer
              │   ├── BufferPaddingExecutor.java      - 填充 RingBuffer 的执行器 (C)
              │   ├── BufferedUidProvider.java        - RingBuffer 中 UID 的提供者 (C)
              │   ├── RejectedPutBufferHandler.java   - 拒绝 Put 到 RingBuffer 的处理器 (C)
              │   ├── RejectedTakeBufferHandler.java  - 拒绝从 RingBuffer 中 Take 的处理器 (C)
              │   └── RingBuffer.java                 - 内含两个环形数组 (C)
              ├── exception
              │   └── UidGenerateException.java       - 运行时异常
              ├── impl
              │   ├── CachedUidGenerator.java         - RingBuffer 存储的 UID 生成器 (C)
              │   └── DefaultUidGenerator.java        - 无 RingBuffer 的默认 UID 生成器 (C)
              ├── utils
              │   ├── DateUtils.java
              │   ├── DockerUtils.java
              │   ├── EnumUtils.java
              │   ├── NamingThreadFactory.java
              │   ├── NetUtils.java
              │   ├── PaddedAtomicLong.java
              │   └── ValuedEnum.java
              └── worker
                  ├── DisposableWorkerIdAssigner.java  - 用完即弃的 WorkerId 分配器 (C)
                  ├── WorkerIdAssigner.java            - WorkerId 分配器 (I)
                  ├── WorkerNodeType.java              - 工作节点类型 (E)
                  ├── dao
                  │   └── WorkerNodeDAO.java           - MyBatis Mapper
                  └── entity
                      └── WorkerNodeEntity.java        - MyBatis Entity
```

### DefaultUidGenerator

UidGenerator 在应用中是以 Spring 组件的形式提供服务，`DefaultUidGenerator` 提供了最简单的 Snowflake 式的生成模式，没有使用任何缓存来预存 UID，在需要生成 ID 的时候即时进行计算。

所以我们结合源码来串一下最简单的默认生成模式的流程。

首先引入`DefaultUidGenerator`配置

```xml
<!-- DefaultUidGenerator -->
<bean id="defaultUidGenerator" class="com.baidu.fsg.uid.impl.DefaultUidGenerator" lazy-init="false">
    <property name="workerIdAssigner" ref="disposableWorkerIdAssigner"/>

    <!--当前时间位数，前文图上是 28 位，这里设置 29 位 -->
    <property name="timeBits" value="29"/>
    <!-- 机器 id 位数，设置 21 位 -->
    <property name="workerBits" value="21"/>
    <!-- 每秒下的并发序列位数，13 bits 可支持每秒 8192 个并发 -->
    <property name="seqBits" value="13"/>
    <!-- 相对于时间基点"2016-09-20"的增量值，单位是秒 -->
    <!-- 用于计算时间戳的差值（当前时间减去固定的开始时间），这样可以使产生的 ID 从更小值开始-->
    <property name="epochStr" value="2016-09-20"/>
</bean>
 
<!-- 用完即弃的 WorkerIdAssigner，依赖 DB 操作 -->
<bean id="disposableWorkerIdAssigner" class="com.baidu.fsg.uid.worker.DisposableWorkerIdAssigner" />

```

配置文件中的配置我都作了注释，这里重点说两个属性：

**epochStr**

是给一个过去时间的字符串，作为时间基点，比如"2016-09-20"，用于计算时间戳的差值（当前时间减去固定的开始时间），这样可以使产生的 ID 从更小值开始。这点从以下的两段源码可以看出来：

```java
public void setEpochStr(String epochStr) {
    if (StringUtils.isNotBlank(epochStr)) {
        this.epochStr = epochStr;
        this.epochSeconds = TimeUnit.MILLISECONDS.toSeconds(DateUtils.parseByDayPattern(epochStr).getTime());
    }
}
 /**
  * Get current second
  */
private long getCurrentSecond() {
    long currentSecond = TimeUnit.MILLISECONDS.toSeconds(System.currentTimeMillis());
    if (currentSecond - epochSeconds > bitsAllocator.getMaxDeltaSeconds()) {
        throw new UidGenerateException("Timestamp bits is exhausted. Refusing UID generate. Now: " + currentSecond);
    }

    return currentSecond;
}

```

**disposableWorkerIdAssigner**

Worker ID 分配器，用于为每个工作机器分配一个唯一的 ID，目前来说是用完即弃，在初始化 Bean 的时候会自动向 MySQL 中插入一条关于该服务的启动信息，待 MySQL 返回其自增 ID 之后，使用该 ID 作为工作机器 ID 并柔和到 UID 的生成当中。

```java
@Transactional
public long assignWorkerId() {
    // build worker node entity
    WorkerNodeEntity workerNodeEntity = buildWorkerNode();

    // add worker node for new (ignore the same IP + PORT)
    workerNodeDAO.addWorkerNode(workerNodeEntity);
    LOGGER.info("Add worker node:" + workerNodeEntity);

    return workerNodeEntity.getId();
}

```

buildWorkerNode() 为获取该启动服务的信息，兼容 Docker 服务。但要注意，无论是 docker 还是用 k8s，需要添加相关的环境变量 env 在配置文件中以便程序能够获取到。

```java
/** Environment param keys 主要是端口和 host */
   private static final String ENV_KEY_HOST = "JPAAS_HOST";
   private static final String ENV_KEY_PORT = "JPAAS_HTTP_PORT";
```

**核心方法**

介绍完上面这些，我们来看下 `defaultUidGenerator` 生成 ID 的核心方法（注意这个方法是`同步`方法）

```java
protected synchronized long nextId() {

   long currentSecond = getCurrentSecond();

   // 时钟向后移动，拒绝生成 id （解决时钟回拨问题）
   if (currentSecond < lastSecond) {
       long refusedSeconds = lastSecond - currentSecond;
       throw new UidGenerateException("Clock moved backwards. Refusing for %d seconds", refusedSeconds);
   }

   // 如果是在同一秒内，那么增加 sequence
   if (currentSecond == lastSecond) {
       sequence = (sequence + 1) & bitsAllocator.getMaxSequence();
       // 如果超过了最大值，那么需要等到下一秒再进行生成
       if (sequence == 0) {
           currentSecond = getNextSecond(lastSecond);
       }

   // 不同秒的情况下，sequence 重新从 0 开始计数
   } else {
       sequence = 0L;
   }
   lastSecond = currentSecond;
   // Allocate bits for UID
   return bitsAllocator.allocate(currentSecond - epochSeconds, workerId, sequence);
}

```

可以看到大部分代码用来处理异常情况，比如时钟回拨问题，这里的做法比较简单，就是直接抛出异常。

最后一行才是根据传入的或计算好的参数进行 ID 的真正分配，通过二进制的移位和或运算得到最终的 long ID 值。

```java
public long allocate(long deltaSeconds, long workerId, long sequence) {

   return (deltaSeconds << timestampShift) | (workerId << workerIdShift) | sequence;
}
```

### CachedUidGenerator

`CachedUidGenerator` 是一个使用 `RingBuffer` 预先缓存 UID 的生成器，在初始化时就会填充整个 `RingBuffer`，并在 take() 时检测到少于指定的填充阈值之后就会异步地再次填充 `RingBuffer`（默认值为 50%），另外可以启动一个定时器周期性检测阈值并及时进行填充。

**RingBuffer**

上文提到 `RingBuffer` 是预先缓存 UID 的生成器，我们先看下它的成员变量情况：

```java
/** 常量配置 */
private static final int START_POINT = -1;
private static final long CAN_PUT_FLAG = 0L;
private static final long CAN_TAKE_FLAG = 1L;
public static final int DEFAULT_PADDING_PERCENT = 50;

/** RingBuffer 的 slot 的大小，每个 slot 持有一个 UID */
private final int bufferSize;
private final long indexMask;
/** 存 UID 的数组 */
private final long[] slots;
/** 存放 UID 状态的数组（是否可读或者可写，或是否可填充、是否可消费） */
private final PaddedAtomicLong[] flags;

/** Tail: 要产生的最后位置序列 */
private final AtomicLong tail = new PaddedAtomicLong(START_POINT);
/** Cursor: 要消耗的当前位置序列 */
private final AtomicLong cursor = new PaddedAtomicLong(START_POINT);

/** 触发填充缓冲区的阈值 */
private final int paddingThreshold; 

/** 放置 缓冲区的拒绝策略 拒绝方式为打印日志 */
private RejectedPutBufferHandler rejectedPutHandler = this::discardPutBuffer;
/** 获取 缓冲区的拒绝策略 拒绝方式为抛出异常并打印日志 */
private RejectedTakeBufferHandler rejectedTakeHandler = this::exceptionRejectedTakeBuffer; 

/** 填充缓冲区的执行者 */
private BufferPaddingExecutor bufferPaddingExecutor;
```

可以看到 `RingBuffer` 内部有两个环形数组，一个用来存放 UID，一个用来存放 UID 的状态，这两个数组的大小都是一样的，也就是 `bufferSize`。

slots 用于存放 UID 的 long 类型数组，flags 的用于存放读写标识的 PaddedAtomicLong 类型数组。这什么用 PaddedAtomicLong？ 上文有提到过`伪共享`的概念，这里就是为了解决这个问题，如果对 `伪共享` 还不太理解的朋友可以看一下上文的参考链接理解一下。

简单讲，由于 slots 实质是属于多读少写的变量，所以使用原生类型的收益更高。而 flags 则是会频繁进行写操作，为了避免伪共享问题所以手工进行补齐。

**RingBuffer 构造方法**

```java
/**
     * 具有缓冲区大小的构造函数，paddingFactor 默认为 {@value #DEFAULT_PADDING_PERCENT}
     * 
     * @param bufferSize 必须是正数和 2 的幂
     */
    public RingBuffer(int bufferSize) {
        this(bufferSize, DEFAULT_PADDING_PERCENT);
    }
    
    /**
     * 具有缓冲区大小和填充因子的构造函数
     * 
     * @param bufferSize 必须是正数和 2 的幂
     * @param paddingFactor (0 - 100) 中的百分比。当剩余可用的 UID 数量达到阈值时，将触发填充缓冲区
     *        Sample: paddingFactor=20, bufferSize=1000 -> threshold=1000 * 20 /100,
     *        当 tail-cursor<threshold 时将触发填充缓冲区
     */
    public RingBuffer(int bufferSize, int paddingFactor) {
        // check buffer size is positive & a power of 2; padding factor in (0, 100)
        Assert.isTrue(bufferSize > 0L, "RingBuffer size must be positive");
        Assert.isTrue(Integer.bitCount(bufferSize) == 1, "RingBuffer size must be a power of 2");
        Assert.isTrue(paddingFactor > 0 && paddingFactor < 100, "RingBuffer size must be positive");

        this.bufferSize = bufferSize;
        this.indexMask = bufferSize - 1;
        this.slots = new long[bufferSize];
        this.flags = initFlags(bufferSize);
        
        this.paddingThreshold = bufferSize * paddingFactor / 100;
    }

```

bufferSize 的默认值 ，如果 sequence 是 13 位，那么默认最大值是 8192，且是支持扩容的。

触发填充缓冲区的阈值也是支持配置的，

```xml
<!-- RingBuffer size 扩容参数，可提高 UID 生成的吞吐量。--> 
<!-- 默认：3， 原 bufferSize=8192, 扩容后 bufferSize= 8192 << 3 = 65536 -->
<property name="boostPower" value="3"></property>

<!-- 指定何时向 RingBuffer 中填充 UID, 取值为百分比 (0, 100), 默认为 50 -->
<!-- 举例：bufferSize=1024, paddingFactor=50 -> threshold=1024 * 50 / 100 = 512. -->
<!-- 当环上可用 UID 数量 < 512 时，将自动对 RingBuffer 进行填充补全 -->
<property name="paddingFactor" value="50"></property>

```

**RingBuffer 的填充和获取**

RingBuffer 的填充和获取操作是线程安全的，但是填充和获取操作的性能会受到 RingBuffer 的大小的影响，先来看下 `put` 操作：

```java
public synchronized boolean put(long uid) {
       long currentTail = tail.get();
       long currentCursor = cursor.get();

       // 当 tail 追上了 cursor 时，表示 RingBuffer 满了，不能再放了
       // Tail 不能超过 Cursor，即生产者不能覆盖未消费的 slot。当 Tail 已赶上 curosr，此时可通过 rejectedPutBufferHandler 指定 PutRejectPolicy
       long distance = currentTail - (currentCursor == START_POINT ? 0 : currentCursor);
       if (distance == bufferSize - 1) {
           rejectedPutHandler.rejectPutBuffer(this, uid);
           return false;
       }

       // 1. 预检查 flag 是否为 CAN_PUT_FLAG，首次 put 时，currentTail 为-1
       int nextTailIndex = calSlotIndex(currentTail + 1);
       if (flags[nextTailIndex].get() != CAN_PUT_FLAG) {
           rejectedPutHandler.rejectPutBuffer(this, uid);
           return false;
       }
       // 2. put UID in the next slot
       slots[nextTailIndex] = uid;
       // 3. update next slot' flag to CAN_TAKE_FLAG
       flags[nextTailIndex].set(CAN_TAKE_FLAG);
       // 4. publish tail with sequence increase by one 移动 tail
       tail.incrementAndGet();

       
       // 上述操作的原子性由“synchronized”保证。换句话说
       // take 操作不能消费我们刚刚放的 UID，直到 tail 移动 (tail.incrementAndGet()) 才可以
       return true;
   }
```

注意 put 方法是 synchronized。再来看下 `take` 方法：

UID 的读取是一个无锁的操作。在获取 UID 之前，还要检查是否达到了 padding 阈值，在另一个线程中会触发 padding buffer 操作，如果没有更多可用的 UID 可以获取，则应用指定的 RejectedTakeBufferHandler

```java
public long take() {
   // spin get next available cursor
   long currentCursor = cursor.get();
   // cursor 初始化为-1，现在 cursor 等于 tail，所以初始化时 nextCursor 为-1
   long nextCursor = cursor.updateAndGet(old -> old == tail.get() ? old : old + 1);

   // check for safety consideration, it never occurs
   // 初始化或者全部 UID 耗尽时 nextCursor == currentCursor
   Assert.isTrue(nextCursor >= currentCursor, "Curosr can't move back");

   // 如果达到阈值，则以异步模式触发填充
   long currentTail = tail.get();
   if (currentTail - nextCursor < paddingThreshold) {
       LOGGER.info("Reach the padding threshold:{}. tail:{}, cursor:{}, rest:{}", paddingThreshold, currentTail,
               nextCursor, currentTail - nextCursor);
       bufferPaddingExecutor.asyncPadding();
   }

   // cursor 追上 tail ，意味着没有更多可用的 UID 可以获取
   if (nextCursor == currentCursor) {
       rejectedTakeHandler.rejectTakeBuffer(this);
   }

   // 1. check next slot flag is CAN_TAKE_FLAG
   int nextCursorIndex = calSlotIndex(nextCursor);
   // 这个位置必须要是可以 TAKE
   Assert.isTrue(flags[nextCursorIndex].get() == CAN_TAKE_FLAG, "Curosr not in can take status");

   // 2. get UID from next slot
   // 3. set next slot flag as CAN_PUT_FLAG.
   long uid = slots[nextCursorIndex];
   // 告知 flags 数组这个位置是可以被重用了
   flags[nextCursorIndex].set(CAN_PUT_FLAG);

   // 注意：步骤 2，3 不能互换。如果我们在获取 slot 的值之前设置 flag，生产者可能会用新的 UID 覆盖 slot，这可能会导致消费者在一个 ring 中  获取 UID 两次
   return uid;
}
```

**BufferPaddingExecutor**

默认情况下，slots 被消费大于 50%的时候进行异步填充，这个填充由 BufferPaddingExecutor 所执行的，下面我们马上看看这个执行者的主要代码。

```java
/**
   * Padding buffer fill the slots until to catch the cursor
   * 该方法被即时填充和定期填充所调用
   */
public void paddingBuffer() {
   LOGGER.info("Ready to padding buffer lastSecond:{}. {}", lastSecond.get(), ringBuffer);

   // is still running
   // 这个是代表填充 executor 在执行，不是 RingBuffer 在执行。避免多个线程同时扩容。
   if (!running.compareAndSet(false, true)) {
       LOGGER.info("Padding buffer is still running. {}", ringBuffer);
       return;
   }

   // fill the rest slots until to catch the cursor
   boolean isFullRingBuffer = false;
   while (!isFullRingBuffer) {
       // 填充完指定 SECOND 里面的所有 UID，直至填满
       List<Long> uidList = uidProvider.provide(lastSecond.incrementAndGet());
       for (Long uid : uidList) {
           isFullRingBuffer = !ringBuffer.put(uid);
           if (isFullRingBuffer) {
               break;
           }
       }
   }

   // not running now
   running.compareAndSet(true, false);
   LOGGER.info("End to padding buffer lastSecond:{}. {}", lastSecond.get(), ringBuffer);
}
```

*   当线程池分发多条线程来执行填充任务的时候，成功抢夺运行状态的线程会真正执行对 RingBuffer 填充，直至全部填满，其他抢夺失败的线程将会直接返回。

*   该类还提供定时填充功能，如果有设置开关则会生效，默认不会启用周期性填充

*   RIngBuffer 的填充时机有 3 个：CachedUidGenerator 时对 RIngBuffer 初始化、RIngBuffer#take() 时检测达到阈值和周期性填充（如果有打开）

**使用 RingBuffer 的 UID 生成器**

最后我们看一下利用 CachedUidGenerator 生成 UID 的代码，CachedUidGenerator 继承了 DefaultUidGenerator，实现了 UidGenerator 接口。

该类在应用中作为 Spring Bean 注入到各个组件中，主要作用是初始化 RingBuffer 和 BufferPaddingExecutor。最重要的方法为 BufferedUidProvider 的提供者，即 lambda 表达式中的 nextIdsForOneSecond(long) 方法。

```java
/**
    * Get the UIDs in the same specified second under the max sequence
    * 
    * @param currentSecond
    * @return UID list, size of {@link BitsAllocator#getMaxSequence()} + 1
    */
   protected List<Long> nextIdsForOneSecond(long currentSecond) {
       // Initialize result list size of (max sequence + 1)
       int listSize = (int) bitsAllocator.getMaxSequence() + 1;
       List<Long> uidList = new ArrayList<>(listSize);

       // Allocate the first sequence of the second, the others can be calculated with the offset
       long firstSeqUid = bitsAllocator.allocate(currentSecond - epochSeconds, workerId, 0L);
       for (int offset = 0; offset < listSize; offset++) {
           uidList.add(firstSeqUid + offset);
       }

       return uidList;
   }
```

获取 ID 是通过委托 RingBuffer 的 take() 方法达成的

```java
@Override
public long getUID() {
    try {
        return ringBuffer.take();
    } catch (Exception e) {
        LOGGER.error("Generate unique id exception. ", e);
        throw new UidGenerateException(e);
    }
}
```

这里通过时序图再串一下 获取 id 的流程。

![](https://tva1.sinaimg.cn/large/008i3skNly1gw4aimxsuxj31430u0414.jpg)

## 几段位运算代码

### 判断是不是 2 的幂

利用 bitCount 函数，原理是计算参数传递的 int 值的二进制值有多少个 1，如果只有 1 个，则说明是 2 的幂，否则不是。

```java
Integer.bitCount(bufferSize) == 1
```

### 根据位数取最大值

```java
// initialize max value
this.maxDeltaSeconds = ~(-1L << timestampBits);
this.maxWorkerId = ~(-1L << workerIdBits);
this.maxSequence = ~(-1L << sequenceBits);
```

比较有意思，是用负 1 的二进制先左移再取反，比如看一下 13 和-1 的二进制值就明白了：

```java
System.out.println(Long.toBinaryString(-1L));
System.out.println(Long.toBinaryString(-1L << 13 ));
System.out.println(Long.toBinaryString( ~(-1L << 13) ));

输出
1111111111111111111111111111111111111111111111111111111111111111
1111111111111111111111111111111111111111111111111110000000000000
1111111111111
```

### parse UID

与上面的有异曲同之处。

```java
// parse UID
long sequence = (uid << (totalBits - sequenceBits)) >>> (totalBits - sequenceBits);
long workerId = (uid << (timestampBits + signBits)) >>> (totalBits - workerIdBits);
long deltaSeconds = uid >>> (workerIdBits + sequenceBits);
```

## 参考

*   [https://www.cnblogs.com/cyfonly/p/5800758.html](https://www.cnblogs.com/cyfonly/p/5800758.html "https://www.cnblogs.com/cyfonly/p/5800758.html")

*   [http://www.semlinker.com/uuid-snowflake/](http://www.semlinker.com/uuid-snowflake/ "http://www.semlinker.com/uuid-snowflake/")

*   [https://programmer.group/snowflake-algorithm-improved-version-snowflake.html](https://programmer.group/snowflake-algorithm-improved-version-snowflake.html "https://programmer.group/snowflake-algorithm-improved-version-snowflake.html")

*   [https://github.com/baidu/uid-generator/blob/master/README.zh\_cn.md](https://github.com/baidu/uid-generator/blob/master/README.zh_cn.md "https://github.com/baidu/uid-generator/blob/master/README.zh_cn.md")

*   [http://blog.chriscs.com/2017/08/02/baidu-uid-generator/](http://blog.chriscs.com/2017/08/02/baidu-uid-generator/ "http://blog.chriscs.com/2017/08/02/baidu-uid-generator/")

*   [http://icyfenix.cn/architect-perspective/general-architecture/diversion-system/cache-middleware.html](http://icyfenix.cn/architect-perspective/general-architecture/diversion-system/cache-middleware.html "http://icyfenix.cn/architect-perspective/general-architecture/diversion-system/cache-middleware.html")
