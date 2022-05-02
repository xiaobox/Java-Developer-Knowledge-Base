# 自顶向下学习 RocketMQ（八）：事务消息原理分析

## RocketMQ 5.0

在了解事务消息原理之前，先说点版本相关的事情，我们的 demo 和原码分析是以 4.9 版本为基础的，而 RocketMQ 的最新版本已经是 5.0 了，看一下 RocketMQ 5.0 的最新进展，从里程碑上来看目前是在 alpha 阶段。

![](https://tva1.sinaimg.cn/large/008i3skNly1gyazsq4j2uj31770fcmyt.jpg)

功能上，看下官宣：

![](https://tva1.sinaimg.cn/large/008i3skNly1gyazthiy15j30je1g4q77.jpg)

不知道以后阿里云上商业版本和开源版本能不能对齐，但从架构上功能上确实比 5.0 之前的有很大调整。至于在云原生领域 是否和 apache pulsar ([https://pulsar.apache.org/](https://pulsar.apache.org/ "https://pulsar.apache.org/")) 呈 PK 之势，就让我们拭目以待吧。

## 事务流程

我们再回顾一下事务流程

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

## 源码流程

有关源码的走读流程，这里有一篇文章该写的都写了，写的挺好的，我就不赘述了：

[https://juejin.cn/post/7017973903561605151#heading-9](https://juejin.cn/post/7017973903561605151#heading-9 "https://juejin.cn/post/7017973903561605151#heading-9")

## 参考

*   [https://developer.aliyun.com/article/797277](https://developer.aliyun.com/article/797277 "https://developer.aliyun.com/article/797277")

*   [https://cloud.tencent.com/developer/article/1648617](https://cloud.tencent.com/developer/article/1648617 "https://cloud.tencent.com/developer/article/1648617")
