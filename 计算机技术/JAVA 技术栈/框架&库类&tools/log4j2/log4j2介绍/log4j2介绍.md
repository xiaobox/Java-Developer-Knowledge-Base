# log4j2介绍

## 为什么使用 Log4J2?

> Apache Log4j 2 is an upgrade to Log4j that provides significant improvements over its predecessor, Log4j 1.x, and provides many of the improvements available in Logback while fixing some inherent problems in Logback’s architecture.

Apache Log4j 2是 Log4j 的升级版，对 Log4j 1.x 进行了重大改进，并提供了 Logback 中可用的许多改进，同时解决了 Logback 体系结构中的一些固有问题。

### 原因

1.  重新配置时，Log4j 1.x 和 Logback 都将丢失事件。 Log4j 2不会。 在 Logback 中，`Appender` 中的异常对应用程序永远是不可见的。 在Log4j 2中，可以将 `Appender` 配置为允许异常渗透到应用程序中。

2.  Log4j 2 包含基于 LMAX Disruptor 库的下一代异步记录器。 在多线程方案中，与 Log4j 1.x 和 Logback 相比，异步 `Logger` 的吞吐量高10倍，延迟降低了几个数量级。

3.  对于独立应用程序来说，Log4j2 没有垃圾，而在稳定状态日志记录期间，web 应用程序的垃圾较少。这样可以减少垃圾收集器上的压力，并可以提供更好的响应时间性能。

4.  Log4j 2使用了一个插件系统，通过添加新的  `Appenders`、`Filters`、`Layouts`、`Lookups` 和 `Pattern Converters`，扩展框架非常容易，而不需要对 Log4j 进行任何更改。

5.  由于插件系统配置更简单。配置中的条目不需要指定类名

6.  支持自定义日志级别。自定义日志级别可以在代码或配置中定义。

7.  支持 lambda 表达式。运行在 Java 8 上的客户端代码只有在启用了请求的日志级别时，才可以使用 lambda表达式惰性地构造日志消息。不需要显式的级别检查，从而使代码更清晰。

8.  支持消息对象。消息支持通过日志系统传递有趣和复杂的构造，并能有效地进行操作。用户可以自由地创建自己的消息类型，并编写自定义 `Layouts`、`Filters` 和 `Lookups` 来操作它们。

9.  Log4j 1.x 支持 `Appender` 上的过滤器。 Logback 添加了 TurboFilter，以允许在 `Logger` 处理事件之前对其进行过滤。 Log4j 2支持过滤器，这些过滤器可以配置为在由记录器处理或在附加器上处理事件之前，由Logger 处理事件。

10. 许多 Logback appender 不接受 layout，只以固定格式发送数据。大多数 Log4j2 `Appender` 接受 `layout`，允许以所需的任何格式传输数据。

11. Log4j 1.x 和 Logback 中的 `layout` 返回一个字符串。 这导致了在Logback 编码器中讨论的问题。 Log4j 2采用了一种更简单的方法，即 `Layouts` 总是返回一个字节数组。这样做的好处是，这意味着它们实际上可以用于任何 `Appender`，而不仅仅是写入 OutputStream 的 `Appender`。

12. Syslog Appender 支持 TCP 和 UDP，也支持 BSD Syslog 和 RFC 5424 格式

13. Log4j 2 利用 Java 5 并发支持，并以最低级别执行锁定。 Log4j 1.x 已知死锁问题。 其中许多已在 Logback 中修复，但许多 Logback 类仍需要较高级别的同步。

14. 它是一个Apache Software Foundation 项目，遵循所有 ASF 项目使用的社区和支持模型。 如果您想贡献或获得提交更改的权利，请按照贡献中列出的路径进行操作。

### 详细解释

#### 针对上面原因中的第2点

更多信息可以查看官网：[https://logging.apache.org/log4j/2.x/performance.html](https://logging.apache.org/log4j/2.x/performance.html "https://logging.apache.org/log4j/2.x/performance.html")

Log4j 2 的异步 Logger 使用 无锁数据结构，而 Logback，Log4j 1.2 和 Log4j 2 的异步附加器使用 ArrayBlockingQueue。对于阻塞队列，多线程应用程序在尝试使日志事件入队时通常会遇到锁争用。

**异步日志记录-峰值吞吐量比较**
![](https://tva1.sinaimg.cn/large/e6c9d24ely1h1rt791jtpj20l10b2dgt.jpg)

#### 针对上面原因中的第7点

如果没有启用相应的日志级别，则可以避免对日志消息的计算，这可能会对使用日志记录的应用程序带来潜在的性能改进。

```纯文本
logger.trace("Number is {}", getRandomNumber());
```

上面这行日志代码中，我们调用了 `getRandomNumber()` 方法来替代日志参数，而不管是什么日志级别 `getRandomNumber()`  方法都会执行，比如日志级别是DEBUG，日志虽然不会被记录，但`getRandomNumber()`  方法会被执行。 换句话说，这个方法的执行可能是不必要的。

在添加对 lambda 表达式的支持之前，我们可以通过在执行 log 语句之前显式地检查日志级别来避免构造没有记录日志的消息:

```纯文本
if (logger.isTraceEnabled()) {
    logger.trace("Number is {}", getRandomNumer());
}
```

通过使用lambda表达式，我们可以进一步简化上面的代码:

```纯文本
logger.trace("Number is {}", () -> getRandomNumber());
```

只有当相应的日志级别被启用时，才会计算lambda表达式。这被称为惰性日志记录。

我们也可以在日志消息中使用多个 lambda 表达式:

```纯文本
logger.trace("Name is {} and age is {}", () -> getName(), () -> getRandomNumber());
```

如果使用了lombok 可以用 `@Log4j2` 注解，相当于

```java
 private static final org.apache.logging.log4j.Logger log = org.apache.logging.log4j.LogManager.getLogger(LogExample.class);

```

**但是**，如果你遵循阿里巴巴的编程规约就只能呵呵了，还是要用 `@Slf4j ` 这个注解。因为要统一使用 SLF4J 这个门面

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h1rt7r8e53j20zx0eoq5f.jpg)

## 同步还是异步？

Log4j 2 中记录日志的方式有**同步日志和异步日志**两种方式。

所谓**同步日志**，即当输出日志时，必须等待日志输出语句执行完毕后，才能执行后面的业务逻辑语句。比如下面这个配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration>

    <Properties>
        <!-- 日志输出级别 -->
        <Property name="LOG_INFO_LEVEL" value="info"/>
        <!-- error级别日志 -->
        <Property name="LOG_ERROR_LEVEL" value="error"/>
        <!-- 在当前目录下创建名为log目录做日志存放的目录 -->
        <Property name="LOG_HOME" value="./log"/>
        <!-- 档案日志存放目录 -->
        <Property name="LOG_ARCHIVE" value="./log/archive"/>
        <!-- 模块名称， 影响日志配置名，日志文件名，根据自己项目进行配置 -->
        <Property name="LOG_MODULE_NAME" value="spring-boot"/>
        <!-- 日志文件大小，超过这个大小将被压缩 -->
        <Property name="LOG_MAX_SIZE" value="100 MB"/>
        <!-- 保留多少天以内的日志 -->
        <Property name="LOG_DAYS" value="15"/>
        <!--输出日志的格式：%d表示日期，%thread表示线程名，%-5level：级别从左显示5个字符宽度， %msg：日志消息，%n是换行符 -->
        <Property name="LOG_PATTERN" value="%d [%t] %-5level %logger{0} - %msg%n"/>
        <!--interval属性用来指定多久滚动一次-->
        <Property name="TIME_BASED_INTERVAL" value="1"/>
    </Properties>

    <Appenders>
        <!-- 控制台输出 -->
        <Console name="STDOUT" target="SYSTEM_OUT">
            <!--输出日志的格式-->
            <PatternLayout pattern="${LOG_PATTERN}"/>
            <!--控制台只输出level及其以上级别的信息（onMatch），其他的直接拒绝（onMismatch）-->
            <ThresholdFilter level="${LOG_INFO_LEVEL}" onMatch="ACCEPT" onMismatch="DENY"/>
        </Console>

        <!-- 这个会打印出所有的info级别以上，error级别以下的日志，每次大小超过size或者满足TimeBasedTriggeringPolicy，则日志会自动存入按年月日建立的文件夹下面并进行压缩，作为存档-->
        <RollingRandomAccessFile name="RollingRandomAccessFileInfo"
                                 fileName="${LOG_HOME}/${LOG_MODULE_NAME}-infoLog.log"
                                 filePattern="${LOG_ARCHIVE}/${LOG_MODULE_NAME}-infoLog-%d{yyyy-MM-dd}-%i.log.gz">
            <Filters>
                <!--如果是error级别拒绝，设置 onMismatch="NEUTRAL" 可以让日志经过后续的过滤器-->
                <ThresholdFilter level="${LOG_ERROR_LEVEL}" onMatch="DENY" onMismatch="NEUTRAL"/>
                <!--如果是info\warn输出-->
                <ThresholdFilter level="${LOG_INFO_LEVEL}" onMatch="ACCEPT" onMismatch="DENY"/>
            </Filters>
            <PatternLayout pattern="${LOG_PATTERN}"/>
            <Policies>
                <!--interval属性用来指定多久滚动一次，根据当前filePattern设置是1天滚动一次-->
                <TimeBasedTriggeringPolicy interval="${TIME_BASED_INTERVAL}"/>
                <SizeBasedTriggeringPolicy size="${LOG_MAX_SIZE}"/>
            </Policies>
            <!-- DefaultRolloverStrategy属性如不设置，则默认同一文件夹下最多保存7个文件-->
            <DefaultRolloverStrategy max="${LOG_DAYS}"/>
        </RollingRandomAccessFile>

        <!--只记录error级别以上的日志，与info级别的日志分不同的文件保存-->
        <RollingRandomAccessFile name="RollingRandomAccessFileError"
                                 fileName="${LOG_HOME}/${LOG_MODULE_NAME}-errorLog.log"
                                 filePattern="${LOG_ARCHIVE}/${LOG_MODULE_NAME}-errorLog-%d{yyyy-MM-dd}-%i.log.gz">
            <Filters>
                <ThresholdFilter level="${LOG_ERROR_LEVEL}" onMatch="ACCEPT" onMismatch="DENY"/>
            </Filters>
            <PatternLayout pattern="${LOG_PATTERN}"/>
            <Policies>
                <TimeBasedTriggeringPolicy interval="${TIME_BASED_INTERVAL}"/>
                <SizeBasedTriggeringPolicy size="${LOG_MAX_SIZE}"/>
            </Policies>
            <DefaultRolloverStrategy max="${LOG_DAYS}"/>
        </RollingRandomAccessFile>

    </Appenders>

    <Loggers>
        <!-- 开发环境使用 -->
        <!--<Root level="${LOG_INFO_LEVEL}">
            <AppenderRef ref="STDOUT"/>
        </Root>-->

        <!-- 测试，生产环境使用 -->
        <Root level="${LOG_INFO_LEVEL}">
            <AppenderRef ref="RollingRandomAccessFileInfo"/>
            <AppenderRef ref="RollingRandomAccessFileError"/>
        </Root>
    </Loggers>

</Configuration>

```

通过`log.info(“是否为异步日志：{}”, AsyncLoggerContextSelector.isSelected());`可以查看是否为异步日志。

Log4j2 提供了两种实现异步日志的方式，一个是通过 `AsyncAppender`，一个是通过 `AsyncLogger`。

**官方推荐使用 Async Logger 的方式**

*   Async Appender。内部使用的一个队列（ArrayBlockingQueue）和一个后台线程，日志先存入队列，后台线程从队列中取出日志。阻塞队列容易受到锁竞争的影响，当更多线程同时记录时性能可能会变差。

*   Async Logger。是 ***log4j2 新增的功能。*** 内部使用的是 LMAX Disruptor 技术，Disruptor 是一个无锁的线程间通信库，它不是一个队列，不需要排队，从而产生更高的吞吐量和更低的延迟。

### Async Appender

Async Appender 是 `log4j2` 最开始的异步日志实现，它把其他 Appender 作为输入，然后把产生 `logEvent`输出到默认的容器 `ArrayBlockingQueue `中，然后使用另外一个线程中来输出日志以实现异步。

但是官方文档也指出：

> 在这种多线程应用的实践中需要主要：阻塞队列很容易发生锁争用，测试表明当大量线程并发写日志的时候，性能甚至会变得更糟糕。所以应该考虑使用 **无锁的 Asyn Loggers** 进行优化。

以下为官方例子：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="debug">
    <Appenders>
        <File name="TEMP" fileName="temp">
            <PatternLayout pattern="%r [%t] %p %c %notEmpty{%ndc }- %m%n"/>
        </File>
        <Async name="ASYNC">
            <AppenderRef ref="TEMP"/>
        </Async>
    </Appenders>
    <Loggers>
        <Root level="debug">
            <AppenderRef ref="ASYNC"/>
        </Root>
    </Loggers>
</Configuration>
```

上面就是一个使用 AsyncAppender 的典型配置，配置 AsyncAppender 后，日志事件写入文件的操作**将在单独的线程中执行**。

AsyncAppender 所支持的所有配置项以及其中每个配置项的作用：

| 名称                   | 类型                   | 描述                                                                                                  | 默认值                       |
| -------------------- | -------------------- | --------------------------------------------------------------------------------------------------- | ------------------------- |
| AppenderRef          | String               | 要异步调用的 Appender 的名称。 可以配置多个 AppenderRef 元素。                                                         |                           |
| blocking             | boolean              | 如果为 true，则 appender 将等待，直到队列中有空闲槽为止。 如果为 false，则在队列已满的情况下将事件写入 error appender。                      | true                      |
| shutdownTimeout      | integer              | Appender 在关闭时应等待多少毫秒来刷新队列中的未完成日志事件。 默认值为零。                                                          | 0(立刻关闭)                   |
| bufferSize           | integer              | 阻塞队列的最大容量，默认值为1024。请注意，在使用 disruptor-style 的 BlockingQueue 时，此缓冲区的大小必须为2的幂。                         | 1024                      |
| errorRef             | String               | 如果由于 appender 中的错误或队列已满而无法调用任何 appender，则要调用的 error appender 的名称。 如果未指定，则错误将被忽略。                    |                           |
| filter               | Filter               | 过滤器                                                                                                 |                           |
| name                 | String               | appender 的名称                                                                                        |                           |
| ignoreExceptions     | boolean              | 用于决定是否需要记录在日志事件处理过程中出现的异常                                                                           | true                      |
| BlockingQueueFactory | BlockingQueueFactory | Buffer的种类(默认ArrayBlockingQueue，能够支持DisruptorBlockingQueue，JCToolsBlockingQueue，LinkedTransferQueue) | ArrayBlockingQueueFactory |

### Async Logger

Log4j2 中的 AsyncLogger 的内部使用了 **Disruptor** 框架。Disruptor 是英国外汇交易公司 LMAX 开发的一个高性能队列，基于 Disruptor 开发的系统单线程能支撑每秒 600万订单。目前，包括 Apache Strom、Log4j2 在内的很多知名项目都应用了 Disruptor 来获取高性能。Disruptor 框架内部核心数据结构为 RingBuffer，其为无锁环形队列。

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h1rt9lseszj20au0agaa9.jpg)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="debug" name="MyApp" packages="">
    <Appenders>
        <Console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n" />
        </Console>
        <RollingFile name="RollingFile" fileName="logs/app.log"
                     filePattern="logs/app-%d{yyyy-MM-dd HH}.log">
            <PatternLayout>
                <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
            </PatternLayout>
            <Policies>
                <SizeBasedTriggeringPolicy size="500MB"/>
            </Policies>
        </RollingFile>
        <RollingFile name="RollingFile2" fileName="logs/app2.log"
                     filePattern="logs/app2-%d{yyyy-MM-dd HH}.log">
            <PatternLayout>
                <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
            </PatternLayout>
            <Policies>
                <SizeBasedTriggeringPolicy size="500MB"/>
            </Policies>
        </RollingFile>
    </Appenders>
    <Loggers>
        <AsyncLogger name="com.abc.Main" level="trace" additivity="false">
            <appender-ref ref="RollingFile"/>
        </AsyncLogger>
        <AsyncLogger name="RollingFile2" level="trace" additivity="false">
            <appender-ref ref="RollingFile2"/>
        </AsyncLogger>
        <Root level="debug">
            <AppenderRef ref="Console"/>
            <AppenderRef ref="RollingFile"/>
        </Root>
    </Loggers>
</Configuration>
```

在加载 log4j2.xml 的启动阶段，如果检测到配置了 AsyncRoot 或 AsyncLogger，将启动一个 disruptor 实例。

### 全局异步模式（性能最好，推荐）

JVM启动参数加上：

```xml
-DLog4jContextSelector=org.apache.logging.log4j.core.async.AsyncLoggerContextSelector
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- Don't forget to set system property
-Dlog4j2.contextSelector=org.apache.logging.log4j.core.async.AsyncLoggerContextSelector
to make all loggers asynchronous. -->
<Configuration status="WARN">
<Appenders>
    <!-- Async Loggers will auto-flush in batches, so switch off immediateFlush. -->
    <RandomAccessFile name="RandomAccessFile" fileName="async.log" immediateFlush="false" append="false">
        <PatternLayout>
            <Pattern>%d %p %c{1.} [%t] %m %ex%n</Pattern>
        </PatternLayout>
    </RandomAccessFile>
</Appenders>
<Loggers>
    <Root level="info" includeLocation="false">
        <AppenderRef ref="RandomAccessFile"/>
    </Root>
</Loggers>
</Configuration>
```

**当使用 `AsyncLoggerContextSelector` 来使所有记录器异步时，请确保使用配置中的普通  和 元素**

### 同步异步混合模式

可以在配置中组合同步和异步记录器。这提供了更大的灵活性，但代价是性能略有下降（与使所有记录器异步相比）。使用  或   配置元素指定需要异步的记录器。配置只能包含一个根记录器（  或  元素），但是可以组合异步和非异步记录器。例如，包含  元素的配置文件也可以包含  和同步记录器的元素。

默认情况下，异步记录器不会将位置传递给 I / O 线程。如果您的某个 layout 或自定义 filter 需要位置信息，则需要在所有相关记录器的配置中设置“includeLocation = true”，包括根记录器。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration>

    <Properties>
        <!-- 日志输出级别 -->
        <Property name="LOG_INFO_LEVEL" value="info"/>
        <!-- error级别日志 -->
        <Property name="LOG_ERROR_LEVEL" value="error"/>
        <!-- 在当前目录下创建名为log目录做日志存放的目录 -->
        <Property name="LOG_HOME" value="./log"/>
        <!-- 档案日志存放目录 -->
        <Property name="LOG_ARCHIVE" value="./log/archive"/>
        <!-- 模块名称， 影响日志配置名，日志文件名，根据自己项目进行配置 -->
        <Property name="LOG_MODULE_NAME" value="spring-boot"/>
        <!-- 日志文件大小，超过这个大小将被压缩 -->
        <Property name="LOG_MAX_SIZE" value="100 MB"/>
        <!-- 保留多少天以内的日志 -->
        <Property name="LOG_DAYS" value="15"/>
        <!--输出日志的格式：%d表示日期，%thread表示线程名，%-5level：级别从左显示5个字符宽度， %msg：日志消息，%n是换行符 -->
        <Property name="LOG_PATTERN" value="%d [%t] %-5level %logger{0} - %msg%n"/>
        <!--interval属性用来指定多久滚动一次-->
        <Property name="TIME_BASED_INTERVAL" value="1"/>
    </Properties>

    <Appenders>
        <!-- 控制台输出 -->
        <Console name="STDOUT" target="SYSTEM_OUT">
            <!--输出日志的格式-->
            <PatternLayout pattern="${LOG_PATTERN}"/>
            <!--控制台只输出level及其以上级别的信息（onMatch），其他的直接拒绝（onMismatch）-->
            <ThresholdFilter level="${LOG_INFO_LEVEL}" onMatch="ACCEPT" onMismatch="DENY"/>
        </Console>

        <!-- 这个会打印出所有的info级别以上，error级别一下的日志，每次大小超过size或者满足TimeBasedTriggeringPolicy，则日志会自动存入按年月日建立的文件夹下面并进行压缩，作为存档-->
        <!--异步日志会自动批量刷新，所以将immediateFlush属性设置为false-->
        <RollingRandomAccessFile name="RollingRandomAccessFileInfo"
                                 fileName="${LOG_HOME}/${LOG_MODULE_NAME}-infoLog.log"
                                 filePattern="${LOG_ARCHIVE}/${LOG_MODULE_NAME}-infoLog-%d{yyyy-MM-dd}-%i.log.gz"
                                 immediateFlush="false">
            <Filters>
                <!--如果是error级别拒绝，设置 onMismatch="NEUTRAL" 可以让日志经过后续的过滤器-->
                <ThresholdFilter level="${LOG_ERROR_LEVEL}" onMatch="DENY" onMismatch="NEUTRAL"/>
                <!--如果是info\warn输出-->
                <ThresholdFilter level="${LOG_INFO_LEVEL}" onMatch="ACCEPT" onMismatch="DENY"/>
            </Filters>
            <PatternLayout pattern="${LOG_PATTERN}"/>
            <Policies>
                <!--interval属性用来指定多久滚动一次，根据当前filePattern设置是1天滚动一次-->
                <TimeBasedTriggeringPolicy interval="${TIME_BASED_INTERVAL}"/>
                <SizeBasedTriggeringPolicy size="${LOG_MAX_SIZE}"/>
            </Policies>
            <!-- DefaultRolloverStrategy属性如不设置，则默认同一文件夹下最多保存7个文件-->
            <DefaultRolloverStrategy max="${LOG_DAYS}"/>
        </RollingRandomAccessFile>

        <!--只记录error级别以上的日志，与info级别的日志分不同的文件保存-->
        <RollingRandomAccessFile name="RollingRandomAccessFileError"
                                 fileName="${LOG_HOME}/${LOG_MODULE_NAME}-errorLog.log"
                                 filePattern="${LOG_ARCHIVE}/${LOG_MODULE_NAME}-errorLog-%d{yyyy-MM-dd}-%i.log.gz"
                                 immediateFlush="false">
            <Filters>
                <ThresholdFilter level="${LOG_ERROR_LEVEL}" onMatch="ACCEPT" onMismatch="DENY"/>
            </Filters>
            <PatternLayout pattern="${LOG_PATTERN}"/>
            <Policies>
                <TimeBasedTriggeringPolicy interval="${TIME_BASED_INTERVAL}"/>
                <SizeBasedTriggeringPolicy size="${LOG_MAX_SIZE}"/>
            </Policies>
            <DefaultRolloverStrategy max="${LOG_DAYS}"/>
        </RollingRandomAccessFile>

    </Appenders>

    <Loggers>
        <!-- 开发环境使用 -->
        <!--<Root level="${LOG_INFO_LEVEL}">
            <AppenderRef ref="STDOUT"/>
        </Root>-->

        <!-- 测试，生产环境使用 -->
        <!-- 当使用<asyncLogger> or <asyncRoot>时，无需设置系统属性"Log4jContextSelector" -->
        <AsyncLogger name="com.jourwon" level="${LOG_INFO_LEVEL}" additivity="false">
            <AppenderRef ref="RollingRandomAccessFileInfo"/>
            <AppenderRef ref="RollingRandomAccessFileError"/>
        </AsyncLogger>

        <Root level="${LOG_INFO_LEVEL}">
            <AppenderRef ref="RollingRandomAccessFileInfo"/>
            <AppenderRef ref="RollingRandomAccessFileError"/>
        </Root>
    </Loggers>

</Configuration>

```

## log4j2 配置文件详解

推荐看看这篇文章，写的挺全的：[https://mp.weixin.qq.com/s?\_\_biz=MzIwMzY1OTU1NQ==\&mid=2247488440\&idx=3\&sn=b2e84f2872780261d853ba6688117d7d\&chksm=96cd53f4a1badae26517bd371cb0196a19c5bfd2d2cd91c7f0261f69dc93552f0f24948137b1\&mpshare=1\&scene=24\&srcid=0408Brw78pw3Y879oNZZjT8T\&sharer\_sharetime=1617877802753\&sharer\_shareid=57b2967278387e4aa7e0b21737c8befa#rd](https://mp.weixin.qq.com/s?__biz=MzIwMzY1OTU1NQ==\&mid=2247488440\&idx=3\&sn=b2e84f2872780261d853ba6688117d7d\&chksm=96cd53f4a1badae26517bd371cb0196a19c5bfd2d2cd91c7f0261f69dc93552f0f24948137b1\&mpshare=1\&scene=24\&srcid=0408Brw78pw3Y879oNZZjT8T\&sharer_sharetime=1617877802753\&sharer_shareid=57b2967278387e4aa7e0b21737c8befa#rd "https://mp.weixin.qq.com/s?__biz=MzIwMzY1OTU1NQ==\&mid=2247488440\&idx=3\&sn=b2e84f2872780261d853ba6688117d7d\&chksm=96cd53f4a1badae26517bd371cb0196a19c5bfd2d2cd91c7f0261f69dc93552f0f24948137b1\&mpshare=1\&scene=24\&srcid=0408Brw78pw3Y879oNZZjT8T\&sharer_sharetime=1617877802753\&sharer_shareid=57b2967278387e4aa7e0b21737c8befa#rd")

另外，就是参考官网的文档了：[http://logging.apache.org/log4j/2.x/manual/layouts.html#PatternLayout](http://logging.apache.org/log4j/2.x/manual/layouts.html#PatternLayout "http://logging.apache.org/log4j/2.x/manual/layouts.html#PatternLayout")

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h1rtarm7suj20px0cht97.jpg)

### 两个模版文件

```xml
<?xml version="1.0" encoding="utf-8"?>
<!--日志级别以及优先级排序: OFF > FATAL > ERROR > WARN > INFO > DEBUG > TRACE > ALL -->
<!--Configuration后面的status，这个用于设置log4j2自身内部的信息输出，可以不设置，当设置成trace时，你会看到log4j2内部各种详细输出-->
 
<!--monitorInterval：Log4j能够自动检测修改配置 文件和重新配置本身，设置间隔秒数-->
<configuration status="INFO" monitorInterval="300">
    <!--定义属性-->
    <properties>
        <property name="log_deploy_path">logs/deploy</property>
        <property name="log_test_path">logs/test</property>
    </properties>
    <!--先定义所有的appender-->
    <appenders>
        <!--这个输出控制台的配置-->
        <Console name="Console" target="SYSTEM_OUT" follow="true">
            <!-- 控制台只输出level及以上级别的信息(onMatch),其他的直接拒绝(onMismatch) -->
            <ThresholdFilter level="debug" onMatch="ACCEPT" onMismatch="DENY"/>
            <!--输出日志的格式-->
            <PatternLayout pattern="%date{yyyy-MM-dd HH:mm:ss.SSS} %level [%thread][%file:%line] - %msg%n"/>
        </Console>
        <!--文件会打印出所有信息，这个log每次运行程序会自动清空，由append属性决定，这个也挺有用的，适合临时测试用-->
        <File name="test_log" fileName="${log_test_path}/testlog.html" append="false">
            <!--此处采用html文件格式进行整理-->
            <HTMLLayout title="日志"/>
            <ThresholdFilter level="info" onMatch="ACCEPT" onMismatch="DENY"/>
            <!--<PatternLayout pattern="%date{yyyy-MM-dd HH:mm:ss.SSS} %level [%thread][%file:%line] - %msg%n"/>-->
        </File>
        <!-- 这个会打印出所有的info及以下级别的信息，每次大小超过size，则这size大小的日志会自动存入按年份-月份建立的文件夹下面并进行压缩，作为存档-->
        <!--${sys:user.home}用户home目录-->
        <RollingFile name="RollingFileInfo" fileName="${log_deploy_path}/info.log"
                     filePattern="${log_deploy_path}/$${date:yyyy-MM}/info-%d{yyyy-MM-dd}-%i.log">
            <!--控制台只输出level及以上级别的信息（onMatch），其他的直接拒绝（onMismatch）-->
            <ThresholdFilter level="info" onMatch="ACCEPT" onMismatch="DENY"/>
            <PatternLayout pattern="[%d{HH:mm:ss:SSS}] [%p] - %l - %m%n"/>
            <Policies>
                <TimeBasedTriggeringPolicy/>
                <SizeBasedTriggeringPolicy size="100 MB"/>
            </Policies>
        </RollingFile>
        <RollingFile name="RollingFileWarn" fileName="${log_deploy_path}/warn.log"
                     filePattern="${log_deploy_path}/$${date:yyyy-MM}/warn-%d{yyyy-MM-dd}-%i.log">
            <ThresholdFilter level="warn" onMatch="ACCEPT" onMismatch="DENY"/>
            <PatternLayout pattern="[%d{HH:mm:ss:SSS}] [%p] - %l - %m%n"/>
            <Policies>
                <TimeBasedTriggeringPolicy/>
                <SizeBasedTriggeringPolicy size="100 MB"/>
            </Policies>
            <!-- DefaultRolloverStrategy属性如不设置，则默认为最多同一文件夹下7个文件，这里设置了20 -->
            <DefaultRolloverStrategy max="20"/>
        </RollingFile>
        <RollingFile name="RollingFileError" fileName="${log_deploy_path}/error.log"
                     filePattern="${log_deploy_path}/$${date:yyyy-MM}/error-%d{yyyy-MM-dd}-%i.log">
            <ThresholdFilter level="error" onMatch="ACCEPT" onMismatch="DENY"/>
            <PatternLayout pattern="[%d{HH:mm:ss:SSS}] [%p] - %l - %m%n"/>
            <Policies>
                <TimeBasedTriggeringPolicy/>
                <SizeBasedTriggeringPolicy size="100 MB"/>
            </Policies>
        </RollingFile>
    </appenders>
    <!--然后定义logger，只有定义了logger并引入的appender，appender才会生效-->
    <loggers>
        <!-- 过滤掉spring和mybatis的一些无用的DEBUG信息 -->
        <logger name="org.springframework.core" level="INFO">
        </logger>
        <logger name="org.springframework.beans" level="INFO">
        </logger>
        <logger name="org.springframework.context" level="INFO">
        </logger>
        <logger name="org.springframework.web" level="INFO">
        </logger>
        <logger name="org.apache.http" level="INFO">
        </logger>
        <logger name="org.mybatis" level="INFO">
        </logger>
        <!-- Root Logger -->
        <root level="all" includeLocation="true">
            <appender-ref ref="Console"/>
            <AppenderRef ref="test_log"/>
            <appender-ref ref="RollingFileInfo"/>
            <appender-ref ref="RollingFileWarn"/>
            <appender-ref ref="RollingFileError"/>
        </root>
    </loggers>
</configuration>
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!--日志级别以及优先级排序: OFF > FATAL > ERROR > WARN > INFO > DEBUG > TRACE > ALL -->
<!--Configuration后面的status,这个用于设置log4j2自身内部的信息输出,可以不设置,当设置成trace时,你会看到log4j2内部各种详细输出-->
<!--monitorInterval：Log4j能够自动检测修改配置 文件和重新配置本身,设置间隔秒数-->
<configuration status="WARN" monitorInterval="1800">

    <Properties>
        <!-- ==============================================公共配置============================================== -->
        <!-- 设置日志文件的目录名称 -->
        <property name="logFileName">qfxLog4jDemoLog</property>

        <!-- 日志默认存放的位置,可以设置为项目根路径下,也可指定绝对路径 -->
        <!-- 存放路径一:通用路径,window平台 -->
        <!-- <property name="basePath">d:/logs/${logFileName}</property> -->
        <!-- 存放路径二:web工程专用,java项目没有这个变量,需要删掉,否则会报异常,这里把日志放在web项目的根目录下 -->
        <!-- <property name="basePath">${web:rootDir}/${logFileName}</property> -->
        <!-- 存放路径三:web工程专用,java项目没有这个变量,需要删掉,否则会报异常,这里把日志放在tocmat的logs目录下 -->
        <property name="basePath">${sys:catalina.home}/logs/${logFileName}</property>

        <!-- 控制台默认输出格式,"%-5level":日志级别,"%l":输出完整的错误位置,是小写的L,因为有行号显示,所以影响日志输出的性能 -->
        <property name="console_log_pattern">%d{yyyy-MM-dd HH:mm:ss.SSS} [%-5level] %l - %m%n</property>
        <!-- 日志文件默认输出格式,不带行号输出(行号显示会影响日志输出性能);%C:大写,类名;%M:方法名;%m:错误信息;%n:换行 -->
        <!-- <property name="log_pattern">%d{yyyy-MM-dd HH:mm:ss.SSS} [%-5level] %C.%M - %m%n</property> -->
        <!-- 日志文件默认输出格式,另类带行号输出(对日志输出性能未知);%C:大写,类名;%M:方法名;%L:行号;%m:错误信息;%n:换行 -->
        <property name="log_pattern">%d{yyyy-MM-dd HH:mm:ss.SSS} [%-5level] %C.%M[%L line] - %m%n</property>

        <!-- 日志默认切割的最小单位 -->
        <property name="every_file_size">20MB</property>
        <!-- 日志默认输出级别 -->
        <property name="output_log_level">DEBUG</property>

        <!-- ===========================================所有级别日志配置=========================================== -->
        <!-- 日志默认存放路径(所有级别日志) -->
        <property name="rolling_fileName">${basePath}/all.log</property>
        <!-- 日志默认压缩路径,将超过指定文件大小的日志,自动存入按"年月"建立的文件夹下面并进行压缩,作为存档 -->
        <property name="rolling_filePattern">${basePath}/%d{yyyy-MM}/all-%d{yyyy-MM-dd-HH}-%i.log.gz</property>
        <!-- 日志默认同类型日志,同一文件夹下可以存放的数量,不设置此属性则默认为7个,filePattern最后要带%i才会生效 -->
        <property name="rolling_max">500</property>
        <!-- 日志默认同类型日志,多久生成一个新的日志文件,这个配置需要和filePattern结合使用;
                如果设置为1,filePattern是%d{yyyy-MM-dd}到天的格式,则间隔一天生成一个文件
                如果设置为12,filePattern是%d{yyyy-MM-dd-HH}到小时的格式,则间隔12小时生成一个文件 -->
        <property name="rolling_timeInterval">12</property>
        <!-- 日志默认同类型日志,是否对封存时间进行调制,若为true,则封存时间将以0点为边界进行调整,
                如:现在是早上3am,interval是4,那么第一次滚动是在4am,接着是8am,12am...而不是7am -->     
        <property name="rolling_timeModulate">true</property>

        <!-- ============================================Info级别日志============================================ -->
        <!-- Info日志默认存放路径(Info级别日志) -->
        <property name="info_fileName">${basePath}/info.log</property>
        <!-- Info日志默认压缩路径,将超过指定文件大小的日志,自动存入按"年月"建立的文件夹下面并进行压缩,作为存档 -->
        <property name="info_filePattern">${basePath}/%d{yyyy-MM}/info-%d{yyyy-MM-dd}-%i.log.gz</property>
        <!-- Info日志默认同一文件夹下可以存放的数量,不设置此属性则默认为7个 -->
        <property name="info_max">100</property>
        <!-- 日志默认同类型日志,多久生成一个新的日志文件,这个配置需要和filePattern结合使用;
                如果设置为1,filePattern是%d{yyyy-MM-dd}到天的格式,则间隔一天生成一个文件
                如果设置为12,filePattern是%d{yyyy-MM-dd-HH}到小时的格式,则间隔12小时生成一个文件 -->
        <property name="info_timeInterval">1</property>
        <!-- 日志默认同类型日志,是否对封存时间进行调制,若为true,则封存时间将以0点为边界进行调整,
                如:现在是早上3am,interval是4,那么第一次滚动是在4am,接着是8am,12am...而不是7am -->     
        <property name="info_timeModulate">true</property>

        <!-- ============================================Warn级别日志============================================ -->
        <!-- Warn日志默认存放路径(Warn级别日志) -->
        <property name="warn_fileName">${basePath}/warn.log</property>
        <!-- Warn日志默认压缩路径,将超过指定文件大小的日志,自动存入按"年月"建立的文件夹下面并进行压缩,作为存档 -->
        <property name="warn_filePattern">${basePath}/%d{yyyy-MM}/warn-%d{yyyy-MM-dd}-%i.log.gz</property>
        <!-- Warn日志默认同一文件夹下可以存放的数量,不设置此属性则默认为7个 -->
        <property name="warn_max">100</property>
        <!-- 日志默认同类型日志,多久生成一个新的日志文件,这个配置需要和filePattern结合使用;
                如果设置为1,filePattern是%d{yyyy-MM-dd}到天的格式,则间隔一天生成一个文件
                如果设置为12,filePattern是%d{yyyy-MM-dd-HH}到小时的格式,则间隔12小时生成一个文件 -->
        <property name="warn_timeInterval">1</property>
        <!-- 日志默认同类型日志,是否对封存时间进行调制,若为true,则封存时间将以0点为边界进行调整,
                如:现在是早上3am,interval是4,那么第一次滚动是在4am,接着是8am,12am...而不是7am -->     
        <property name="warn_timeModulate">true</property>

        <!-- ============================================Error级别日志============================================ -->
        <!-- Error日志默认存放路径(Error级别日志) -->
        <property name="error_fileName">${basePath}/error.log</property>
        <!-- Error日志默认压缩路径,将超过指定文件大小的日志,自动存入按"年月"建立的文件夹下面并进行压缩,作为存档 -->
        <property name="error_filePattern">${basePath}/%d{yyyy-MM}/error-%d{yyyy-MM-dd}-%i.log.gz</property>
        <!-- Error日志默认同一文件夹下可以存放的数量,不设置此属性则默认为7个 -->
        <property name="error_max">100</property>
        <!-- 日志默认同类型日志,多久生成一个新的日志文件,这个配置需要和filePattern结合使用;
                如果设置为1,filePattern是%d{yyyy-MM-dd}到天的格式,则间隔一天生成一个文件
                如果设置为12,filePattern是%d{yyyy-MM-dd-HH}到小时的格式,则间隔12小时生成一个文件 -->
        <property name="error_timeInterval">1</property>
        <!-- 日志默认同类型日志,是否对封存时间进行调制,若为true,则封存时间将以0点为边界进行调整,
                如:现在是早上3am,interval是4,那么第一次滚动是在4am,接着是8am,12am...而不是7am -->     
        <property name="error_timeModulate">true</property>

        <!-- ============================================控制台显示控制============================================ -->
        <!-- 控制台显示的日志最低级别 -->
        <property name="console_print_level">DEBUG</property>

    </Properties>

    <!--定义appender -->
    <appenders>
        <!-- =======================================用来定义输出到控制台的配置======================================= -->
        <Console name="Console" target="SYSTEM_OUT">
            <!-- 设置控制台只输出level及以上级别的信息(onMatch),其他的直接拒绝(onMismatch)-->
            <ThresholdFilter level="${console_print_level}" onMatch="ACCEPT" onMismatch="DENY"/>
            <!-- 设置输出格式,不设置默认为:%m%n -->
            <PatternLayout pattern="${console_log_pattern}"/>
        </Console>

        <!-- ================================打印root中指定的level级别以上的日志到文件================================ -->
        <RollingFile name="RollingFile" fileName="${rolling_fileName}" filePattern="${rolling_filePattern}">
            <PatternLayout pattern="${log_pattern}"/>
            <Policies>
                <TimeBasedTriggeringPolicy interval="${rolling_timeInterval}" modulate="${warn_timeModulate}"/>
                <SizeBasedTriggeringPolicy size="${every_file_size}"/>
            </Policies>
            <!-- 设置同类型日志,同一文件夹下可以存放的数量,如果不设置此属性则默认存放7个文件 -->
            <DefaultRolloverStrategy max="${rolling_max}" />
        </RollingFile>

        <!-- =======================================打印INFO级别的日志到文件======================================= -->
        <RollingFile name="InfoFile" fileName="${info_fileName}" filePattern="${info_filePattern}">
            <PatternLayout pattern="${log_pattern}"/>
            <Policies>
                <TimeBasedTriggeringPolicy interval="${info_timeInterval}" modulate="${info_timeModulate}"/>
                <SizeBasedTriggeringPolicy size="${every_file_size}"/>
            </Policies>
            <DefaultRolloverStrategy max="${info_max}" />
            <Filters>
                <ThresholdFilter level="WARN" onMatch="DENY" onMismatch="NEUTRAL"/>  
                <ThresholdFilter level="INFO" onMatch="ACCEPT" onMismatch="DENY"/>
            </Filters>
        </RollingFile>

        <!-- =======================================打印WARN级别的日志到文件======================================= -->
        <RollingFile name="WarnFile" fileName="${warn_fileName}" filePattern="${warn_filePattern}">
            <PatternLayout pattern="${log_pattern}"/>
            <Policies>
                <TimeBasedTriggeringPolicy interval="${warn_timeInterval}" modulate="${warn_timeModulate}"/>
                <SizeBasedTriggeringPolicy size="${every_file_size}"/>
            </Policies>
            <DefaultRolloverStrategy max="${warn_max}" />
            <Filters>
                <ThresholdFilter level="ERROR" onMatch="DENY" onMismatch="NEUTRAL"/>  
                <ThresholdFilter level="WARN" onMatch="ACCEPT" onMismatch="DENY"/>
            </Filters>
        </RollingFile>

        <!-- =======================================打印ERROR级别的日志到文件======================================= -->
        <RollingFile name="ErrorFile" fileName="${error_fileName}" filePattern="${error_filePattern}">
            <PatternLayout pattern="${log_pattern}"/>
            <Policies>
                <TimeBasedTriggeringPolicy interval="${error_timeInterval}" modulate="${error_timeModulate}"/>
                <SizeBasedTriggeringPolicy size="${every_file_size}"/>
            </Policies>
            <DefaultRolloverStrategy max="${error_max}" />
            <Filters>
                <ThresholdFilter level="FATAL" onMatch="DENY" onMismatch="NEUTRAL"/>  
                <ThresholdFilter level="ERROR" onMatch="ACCEPT" onMismatch="DENY"/>
            </Filters>
        </RollingFile>
    </appenders>

    <!--定义logger,只有定义了logger并引入的appender,appender才会生效-->
    <loggers>
        <!-- 设置打印sql语句配置开始,以下两者配合使用,可以优化日志的输出信息,减少一些不必要信息的输出 -->
        <!-- 设置java.sql包下的日志只打印DEBUG及以上级别的日志,此设置可以支持sql语句的日志打印 -->
        <logger name="java.sql" level="DEBUG" additivity="false">
            <appender-ref ref="Console"/>
        </logger>
        <!-- 设置org.mybatis.spring包下的日志只打印WARN及以上级别的日志 -->
        <logger name="org.mybatis.spring" level="WARN" additivity="false">
            <appender-ref ref="Console"/>
        </logger>
        <!-- 设置org.springframework包下的日志只打印WARN及以上级别的日志 -->
        <logger name="org.springframework" level="WARN" additivity="false">
            <appender-ref ref="Console"/>
        </logger>
        <!-- 设置com.qfx.workflow.service包下的日志只打印WARN及以上级别的日志 -->
        <logger name="com.qfx.workflow.service" level="WARN" additivity="false">
            <appender-ref ref="Console"/>
        </logger>
        <!-- 设置打印sql语句配置结束 -->

        <!--建立一个默认的root的logger-->
        <root level="${output_log_level}">
            <appender-ref ref="RollingFile"/>
            <appender-ref ref="Console"/>
            <appender-ref ref="InfoFile"/>
            <appender-ref ref="WarnFile"/>
            <appender-ref ref="ErrorFile"/>
        </root>
    </loggers>

</configuration>
```

## 参考：

*   [https://www.baeldung.com/log4j-2-lazy-logging](https://www.baeldung.com/log4j-2-lazy-logging "https://www.baeldung.com/log4j-2-lazy-logging")

*   [https://logging.apache.org/log4j/2.x/](https://logging.apache.org/log4j/2.x/ "https://logging.apache.org/log4j/2.x/")

*   [https://www.iocoder.cn/Spring-Boot/Logging/?self](https://www.iocoder.cn/Spring-Boot/Logging/?self "https://www.iocoder.cn/Spring-Boot/Logging/?self")

*   [https://bryantchang.github.io/2018/11/18/log4j-async/](https://bryantchang.github.io/2018/11/18/log4j-async/ "https://bryantchang.github.io/2018/11/18/log4j-async/")

*   [https://www.cnblogs.com/yeyang/p/7944906.html](https://www.cnblogs.com/yeyang/p/7944906.html "https://www.cnblogs.com/yeyang/p/7944906.html")

*   [https://my.oschina.net/u/1584802/blog/4644029](https://my.oschina.net/u/1584802/blog/4644029 "https://my.oschina.net/u/1584802/blog/4644029")
