# 自顶向下学习 RocketMQ（二）：SpringCloud 整合 RocketMQ

## why

上文中我们讨论了 RocketMQ 的安装问题，有些重要的问题忘了说，即：

*   **为什么要用消息队列**

*   **消息队列有什么用？**

*   **用了消息队列有什么好处？**

来讨论为什么之前，先来看一下消息模型，即 “是什么？”，我们引用 RocketMQ 的消息模型：

> RocketMQ 主要由 Producer、Broker、Consumer 三部分组成，其中 Producer 负责生产消息，Consumer 负责消费消息，Broker 负责存储消息。Broker 在实际部署过程中对应一台服务器，每个 Broker 可以存储多个 Topic 的消息，每个 Topic 的消息也可以分片存储于不同的 Broker。Message Queue 用于存储消息的物理地址，每个 Topic 中的消息地址存储于多个 Message Queue 中。ConsumerGroup 由多个 Consumer 实例构成。

可以看到，主要是有 Producer、Broker、Consumer 这三部分。利用这样的模型，一般来说我们可以做到几点应用：

*   异步

*   解耦

*   削峰填谷

### 异步

相对同步，异步理论上执行的速度更快，效率更高，主线程执行完自己的逻辑然后发一条消息给消息队列就结束了，另外一个异步线程去订阅然后消费这条消息。

### 解耦

其实异步和解耦关系很密切，如果你把一个业务从同步的改成异步的了，实际上它从业务上就已经解耦了，至于形式上，无论你是用多个微服务进行拆分解耦，还是多个线程进行拆分解耦，都是解耦的。举个比较常见的例子，也是异步场景的，比如电商的用户下单购买后，增加消费积分场景：订单服务主业务逻辑结束后会给消息队列发一条增加消费积分的消息，下游营销服务会去订阅这条消息消费它，异步地执行增加积分的逻辑。

当然解耦后，系统间调用关系从大的业务上是同步的还是异步的，完全是由业务自己决定的。

### 削峰填谷

说白了，就是根据系统的处理能力来处理信息。在我们没有用消息队列之前，系统接收多少请求就都要处理完响应回去，处理的能力是根据单机或集群的处理能力而定，当然这是有上限的，虽然可以扩展，但可扩展的粒度比较粗： scale out 或 scale up，要么增加机器，要么扩展单机性能。如果我只是有某一类请求有问题处理不过来，其他的没啥事儿，这种扩展粒度就不太合适了。

利用消息队列，我们完全可以实现灵活的扩展，即将更细粒度的请求，解耦抽象为消息，发送到消息队列，让下游服务根据自身的能力去进消费。这里还涉及一个“流控”的问题，即像控制水流的流速一样，我们也要根据上下游系统及消息队列的处理能力来控制消息的流量，以 RocketMQ 为例：

*   生产者流控，因为 broker 处理能力达到瓶颈

*   消费者流控，因为消费能力达到瓶颈。

所谓削峰填谷，讲究的是一个平衡，这个平衡是系统和消息队列处理能力与消息流量之间的平衡。峰值的时候控一控让它下来，低谷的时候填一填让它上来，仅此而已。

## Spring Cloud Stream

聊完 why, 我们回到本文的正题，既然是 SpringCloud 的整合，那我们先聊一下 SpringCloud 吧。

Spring Cloud 体系内本身是有基于消息驱动的微服务框架的，即 Spring Cloud Stream。

![](https://tva1.sinaimg.cn/large/008i3skNly1gwvzyme1dsj30ei093q36.jpg)

> Spring Cloud Stream is a framework for building highly scalable event-driven microservices connected with shared messaging systems.
> The framework provides a flexible programming model built on already established and familiar Spring idioms and best practices, including support for persistent pub/sub semantics, consumer groups, and stateful partitions.

Spring Cloud Stream 提供了消息中间件配置的统一抽象，推出了 publish-subscribe、consumer groups、partition 这些统一的概念，有效的简化了上层研发人员对 MQ 使用的复杂度，让开发人员更多的精力投入到核心业务的处理。

**Spring Cloud Stream 解决什么问题？**

*   无感知的使用消息中间件
    Stream 解决了开发人员无感知的使用消息中间件的问题，因为 Stream 对消息中间件的进一步封装，可以做到代码层面对中间件的无感知。

*   中间件和服务的高度解耦
    Spring Cloud Stream 进行了配置隔离，只需要调整配置，开发中可以动态的切换中间件（如 rabbitmq 切换为 kafka)，使得微服务开发的高度解耦，服务可以关注更多自己的业务流程。

### 应用模型

![](https://tva1.sinaimg.cn/large/008i3skNly1gww5jr44nwj30ad08mdfz.jpg)

Spring Cloud Stream 由一个中立的中间件内核组成。Spring Cloud Stream 会注入输入和输出的 channels，应用程序通过这些 channels 与外界通信，而 channels 则是通过一个明确的中间件 Binder 与外部 brokers 连接。

列举下 Binder 的实现：

*   RabbitMQ

*   Apache Kafka

*   Amazon Kinesis

*   Google PubSub (partner maintained)

*   Solace PubSub+ (partner maintained)

*   Azure Event Hubs (partner maintained)

*   Apache RocketMQ (partner maintained)

另外还有一个概念叫 **Binding**,Binding 在消息中间件与应用程序提供的 Provider 和 Consumer 之间提供了一个桥梁，实现了开发者只需使用应用程序的 Provider 或 Consumer 生产或消费数据即可，屏蔽了开发者与底层消息中间件的接触。Binding 包括 Input Binding 和 Output Binding。

注意这里的 input 和 output 是站在生产者和消费者的角度，而不是 broker 的角度，如果你生产消息，那么对应使用的应该是 output, 如果你消费消息，那么对应使用的应该是 input。

![](https://tva1.sinaimg.cn/large/008i3skNly1gww5tm9060j30q608swf2.jpg)

## 接入 RocketMQ

### 依赖

使用 SpringCloud SpringBoot SpringCloud Alibaba 的组合，版本信息如下：

```xml
  <spring.boot.version>2.3.2.RELEASE</spring.boot.version>
  <spring.cloud.version>Hoxton.SR9</spring.cloud.version>
  <spring.cloud.alibaba.version>2.2.6.RELEASE</spring.cloud.alibaba.version>
```

依赖包

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-rocketmq</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### 配置

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
        # 定义 name 为 input 的 binding
        input:
          content-type: application/json
          destination: test-topic
          group: test-group
        # 定义 name 为 output 的 binding
        output:
          content-type: application/json
          destination: test-topic
      rocketmq:
        binder:
          # 配置 rocketmq 的 nameserver 地址
          name-server: 127.0.0.1:9876

```

### 生产&消费

**生产者 controller**

```java
 @GetMapping("/produce")
    public void produceMsg() {

        Map<String, Object> headers = new HashMap<>();
        headers.put(MessageConst.PROPERTY_TAGS, "test");
        Message message = MessageBuilder.createMessage("Hello RocketMQ!", new MessageHeaders(headers));
        output.send(message);
        System.out.println("发送了消息 "+message);

    }
```

**消费者订阅**

```java
@Service
public class ReceiveService {

    /**
     * 订阅消息
     * @param receiveMsg
     */
    @StreamListener("input")
    public void receiveInput1(String receiveMsg) {
        System.out.println(" 接受到消息 input receive: " + receiveMsg);
    }
}

```

### 测试

首先启动 RocketMQ(nameserver 和 broker) ，然后启动服务，调用 controller 接口 发送消息，查看接收到的消息内容。

也可以通过 dashboard 查看消息的接收状态。

![](https://tva1.sinaimg.cn/large/008i3skNly1gwx5que2nhj31gn0cj3zp.jpg)

## 参考

*   [https://spring.io/projects/spring-cloud-stream](https://spring.io/projects/spring-cloud-stream "https://spring.io/projects/spring-cloud-stream")

*   [https://github.com/alibaba/spring-cloud-alibaba/blob/master/spring-cloud-alibaba-examples/rocketmq-example/readme-zh.md](https://github.com/alibaba/spring-cloud-alibaba/blob/master/spring-cloud-alibaba-examples/rocketmq-example/readme-zh.md "https://github.com/alibaba/spring-cloud-alibaba/blob/master/spring-cloud-alibaba-examples/rocketmq-example/readme-zh.md")

*   [https://docs.spring.io/spring-cloud-stream/docs/3.1.5/reference/html/spring-cloud-stream.html#spring-cloud-stream-reference](https://docs.spring.io/spring-cloud-stream/docs/3.1.5/reference/html/spring-cloud-stream.html#spring-cloud-stream-reference "https://docs.spring.io/spring-cloud-stream/docs/3.1.5/reference/html/spring-cloud-stream.html#spring-cloud-stream-reference")

*   [https://www.cnblogs.com/binyue/p/12222198.html](https://www.cnblogs.com/binyue/p/12222198.html "https://www.cnblogs.com/binyue/p/12222198.html")
