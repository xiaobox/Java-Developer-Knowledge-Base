# 自顶向下学习 RocketMQ（一）QuickStart

## 安装部署 RocketMQ

采用源码编译安装，注意请提前将 maven 安装调试好。

*   操作系统 macOS

*   RocketMQ 版本：4.9.2

*   Maven 版本 3.3.9

*   JDK 版本 1.8.0\_181

以下安装是**单 Master 模式**，非集群模式，仅适用于本地开发和测试环境。

### 下载源码包

```bash
wget https://dlcdn.apache.org/rocketmq/4.9.2/rocketmq-all-4.9.2-source-release.zip
```

### 解压源码并进行编译

```纯文本
> unzip rocketmq-all-4.9.2-source-release.zip
> cd rocketmq-all-4.9.2/
> mvn -Prelease-all -DskipTests clean install -U
> cd distribution/target/rocketmq-4.9.2/rocketmq-4.9.2
```

### 启动 Name Server

在上一步的目录下，执行：

```纯文本
> nohup sh bin/mqnamesrv &
> tail -f ~/logs/rocketmqlogs/namesrv.log

021-11-24 16:50:14 INFO main - tls.client.keyPassword = null
2021-11-24 16:50:14 INFO main - tls.client.certPath = null
2021-11-24 16:50:14 INFO main - tls.client.authServer = false
2021-11-24 16:50:14 INFO main - tls.client.trustCertPath = null
2021-11-24 16:50:14 INFO main - Using JDK SSL provider
2021-11-24 16:50:15 INFO main - SSLContext created for server
2021-11-24 16:50:15 INFO NettyEventExecutor - NettyEventExecutor service started
2021-11-24 16:50:15 INFO main - Try to start service thread:FileWatchService started:false lastThread:null
2021-11-24 16:50:15 INFO FileWatchService - FileWatchService service started
2021-11-24 16:50:15 INFO main - The Name Server boot success. serializeType=JSON

```

### 启动 Broker

```bash
> nohup sh bin/mqbroker -n localhost:9876 &
> tail -f ~/logs/rocketmqlogs/broker.log 

2021-11-24 16:52:22 INFO main - The broker[localhost, 10.3.10.244:10911] boot success. serializeType=JSON and name server is localhost:9876
2021-11-24 16:52:32 INFO BrokerControllerScheduledThread1 - dispatch behind commit log 0 bytes
2021-11-24 16:52:32 INFO BrokerControllerScheduledThread1 - Slave fall behind master: 202890 bytes
2021-11-24 16:52:32 INFO brokerOutApi_thread_2 - register broker[0]to name server localhost:9876 OK
2021-11-24 16:53:02 INFO brokerOutApi_thread_3 - register broker[0]to name server localhost:9876 OK

```

## 安装 RocketMQ-Dashboard

可视化工具，可以在浏览器中查看消息队列的状态。

同样用源码安装，当然也可以用 docker  安装。

### 下载源码包

```bash
 wget https://github.com/apache/rocketmq-dashboard/archive/refs/tags/rocketmq-dashboard-1.0.0.zip
```

### 编译运行

```bash

> unzip rocketmq-dashboard-1.0.0.zip
> cd rocketmq-dashboard-rocketmq-dashboard-1.0.0
> mvn spring-boot:run
```

这时候会出现如下错误：

```纯文本
Caused by: org.apache.rocketmq.remoting.exception.RemotingConnectException: connect to null failed
  at org.apache.rocketmq.remoting.netty.NettyRemotingClient.invokeSync(NettyRemotingClient.java:394)
  at org.apache.rocketmq.client.impl.MQClientAPIImpl.getBrokerClusterInfo(MQClientAPIImpl.java:1333)
  at org.apache.rocketmq.tools.admin.DefaultMQAdminExtImpl.examineBrokerClusterInfo(DefaultMQAdminExtImpl.java:306)
  at org.apache.rocketmq.tools.admin.DefaultMQAdminExt.examineBrokerClusterInfo(DefaultMQAdminExt.java:257)
  at org.apache.rocketmq.dashboard.service.client.MQAdminExtImpl.examineBrokerClusterInfo(MQAdminExtImpl.java:204)
  at org.apache.rocketmq.dashboard.service.client.MQAdminExtImpl$$FastClassBySpringCGLIB$$a15c4ca6.invoke(<generated>)
  at org.springframework.cglib.proxy.MethodProxy.invoke(MethodProxy.java:218)
  at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.invokeJoinpoint(CglibAopProxy.java:769)
  at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:163)
  at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.proceed(CglibAopProxy.java:747)
  at org.springframework.aop.aspectj.MethodInvocationProceedingJoinPoint.proceed(MethodInvocationProceedingJoinPoint.java:88)
  at org.apache.rocketmq.dashboard.aspect.admin.MQAdminAspect.aroundMQAdminMethod(MQAdminAspect.java:52)
  at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
  at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
  at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
  at java.lang.reflect.Method.invoke(Method.java:498)

```

明显是连不上服务器，需要修改配置文件：

```bash
> cd src/main/resources/
```

打开  application.properties  文件

```.properties
#
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

server.address=0.0.0.0
server.port=8080

### SSL setting
#server.ssl.key-store=classpath:rmqcngkeystore.jks
#server.ssl.key-store-password=rocketmq
#server.ssl.keyStoreType=PKCS12
#server.ssl.keyAlias=rmqcngkey

#spring.application.index=true
spring.application.name=rocketmq-dashboard
spring.http.encoding.charset=UTF-8
spring.http.encoding.enabled=true
spring.http.encoding.force=true
logging.level.root=INFO
logging.config=classpath:logback.xml
#if this value is empty,use env value rocketmq.config.namesrvAddr  NAMESRV_ADDR | now, you can set it in ops page.default localhost:9876
rocketmq.config.namesrvAddr=
#if you use rocketmq version < 3.5.8, rocketmq.config.isVIPChannel should be false.default true
rocketmq.config.isVIPChannel=
#timeout for mqadminExt, default 5000ms
rocketmq.config.timeoutMillis=
#rocketmq-console's data path:dashboard/monitor
rocketmq.config.dataPath=/tmp/rocketmq-console/data
#set it false if you don't want use dashboard.default true
rocketmq.config.enableDashBoardCollect=true
#set the message track trace topic if you don't want use the default one
rocketmq.config.msgTrackTopicName=
rocketmq.config.ticketKey=ticket

#Must create userInfo file: ${rocketmq.config.dataPath}/users.properties if the login is required
rocketmq.config.loginRequired=false

#set the accessKey and secretKey if you used acl
#rocketmq.config.accessKey=
#rocketmq.config.secretKey=
rocketmq.config.useTLS=false

```

根据注释我们得知，需要将 rocketmq.config.namesrvAddr 设置为：localhost:9876, 再次启动成功。

访问：[http://localhost:8080/](http://localhost:8080/ "http://localhost:8080/") ，当然你也可以修改地址或者更改端口号。同样是修改 application.properties 文件，修改成类似这样的配置：

```.properties
server.port=8083
server.servlet.context-path=/rocketmq-dashboard

```

![](https://tva1.sinaimg.cn/large/008i3skNly1gwqd0w4fc0j31fi0np40x.jpg)

## 测试消息的发送和接收

```bash
> export NAMESRV_ADDR=localhost:9876
> sh bin/tools.sh org.apache.rocketmq.example.quickstart.Producer
 SendResult [sendStatus=SEND_OK, msgId= ...

> sh bin/tools.sh org.apache.rocketmq.example.quickstart.Consumer
 ConsumeMessageThread_%d Receive New Messages: [MessageExt...

```

## 关闭 nameServer 和 broker

```bash
> sh bin/mqshutdown broker
The mqbroker(36695) is running...
Send shutdown request to mqbroker(36695) OK

> sh bin/mqshutdown namesrv
The mqnamesrv(36664) is running...
Send shutdown request to mqnamesrv(36664) OK

```

## 其他高可用部署方式

*   多 Master 模式

*   多 Master 多 Slave 模式-异步复制

*   多 Master 多 Slave 模式-同步双写

具体部署方式可参考：[文档](https://github.com/apache/rocketmq/blob/master/docs/cn/operation.md "文档")

这里贴一个比较完整的生产 K8S 部署方案：

[https://cloud.tencent.com/developer/article/1785729](https://cloud.tencent.com/developer/article/1785729 "https://cloud.tencent.com/developer/article/1785729")

上面这个方案是一个双主双从的配置，每个 statefulset 只有一个副本，每个 Pod  拥有自己的配置文件，根据 RocketMQ 文档的说明来配置，没有什么问题。但当我们要扩展多 Master, 多 Slave 的时候，就会比较麻烦，要再建 Master Slave 对，增加 statefulset。

下面有一个部署技巧可以解决这个的问题。

### 通过 statefulset 副本扩展集群规模

> 以上 Broker 与 Slave 配对是通过指定相同的 BrokerName 参数来配对，Master 的 BrokerId 必须是 0，Slave 的 BrokerId 必须是大于 0 的数。

根据文档得知 Master 和 Slave 是根据 BrokerName 来配对的，所以上面的部署配置中也对同一对的主从设置了相同的 BrokerName。

**如果没有配置 BrokerName 有默认值吗？**

我们来看下 RocketMQ 的源码：

```java

public class BrokerConfig {
    private static final InternalLogger log = InternalLoggerFactory.getLogger(LoggerName.COMMON_LOGGER_NAME);

    private String rocketmqHome = System.getProperty(MixAll.ROCKETMQ_HOME_PROPERTY, System.getenv(MixAll.ROCKETMQ_HOME_ENV));
    @ImportantField
    private String namesrvAddr = System.getProperty(MixAll.NAMESRV_ADDR_PROPERTY, System.getenv(MixAll.NAMESRV_ADDR_ENV));
    @ImportantField
    private String brokerIP1 = RemotingUtil.getLocalAddress();
    private String brokerIP2 = RemotingUtil.getLocalAddress();
    @ImportantField
    private String brokerName = localHostName();
    @ImportantField
    private String brokerClusterName = "DefaultCluster";

```

```java
  */
    @ImportantField
    private boolean aclEnable = false;

    private boolean storeReplyMessageEnable = true;

    private boolean autoDeleteUnusedStats = false;

    public static String localHostName() {
        try {
            return InetAddress.getLocalHost().getHostName();
        } catch (UnknownHostException e) {
            log.error("Failed to obtain the host name", e);
        }

        return "DEFAULT_BROKER";
    }

```

可见，brokerName 有默认值 ，即如果没有主动 set brokerName，那么获取的是 localhostName。

pod 的 hostname 即为 pod 的名字，如果我们的 pod 名字是 broker-0，那么它的 hostname 就是 broker-0。联系上面 RocketMQ 的源码，brokerName 的值如果不设置拿到的就是 hostname, 即 pod Name。

由于我们部署 RocketMQ 时用的是有状态服务 (statefulset)，statefulset 的 Pod 的命名是有规则的：

> 对于一个拥有 N 个副本的 StatefulSet，Pod 被部署时是按照 {0 …… N-1} 的序号顺序创建的。

于是我们就可以利用 statefulset 的命名规则进行集群的扩展了，例如我要设置一个双主双从的集群，那么只需要将 master 和 slave 的 statefulset 的副本数设置为 2 就可以了，结果类似这样：

```纯文本
rocketmq-broker-master-0                               1/1     Running   1          462d
rocketmq-broker-master-1                               1/1     Running   0          462d
rocketmq-broker-slave-0                                1/1     Running   1          462d
rocketmq-broker-slave-1                                1/1     Running   0          462d
```

可见，根据命名规则 RocketMQ 的 Master 和 Slave 可以根据 brokerName（即 pod name）配对正确。那么如果我想要扩展集群，那么我只需要把 master 和 slave 的 statefulset 的副本数设置为 2、3、4、5... 就可以了。

以上就完成了只需要改两个 statefulset 的副本数就可以扩展集群的操作。一个小技巧分享给大家。

## 参考

*   [https://github.com/apache/rocketmq/tree/master/docs/cn](https://github.com/apache/rocketmq/tree/master/docs/cn "https://github.com/apache/rocketmq/tree/master/docs/cn")

*   [https://rocketmq.apache.org/docs/quick-start/](https://rocketmq.apache.org/docs/quick-start/ "https://rocketmq.apache.org/docs/quick-start/")

*   [https://github.com/apache/rocketmq-dashboard](https://github.com/apache/rocketmq-dashboard "https://github.com/apache/rocketmq-dashboard")

*   [https://cloud.tencent.com/developer/article/1785729](https://cloud.tencent.com/developer/article/1785729 "https://cloud.tencent.com/developer/article/1785729")
