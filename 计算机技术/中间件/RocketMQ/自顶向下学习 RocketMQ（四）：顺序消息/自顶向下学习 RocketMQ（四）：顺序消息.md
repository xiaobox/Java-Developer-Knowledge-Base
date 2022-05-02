# 自顶向下学习 RocketMQ（四）：顺序消息

## 顺序消息

顺序消息是消息队列 RocketMQ 提供的一种对消息发送和消费顺序有严格要求的消息。对于一个指定的 Topic，消息严格按照先进先出（FIFO）的原则进行消息发布和消费，即先发布的消息先消费，后发布的消息后消费。

顺序消息分为 `分区顺序消息` 和 `全局顺序消息`

### 分区顺序消息

> 分区顺序 对于指定的一个 Topic，所有消息根据 sharding key 进行区块分区。 同一个分区内的消息按照严格的 FIFO 顺序进行发布和消费。 Sharding key 是顺序消息中用来区分不同分区的关键字段，和普通消息的 Key 是完全不同的概念。 适用场景：性能要求高，以 sharding key 作为分区字段，在同一个区块中严格的按照 FIFO 原则进行消息发布和消费的场景。

*   Sharding Key：顺序消息中用来区分 Topic 中不同分区的关键字段，和普通消息的 Key 是完全不同的概念。消息队列 RocketMQ 会将设置了相同 Sharding Key 的消息路由到同一个分区下，同一个分区内的消息将按照消息发布顺序进行消费。

*   分区：即 Topic Partition，每个 Topic 包含一个或多个分区，Topic 中的消息会分布在这些不同的分区中。本文中的逻辑分区指的就是 Topic 的分区。

*   物理分区：区别于逻辑分区，消息实际存储的单元，每个物理分区都会分配到某一台机器指定节点上。

![](https://tva1.sinaimg.cn/large/008i3skNly1gxliflmmglj31lg0u0jur.jpg)

一般场景下，消息的发送顺序和消息生产的绝对时间顺序保持一致，生产者需要自己保证消息发送的顺序和生产顺序一致，建议使用单线程发送，若使用多线程发送消息，可能会造成消息发送顺序乱序。

Topic 中的每个逻辑分区可以对应多个物理分区，当消息按照顺序发送到 Topic 中的逻辑分区时，每个分区的消息将按照负载均匀的存储到对应的多个物理分区中，在物理分区中消息的存储可以不用保持顺序，但消息队列 RocketMQ 会记录消息在逻辑分区和物理分区中的映射关系及存储位置。

即使同一逻辑分区的消息被存储在不同的物理分区中且没有保持消息的顺序，消息队列 RocketMQ 服务端在投递消息时，最终还是会按照消息在逻辑队列中存储的顺序投递给 Consumer，Consumer 消费消息时，同一 Sharding Key 的消息使用单并发消费，保证消息消费顺序和存储顺序一致，最终实现消费顺序和发布顺序的一致

**适用场景**

适用于性能要求高，以 Sharding Key 作为分区字段，在同一个区块中严格地按照先进先出（FIFO）原则进行消息发布和消费的场景。

**业务例子**

*   用户注册需要发送验证码，以用户 ID 作为 Sharding Key，那么同一个用户发送的消息都会按照发布的先后顺序来消费。

*   电商的订单创建，以订单 ID 作为 Sharding Key，那么同一个订单相关的创建订单消息、订单支付消息、订单退款消息、订单物流消息都会按照发布的先后顺序来消费。

**示例**

代码依然使用 SpringCloud 整合 RocketMq 的方式

不同于之前文章中使用默认的 input output ，本例中我们使用自定义的方式来实现：

首先声明 Source 接口，注意这个接口不用实现，框架会有默认实现

```java
public interface MySource {

    @Output("output-order")
    MessageChannel output4Order();
}

```

在消息发送时引入

```java
@Autowired
private MySource mySource;
```

发送消息，消息内容为 order 实体：

```java
 // 发送 3 条相同 id 的消息
Long id = new Random().nextLong();
for (int i = 0; i < 3; i++) {
    // 创建 Message
    Order order = Order.builder().id(id).desc("test").build();

    Message message = MessageBuilder.createMessage(order, new MessageHeaders(headers));
    mySource.output4Order().send(message);
    System.out.println("发送了消息 " + message);
}
```

配置文件

```yaml
spring:
  mvc:
    throw-exception-if-no-handler-found: true # 处理 404 问题
  resources:
    add-mappings: false # 关闭 404 资源映射
  application:
    name: mq-example
  cloud:
    stream:
      bindings:
        # 定义 name 为 input 的 binding 消费
        input:
          content-type: application/json
          destination: test-topic
          group: consumer-group
        # 定义 name 为 output 的 binding 生产
        output-order:
          content-type: application/json
          destination: test-topic
          # Producer 配置项，对应 ProducerProperties 类
          producer:
            partition-key-expression: payload['id'] # 分区 key 表达式。该表达式基于 Spring EL，从消息中获得分区 key。
      rocketmq:
        # RocketMQ Binder 配置项，对应 RocketMQBinderConfigurationProperties 类
        binder:
          # 配置 rocketmq 的 nameserver 地址
          name-server: 127.0.0.1:9876
        bindings:
          output-order:
            # RocketMQ Producer 配置项，对应 RocketMQProducerProperties 类
            producer:
              group: producer-group # 生产者分组
              sync: true # 是否同步发送消息，默认为 false 异步。
          input:
            # RocketMQ Consumer 配置项，对应 RocketMQConsumerProperties 类
            consumer:
              enabled: true # 是否开启消费，默认为 true
              broadcasting: false # 是否使用广播消费，默认为 false 使用集群消费
              orderly: true # 是否顺序消费，默认为 false 并发消费。

```

为了方便这里我将生产和消息端的配置写在一起了，实际上生产的情况应该是分开的。

注意几点：

*   由于是顺序消息，producer 要配置成 同步发送

*   partition-key-expression，如果我们想从消息的 headers 中获得 Sharding key，可以设置为 headers\['partitionKey']

*   orderly 消费时要配置成顺序消费

最后可以输出一下结果，看一下线程 ID 和队列 ID

```java
 @StreamListener("input")
    public void receiveInput1(String receiveMsg, GenericMessage message, @Headers Map headers) {

        System.out.println(message.toString());
        System.out.println("线程 ID: " + Thread.currentThread().getId() + " 接受到消息 input receive: " + receiveMsg);
    }
```

![](https://tva1.sinaimg.cn/large/008i3skNly1gxllyc7okqj30o206mq4s.jpg)

列队 ID 相同，证明是顺序消费。

### 全局顺序消息

> 全局顺序 对于指定的一个 Topic，所有消息按照严格的先入先出（FIFO）的顺序进行发布和消费。 适用场景：性能要求不高，所有的消息严格按照 FIFO 原则进行消息发布和消费的场景

默认 Topic 对应多个队列，当设置 Topic 只有 1 个队列可以实现全局有序，创建 Topic 时手动设置。此类场景极少，性能差，通常不推荐使用。

**适用场景**

适用于性能要求不高，所有的消息严格按照 FIFO 原则来发布和消费的场景。

**示例**

在证券处理中，以人民币兑换美元为 Topic，在价格相同的情况下，先出价者优先处理，则可以按照 FIFO 的方式发布和消费全局顺序消息。

## 常见问题

### 同一条消息是否可以既是顺序消息，又是定时消息和事务消息？

不可以。顺序消息、定时消息、事务消息是不同的消息类型，三者是互斥关系，不能叠加在一起使用。

### 顺序消息支持哪种消息发送方式？

顺序消息只支持可靠同步发送方式，不支持异步发送方式，否则将无法严格保证顺序。

### 顺序消息是否支持集群消费和广播消费？

顺序消息暂时仅支持集群消费模式，不支持广播消费模式。

## 参考

*   [https://github.com/alibaba/spring-cloud-alibaba/tree/master/spring-cloud-alibaba-examples/rocketmq-example](https://github.com/alibaba/spring-cloud-alibaba/tree/master/spring-cloud-alibaba-examples/rocketmq-example "https://github.com/alibaba/spring-cloud-alibaba/tree/master/spring-cloud-alibaba-examples/rocketmq-example")

*   [https://spring-cloud-alibaba-group.github.io/github-pages/hoxton/zh-cn/index.html](https://spring-cloud-alibaba-group.github.io/github-pages/hoxton/zh-cn/index.html "https://spring-cloud-alibaba-group.github.io/github-pages/hoxton/zh-cn/index.html")

*   [https://www.iocoder.cn/Spring-Cloud-Alibaba/RocketMQ/?self](https://www.iocoder.cn/Spring-Cloud-Alibaba/RocketMQ/?self "https://www.iocoder.cn/Spring-Cloud-Alibaba/RocketMQ/?self")
