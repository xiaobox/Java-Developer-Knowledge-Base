# 自顶向下学习 RocketMQ（六）：定时消息

## 定义和原理

**定时消息（延迟队列）** 是指消息发送到 broker 后，不会立即被消费，等待特定时间投递给真正的 topic。

broker 有配置项 messageDelayLevel，默认值为 `1s` `5s` `10s` `30s` `1m` `2m` `3m` `4m` `5m` `6m` `7m` `8m` `9m` `10m` `20m` `30m` `1h` `2h`  18 个 level , 可以配置自定义 messageDelayLevel。

注意，messageDelayLevel 是 broker 的属性，不属于某个 topic。发消息时，设置 delayLevel 等级即可：msg.setDelayLevel(level)。level 有以下三种情况：

*   level == 0，消息为非延迟消息

*   1<=level<=maxLevel，消息延迟特定时间，例如 level==1，延迟 1s

*   level > maxLevel，则 level== maxLevel，例如 level==20，延迟 2h

定时消息会暂存在名为 SCHEDULE\_TOPIC\_XXXX 的 topic 中，并根据 delayTimeLevel 存入特定的 queue，queueId = delayTimeLevel – 1，即一个 queue 只存相同延迟的消息，保证具有相同发送延迟的消息能够顺序消费。broker 会调度地消费 SCHEDULE\_TOPIC\_XXXX，将消息写入真实的 topic。

**RocketMQ 暂时不支持任意时间的定时**

简化一个实现原理方案示意图：

![](https://tva1.sinaimg.cn/large/008i3skNly1gy59zalb1bj30pa0a4q38.jpg)

分为两个部分：

*   消息的写入

*   消息的 Schedule

消息写入中：

1.  在写入 CommitLog 之前，如果是延迟消息，替换掉消息的 Topic 和 queueId（被替换为延迟消息特定的 Topic，queueId 则为延迟级别对应的 id)

2.  消息写入 CommitLog 之后，提交 dispatchRequest 到 DispatchService

3.  因为在第①步中 Topic 和 QueueId 被替换了，所以写入的 ConsumeQueue 实际上非真正消息应该所属的 ConsumeQueue，而是写入到 ScheduledConsumeQueue 中（这个特定的 Queue 存放不会被消费）

Schedule 过程中：

1.  给每个 Level 设置定时器，从 ScheduledConsumeQueue 中读取信息

2.  如果 ScheduledConsumeQueue 中的元素已近到时，那么从 CommitLog 中读取消息内容，恢复成正常的消息内容写入 CommitLog

3.  写入 CommitLog 后提交 dispatchRequest 给 DispatchService

4.  因为在写入 CommitLog 前已经恢复了 Topic 等属性，所以此时 DispatchService 会将消息投递到正确的 ConsumeQueue 中

## Demo

### 配置

由于 spring cloud alibaba 低版本的 rocketmq 定时消息功能有问题，不能实现，所以必须换高版本的，下面是我使用的版本信息：

```xml
<spring.boot.version>2.3.12.RELEASE</spring.boot.version>
<spring.cloud.version>Hoxton.SR12</spring.cloud.version>
<spring.cloud.alibaba.version>2.2.7.RELEASE</spring.cloud.alibaba.version>

```

引入的是 starter

```xml
 <dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-rocketmq</artifactId>
</dependency>
```

分享一下我的配置文件：

```yaml
spring:

  application:
    name: mq-example
  cloud:
    stream:
      bindings:
        # 定义 name 为 input 的 binding 消费
        input:
          content-type: application/json
          destination: test-topic3
          group: consumer-group
        # 定义 name 为 output 的 binding 生产
        output-order:
          content-type: application/json
          destination: test-topic3
          # Producer 配置项，对应 ProducerProperties 类
          #producer:
            # partition-key-expression: payload['id'] # 分区 key 表达式。该表达式基于 Spring EL，从消息中获得分区 key。
            #partitionCount: 3  # 分区数量
      rocketmq:
        # RocketMQ Binder 配置项，对应 RocketMQBinderConfigurationProperties 类
        binder:
          # 配置 rocketmq 的 nameserver 地址
          name-server: 127.0.0.1:9876
          group: rocketmq-group
        bindings:
          output-order:
            # RocketMQ Producer 配置项，对应 RocketMQProducerProperties 类
            producer:
              #group: producer-group # 生产者分组
              sync: true # 是否同步发送消息，默认为 false 异步。
          input:
            # RocketMQ Consumer 配置项，对应 RocketMQConsumerProperties 类
            consumer:
              #group: consumer-group # 消费者分组
              enabled: true # 是否开启消费，默认为 true
              broadcasting: false # 是否使用广播消费，默认为 false 使用集群消费
              orderly: false # 是否顺序消费，默认为 false 并发消费。

```

![](https://tva1.sinaimg.cn/large/008i3skNly1gy596dsyqgj30l90f2406.jpg)

这里注意红框部分在低版本有说要改成 true 才可以发送定时消息，我在高版本测试不用，true 和 false 都可以。

### 发送消息

消息发送部分几乎和之前一样，只是多加一了个 header `PROPERTY_DELAY_TIME_LEVEL`, 这里我写的是 2，即延迟 5 秒。

```java
   Map<String, Object> headers = Maps.newHashMapWithExpectedSize(16);
        headers.put(MessageConst.PROPERTY_DELAY_TIME_LEVEL, 2);
        headers.put(MessageConst.PROPERTY_TAGS, "test03");

        Order order = Order.builder().id(1L).desc("test").build();
        Message message = MessageBuilder.createMessage(order, new MessageHeaders(headers));
        mySource.output4Order().send(message);

```

### 接收消息

接收和之前没什么区别

```java
@Service
public class ReceiveService {

    /**
     * 订阅消息
     *
     * @param receiveMsg
     */
    @StreamListener("input")
    public void receiveInput1(String receiveMsg, GenericMessage message, @Headers Map headers) {

        System.out.println(message.toString());
        System.out.println("线程 ID: " + Thread.currentThread().getId() + " 接受到消息 input receive: " + receiveMsg);
    }

```

### 效果

```text
18:00:16.673 [http-nio-8080-exec-1] INFO   the message has sent, ......
18:00:21.685 [ConsumeMessageThread_1] INFO   接收到消息：{"id":1,"desc":"test"}
```

可以看到发送到接收，相距 5 秒。

## 应用场景

*   若生成订单 30 分钟未支付则自动取消

## 参考

*   [https://github.com/apache/rocketmq/blob/master/docs/cn/features.md](https://github.com/apache/rocketmq/blob/master/docs/cn/features.md "https://github.com/apache/rocketmq/blob/master/docs/cn/features.md")

*   [https://www.cnblogs.com/hzmark/p/mq-delay-msg.html](https://www.cnblogs.com/hzmark/p/mq-delay-msg.html "https://www.cnblogs.com/hzmark/p/mq-delay-msg.html")
