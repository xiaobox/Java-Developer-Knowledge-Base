# apisix 基础知识

## APISIX

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h1rlcnzg5gj21f40u0ado.jpg)

## APISIX Ingress

### k8s ingress

使用Kubernetes Ingress来统一网络入口。Kubernetes Ingress声明了一个应用层（OSI七层）的负载均衡器，可以根据HTTP请求的内容将来自同一个TCP端口的请求分发到不同的Kubernetes Service

在k8s里，Ingress是一个可以允许集群外部访问集群内部服务的控制器。通过配置一条条规则（rules）来规定进来的连接被分配到后端的哪个服务。

引入Ingress的好处就是有一个集中的路由配置项，同时可实现基于内容的七层负载均衡

**功能包括：**

*   **按HTTP请求的URL进行路由**
    同一个TCP端口进来的流量可以根据URL路由到Cluster中的不同服务，如下图所示：

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h1rld8wzcqj20fz0bpt92.jpg)

*   **按HTTP请求的Host进行路由**
    同一个IP进来的流量可以根据HTTP请求的Host路由到Cluster中的不同服务，如下图所示：

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h1rldzf1qlj20fw0ca74n.jpg)

**华为云 ELB+K8s的实现：**[https://support.huaweicloud.com/usermanual-cce/cce\_01\_0014.html](https://support.huaweicloud.com/usermanual-cce/cce_01_0014.html "https://support.huaweicloud.com/usermanual-cce/cce_01_0014.html")

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h1rlesc8x9j21200u0n0j.jpg)

**ELB+ingress+service的实现：**

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h1rlftfjgwj20eu0ildgi.jpg)

K8S Ingress提供了一个基础的七层网关功能的抽象定义，其作用是对外提供一个七层服务的统一入口，并根据URL/HOST将请求路由到集群内部不同的服务上。

### APISIX ingress Controller

Apache APISIX Ingress Controller 通过 CRD 的方式对 Kubernetes 进行了扩展，你也可以通过发布 ApisixRoute 等这种自定义资源来完成 Kubernetes 中服务的对外暴露。

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h1rlgbuvrhj21qg0s0agq.jpg)
就不能用 dashboard 配置啦，如果裸用 apisix 是可以结合 dashboard的。

用声明式配置，说白了就是写配置文件。

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h1rlgr1b2cj20u00mjabd.jpg)

controller 和 proxy 都可以单独缩扩容。一般只对 proxy 做。
apisix 可以部署在 K8s 上，也可以单独运行在物理机上，只要能够在网络层面和 apisix ingress controller 通就行。

[https://apisix.apache.org/zh/docs/apisix/discovery](https://apisix.apache.org/zh/docs/apisix/discovery "https://apisix.apache.org/zh/docs/apisix/discovery")[https://www.apiseven.com/zh/blog/apisix-ingress-details](https://www.apiseven.com/zh/blog/apisix-ingress-details "https://www.apiseven.com/zh/blog/apisix-ingress-details")
