# 自顶向下学习 RocketMQ（五）：顺序消息原理

## 回顾

上文中我们对 RocketMQ 的 `顺序消息` 进行了 spring cloud 版本的演示，这里再回顾一下：

顺序消息分为 `分区顺序消息` 和 `全局顺序消息`。

全局顺序消息其实是分区顺序消息的一种特殊情况，即如果只有一个分区且同一时间只有一个消费者线程进行消费，那么就可以看作是全局顺序消息。

在 RocketMQ 创建 topic 时默认队列（分区）数量是：8 ，是针对所有 topic 的

![](https://tva1.sinaimg.cn/large/008i3skNly1gy1pygi7vsj313y0ocjwd.jpg)

如果要单独设置一个 topic 的队列（分区）数量，在 spring cloud alibaba 中可以这样配置：

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
          producer:
            partitionCount: 1  # 分区数量

```

注意这里 `partitionCount`，如将该值设置为 1，则 broker 会将消息发送到同一个分区中。

## 原理

本文我们重点了解一下，RocketMQ 的顺序消息的实现原理。

在 MQ 的模型中，顺序需要由 3 个阶段去保障：

*   消息被发送时保持顺序

*   消息被存储时保持和发送的顺序一致

*   消息被消费时保持和存储的顺序一致

**RocketMQ 要想实现顺序消息，核心就是 Producer 同步发送，确保一组顺序消息被发送到同一个分区队列，然后 Consumer 确保同一个队列只被一个线程消费**

### 发送有序

这里我们串一下代码，看一下 producer 发送消息的时候是怎么实现顺序发送的：

```java
private SendResult sendDefaultImpl(Message msg, CommunicationMode communicationMode, SendCallback sendCallback, long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
        this.makeSureStateOK();
        Validators.checkMessage(msg, this.defaultMQProducer);
        long invokeID = this.random.nextLong();
        long beginTimestampFirst = System.currentTimeMillis();
        long beginTimestampPrev = beginTimestampFirst;
        TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
        if (topicPublishInfo != null && topicPublishInfo.ok()) {
            boolean callTimeout = false;
            MessageQueue mq = null;
            Exception exception = null;
            SendResult sendResult = null;
            int timesTotal = communicationMode == CommunicationMode.SYNC ? 1 + this.defaultMQProducer.getRetryTimesWhenSendFailed() : 1;
            int times = 0;
            String[] brokersSent = new String[timesTotal];

            while(true) {
                label122: {
                    String info;
                    if (times < timesTotal) {
                        info = null == mq ? null : mq.getBrokerName();
                        MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, info);
                        if (mqSelected != null) {
                            mq = mqSelected;
                            brokersSent[times] = mqSelected.getBrokerName();

                            long endTimestamp;
                            try {
                                beginTimestampPrev = System.currentTimeMillis();
                                long costTime = beginTimestampPrev - beginTimestampFirst;
                                if (timeout >= costTime) {
                                    sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout - costTime);
                                    endTimestamp = System.currentTimeMillis();
                                    this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
                                    switch(communicationMode) {
                                    case ASYNC:
                                        return null;
                                    case ONEWAY:
                                        return null;
                                    case SYNC:
                                        if (sendResult.getSendStatus() == SendStatus.SEND_OK || !this.defaultMQProducer.isRetryAnotherBrokerWhenNotStoreOK()) {
                                            return sendResult;
                                        }
                                    default:
                                        break label122;
                                    }
                                }

                                callTimeout = true;
                            } catch (RemotingException var26) {
                                endTimestamp = System.currentTimeMillis();
                                this.updateFaultItem(mqSelected.getBrokerName(), endTimestamp - beginTimestampPrev, true);
                                this.log.warn(String.format("sendKernelImpl exception, resend at once, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mqSelected), var26);
                                this.log.warn(msg.toString());
                                exception = var26;
                                break label122;
                            } catch (MQClientException var27) {
                                endTimestamp = System.currentTimeMillis();
                                this.updateFaultItem(mqSelected.getBrokerName(), endTimestamp - beginTimestampPrev, true);
                                this.log.warn(String.format("sendKernelImpl exception, resend at once, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mqSelected), var27);
                                this.log.warn(msg.toString());
                                exception = var27;
                                break label122;
                            } catch (MQBrokerException var28) {
                                endTimestamp = System.currentTimeMillis();
                                this.updateFaultItem(mqSelected.getBrokerName(), endTimestamp - beginTimestampPrev, true);
                                this.log.warn(String.format("sendKernelImpl exception, resend at once, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mqSelected), var28);
                                this.log.warn(msg.toString());
                                exception = var28;
                                switch(var28.getResponseCode()) {
                                case 1:
                                case 14:
                                case 16:
                                case 17:
                                case 204:
                                case 205:
                                    break label122;
                                default:
                                    if (sendResult != null) {
                                        return sendResult;
                                    }

                                    throw var28;
                                }
                            } catch (InterruptedException var29) {
                                endTimestamp = System.currentTimeMillis();
                                this.updateFaultItem(mqSelected.getBrokerName(), endTimestamp - beginTimestampPrev, false);
                                this.log.warn(String.format("sendKernelImpl exception, throw exception, InvokeID: %s, RT: %sms, Broker: %s", invokeID, endTimestamp - beginTimestampPrev, mqSelected), var29);
                                this.log.warn(msg.toString());
                                this.log.warn("sendKernelImpl exception", var29);
                                this.log.warn(msg.toString());
                                throw var29;
                            }
                        }
                    }

                    if (sendResult != null) {
                        return sendResult;
                    }

                    info = String.format("Send [%d] times, still failed, cost [%d]ms, Topic: %s, BrokersSent: %s", times, System.currentTimeMillis() - beginTimestampFirst, msg.getTopic(), Arrays.toString(brokersSent));
                    info = info + FAQUrl.suggestTodo("http://rocketmq.apache.org/docs/faq/");
                    MQClientException mqClientException = new MQClientException(info, (Throwable)exception);
                    if (callTimeout) {
                        throw new RemotingTooMuchRequestException("sendDefaultImpl call timeout");
                    }

                    if (exception instanceof MQBrokerException) {
                        mqClientException.setResponseCode(((MQBrokerException)exception).getResponseCode());
                    } else if (exception instanceof RemotingConnectException) {
                        mqClientException.setResponseCode(10001);
                    } else if (exception instanceof RemotingTimeoutException) {
                        mqClientException.setResponseCode(10002);
                    } else if (exception instanceof MQClientException) {
                        mqClientException.setResponseCode(10003);
                    }

                    throw mqClientException;
                }

                ++times;
            }
        } else {
            List<String> nsList = this.getmQClientFactory().getMQClientAPIImpl().getNameServerAddressList();
            if (null != nsList && !nsList.isEmpty()) {
                throw (new MQClientException("No route info of this topic, " + msg.getTopic() + FAQUrl.suggestTodo("http://rocketmq.apache.org/docs/faq/"), (Throwable)null)).setResponseCode(10005);
            } else {
                throw (new MQClientException("No name server address, please set it." + FAQUrl.suggestTodo("http://rocketmq.apache.org/docs/faq/"), (Throwable)null)).setResponseCode(10004);
            }
        }
    }
```

上面是消息发送的代码，下面梳理下主要流程：

消息发送时，先根据 Topic 从 Broker 拉取 TopicPublishInfo 信息，它里面包含了 Topic 下所有的 MessageQueue。

```java
 TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
```

```java
 private TopicPublishInfo tryToFindTopicPublishInfo(String topic) {
        TopicPublishInfo topicPublishInfo = (TopicPublishInfo)this.topicPublishInfoTable.get(topic);
        if (null == topicPublishInfo || !topicPublishInfo.ok()) {
            this.topicPublishInfoTable.putIfAbsent(topic, new TopicPublishInfo());
            this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic);
            topicPublishInfo = (TopicPublishInfo)this.topicPublishInfoTable.get(topic);
        }

        if (!topicPublishInfo.isHaveTopicRouterInfo() && !topicPublishInfo.ok()) {
            this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic, true, this.defaultMQProducer);
            topicPublishInfo = (TopicPublishInfo)this.topicPublishInfoTable.get(topic);
            return topicPublishInfo;
        } else {
            return topicPublishInfo;
        }
    }
```

```java
public class TopicPublishInfo {
    private boolean orderTopic = false;
    private boolean haveTopicRouterInfo = false;
    private List<MessageQueue> messageQueueList = new ArrayList();
    private volatile ThreadLocalIndex sendWhichQueue = new ThreadLocalIndex();
    private TopicRouteData topicRouteData;

    public TopicPublishInfo() {
    }
    ...
```

选取一个目标队列：

```java
 MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, info);
```

接着调用核心发送方法，将消息发送到 broker

```java
sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout - costTime);
```

必须使用同步发送，异步/单向发送都无法保证消息被有序写入队列

sendKernelImpl 方法中有同/异步的判断，这里应该是走的 `case SYNC`

```java
case ASYNC:
    Message tmpMessage = msg;
    if (msgBodyCompressed) {
        tmpMessage = MessageAccessor.cloneMessage(msg);
        msg.setBody(prevBody);
    }

    long costTimeAsync = System.currentTimeMillis() - beginStartTime;
    if (timeout < costTimeAsync) {
        throw new RemotingTooMuchRequestException("sendKernelImpl call timeout");
    }

    sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(brokerAddr, mq.getBrokerName(), tmpMessage, requestHeader, timeout - costTimeAsync, communicationMode, sendCallback, topicPublishInfo, this.mQClientFactory, this.defaultMQProducer.getRetryTimesWhenSendAsyncFailed(), context, this);
    break;
case ONEWAY:
case SYNC:
    long costTimeSync = System.currentTimeMillis() - beginStartTime;
    if (timeout < costTimeSync) {
        throw new RemotingTooMuchRequestException("sendKernelImpl call timeout");
    }

    sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(brokerAddr, mq.getBrokerName(), msg, requestHeader, timeout - costTimeSync, communicationMode, context, this);
    break;
default:
    assert false;
```

Producer 保证发送有序，只要保证相同 ShardingKey 的消息发送到同一队列即可，以 spring cloud stream 的实现为例，可以查看   `PartitionHandler` 类中的 `determinePartition` 方法

```java
 public int determinePartition(Message<?> message) {
        Object key = this.extractKey(message);
        int partition;
        if (this.producerProperties.getPartitionSelectorExpression() != null) {
            partition = (Integer)this.producerProperties.getPartitionSelectorExpression().getValue(this.evaluationContext, key, Integer.class);
        } else {
            partition = this.partitionSelectorStrategy.selectPartition(key, this.partitionCount);
        }

        return Math.abs(partition % this.partitionCount);
    }

```

可以看到  partition 的值如果之前配置了分区 key 表达式如：

```yaml
 producer:
   partition-key-expression: payload['id']
```

则值是表达式的值，如没有配置，则走默认策略，默认策略的实现取的是消息的 hash:

```java
private static class DefaultPartitionSelector implements PartitionSelectorStrategy {
        private DefaultPartitionSelector() {
        }

        public int selectPartition(Object key, int partitionCount) {
            int hashCode = key.hashCode();
            if (hashCode == -2147483648) {
                hashCode = 0;
            }

            return Math.abs(hashCode);
        }
    }
```

**最后队列的选择是利用 partition 和队列（分区）总数取模后得到的结果。** 这样就可以保证相同 ShardingKey 的消息发送到同一队列了。

整体的流程如下图：

![](https://tva1.sinaimg.cn/large/008i3skNly1gy1uy78ln0j30ss0lhq3z.jpg)

消息发送后，由于队列本身的 FIFO 特性，它保存到 broker 端也是有序的。

### 消费有序

Consumer 默认是多线程并发消费同一个 MessageQueue 的，即使消息是顺序到达的，也不能保证消息顺序消费。

那么 RocketMQ 如何保证消息顺序消费呢？

与 producer 一样，我们按照 consumer 的流程串一下代码

consumer 启动时，在 MQClientInstance 类的 start 方法中进行了重平衡操作：

```java
public void start() throws MQClientException {

        synchronized (this) {
            switch (this.serviceState) {
                case CREATE_JUST:
                    this.serviceState = ServiceState.START_FAILED;
                    // If not specified,looking address from name server
                    if (null == this.clientConfig.getNamesrvAddr()) {
                        this.mQClientAPIImpl.fetchNameServerAddr();
                    }
                    // Start request-response channel
                    this.mQClientAPIImpl.start();
                    // Start various schedule tasks
                    this.startScheduledTask();
                    // Start pull service
                    this.pullMessageService.start();
                    // Start rebalance service
                    this.rebalanceService.start();
                    // Start push service
                    this.defaultMQProducer.getDefaultMQProducerImpl().start(false);
                    log.info("the client factory [{}] start OK", this.clientId);
                    this.serviceState = ServiceState.RUNNING;
                    break;
                case START_FAILED:
                    throw new MQClientException("The Factory object[" + this.getClientId() + "] has been created before, and failed.", null);
                default:
                    break;
            }
        }
    }
```

就是这一行 `rebalanceService.start()` ，它的目的是给自己分配 MessageQueue。要保证一个队列被一个消费者消费，那么消费者在进行消息拉取消费时就必须向 mq 服务器申请队列锁。如果申请到琐，则拉取消息，否则放弃消息拉取，等到下一个队列负载周期 (20s) 再试。

申请锁可以参考  `RebalanceImpl`类 `updateProcessQueueTableInRebalance` 和 `lock` 方法中的代码：

```java
 //如果是顺序消息，需要向 Broker 申请锁队列，加锁成功才开始消费。
for (MessageQueue mq : mqSet) {
    if (!this.processQueueTable.containsKey(mq)) {
        if (isOrder && !this.lock(mq)) {
            log.warn("doRebalance, {}, add a new mq failed, {}, because lock failed", consumerGroup, mq);
            continue;
        }

        this.removeDirtyOffset(mq);
        ProcessQueue pq = new ProcessQueue();
```

```java
public boolean lock(final MessageQueue mq) {
    // 查找 Broker Master 主机地址
    FindBrokerResult findBrokerResult = this.mQClientFactory.findBrokerAddressInSubscribe(mq.getBrokerName(), MixAll.MASTER_ID, true);
    if (findBrokerResult != null) {
        // 构建请求体
        LockBatchRequestBody requestBody = new LockBatchRequestBody();
        requestBody.setConsumerGroup(this.consumerGroup);// 消费组
        requestBody.setClientId(this.mQClientFactory.getClientId());// 客户端实例 ID
        requestBody.getMqSet().add(mq);// 申请锁哪些队列

        try {
            // 发送请求，Broker 返回锁住的队列集合
            Set<MessageQueue> lockedMq =
                this.mQClientFactory.getMQClientAPIImpl().lockBatchMQ(findBrokerResult.getBrokerAddr(), requestBody, 1000);
            for (MessageQueue mmqq : lockedMq) {
                ProcessQueue processQueue = this.processQueueTable.get(mmqq);
                if (processQueue != null) {
                    processQueue.setLocked(true);
                    processQueue.setLastLockTimestamp(System.currentTimeMillis());
                }
            }
            // 目标队列在里面，就说明加锁成功了
            boolean lockOK = lockedMq.contains(mq);
            log.info("the message queue lock {}, {} {}",
                     lockOK ? "OK" : "Failed",
                     this.consumerGroup,
                     mq);
            return lockOK;
        } catch (Exception e) {
            log.error("lockBatchMQ exception, " + mq, e);
        }
    }
    return false;
}
```

这个锁是 Broker 维护的全局锁。

一旦加锁成功，就会开始构建 PullRequest 对象开始拉取消息，消息拉取部分的代码实现在 PullMessageService 中，拉取成功后，在 PullCallback 里会将拉取到的消息填充到 ProcessQueue，然后提交消费请求，让 ConsumeMessageOrderlyService 开始消费消息。

消费消息时，先获取 MessageQueue 的锁对象，然后通过 synchronized 关键字保证只有一个线程消费

![](https://tva1.sinaimg.cn/large/008i3skNly1gy2w60mvdzj30zo0jwado.jpg)

对于顺序消息，当消费者消费消息失败后，消息队列 RocketMQ 会自动不断地进行消息重试（每次间隔时间为 1 秒），重试最大值是 Integer.MAX\_VALUE

```java
case SUSPEND_CURRENT_QUEUE_A_MOMENT:
    this.getConsumerStatsManager().incConsumeFailedTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(), msgs.size());
    // 校验最大重试次数，默认 Integer.MAX_VALUE
    if (checkReconsumeTimes(msgs)) {
        // 标记消息等待重新消费
        consumeRequest.getProcessQueue().makeMessageToConsumeAgain(msgs);
        // 提交消费请求，稍后重试
        this.submitConsumeRequestLater(
            consumeRequest.getProcessQueue(),
            consumeRequest.getMessageQueue(),
            context.getSuspendCurrentQueueTimeMillis());
        continueConsume = false;
    } else {
        commitOffset = consumeRequest.getProcessQueue().commit();
    }
    break;

```

最后补充一点，在消费的过程中，会对处理队列 (ProccessQueue) 进行加锁，保证处理中的消息消费完成，发生队列负载后，其他消费者才能继续消费。

例如队列 q3 目前是分配给消费者 C2 进行消费，已将拉取了 32 条消息在线程池中处理，然后对消费者进行了扩容，分配给 C2 的 q3 队列，被分配给 C3 了，由于 C2 已将处理了一部分，位点信息还没有提交，如果 C3 立马去消费 q3 队列中的消息，那存在一部分数据会被重复消费，故在 C2 消费者在消费 q3 队列的时候，消息没有消费完成，那负载队列就不能丢弃该队列，就不会在 broker 端释放琐，其他消费者就无法从该队列消费，尽最大可能保证了消息的重复消费，保证顺序性语义

消费总结 ：

1.  创建消息拉取任务时，消息客户端向 broker 端申请锁定 MessageQueue，使得一个 MessageQueue 同一个时刻只能被一个消费客户端消费

2.  消息消费时，多线程针对同一个消息队列的消费先尝试使用 synchronized 申请独占锁，加锁成功才能进行消费，使得一个 MessageQueue 同一个时刻只能被一个消费客户端中一个线程消费

3.  RocketMQ 中每一个消费组一个单独的线程池并发消费拉取到的消息，即消费端是多线程消费。而顺序消费的并发度等于该消费者分配到的队列数。消费并行度理论上不会有太大问题，因为 MessageQueue 的数量可以调整。

4.  在消费的过程中，会对处理队列 (ProccessQueue) 进行加锁，保证处理中的消息消费完成

5.  顺序消息一旦消费失败，默认会一直重试，不会跳过，因为一旦跳过就失去顺序消息的语义了

### 顺序消息可能存在的问题

**消息阻塞**

在顺序性语义的要求下，如果一条消息没有被成功消费，下一条消息就不能被消费，否则就不是顺序消费了。一条消息失败，如果跳过去消费其他消息，那就违背了顺序消费的语义。

建议在使用顺序消息时，务必保证应用能够及时监控并处理消费失败的情况，避免阻塞现象的发生。可以提供一些策略，由用户根据错误类型来决定是否跳过，并且提供重试队列之类的功能，在跳过之后用户可以在“其他”地方重新消费到这条消息。

**failover 失效**

发送顺序消息无法利用集群的 Failover 特性，因为不能更换 MessageQueue 进行重试

## 参考

*   [https://mp.weixin.qq.com/s/Ff3FTnkQmiOPZ69r3ek7sQ](https://mp.weixin.qq.com/s/Ff3FTnkQmiOPZ69r3ek7sQ "https://mp.weixin.qq.com/s/Ff3FTnkQmiOPZ69r3ek7sQ")

*   [https://spring.io/blog/2021/02/03/](https://spring.io/blog/2021/02/03/ "https://spring.io/blog/2021/02/03/")

*   [https://www.zhihu.com/question/30195969/answer/142416274demystifying-spring-cloud-stream-producers-with-apache-kafka-partitions](https://www.zhihu.com/question/30195969/answer/142416274demystifying-spring-cloud-stream-producers-with-apache-kafka-partitions "https://www.zhihu.com/question/30195969/answer/142416274demystifying-spring-cloud-stream-producers-with-apache-kafka-partitions")
