# API 网关选型及包含 BFF 的架构设计

## 一 背景介绍

下图是我从网络上找到的一个微服务架构的简单架构图，如图可见 API Gateway 在其中起到一个承上启下的作用，是关键组件。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8d4456b9963e4119887c7562613abd0b\~tplv-k3u1fbpfcp-zoom-1.image)

图片来源于网络

在更通用的场景下我们会使用 NGINX 这样的软件做前置，用来处理SLB负载均衡过来的流量，作用是反向代理、集群负载均衡、转发、日志收集等功能。

然后再将 NGINX 的请求 proxy 到 API Gateway 做统一网关处理。

在上面的这个场景下 API Gateway 可以包含以下功能：

*   安全

*   限流

*   缓存

*   熔断

*   重试

*   负载

*   反向路由

*   认证、鉴权

*   日志收集和监控

*   其他

熟悉 NGINX 的朋友应该可以看出来，上面列出的这些功能和 NGINX 的部分功能是重合的，不过由于架构结构不同，在上面我提到的场景中，即 NGINX 在前 API gateway 在后的结构中，他们两者关注的维度也不一样，所以即使有重合也正常。

## 二 架构调整

下图是我基于云原生微服务架构设计的架构图其中前端流量是通过 SLB -> NGINX -> API Gateway 再到具体服务。&#x20;

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b3ffb0c1def147639509c45e247f649c\~tplv-k3u1fbpfcp-zoom-1.image)

## 三 java技术栈的 API Gateway 选型

由于后端采用java 的 spring cloud 开发的，所以在语言一致性上更倾向 java 语言开发的组件。如上图虽然在 API Gateway 的位置上写的是 spring cloud gateway，然而也可以采用像 zuul、zuul2 这些同样是 java 语言开发的组件。对于具体 zuul 和 spring gateway的选型，是这样考虑的：

|       | spring cloud gateway   | zuul                                |
| ----- | ---------------------- | ----------------------------------- |
| 性能    | 性能比 Netflix Zuul 好将近一倍 | Zuul1 的性能较差 Zuul2 较 Zuul1 有较大的提升    |
| 社区和文档 | spring社区非常活跃           | 一般                                  |
| 可维护性  | 基于spring官方维护性强         | 经常跳票、Spring Cloud暂时还没有对Zuul2.0的整合计划 |
| 亮点    | 异步、配置灵活                | 成熟、简单门槛低                            |
| 不足    | 早期产品、新版本踩坑             | 性能一般、可编程一般                          |

Spring Cloud Gateway 的性能比 Zuul 好基本上已经是业界公认的了，实际上，Spring Cloud Gateway 官方也发布过一个性能测试，这里节选如下数据：&#x20;

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/279a14e3d66c414997baad0d86f0c8d9\~tplv-k3u1fbpfcp-zoom-1.image)

Spring Cloud Gateway 构建于 Spring 5+，基于 Spring Boot 2.x 响应式的、非阻塞式的 API。同时，它支持 websockets，和 Spring 框架紧密集成。从目前来看，gateway替代zuul是趋势。**基于以上这些，综合考虑在架构中使用Spring Cloud Gateway**。

## 四 非java技术栈的 API Gateway 选型

现代 API Gateway 越来越需要或者流行**可编程**网关了。上面介绍的都是基于 java 语言开发的可编程的 API Gateway。下面我们来聊聊非 java 语言开发的网关。从前面的架构图上看，我们完全可以**将 NGINX 和 API Gateway 合并起来**，他们的功能的重合点自然消除了，也能降低架构的复杂性和运维成本。

NGINX 是一款优秀的软件，然而它在动态性方面的不足导致不太灵活，后面出现的 OpenResty、tengine 这些基于NGINX 和 Lua 的软件在动态性、灵活方面有本质上的改善，加上基于Lua脚本和插件，可以实现所谓的**可编程**。

市面上基于OpenResty 以 API Gateway 为应用场景的应用软件有 **Kong、APISIX、tyk** 等。以下是CNCFland scape 的一个概览

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f581dba0c83c4b768d9074ad5c523828\~tplv-k3u1fbpfcp-zoom-1.image)

比较了一下 NGING 和 KONG&#x20;

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eaa111c715044cbfa46e730a866d3acb\~tplv-k3u1fbpfcp-zoom-1.image)

**经过考虑，在架构上，后期有可能将 NGINX、Spring Cloud Gateway 替换成KONG 或其他软件。**

比较了一下，目前最火的应用是Kong，另一个国产的 APISIX 趋势也是很猛，且他们的技术栈雷同，所以我在选型上找到了APISIX的作者做的对比：

从 API 网关核心功能点来看，两者均已覆盖：&#x20;

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e58ff2ea8b5b47cbb5a290e76b0abaff\~tplv-k3u1fbpfcp-zoom-1.image)

更详细的比较：&#x20;

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5369236578204f299a4c48a500b0feca\~tplv-k3u1fbpfcp-zoom-1.image)

通过性能测试可以看到，在不开启插件的情况下，Apache APISIX 的性能（QPS 和延迟）是 Kong 的2倍，但开启了两个常用插件后，性能就是 Kong 的十倍了。

**无论从性能、可用性、可编程代码量等各个维度APISIX都是非常优秀的，目前唯一担心的就是这种早期项目没有太多大规模应用实践，如果上生产还是有风险，可在测试环境调研，并等待有更多生产实践作为依据。** 当然如果架构师认为风险并不大，且经过了测试调研也是可以上的。😁

## 五 BFF 层建设迭代

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d57618cb62f04074b88523f5323c4a36\~tplv-k3u1fbpfcp-zoom-1.image)

前面我们将 API Gateway 的网关选型介绍了一下，请求通过网关后一般不会直接打到具体微服务上的，而是会通过BFF层，所谓的BFF，即 backend for frontend 面向前端的后端。具体来说它的职能包括：

*   api数据裁剪

*   接口编排

*   接口调用

这层有的公司会按业务进行多个BFF的建设，在BFF中又有可能拆成多个服务，比如支撑首页的，支持列表页的，或者只有一个服务，支撑某个应用的所有请求的。

有了BFF层，前后端就会更好的解耦，前端不用再调用多个接口，然后再组织数据，微服务后端也只需要关心自己服务边界内的事情。

然而在实践的过程中会出现一些问题：

*   大量业务逻辑从前后端集中在了BFF层

*   BFF层逻辑复杂，代码量越业越大，难以维护

*   BFF API版本维护复杂

*   前端端接口职责不清，扯皮的结果就是放在BFF层

以上是我真实遇到过的场景。所以在后面的架构设计和实施中，这些情况会尽量避免，但没有从技术上解决根本问题。直到 **GraphQL** 的出现，让我眼前一亮，给了我一个很好的解决方案。关于GraphQL的搭建，数据交换等细节这里就不展开说了，感兴趣的可以从网上找到很多资料。

下图是我从网络上找的一个符合我心目中的理想架构。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b936686f0bdc483bacbec5a0b8c4296f\~tplv-k3u1fbpfcp-zoom-1.image)

说起来简单，做起来没那么容易 ，细节是魔鬼，每利用一个新的技术都会经历一波打怪升级的过程。不过总体来说利用GraphQL确实能从理论上解决上面说所的问题。而重点是如何将它结合进你的系统架构中，并且发挥出它的优势。**架构很多时候是在做权衡和选择**
