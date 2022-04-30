# Kubernetes 监控体系

# 基本概念

### cAdvisor

![](https://tva1.sinaimg.cn/large/008i3skNly1gwfv8lvm2yj31it0o2q4h.jpg)

Docker 是一个开源的应用容器引擎，让开发者可以打包他们的应用以及依赖包到一个可移植的容器中，然后发布到任何流行的 Linux/Windows/Mac 机器上。容器镜像正成为一个新的标准化软件交付方式。为了能够获取到 Docker 容器的运行状态，用户可以通过 Docker 的 stats 命令获取到当前主机上运行容器的统计信息，可以查看容器的 CPU 利用率、内存使用量、网络 IO 总量以及磁盘 IO 总量等信息。

显然如果我们想对监控数据做存储以及可视化的展示，那么 docker 的 stats 是不能满足的。

为了解决 docker stats 的问题（存储、展示），谷歌开源的 cadvisor 诞生了，cadvisor 不仅可以搜集一台机器上所有运行的容器信息，还提供基础查询界面和 http 接口，方便其他组件如 Prometheus 进行数据抓取，或者 cAdvisor + influxDB + grafana 搭配使用。

cAdvisor 可以对节点机器上的资源及容器进行实时监控和性能数据采集，包括 CPU 使用情况、内存使用情况、网络吞吐量及文件系统使用情况

**监控原理**

cAdvisor 使用 Go 语言开发，利用 Linux 的 cgroups 获取容器的资源使用信息，在 K8S 中集成在 Kubelet 里作为默认启动项，官方标配。

Docker 是基于 Namespace、Cgroups 和联合文件系统实现的

Cgroups 不仅可以用于容器资源的限制，还可以提供容器的资源使用率。不管用什么监控方案，底层数据都来源于 Cgroups

Cgroups 的工作目录 /sys/fs/cgroup 下包含了 Cgroups 的所有内容。Cgroups 包含了很多子系统，可以对 CPU，内存，PID，磁盘 IO 等资源进行限制和监控。

cAdvisor 运行原理，如下图

![](https://tva1.sinaimg.cn/large/008i3skNly1gwfvbelfamj30u0129gr5.jpg)

### Prometheus

![](https://tva1.sinaimg.cn/large/008i3skNly1gwfxqhgv1mj30ku095mxf.jpg)

Prometheus 是一套开源的监控报警系统。主要特点包括多维数据模型、灵活查询语句 PromQL 以及数据可视化展示等

**架构图**

![](https://tva1.sinaimg.cn/large/008i3skNly1gwfyo8qx6dj311j0mjju0.jpg)

**基本原理**

Prometheus 的基本原理是通过 HTTP 协议周期性抓取被监控组件的状态，任意组件只要提供对应的 HTTP 接口就可以接入监控。不需要任何 SDK 或者其他的集成过程。这样做非常适合做虚拟化环境监控系统，比如 VM、Docker、Kubernetes 等。输出被监控组件信息的 HTTP 接口被叫做 exporter 。目前互联网公司常用的组件大部分都有 exporter 可以直接使用，比如 Varnish、Haproxy、Nginx、MySQL、Linux 系统信息（包括磁盘、内存、CPU、网络等等）。

**服务过程**

*   Prometheus Daemon 负责定时去目标上抓取 metrics（指标）数据，每个抓取目标需要暴露一个 http 服务的接口给它定时抓取。Prometheus 支持通过配置文件、文本文件、Zookeeper、Consul、DNS SRV Lookup 等方式指定抓取目标。Prometheus 采用 PULL 的方式进行监控，即服务器可以直接通过目标 PULL 数据或者间接地通过中间网关来 Push 数据。

*   Prometheus 在本地存储抓取的所有数据，并通过一定规则进行清理和整理数据，并把得到的结果存储到新的时间序列中。

*   Prometheus 通过 PromQL 和其他 API 可视化地展示收集的数据。Prometheus 支持很多方式的图表可视化，例如 Grafana、自带的 Promdash 以及自身提供的模版引擎等等。Prometheus 还提供 HTTP API 的查询方式，自定义所需要的输出。

*   PushGateway 支持 Client 主动推送 metrics 到 PushGateway，而 Prometheus 只是定时去 Gateway 上抓取数据。

*   Alertmanager 是独立于 Prometheus 的一个组件，可以支持 Prometheus 的查询语句，提供十分灵活的报警方式。

### Operator

Operator 是 CoreOS 推出的旨在**简化复杂有状态应用管理的框架**，它是一个感知应用状态的控制器，通过扩展 Kubernetes API 来自动创建、管理和配置应用实例。

Operator 基于 CustomResourceDefinition(CRD) 扩展了新的应用资源，并通过控制器来保证应用处于预期状态。比如 etcd operator 通过下面的三个步骤模拟了管理 etcd 集群的行为：

1.  通过 Kubernetes API 观察集群的当前状态；

2.  分析当前状态与期望状态的差别；

3.  调用 etcd 集群管理 API 或 Kubernetes API 消除这些差别。

![](https://tva1.sinaimg.cn/large/008i3skNly1gwg0p6nbprj31eq0lc76s.jpg)

### Prometheus Operator

为了在 Kubernetes 能够方便的管理和部署 Prometheus，我们使用 ConfigMap 了管理 Prometheus 配置文件。每次对 Prometheus 配置文件进行升级时，我们需要手动移除已经运行的 Pod 实例，从而让 Kubernetes 可以使用最新的配置文件创建 Prometheus。 而如果当应用实例的数量更多时，通过手动的方式部署和升级 Prometheus 过程繁琐并且效率低下。

从本质上来讲 Prometheus 属于是典型的有状态应用，而其又包含了一些自身特有的运维管理和配置管理方式。而这些都无法通过 Kubernetes 原生提供的应用管理概念实现自动化。为了简化这类应用程序的管理复杂度，CoreOS 率先引入了 Operator 的概念，并且首先推出了针对在 Kubernetes 下运行和管理 Etcd 的 Etcd Operator。并随后推出了 Prometheus Operator。

从概念上来讲 Operator 就是针对管理特定应用程序的，在 Kubernetes 基本的 Resource 和 Controller 的概念上，以扩展 Kubernetes api 的形式。帮助用户创建，配置和管理复杂的有状态应用程序。从而实现特定应用程序的常见操作以及运维自动化。

在 Kubernetes 中我们使用 Deployment、DamenSet，StatefulSet 来管理应用 Workload，使用 Service，Ingress 来管理应用的访问方式，使用 ConfigMap 和 Secret 来管理应用配置。我们在集群中对这些资源的创建，更新，删除的动作都会被转换为事件 (Event)，Kubernetes 的 Controller Manager 负责监听这些事件并触发相应的任务来满足用户的期望。这种方式我们成为声明式，用户只需要关心应用程序的最终状态，其它的都通过 Kubernetes 来帮助我们完成，通过这种方式可以大大简化应用的配置管理复杂度。

而除了这些原生的 Resource 资源以外，Kubernetes 还允许用户添加自己的自定义资源 (Custom Resource)。并且通过实现自定义 Controller 来实现对 Kubernetes 的扩展。

如下所示，是 Prometheus Operator 的架构示意图：

![](https://tva1.sinaimg.cn/large/008i3skNly1gwg0u0uxedj31dw0t2mzu.jpg)

Prometheus 的本职就是一组用户自定义的 CRD 资源以及 Controller 的实现，Prometheus Operator 负责监听这些自定义资源的变化，并且根据这些资源的定义自动化的完成如 Prometheus Server 自身以及配置的自动化管理工作。

**简言之，Prometheus Operator 能够帮助用户自动化的创建以及管理 Prometheus Server 以及其相应的配置。**

### HPA

Horizontal Pod Autoscaler ，K8S 中的一个概念，可以自动调整 Pod 的数量，以达到指定的目标值。

Pod 水平自动扩缩（Horizontal Pod Autoscaler） 可以基于 CPU 利用率自动扩缩 ReplicationController、Deployment、ReplicaSet 和 StatefulSet 中的 Pod 数量。 除了 CPU 利用率，也可以基于其他应程序提供的 `自定义度量指标 `来执行自动扩缩。 Pod 自动扩缩不适用于无法扩缩的对象，比如 DaemonSet。

![](https://tva1.sinaimg.cn/large/008i3skNly1gwfy2haq8kj305k046wea.jpg)

### Heapster

Heapster 是容器集群监控和性能分析工具，天然的支持 Kubernetes 和 CoreOS。

Heapster 首先从 K8S Master 获取集群中所有 Node 的信息，然后通过这些 Node 上的 kubelet 获取有用数据，而 kubelet 本身的数据则是从 cAdvisor 得到。所有获取到的数据都被推到 Heapster 配置的后端存储中，并还支持数据的可视化。现在后端存储 + 可视化的方法，如 InfluxDB + grafana。

Heapster 可以收集 Node 节点上的 cAdvisor 数据，还可以按照 kubernetes 的资源类型来集合资源，比如 Pod、Namespace 域，可以分别获取它们的 CPU、内存、网络和磁盘的 metric。默认的 metric 数据聚合时间间隔是 1 分钟。

注意 ：**Kubernetes 1.11 不建议使用 Heapster**，就 SIG Instrumentation 而言，这是为了转向新的 Kubernetes 监控模型的持续努力的一部分。**仍使用 Heapster 进行自动扩展的集群应迁移到 metrics-server 和自定义指标 API**。

![](https://tva1.sinaimg.cn/large/008i3skNly1gwfyt4e4aej30g806m0su.jpg)

### Metrics Server

![](https://tva1.sinaimg.cn/large/008i3skNly1gwfz2g8rahj315o0j1776.jpg)

kubernetes 集群资源监控之前可以通过 `heapster` 来获取数据，在 `1.11` 开始开始逐渐废弃 `heapster` 了，采用 `metrics-server` 来代替，`metrics-server` 是集群的核心监控数据的聚合器，它从 kubelet 公开的 Summary API 中采集指标信息，`metrics-server` 是扩展的 APIServer，依赖于 kube-aggregator，因为我们需要在 APIServer 中开启相关参数。

`Metrics Server` 并不是 kube-apiserver 的一部分，而是通过 Aggregator 这种插件机制，在独立部署的情况下同 kube-apiserver 一起统一对外服务的。

**Aggregator**

> **通过聚合层扩展 Kubernetes API**

使用聚合层（Aggregation Layer），用户可以通过额外的 API 扩展 Kubernetes， 而不局限于 Kubernetes 核心 API 提供的功能。这里的附加 API 可以是现成的解决方案比如 metrics server, 或者你自己开发的 API。

聚合层不同于 定制资源（Custom Resources）。 后者的目的是让 kube-apiserver 能够认识新的对象类别（Kind）。

> **聚合层**

聚合层在 kube-apiserver 进程内运行。在扩展资源注册之前，聚合层不做任何事情。 要注册 API，用户必须添加一个 APIService 对象，用它来“申领” Kubernetes API 中的 URL 路径。 自此以后，聚合层将会把发给该 API 路径的所有内容（例如 /apis/myextension.mycompany.io/v1/…） 转发到已注册的 APIService。

> APIService 的最常见实现方式是在集群中某 Pod 内运行 扩展 API 服务器。 如果你在使用扩展 API 服务器来管理集群中的资源，该扩展 API 服务器（也被写成“extension-apiserver”） 一般需要和一个或多个控制器一起使用。 apiserver-builder 库同时提供构造扩展 API 服务器和控制器框架代码。

这里，Aggregator APIServer 的工作原理，可以用如下所示的一幅示意图来表示清楚 ：

![](https://tva1.sinaimg.cn/large/008i3skNly1gwfz5teuxdj30jc09paaf.jpg)

因为 k8s 的 api-server 将所有的数据持久化到了 etcd 中，显然 k8s 本身不能处理这种频率的采集，而且这种监控数据变化快且都是临时数据，因此需要有一个组件单独处理他们，于是 metric-server 的概念诞生了。

Metrics server 出现后，新的 Kubernetes 监控架构将变成下图的样子

*   核心流程（黑色部分）：这是 Kubernetes 正常工作所需要的核心度量，从 Kubelet、cAdvisor 等获取度量数据，再由 metrics-server 提供给 Dashboard、HPA 控制器等使用。

*   监控流程（蓝色部分）：基于核心度量构建的监控流程，比如 Prometheus 可以从 metrics-server 获取核心度量，从其他数据源（如 Node Exporter 等）获取非核心度量，再基于它们构建监控告警系统。

![](https://tva1.sinaimg.cn/large/008i3skNly1gwfzb025saj30kc0f90u3.jpg)

注意：

*   metrics-sevrer 的数据存在内存中。

*   metrics-server 主要针对 node、pod 等的 cpu、网络、内存等系统指标的监控

### kube-state-metrics

已经有了 cadvisor、heapster、metric-server，几乎容器运行的所有指标都能拿到，但是下面这种情况却无能为力：

*   我调度了多少个 replicas？现在可用的有几个？

*   多少个 Pod 是 running/stopped/terminated 状态？

*   Pod 重启了多少次？

*   我有多少 job 在运行中

而这些则是 kube-state-metrics 提供的内容，它基于 client-go 开发，轮询 Kubernetes API，并将 Kubernetes 的结构化信息转换为 metrics。

**kube-state-metrics 与 metrics-server 对比**

我们服务在运行过程中，我们想了解服务运行状态，pod 有没有重启，伸缩有没有成功，pod 的状态是怎么样的等，这时就需要 kube-state-metrics，它主要关注 deployment,、node 、 pod 等内部对象的状态。而 metrics-server 主要用于监测 node，pod 等的 CPU，内存，网络等系统指标。

*   metric-server（或 heapster）是从 api-server 中获取 cpu、内存使用率这种监控指标，并把他们发送给存储后端，如 influxdb 或云厂商，他当前的核心作用是：为 HPA 等组件提供决策指标支持。

*   kube-state-metrics 关注于获取 k8s 各种资源的最新状态，如 deployment 或者 daemonset，之所以没有把 kube-state-metrics 纳入到 metric-server 的能力中，是因为他们的关注点本质上是不一样的。**metric-server 仅仅是获取、格式化现有数据，写入特定的存储，实质上是一个监控系统。而 kube-state-metrics 是将 k8s 的运行状况在内存中做了个快照，并且获取新的指标，但他没有能力导出这些指标**

*   换个角度讲，kube-state-metrics 本身是 metric-server 的一种数据来源，虽然现在没有这么做。

*   另外，像 Prometheus 这种监控系统，并不会去用 metric-server 中的数据，他都是自己做指标收集、集成的（Prometheus 包含了 metric-server 的能力），但 Prometheus 可以监控 metric-server 本身组件的监控状态并适时报警，这里的监控就可以通过 kube-state-metrics 来实现，如 metric-serverpod 的运行状态。

![](https://tva1.sinaimg.cn/large/008i3skNly1gwfzwltdicj30gy08l0t2.jpg)

### custom-metrics-apiserver

kubernetes 的监控指标分为两种

*   Core metrics（核心指标）：从 Kubelet、cAdvisor 等获取度量数据，再由 metrics-server 提供给 Dashboard、HPA 控制器等使用。

*   Custom Metrics（自定义指标）：由 Prometheus Adapter 提供 API [custom.metrics.k8s.io](http://custom.metrics.k8s.io "custom.metrics.k8s.io")，由此可支持任意 Prometheus 采集到的指标。

以下是官方 metrics 的项目介绍：

Resource Metrics API（核心 api）

*   Heapster

*   Metrics Server

Custom Metrics API：

*   Prometheus Adapter

*   Microsoft Azure Adapter

*   Google Stackdriver

*   Datadog Cluster Agent

核心指标只包含 node 和 pod 的 cpu、内存等，一般来说，核心指标作 HPA 已经足够，但如果想根据自定义指标：如请求 qps/5xx 错误数来实现 HPA，就需要使用自定义指标了，目前 Kubernetes 中自定义指标一般由 Prometheus 来提供，再利用 `k8s-prometheus-adpater` 聚合到 apiserver，实现和核心指标（metric-server) 同样的效果。

HPA 请求 metrics 时，kube-aggregator(apiservice 的 controller) 会将请求转发到 adapter，adapter 作为 kubernentes 集群的 pod，实现了 Kubernetes resource metrics API 和 custom metrics API，它会根据配置的 rules 从 Prometheus 抓取并处理 metrics，在处理（如重命名 metrics 等）完后将 metric 通过 custom metrics API 返回给 HPA。最后 HPA 通过获取的 metrics 的 value 对 Deployment/ReplicaSet 进行扩缩容。

adapter 作为 extension-apiserver（即自己实现的 pod)，充当了代理 kube-apiserver 请求 Prometheus 的功能。

![](https://tva1.sinaimg.cn/large/008i3skNly1gwg06qeh3zj311u0f5wfy.jpg)

**其实 k8s-prometheus-adapter 既包含自定义指标，又包含核心指标，即如果安装了 prometheus，且指标都采集完整，k8s-prometheus-adapter 可以替代 metrics server。**

## Prometheus 部署方案

prometheus operator

*   [https://github.com/prometheus-operator/prometheus-operator](https://github.com/prometheus-operator/prometheus-operator "https://github.com/prometheus-operator/prometheus-operator")

kube-prometheus

*   [https://github.com/prometheus-operator/kube-prometheus](https://github.com/prometheus-operator/kube-prometheus "https://github.com/prometheus-operator/kube-prometheus")

在集群外部署

*   [https://www.qikqiak.com/post/monitor-external-k8s-on-prometheus/](https://www.qikqiak.com/post/monitor-external-k8s-on-prometheus/ "https://www.qikqiak.com/post/monitor-external-k8s-on-prometheus/")

kube-prometheus 既包含了 Operator，又包含了 Prometheus 相关组件的部署及常用的 Prometheus 自定义监控，具体包含下面的组件

*   The Prometheus Operator：创建 CRD 自定义的资源对象

*   Highly available Prometheus：创建高可用的 Prometheus

*   Highly available Alertmanager：创建高可用的告警组件

*   Prometheus node-exporter：创建主机的监控组件

*   Prometheus Adapter for Kubernetes Metrics APIs：创建自定义监控的指标工具（例如可以通过 nginx 的 request 来进行应用的自动伸缩）

*   kube-state-metrics：监控 k8s 相关资源对象的状态指标

*   Grafana：进行图像展示

## 我们的做法

我们的做法，其实跟 kube-prometheus 的思路差不多，只不过我们没有用 Operator ，是自己将以下这些组件的 yaml 文件用 helm 组织了起来而已：

*   kube-state-metrics

*   prometheus

*   alertmanager

*   grafana

*   k8s-prometheus-adapter

*   node-exporter

当然 kube-prometheus 也有 helm charts 由 prometheus 社区提供：[https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack "https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack")

这么干的原因是：这样的灵活度是最高的，虽然在第一次初始化创建这些脚本的时候麻烦了些。 不过还有一个原因是我们当时部署整个基于 prometheus 的监控体系时，kube-prometheus 这个项目还在早期，没有引起我们的关注。如果在 2021 年年初或 2020 年年底的时候创建的话，可能就会直接上了。

## 参考

*   [https://blog.opskumu.com/cadvisor.html](https://blog.opskumu.com/cadvisor.html "https://blog.opskumu.com/cadvisor.html")

*   [https://prometheus.io/](https://prometheus.io/ "https://prometheus.io/")

*   [https://kubernetes.io/zh/docs/tasks/run-application/horizontal-pod-autoscale/](https://kubernetes.io/zh/docs/tasks/run-application/horizontal-pod-autoscale/ "https://kubernetes.io/zh/docs/tasks/run-application/horizontal-pod-autoscale/")

*   [https://www.cnblogs.com/chenqionghe/p/10494868.html](https://www.cnblogs.com/chenqionghe/p/10494868.html "https://www.cnblogs.com/chenqionghe/p/10494868.html")

*   [https://www.qikqiak.com/post/k8s-operator-101/](https://www.qikqiak.com/post/k8s-operator-101/ "https://www.qikqiak.com/post/k8s-operator-101/")

*   [https://kubernetes.io/zh/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/](https://kubernetes.io/zh/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/ "https://kubernetes.io/zh/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/")

*   [https://segmentfault.com/a/1190000017875641](https://segmentfault.com/a/1190000017875641 "https://segmentfault.com/a/1190000017875641")

*   [https://segmentfault.com/a/1190000038888544](https://segmentfault.com/a/1190000038888544 "https://segmentfault.com/a/1190000038888544")

*   [https://yasongxu.gitbook.io/](https://yasongxu.gitbook.io/ "https://yasongxu.gitbook.io/")

*   [https://mp.weixin.qq.com/s/p4FAFKHi8we4mrD7OIk7IQ](https://mp.weixin.qq.com/s/p4FAFKHi8we4mrD7OIk7IQ "https://mp.weixin.qq.com/s/p4FAFKHi8we4mrD7OIk7IQ")

*   [https://kubernetes.feisky.xyz/apps/index/operator](https://kubernetes.feisky.xyz/apps/index/operator "https://kubernetes.feisky.xyz/apps/index/operator")

*   [https://yunlzheng.gitbook.io/prometheus-book/part-iii-prometheus-shi-zhan/operator/what-is-prometheus-operator](https://yunlzheng.gitbook.io/prometheus-book/part-iii-prometheus-shi-zhan/operator/what-is-prometheus-operator "https://yunlzheng.gitbook.io/prometheus-book/part-iii-prometheus-shi-zhan/operator/what-is-prometheus-operator")
