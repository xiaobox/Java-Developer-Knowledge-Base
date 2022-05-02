# 自顶向下学习 RocketMQ（七）：事务消息

## 定义

> RocketMQ 事务消息（Transactional Message）是指应用本地事务和发送消息操作可以被定义到全局事务中，要么同时成功，要么同时失败。RocketMQ 的事务消息提供类似 X/Open XA 的分布事务功能，通过事务消息能达到分布式事务的最终一致。

## Demo

下面的例子，还是以 spring cloud stream 编程模型为基础，结合 spring cloud alibaba RocketMQ 的实现，演示了如何使用事务消息。

### 流程

事务消息交互流程如下图所示：

![](https://tva1.sinaimg.cn/large/008i3skNly1gy8lci85hdj31id0ggq4r.jpg)

事务消息发送步骤如下：

1.  生产者将半事务消息发送至消息队列 RocketMQ 版服务端。

2.  消息队列 RocketMQ 版服务端将消息持久化成功之后，向生产者返回 Ack 确认消息已经发送成功，此时消息为半事务消息。

3.  生产者开始执行本地事务逻辑。

4.  生产者根据本地事务执行结果向服务端提交二次确认结果（Commit 或是 Rollback），服务端收到确认结果后处理逻辑如下：

    *   二次确认结果为 Commit：服务端将半事务消息标记为可投递，并投递给消费者。

    *   二次确认结果为 Rollback：服务端不会将该消息投递给消费者，并按照如下逻辑进行回查处理。

事务消息回查步骤如下：

1.  在断网或者是生产者应用重启的特殊情况下，上述步骤 4 提交的二次确认最终未到达服务端，经过固定时间后，服务端将对消息生产者即生产者集群中任一生产者实例发起消息回查。

2.  生产者收到消息回查后，需要检查对应消息的本地事务执行的最终结果。

3.  生产者根据检查得到的本地事务的最终状态再次提交二次确认，服务端仍按照步骤 4 对半事务消息进行处理。

### 配置

和之前一样，我将生产者消息者配置在一起了，先来看一下配置文件：

```yaml
spring:
  application:
    name: mq-example
  cloud:
    stream:
      bindings:

        input-transaction:
          content-type: application/json
          destination: TransactionTopic
          group: transaction-consumer-group
       
        output-transaction:
          content-type: application/json
          destination: TransactionTopic

      rocketmq:
        # RocketMQ Binder 配置项，对应 RocketMQBinderConfigurationProperties 类
        binder:
          # 配置 rocketmq 的 nameserver 地址
          name-server: 127.0.0.1:9876
          group: rocketmq-group
        bindings:
          output-transaction:
            #  对应 RocketMQProducerProperties 类
            producer:
              producerType: Trans
              group: transaction-producer-group # 生产者分组
              transactionListener: myTransactionListener
         

```

这里需要注意生产者类型为：`Trans` 即，事务消息。

还配置了相应的生产者组和消费者组，这里回顾一下这两个概念

**消费者组：**`同一类` Producer 的集合，这类 Producer 发送同一类消息且发送逻辑一致。如果发送的是事务消息且原始生产者在发送之后崩溃，则 Broker 服务器会联系同一生产者组的其他生产者实例以提交或回溯消费。

**生产者组：**`同一类` Consumer 的集合，这类 Consumer 通常消费同一类消息且消费逻辑一致。消费者组使得在消息消费方面，实现`负载均衡`和`容错`的目标变得非常容易。要注意的是，消费者组的消费者实例必须订阅完全相同的 Topic。RocketMQ 支持两种消息模式：集群消费（Clustering）和广播消费（Broadcasting）。

### 实现

transactionListener 为我们自定义的事务监听器，具体代码见下文：

```java
@Component("myTransactionListener")
public class TransactionListenerImpl implements TransactionListener {

    @Override
    public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        Object num = msg.getProperty("test");

        if ("1".equals(num)) {
            System.out.println("executer: " + new String(msg.getBody()) + " unknown");
            return LocalTransactionState.UNKNOW;
        } else if ("2".equals(num)) {
            System.out.println("executer: " + new String(msg.getBody()) + " rollback");
            return LocalTransactionState.ROLLBACK_MESSAGE;
        }
        System.out.println("executer: " + new String(msg.getBody()) + " commit");
        return LocalTransactionState.COMMIT_MESSAGE;
    }

    @Override
    public LocalTransactionState checkLocalTransaction(MessageExt msg) {
        System.out.println("check: " + new String(msg.getBody()));
        return LocalTransactionState.COMMIT_MESSAGE;
    }

}

```

以上代码是参考官方的 Demo, 可以看到根据 `num` 的不同，返回不同的事务状态

*   如果 `num` 为 `1`，则返回 `UNKNOW`，表示本地事务状态未知，需要定期回查事务状态，则会执行 checkLocalTransaction 方法。

*   如果 `num` 为 `2`，则返回 `ROLLBACK_MESSAGE`，表示本地事务状态为回滚，则 broker 会回滚之前的提交的事务消息，即不投递消息。

*   如果 `num` 为 `3`，则返回 `COMMIT_MESSAGE`，表示本地事务状态为提交，则 broker 会投递消息。

发送消息跟之前的代码类似：

```java
 @GetMapping("/send_transaction")
    public void sendTransaction() {

        String msg = "这是一条事务消息";
        Integer num = 2;

        MessageBuilder builder = MessageBuilder.withPayload(msg)
                .setHeader(MessageHeaders.CONTENT_TYPE, MimeTypeUtils.APPLICATION_JSON);
        builder.setHeader("test", String.valueOf(num));
        builder.setHeader(RocketMQConst.USER_TRANSACTIONAL_ARGS, "binder");
        Message message = builder.build();
        mySource.outputTransaction().send(message);
    }
```

为了保证可靠消息最终一致性，需要有一个数据库表记录事务状态，

![](https://tva1.sinaimg.cn/large/008i3skNly1gy8ol2kum3j30u00hajt8.jpg)

事务开始的时候先将 UNKNOW 状态存起来，当事务异常时，返回 ROLLBACK\_MESSAGE 状态，并且在数据库表中记录此状态。当事务提交成功时，将状态修改为 COMMIT\_MESSAGE。

有了事务消息表，checkLocalTransaction 方法就可以依据此表进行事务状态的查询。

当然，如果按上图所示，一个完整的分布式事务跨越 A、B 两个系统，如果 B 系统事务失败回滚时，考虑 A 系统的事务是否需要回滚，如需要，还需要 A 系统提供回滚接口，供 B 系统调用。

## 参考

*   [https://www.alibabacloud.com/help/zh/doc-detail/43348.htm](https://www.alibabacloud.com/help/zh/doc-detail/43348.htm "https://www.alibabacloud.com/help/zh/doc-detail/43348.htm")

*   [https://www.iocoder.cn/Spring-Cloud-Alibaba/RocketMQ/](https://www.iocoder.cn/Spring-Cloud-Alibaba/RocketMQ/ "https://www.iocoder.cn/Spring-Cloud-Alibaba/RocketMQ/")
