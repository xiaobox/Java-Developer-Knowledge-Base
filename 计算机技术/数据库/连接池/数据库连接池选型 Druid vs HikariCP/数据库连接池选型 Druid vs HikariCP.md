# 数据库连接池选型 Druid vs HikariCP

## 这里主要比较HikariCP 和阿里的Druid

这里有来自Druid的竞品对比：[https://github.com/alibaba/druid/wiki/Druid连接池介绍](https://github.com/alibaba/druid/wiki/Druid连接池介绍 "https://github.com/alibaba/druid/wiki/Druid连接池介绍")

![](https://mmbiz.qpic.cn/mmbiz_png/YZibCWq4rxD8Tlf8nQn7eRjs0QbfksOrNvrjwz4WZKZTEEzKltG9vPic4hnTBlDahhwzXicsUq3kd9cLwAeRUV7dA/640?wx_fmt=png\&wxfrom=5\&wx_lazy=1\&wx_co=1)

springboot 现在官方默认的数据库连接池是 HikariCP，HikariCP的性能从测试的数据上来看也是最高的。

![](https://mmbiz.qpic.cn/mmbiz_png/YZibCWq4rxD8Tlf8nQn7eRjs0QbfksOrN9wahOY7GibsuoTXiccpJTTpW7IjWh6FjbWm4oUaibrAHT6ZS377Y6JkIQ/640?wx_fmt=png\&wxfrom=5\&wx_lazy=1\&wx_co=1)

所以我们主要对比Druid和HikariCP

### 先来看下这个著名的issue

*   一个印度小哥提的 issue

![](https://mmbiz.qpic.cn/mmbiz_png/YZibCWq4rxD8Tlf8nQn7eRjs0QbfksOrNlH24eB50r5jSP15V0fTIMZd8nNJS6c2RR5fBc7BxBEaic6SDS3Auqdg/640?wx_fmt=png\&wxfrom=5\&wx_lazy=1\&wx_co=1)

*   brettwooldridge 这边主要针对性能和在中国以外的地方用的少的问题

![](https://mmbiz.qpic.cn/mmbiz_png/YZibCWq4rxD8Tlf8nQn7eRjs0QbfksOrNVOTIgiccOQlHHRmTyQu8748VV2FJW8zZUcSZ4VQIqFDmFIFbHyVFPoA/640?wx_fmt=png\&wxfrom=5\&wx_lazy=1\&wx_co=1)

![](https://mmbiz.qpic.cn/mmbiz_png/YZibCWq4rxD8Tlf8nQn7eRjs0QbfksOrN7KdDtLFGe2LAYIiaH45TDneyiaW7lyj9Y0m9j07P6eYLNPnicvXShibicsg/640?wx_fmt=png\&wxfrom=5\&wx_lazy=1\&wx_co=1)

*   温绍这边说，由于使用公平锁所以降低了性能，至于为什么是因为在生产环境中遇到的一些问题，使设计使然。

![](https://mmbiz.qpic.cn/mmbiz_png/YZibCWq4rxD8Tlf8nQn7eRjs0QbfksOrN23tDUaaatOEPfcVSgXpE8jicIr575tvvibcUMYO8ibib6SIavSQcYKVLCA/640?wx_fmt=png\&wxfrom=5\&wx_lazy=1\&wx_co=1)

*   温绍同时也强调我们淘宝体量大，并发高，顺便甩了个带有马爸爸照片的链接，让他了解一下淘宝

![](https://mmbiz.qpic.cn/mmbiz_png/YZibCWq4rxD8Tlf8nQn7eRjs0QbfksOrNmPs8qotCAJrAnUxW9X1z3Z5gKib77DF8q9mvU8haJicAP6DsvH5JV6ag/640?wx_fmt=png\&wxfrom=5\&wx_lazy=1\&wx_co=1)

![](https://mmbiz.qpic.cn/mmbiz_jpg/YZibCWq4rxD8Tlf8nQn7eRjs0QbfksOrNsZBuWsPNlOA5SOxbsBqSgsy3R0pH7XxHWGV2eklBph3JgbqcAQGyhA/640?wx_fmt=jpeg\&wxfrom=5\&wx_lazy=1\&wx_co=1)

*   brettwooldridge 这边回应 :比量是吧？（内心潜台词）

*   wix.com托管着超过1.09亿个网站，每天处理的请求超过10亿个

*   Atlassian的产品拥有数百万的客户

*   HikariCP是使用Play框架，Slick，JOOS构建的每个应用的默认连接池

*   老子现在是spring boot的默认连接池

*   HikariCP每月从中央Maven存储库中解析超过300,000次。

![](https://mmbiz.qpic.cn/mmbiz_png/YZibCWq4rxD8Tlf8nQn7eRjs0QbfksOrN0wG6FXick0ic1icPT8WAXnVluOrAxv4UNkbvlRX87UdYRQB4DKr2A10Sw/640?wx_fmt=png\&wxfrom=5\&wx_lazy=1\&wx_co=1)

*   同时也甩了个链接，让你看看我HikariCP的名望

![](https://mmbiz.qpic.cn/mmbiz_png/YZibCWq4rxD8Tlf8nQn7eRjs0QbfksOrN1bwdYPWpwrfnXShOFwn6NLxia1K1cC0SIqn3PNqbkQibIEeevsMJXxrg/640?wx_fmt=png\&wxfrom=5\&wx_lazy=1\&wx_co=1)

### 看完热闹，说回正题

*   功能角度考虑，Druid 功能更全面，除具备连接池基本功能外，还支持sql级监控、扩展、SQL防注入等。最新版甚至有集群监控

*   单从性能角度考虑，从数据上确实HikariCP要强，但Druid有更多、更久的生产实践，它可靠。

*   单从监控角度考虑，如果我们有像skywalking、prometheus等组件是可以将监控能力交给这些的 HikariCP 也可以将metrics暴露出去。

### 总结

由于我们的系统架构上有专门用于监控的系统（skywalking、prometheus），外加使用了阿里云的RDS，RDS也有完整数据库监控指标。所以我们可以将监控的功能交给这些系统，让数据库连接池专心做好连接池的本职工作，所以我们选择性能更好的 HikariCP 做为数据库连接池。由于我们使用了Spring boot ,HikariCP 是内置的，也更方便配置使用，能做到开箱即用。
