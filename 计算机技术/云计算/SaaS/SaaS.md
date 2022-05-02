# SaaS

软件即服务，或简称 SaaS，是一种用于向用户提供软件的基于云的方法。SaaS 用户可以订阅应用程序，而无需一次性购买并安装应用程序。用户可以在任何兼容设备上通过 Internet 登录和使用 SaaS 应用程序。实际的应用程序可能在远离用户位置的[云](https://www.cloudflare.com/learning/cloud/what-is-the-cloud/ "云")服务器中运行。

![](https://cf-assets.www.cloudflare.com/slt3lc6tev37/lFmdhi3Yhkb9IoMGcviQd/20f9e97bc317146a3a2d7793f3a586a8/saas-application-vs-on-premises.svg)

SaaS 应用程序可以通过浏览器或应用程序访问。用户通过浏览器访问的在线电子邮件应用程序（例如 Gmail 和 Office 365）就是 SaaS 应用程序的常见示例。

SaaS 与安装在用户计算机上的软件之间的区别有点像流播放电视节目和购买全部整季电视节目 DVD 之间的区别。

![](https://cf-assets.www.cloudflare.com/slt3lc6tev37/nYGOTtGurrQP09eqHsPPc/061e3045461d9fcea702db8ae9a45a67/saas-application-remote-access.svg)

购买 DVD 电视节目的人只需支付一次；但是，他们将需要存储和维护 DVD，并且如果他们改变了硬件（例如，如果他们将 DVD 播放器替换为蓝光播放器），那么他们将需要再次购买物理媒体。而流播放节目则意味着第三方处理所有存储和升级，而用户所需要做的就是按下播放按钮即可。但是，流播放依赖于互联网连接，用户通常需要支付持续性的月租费才能维持其访问权限。

## “即服务”是什么意思？

可以类比代客泊车与租用停车位之间的区别。代客泊车是一项服务，而停车位是一种产品，即使两者都能为客户带来相同的利益：一个可以停车的地方。

传统上，软件供应商将其软件作为产品出售给用户。但是，在 SaaS 模型中，他们通过云为用户积极提供和维护软件。他们托管和维护运行应用程序所需的数据库和代码，并且在服务器上运行该应用程序。因此，SaaS 更像是服务而不是产品。

## 什么是云？

“云”是指各种数据中心（用于托管数据库和运行应用程序代码）中的远程 Web 服务器。云提供商通过 Internet 向客户或最终用户提供服务。（请参见[什么是云](https://www.cloudflare.com/learning/cloud/what-is-the-cloud/ "什么是云")？）

## 三种主要的云服务模型是什么？

![](https://www.cloudflare.com/img/learning/serverless/glossary/platform-as-a-service-paas/saas-paas-iaas-cloud-pyramid.svg)

SaaS 是三大主要云服务模型之一。云服务模型是云提供商（即拥有和运营在各个数据中心的服务器的公司）向用户和企业提供的服务类别。

三种云服务模型是：

*   [**IaaS (Infrastructure-as-a-Service)**](https://www.cloudflare.com/learning/cloud/what-is-iaas/ "IaaS (Infrastructure-as-a-Service)"): Cloud computing infrastructure – servers, databases, etc. – that a cloud provider manages. Companies can build their own applications on IaaS instead of maintaining their applications' backends themselves.

*   [**PaaS（平台即服务）**](https://www.cloudflare.com/learning/serverless/glossary/platform-as-a-service-paas/ "PaaS（平台即服务）")：PaaS 比 IaaS 高出一个级别，它包括开发工具、基础设施以及对构建应用程序的其他支持。

*   **SaaS（软件即服务）**：完整构建的云应用程序。

## 使用 SaaS 的优点和缺点是什么？

SaaS 模型具有许多优点和缺点，尽管对于现代企业和用户而言，SaaS 的优点通常要大于缺点。以下是使用 SaaS 应用程序的一些优点和缺点：

*   **可从任意位置使用任意设备访问**。通常，用户可以从任意位置、通过任意设备登录 SaaS 应用程序。这提供了很大的灵活性。企业可以让员工更好地在全球范围内开展业务，并且用户无论身在何处都可以访问他们的文件。此外，大多数用户会使用多台设备并经常更换设备；用户在每次换用新设备时，无需重新安装 SaaS 应用程序或购买新许可证。

*   **优势：无需更新或安装** SaaS 提供商会不断更新和修补应用程序。

*   **优势：可扩展性**  SaaS 提供商负责处理应用程序的扩展，例如，随着使用量的增加而增加更多数据库空间或提升计算能力。

*   **优势：节省成本**。SaaS 有助于企业削减内部 IT 成本和开销。SaaS 提供商负责维护支持应用程序的服务器和基础设施，而企业的唯一成本就是需要付费订阅应用程序。

*   **需要进行更强有力的访问控制**。SaaS 应用程序的可访问性更强，这也意味着验证用户身份和[控制访问级别](https://www.cloudflare.com/learning/security/what-is-access-control/ "控制访问级别")就显得非常重要。使用 SaaS 后，企业资产不再保持在内部网络中，也不再与外界隔离。相反，用户访问权限是基于用户身份进行分配：如果某人具有正确的登录凭据，则授予他们访问权限。因此，强身份验证就变得至关重要。

*   **缺点：供应商绑定**。企业可能会过度依赖 SaaS 应用程序提供商。如果企业的整个数据库都存储在旧应用程序中，则迁移到新应用程序既耗时又昂贵。

*   **缺点（对于企业而言）：安全与合规问题**。就 SaaS 应用程序而言，保护这些应用程序及其数据的责任从企业内部 IT 团队转移给了外部 SaaS 提供商。对于中小型企业来说，这不算是缺点，因为大型云提供商通常有更多资源，可以确保强大的安全性。但是，如果大型企业需要遵循严格的安全性或监管标准，这就可能是一个挑战。在某些情况下，企业可能无法通过执行[渗透测试](https://www.cloudflare.com/learning/security/glossary/what-is-penetration-testing/ "渗透测试")等方法自行评估其应用程序的安全性。基本上，他们需要从外部 SaaS 提供商口中了解应用程序的安全性如何。

## SaaS 公司有哪些示例？

如上所述，在线电子邮件提供商属于 SaaS 类别。其他知名的 SaaS 公司包括 Salesforce、Slack、MailChimp 和 Dropbox。

## 参考

*   [https://www.cloudflare.com/zh-cn/learning/cloud/what-is-saas/](https://www.cloudflare.com/zh-cn/learning/cloud/what-is-saas/ "https://www.cloudflare.com/zh-cn/learning/cloud/what-is-saas/")
