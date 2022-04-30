# 自顶向下学习 RocketMQ（十）：消息重投

# 消息重投

> 生产者在发送消息时，同步消息失败会重投，异步消息有重试，oneway 没有任何保证。消息重投保证消息尽可能发送成功、不丢失，但可能会造成消息重复，消息重复在 RocketMQ 中是无法避免的问题。消息重复在一般情况下不会发生，当出现消息量大、网络抖动，消息重复就会是大概率事件。另外，生产者主动重发、consumer 负载变化也会导致重复消息。如下方法可以设置消息重试策略：

*   retryTimesWhenSendFailed: 同步发送失败重投次数，默认为 2，因此生产者会最多尝试发送 retryTimesWhenSendFailed + 1 次。不会选择上次失败的>broker，尝试向其他 broker 发送，最大程度保证消息不丢。超过重投次数，抛出异常，由客户端保证消息不丢。当出现 RemotingException、>MQClientException 和部分 MQBrokerException 时会重投。

*   retryTimesWhenSendAsyncFailed: 异步发送失败重试次数，异步重试不会选择其他 broker，仅在同一个 broker 上做重试，不保证消息不丢。

*   retryAnotherBrokerWhenNotStoreOK: 消息刷盘（主或备）超时或 slave 不可用（返回状态非 SEND\_OK），是否尝试发送到其他 broker，默认 false。十分重要消息可以开启。

此外，只有 普通消息 具有发送重试机制，顺序消息是没有的。

### retryTimesWhenSendFailed

同步发送失败策略

```java
DefaultMQProducer producer = new DefaultMQProducer("pg"); 
producer.setNamesrvAddr("rocketmqOS:9876"); 
// 设置同步发送失败时重试发送的次数，默认为 2 次 
producer.setRetryTimesWhenSendFailed(3); 
// 设置发送超时时限为 5s，默认 3s 
producer.setSendMsgTimeout(5000);
```

在我们 Spring Cloud Stream + Spring Cloud Alibaba RocketMQ 的 例子的配置里，我们可以这样配置：

![](https://tva1.sinaimg.cn/large/008i3skNly1gyhzmx1n0zj30pg0osq66.jpg)

通过源码可以看到，它的默认值是 2：

![](https://tva1.sinaimg.cn/large/008i3skNly1gyhzkps8kfj30ls0cfmyu.jpg)

### retryTimesWhenSendAsyncFailed

异步发送失败策略

```java
DefaultMQProducer producer = new DefaultMQProducer("pg"); 
producer.setNamesrvAddr("rocketmqOS:9876"); 
// 指定异步发送失败后不进行重试发送 
producer.setRetryTimesWhenSendAsyncFailed(0);
```

在我们 Spring Cloud Stream + Spring Cloud Alibaba RocketMQ 的 例子的配置里，我们可以这样配置：

![](https://tva1.sinaimg.cn/large/008i3skNly1gyhzptr784j30nl0llju7.jpg)

通过源码可以看到，它的默认值也是 2：

![](https://tva1.sinaimg.cn/large/008i3skNly1gyhzqn64hcj30l90d40un.jpg)

### retryAnotherBrokerWhenNotStoreOK

消息刷盘失败策略

消息刷盘超时（ Master 、 Slave ），默认是不会将消息尝试发送到其他 Broker。对于重要消息可以通过在 Broker 的配置文件设置 retryAnotherBrokerWhenNotStoreOK 属性为 true 来开启。

在我们 Spring Cloud Stream + Spring Cloud Alibaba RocketMQ 的 例子的配置里，我们可以这样配置：

![](https://tva1.sinaimg.cn/large/008i3skNly1gyhzuhe4zwj30nd0l2go7.jpg)

## 消息重试

> Consumer 消费消息失败后，要提供一种重试机制，令消息再消费一次。Consumer 消费消息失败通常可以认为有以下几种情况：

*   由于消息本身的原因，例如反序列化失败，消息数据本身无法处理（例如话费充值，当前消息的手机号被注销，无法充值）等。这种错误通常需要跳过这条消息，再消费其它消息，而这条失败的消息即使立刻重试消费，99%也不成功，所以最好提供一种定时重试机制，即过 10 秒后再重试。

*   由于依赖的下游应用服务不可用，例如 db 连接不可用，外系统网络不可达等。遇到这种错误，即使跳过当前失败的消息，消费其他消息同样也会报错。这种情况建议应用 sleep 30s，再消费下一条消息，这样可以减轻 Broker 重试消息的压力。

> RocketMQ 会为每个消费组都设置一个 Topic 名称为“%RETRY%+consumerGroup”的重试队列（这里需要注意的是，这个 Topic 的重试队列是针对消费组，而不是针对每个 Topic 设置的），用于暂时保存因为各种异常而导致 Consumer 端无法消费的消息。考虑到异常恢复起来需要一些时间，会为重试队列设置多个重试级别，每个重试级别都有与之对应的重新投递延时，重试次数越多投递延时就越大。RocketMQ 对于重试消息的处理是先保存至 Topic 名称为“SCHEDULE\_TOPIC\_XXXX”的延迟队列中，后台定时任务按照对应的时间进行 Delay 后重新保存至“%RETRY%+consumerGroup”的重试队列中。

**消费者消费某条消息失败后，会根据消息重试机制将该消息重新投递，若达到重试次数后消息还没有成功被消费，则消息将被投入死信队列。一条消息无论重试多少次，这些重试消息的 Message ID 不会改变。**

### suspendCurrentQueueTimeMillis

同步消费（顺序消息）消息模式下消费失败后再次消费的时间间隔。 默认值：1000 ms

在我们 Spring Cloud Stream + Spring Cloud Alibaba RocketMQ 的 例子的配置里，我们可以这样配置：

![](https://tva1.sinaimg.cn/large/008i3skNly1gyi0975pwij30lh0jsdi6.jpg)

顺序消息的重试是无休止的，不间断的，直至消费成功，所以，对于顺序消息的消费， 务必要保证应用能够及时监控并处理消费失败的情况，避免消费被永久性阻塞。

**顺序消息没有发送失败重试机制，但具有消费失败重试机制**

### MaxReconsumeTimes

无序消息（包括普通消息、延时消息、定时消息和事务消息）的最大重试次数可通过自定义参数 MaxReconsumeTimes 取值进行配置。默认值为 16 次，该参数取值无最大限制，建议使用默认值。

间隔时间根据重试次数阶梯变化，取值范围：1 秒\~2 小时。不支持自定义配置。

![](https://tva1.sinaimg.cn/large/008i3skNly1gyi0q5hu9xj31150d2js6.jpg)

若最大重试次数小于等于 16 次，则间隔时间按照无序消息重试间隔时间阶梯变化。若最大重试次数大于 16 次，则超过 16 次的间隔时间均为 2 小时。

### delayLevelWhenNextConsume

异步消费消息模式下消费失败重试策略：

*   \-1, 不重复，直接放入死信队列

*   0,broker 控制重试策略

*   0,client 控制重试策略

默认值：0.

在我们 Spring Cloud Stream + Spring Cloud Alibaba RocketMQ 的 例子的配置里，我们可以这样配置：

![](https://tva1.sinaimg.cn/large/008i3skNly1gyi0lw7v3nj30k90jjq58.jpg)

## 死信队列

当一条消息初次消费失败，消息队列会自动进行消费重试；达到最大重试次数后，若消费依然失败，则表明消费者在正常情况下无法正确地消费该消息，此时，消息队列不会立刻将消息丢弃，而是将其发送到该消费者对应的特殊队列中。

正常情况下无法被消费的消息称为 死信消息（Dead-Letter Message），存储死信消息的特殊队列称为 死信队列（Dead-Letter Queue）。

对于 无序消息集群消费 下的重试消费，默认允许每条消息最多重试 16 次，如果消息重试 16 次后仍然失败，消息将被投递至 死信队列

### 特征

*   不会再被消费者正常消费

*   有效期与正常消息相同，均为 3 天，3 天后会被自动删除

### 特性

*   一个死信队列对应一个 Group ID， 而不是对应单个消费者实例。名称为 %DLQ%consumerGroup\@consumerGroup

*   如果一个 Group ID 未产生死信消息，则不会为其创建相应的死信队列

*   一个死信队列包含了对应 Group ID 产生的所有死信消息，不论该消息属于哪个 Topic

## 参考

*   [https://github.com/apache/rocketmq/blob/master/docs/cn/features.md](https://github.com/apache/rocketmq/blob/master/docs/cn/features.md "https://github.com/apache/rocketmq/blob/master/docs/cn/features.md")

*   [https://help.aliyun.com/document\_detail/43490.html#table-4i1-8kq-6vt](https://help.aliyun.com/document_detail/43490.html#table-4i1-8kq-6vt "https://help.aliyun.com/document_detail/43490.html#table-4i1-8kq-6vt")

*   [https://github.com/alibaba/spring-cloud-alibaba/blob/rocketmq/spring-cloud-alibaba-docs/src/main/asciidoc-zh/rocketmq-new.adoc](https://github.com/alibaba/spring-cloud-alibaba/blob/rocketmq/spring-cloud-alibaba-docs/src/main/asciidoc-zh/rocketmq-new.adoc "https://github.com/alibaba/spring-cloud-alibaba/blob/rocketmq/spring-cloud-alibaba-docs/src/main/asciidoc-zh/rocketmq-new.adoc")

*   [https://www.codeleading.com/article/57335926159/](https://www.codeleading.com/article/57335926159/ "https://www.codeleading.com/article/57335926159/")

*   [https://gitbook.cn/books/5d340810c43fe20aeadc88db/index.html](https://gitbook.cn/books/5d340810c43fe20aeadc88db/index.html "https://gitbook.cn/books/5d340810c43fe20aeadc88db/index.html")
