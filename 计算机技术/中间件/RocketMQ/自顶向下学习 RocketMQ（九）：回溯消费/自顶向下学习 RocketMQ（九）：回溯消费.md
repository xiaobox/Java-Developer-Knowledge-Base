# 自顶向下学习 RocketMQ（九）：回溯消费

## 定义

> 回溯消费是指 Consumer 已经消费成功的消息，由于业务上需求需要重新消费，要支持此功能，Broker 在向 Consumer 投递成功消息后，消息仍然需要保留。并且重新消费一般是按照时间维度，例如由于 Consumer 系统故障，恢复后需要重新消费 1 小时前的数据，那么 Broker 要提供一种机制，可以按照时间维度来回退消费进度。RocketMQ 支持按照时间回溯消费，时间维度精确到毫秒。

## Demo

我们仍然是利用 Spring Cloud Stream 的编程模型 + Spring Cloud Alibaba RocketMQ 来实现。

### 理论

在消费时，可以设置一个字段 ConsumeFromWhere（源码位置在：org.apache.rocketmq.common.consumer.ConsumeFromWhere），从哪开始消费。可选参数，去掉 Deprecated 的，剩下的就是

```java
public enum ConsumeFromWhere {
  CONSUME_FROM_LAST_OFFSET,
  CONSUME_FROM_FIRST_OFFSET,
  CONSUME_FROM_TIMESTAMP,
}
```

*   CONSUME\_FROM\_LAST\_OFFSET：从最后的偏移量开始消费

*   CONSUME\_FROM\_FIRST\_OFFSET：从最小偏移量开始消费

*   CONSUME\_FROM\_TIMESTAMP：从某个时间开始消费

我们需要设置从某个时间开始消费，即配置 `CONSUME_FROM_TIMESTAMP` 并设置好具体的时间点。

### 实现

首先还是看一下配置文件

```yaml
server:
  port: 8080
  servlet:
    context-path: /mq-example

spring:
 
  application:
    name: mq-example
  cloud:
    stream:
      bindings:

        input-backtracking:
          content-type: application/json
          destination: test-topic3
          group: backtracking-consumer-group

        # 定义 name 为 output 的 binding 生产
        output-order:
          content-type: application/json
          destination: test-topic3

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
          input-backtracking: # 回溯消息配置
            # com.alibaba.cloud.stream.binder.rocketmq.properties.RocketMQConsumerProperties
            consumer:
              consumeFromWhere: CONSUME_FROM_TIMESTAMP
              consumeTimestamp: 20220117110148
              enabled: true # 是否开启消费，默认为 true
              broadcasting: false # 是否使用广播消费，默认为 false 使用集群消费

```

这里我们仍然用之前的 ouput-order 作为生产者，生产消息。

消息者配置上主要注意 `input-backtracking` 节点中的属性配置：

*   consumeFromWhere 即上文提到的从哪儿开始消费，这里我们指定时间消费

*   consumeTimestamp 即指定的时间点

程序入口：

```java
@SpringBootApplication
@EnableBinding({ MySource.class})
public class MqBootstrapApplication {
    public static void main(String[] args) {
        SpringApplication.run(MqBootstrapApplication.class);
    }

}

```

要加上 `@EnableBinding`

MySource:

```java
public interface MySource {

    @Output("output-order")
    MessageChannel output4Order();

    @Input("input-backtracking")
    MessageChannel inputBackTracking();

}

```

controller 生产消息：

```java
@GetMapping("/produce")
    public void produceMsg() {

        Map<String, Object> headers = Maps.newHashMapWithExpectedSize(16);
        headers.put(MessageConst.PROPERTY_DELAY_TIME_LEVEL, 2);
        headers.put(MessageConst.PROPERTY_TAGS, "test03");

        Order order = Order.builder().id(1L).desc("test").build();
        Message message = MessageBuilder.createMessage(order, new MessageHeaders(headers));
        mySource.output4Order().send(message);

    }
```

ReceiveService 消费消息：

```java

@Service
@Log4j2
public class ReceiveService {

    @StreamListener("input-backtracking")
    public void receiveBackTrackingInput(String receiveMsg, GenericMessage message, @Headers Map headers) {

        log.info("接收到回溯消息：{}", receiveMsg);

    }

}

```

### 测试

可以先调用 controller 生产消息，或者不用 Demo 中的生产者生产消息，找一个之前发过消息的 topic , 看一下它的消息轨迹，找到存储时间

![](https://tva1.sinaimg.cn/large/008i3skNly1gygnui110fj31aq0d4wfo.jpg)

如果你用之前发过消息的 topic 记得修改配置文件中的 topic名称 ：

![](https://tva1.sinaimg.cn/large/008i3skNly1gygnz1spkoj30ef0b7gmd.jpg)

确认找到的这条消息已经被消费过（因为要测回溯，至少是二次消费），将 `consumeTimestamp` 的时间配置在 存储时间之后。

这时启动项目，观察 `ReceiveService` 的输出：

```纯文本
 接收到回溯消息：{"id":1,"desc":"test"}
```

证明消息回溯消费成功。

## 参考

*   [https://github.com/alibaba/spring-cloud-alibaba/blob/rocketmq/spring-cloud-alibaba-docs/src/main/asciidoc-zh/rocketmq-new.adoc](https://github.com/alibaba/spring-cloud-alibaba/blob/rocketmq/spring-cloud-alibaba-docs/src/main/asciidoc-zh/rocketmq-new.adoc "https://github.com/alibaba/spring-cloud-alibaba/blob/rocketmq/spring-cloud-alibaba-docs/src/main/asciidoc-zh/rocketmq-new.adoc")

*   [https://www.niewenjun.com/2020/05/09/fen-bu-shi/rocketmq/#toc-heading-29](https://www.niewenjun.com/2020/05/09/fen-bu-shi/rocketmq/#toc-heading-29 "https://www.niewenjun.com/2020/05/09/fen-bu-shi/rocketmq/#toc-heading-29")
