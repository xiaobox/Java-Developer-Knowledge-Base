# APM 组件选型

## 监控之 what\&why

常用监控手段

*   按监控层次分：业务监控、应用监控和基础监控等；

*   按监控日志来源分：基于日志文件监控、基于数据库监控和基于网络监控等；

*   按监控领域分：前端监控、后端监控、全链路监控、业务间监控等；

*   按监控目标分：系统故障监控、业务指标监控、应用性能监控、用户行为监控、安全合规监控等。

**监控首先要解决的是目标设定，到底要解决什么问题，关注什么指标。**

**我们的定位是APM 即应用性能监控。解决微服务架构下：**

*   **服务间依赖关系梳理、查询**

*   **全局依赖关系拓扑**

*   **调用链跟踪、拓扑、查询**

*   服务响应时间监测（最长、最短、平均）

*   服务JVM性能监测和告警

*   Dashboard（图表展示）

进而解决

*   服务问题快速诊断、定位

*   对于自己的调用情况，方便作容量规划，同时对于突发的请求也能进行异常告警和应急准备

打造监控闭环：监控不是目的，目的是告警，告警不是目的，目的是解决问题。

## APM

APM被作为一个细分领域的IT解决方案行业被单独提出来还是在近几年的事情，大概在2010年左右。 厂商有：appdynamics、听云、OneAPM等

![](https://mmbiz.qpic.cn/mmbiz_png/YZibCWq4rxDicVvt0TshMBmA9Rhj9qwmjAFD6lXta0icqLHjqLt3PSZ95PBNOmypJGjZoSruicADaapiaUkuqjLiazAA/640?wx_fmt=png\&wxfrom=5\&wx_lazy=1\&wx_co=1)

![](https://mmbiz.qpic.cn/mmbiz_png/YZibCWq4rxDicVvt0TshMBmA9Rhj9qwmjAwT0efg8t6xFfsAic4Pg8tUciaPRE0IAvJEAYneibTyF4OtstXerzuPribw/640?wx_fmt=png\&wxfrom=5\&wx_lazy=1\&wx_co=1)

**APM五大维度**

![](https://mmbiz.qpic.cn/mmbiz_png/YZibCWq4rxDicVvt0TshMBmA9Rhj9qwmjASjpAL8OCtEtoibRU2DLFhoUPkjXia1CFrUvqs5OUj8jNeuotolFfRBVg/640?wx_fmt=png\&wxfrom=5\&wx_lazy=1\&wx_co=1)

*   **终端用户体验** &#x20;

    衡量从用户请求到数据再返回的流量传输是捕获最终用户体验（EUE）的一部分。此测量的结果称为实时应用程序监视（又称自顶向下监视），它具有被动和主动两个组件。**被动监控** 通常是使用网络端口镜像实现的无代理设备。**主动监控** 由预定义的合成探针和Web机器人组成，用于报告系统可用性和业务事务（即业务方自行埋点）。 &#x20;

*   **应用架构映射** &#x20;

    应用程序发现和依赖关系映射（ADDM）解决方案用于自动执行将事务和应用程序映射到底层基础架构组件的过程。 &#x20;

*   **应用事务的分析** &#x20;

    关注用户定义的事务或对业务社区有一定意义的URL页面定义。 &#x20;

*   **深度应用诊断** &#x20;

    深度应用诊断（DDCM）需要安装代理，通常针对中间件，侧重于Web，应用程序和消息服务器。 &#x20;

*   **数据分析** &#x20;

    获得一组通用的度量标准以收集和报告每个应用程序非常重要，然后标准化有关数据并呈现应用程序性能数据的常见视图。

APM被形象的称为应用程序的私人医生，越来越收到企业的青睐，比起通过日志方式记录关键数据显然要更加实用，APM主要包含如下核心功能：

*   **应用系统存活检测** &#x20;

*   **应用程序性能指标检测(CPU利用率、内存利用率等)**

*   **应用程序关键事件检测** &#x20;

*   **检测数据持久化存储并能够多维度查询**

*   **服务调用跟踪**

*   **监控告警**

## 一般做法

下面三个维度是有重合部分的，比如JVM监控等。

*   Logging:ELK

*   Metrics:Prometheus

*   tracing:本文选型

![](https://mmbiz.qpic.cn/mmbiz_png/YZibCWq4rxDicVvt0TshMBmA9Rhj9qwmjA3QalH5yTsgbcF2ew9sLvahzBaklKnpdN0KFHHAnXmFDq4JTuS9NQBQ/640?wx_fmt=png\&wxfrom=5\&wx_lazy=1\&wx_co=1)

先看一下演进的历史： &#x20;

![](https://mmbiz.qpic.cn/mmbiz_png/YZibCWq4rxDicVvt0TshMBmA9Rhj9qwmjAkKKX3n8tMO4UD5FiaUsicC6vibfrn5ZuYdvcHp5PsNND8ApkOt0f9hcKg/640?wx_fmt=png\&wxfrom=5\&wx_lazy=1\&wx_co=1)

由于pinpoint和skywalking从工作原理、性能、功能等方面很像，由于我们不需要追求那么好的UI，以及考虑到社区和apache的背书在这两者中我们选择了skywalking。而Uber的jaeger较新相对来说社区和文档的支持没有前者友好，就不在我们的选型范围了。

我们的选型主要针对 `zipkin` `cat` 和`skywalking`进行 &#x20;

|                       |                                   |                                        |                         |
| --------------------- | --------------------------------- | -------------------------------------- | ----------------------- |
|                       | CAT                               | Zipkin                                 | Apache Skywalking       |
| 调用链可视化                | 有                                 | 有                                      | 有                       |
| 聚合报表                  | 非常丰富                              | 少                                      | 较丰富                     |
| 服务依赖图                 | 简单                                | 简单                                     | 好                       |
| 埋点方式                  | 侵入                                | 侵入                                     | 非侵入，运行期字节码增强            |
| VM指标监控                | 好                                 | 无                                      | 有                       |
| 告警支持                  | 有                                 | 无                                      | 有                       |
| 多语言支持                 | java/.Net/C/C++/NodeJS/Python/Go等 | 丰富                                     | java/.Net/NodeJS/PHP/Go |
| 存储机制                  | Mysql,本地文件，HDFS（调用链）              | 可选in memory,mysql,ES(生产)，Cassandra(生产) | H2，ES（生产），mysql,TIDB等   |
| 社区支持                  | 主要在国内，点评、美团                       | 文档丰富，国外主流                              | Apache支持，国内社区好          |
| 国内案例                  | 点评、携程、陆金所、拍拍贷等                    | 京东、阿里定制不开源                             | 华为、小米、当当、微众银行           |
| APM                   | Yes                               | No                                     | Yes                     |
| 祖先源头                  | eBay CAL                          | Google Dapper                          | Google Dapper           |
| 同类产品                  | 暂无                                | Uber jaeger,Spring Cloud Sleuth        | Naver Pinpoint          |
| GitHub starts(2020.7) | 13.7k                             | 13.2k                                  | 14k                     |
| 亮点                    | 企业生产级，报表丰富                        | 社区生态好                                  | 非侵入，Apache背书            |
| 不足                    | 用户体验一般，社区一般                       | APM报表能力弱                               | 时间不长，文档一般               |

基于以上，我的建议是： &#x20;

*   zipkin欠缺APM报表能力，不建议 &#x20;

*   企业生产级，推荐CAT &#x20;

*   关注和试点 Skywalking

*   用好调用链监控，难点在于后期的企业定制化和自研能力

## 参考：

*   [https://www.infoq.cn/article/KYxDaw2qiZ7rm\*7Ej3ps](https://www.infoq.cn/article/KYxDaw2qiZ7rm*7Ej3ps "https://www.infoq.cn/article/KYxDaw2qiZ7rm*7Ej3ps")

*   [https://skywalking.apache.org/zh/blog/2019-03-29-introduction-of-skywalking-and-simple-practice.html](https://skywalking.apache.org/zh/blog/2019-03-29-introduction-of-skywalking-and-simple-practice.html "https://skywalking.apache.org/zh/blog/2019-03-29-introduction-of-skywalking-and-simple-practice.html")

*   [https://developer.aliyun.com/article/272142](https://developer.aliyun.com/article/272142 "https://developer.aliyun.com/article/272142")
