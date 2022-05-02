# 详解 iptables

## 基本概念

### iptables 是什么？

在 **netfilter** 的 [官网](https://www.netfilter.org/projects/iptables/index.html "官网") 找到的如下解释：

> **iptables** is the userspace command line program used to configure the Linux 2.4.x and later packet filtering ruleset. It is targeted towards system administrators.
>
> Since Network Address Translation is also configured from the packet filter ruleset, **iptables** is used for this, too.
>
> The **iptables** package also includes **ip6tables. ip6tables** is used for configuring the IPv6 packet filter.

*   iptables 是用于配置 Linux 2.4.x 及更高版本包过滤规则集的用户空间命令行程序。 它针对系统管理员。

*   由于网络地址转换 (NAT) 也是从包过滤规则集配置的，iptables 也用于此。

*   iptables 包还包括 ip6tables。 ip6tables 用于配置 IPv6 包过滤器。

iptables 源码地址： [https://git.netfilter.org/iptables](https://git.netfilter.org/iptables "https://git.netfilter.org/iptables")

### netfilter 是什么？

来自维基百科的解释：

> netfilter，在 Linux 内核中的一个软件框架，用于管理网络数据包。不仅具有网络地址转换（NAT）的功能，也具备数据包内容修改、以及数据包过滤等防火墙功能。利用运作于用户空间的应用软件，如 iptables、nftables、ebtables 和 arptables 等，来控制 netfilter，系统管理者可以管理通过 Linux 操作系统的各种网络数据包。1990 年代，netfilter 在 Linux 2.3.15 版时进入 Linux 内核，正式应用于 Linux 2.4 版。

netfilter 的主要功能包括：

*   网络地址转换 (Network Address Translate)

*   数据包内容修改

*   以及数据包过滤的防火墙功能

linux 的绝大多数功能都是以模块的形式扩充出来的，netfilter 也是以模块的形式存在于 linux 中，当 linux 多了一个 netfilter 模块，linux 防火墙功能也就多了一项。

netfilter 本身并不对数据包进行过滤，它只是允许过滤的数据包的函数挂接到内核中合适的位置。netfilter 项目在内核中还提供了一些基础设施，比如链接跟踪和日志记录，任何 iptables 策略都可以使用这些设施来执行特定数据包的处理。

netfilter 模块存放的目录：

*   `/lib/modules/<uname -r>/kernel/net/ipv4/netfilter/`

*   `/lib/modules/<uname -r>/kernel/net/ipv6/netfilter/`

不仅是 netfilter 有模块，iptables 也有模块，这些模块就位于`/lib64/xtables/`(32bit 系统在`/lib/xtables/`) 目录下，其中以 libxt 开头的是 iptables 模块，这些模块与 netfilter 模块是一一相对应的。例如`/lib/modules/<uname -r>/kernel/net/netfilter/xt_conntrack.ko`模块，在`/lib64/xtables/libxt_conntrack.so`与之相对应。当下达与 xt\_conntrack.ko 相关的指令时，iptables 会根据 libxt\_conntrack.so 模块的指示去检查语法是否正确。并将 netfilter 相应模块载入到系统内存，iptables 最后将规则写入到规则数据库中。

### netfilter 和 iptables 是什么关系？

在很多场景下，大家用 iptabes 配置防火墙规则，而实际上 iptables 其实不是真正的防火墙，我们可以把它理解成一个客户端代理，用户通过 iptables 这个代理，将用户的安全设定执行到对应的”安全框架”中，这个”安全框架”才是真正的防火墙，这个框架的名字叫 netfilter

*   **netfilter** 才是防火墙真正的安全框架（framework），netfilter 位于内核空间。

*   **iptables** 其实是一个命令行工具，位于用户空间，我们用这个工具操作真正的框架。

## iptables 基础概念

### 链

iptables 在普遍的应用场景中被用作配置防火墙，如果我们想要防火墙能够达到”防火”的目的，则需要在内核中设置关卡，所有进出的报文都要通过这些关卡，经过检查后，符合放行条件的才能放行，符合阻拦条件的则需要被阻止，于是，就出现了 input 关卡和 output 关卡，但是，这个关卡上可能不止有一条规则，而是有很多条规则，当我们把这些规则串到一个链条上的时候，就形成了”链”。

![](https://tva1.sinaimg.cn/large/008i3skNly1gu16dwdasbj60nu0cn3zh02.jpg)

总结下 5 链：

*   PREROUTING 数据包刚进入网络层 , 路由之前

*   INPUT 路由判断，流入用户空间

*   OUTPUT 用户空间发出，后接路由判断出口的网络接口

*   FORWARD 路由判断不进入用户空间，只进行转发

*   POSTROUTING 数据包通过网络接口出去

根据实际情况的不同，报文经过”链”可能不同。如果报文需要转发，那么报文则不会经过 input 链发往用户空间，而是直接在内核空间中经过 forward 链和 postrouting 链转发出去的。

![](https://tva1.sinaimg.cn/large/008i3skNly1gu161iott6j61bk0r677302.jpg)

所以，根据上图，我们能够想象出某些常用场景中，报文的流向：

*   到本机某进程的报文：PREROUTING –> INPUT

*   由本机转发的报文：PREROUTING –> FORWARD –> POSTROUTING

*   由本机的某进程发出报文（通常为响应报文）：OUTPUT –> POSTROUTING

每个经过这个”关卡”的报文，都要将这条”链”上的所有规则匹配一遍，如果有符合条件的规则，则执行规则对应的动作。

### 表

为什么称为 ip"tables" 呢？ 因为这个防火墙软件里面有多个表格 (table) ，每个表格都定义出自己的默认政策与规则， 且每个表格的用途都不相同。每个“表”指的是不同类型的数据包处理流程。

预设的情况下，Linux 的 iptables 至少就有三个表。

![](https://tva1.sinaimg.cn/large/008i3skNly1gu170wsidrj60ja0a6mxw02.jpg)

### 表链关系

每个”链”上都放置了一串规则，但是这些规则有些很相似，比如，A 类规则都是对 IP 或者端口的过滤，B 类规则是修改报文。我们是不是能把实现相同功能的规则放在一起呢？可以的，我们把具有相同功能的规则的组成一个集合，也就是上文说的“表”。

![](https://tva1.sinaimg.cn/large/008i3skNly1gu17ia8zioj60fa08u0ti02.jpg)

iptables 为我们提供了如下规则的分类，或者说，iptables 为我们提供了如下”表”

*   filter 表：负责过滤功能，防火墙；内核模块：iptables\_filter

*   nat 表：network address translation，网络地址转换功能；内核模块：iptable\_nat

*   mangle 表：拆解报文，做出修改，并重新封装 的功能；iptable\_mangle

*   raw 表：关闭 nat 表上启用的连接追踪机制；iptable\_raw

我们自定义的所有规则，都是这四种分类中的规则，或者说，所有规则都存在于这 4 张”表”中

具体来说：

| 链                    | 表                                                 |
| -------------------- | ------------------------------------------------- |
| PREROUTING 的规则可以存在于  | raw 表，mangle 表，nat 表。                             |
| INPUT 的规则可以存在于       | mangle 表，filter 表，（centos7 中还有 nat 表，centos6 中没有） |
| FORWARD 的规则可以存在于     | mangle 表，filter 表。                                |
| OUTPUT 的规则可以存在于      | raw 表 mangle 表，nat 表，filter 表。                    |
| POSTROUTING 的规则可以存在于 | mangle 表，nat 表。                                   |

**我们在实际的使用过程中，往往是通过”表”作为操作入口，对规则进行定义的**，之所以按照上述过程介绍 iptables，是因为从”关卡”的角度更容易从入门的角度理解，但是为了以便在实际使用的时候，更加顺畅的理解它们，此处我们还要将各”表”与”链”的关系罗列出来：

| 表（功能）  | 链（钩子）：                                                                     |
| ------ | -------------------------------------------------------------------------- |
| raw    | 表中的规则可以被哪些链使用：PREROUTING，OUTPUT                                            |
| mangle | 表中的规则可以被哪些链使用：PREROUTING，INPUT，FORWARD，OUTPUT，POSTROUTING                  |
| nat    | 表中的规则可以被哪些链使用：PREROUTING，OUTPUT，POSTROUTING（centos7 中还有 INPUT，centos6 中没有） |
| filter | 表中的规则可以被哪些链使用：INPUT，FORWARD，OUTPUT                                         |

#### 优先级

数据包经过一个”链”的时候，会将当前链的所有规则都匹配一遍，但是匹配时总归要有顺序，我们应该一条一条的去匹配，而且相同功能类型的规则会汇聚在一张”表”中，哪些”表”中的规则会放在”链”的最前面执行呢？这时候就需要有一个优先级的问题

优先级次序（由高而低）：

raw –> mangle –> nat –> filter

#### 数据经过防火墙的流程

![](https://tva1.sinaimg.cn/large/008i3skNly1gu199o4ajkj61ne0u041q02.jpg)

### 规则

规则：根据指定的匹配条件来尝试匹配每个流经此处的报文，一旦匹配成功，则由规则后面指定的处理动作进行处理；

规则由匹配条件和处理动作组成。

#### 匹配条件

匹配条件分为基本匹配条件与扩展匹配条件

*   基本匹配条件： &#x20;

    源地址 Source IP，目标地址 Destination IP &#x20;

    上述内容都可以作为基本匹配条件。

*   扩展匹配条件： &#x20;

    除了上述的条件可以用于匹配，还有很多其他的条件可以用于匹配，这些条件泛称为扩展条件，这些扩展条件其实也是 netfilter 中的一部分，只是以模块的形式存在，如果想要使用这些条件，则需要依赖对应的扩展模块。 &#x20;

    源端口 Source Port, 目标端口 Destination Port 可以作为扩展匹配条件

#### 处理动作

处理动作在 iptables 中被称为 target（这样说并不准确，我们暂且这样称呼），动作也可以分为基本动作和扩展动作。 &#x20;
此处列出一些常用的动作，之后的文章会对它们进行详细的示例与总结：

*   ACCEPT：允许数据包通过。

*   DROP：直接丢弃数据包，不给任何回应信息，这时候客户端会感觉自己的请求泥牛入海了，过了超时时间才会有反应。

*   REJECT：拒绝数据包通过，必要时会给数据发送端一个响应的信息，客户端刚请求就会收到拒绝的信息。

*   SNAT：源地址转换，解决内网用户用同一个公网地址上网的问题。

*   MASQUERADE：是 SNAT 的一种特殊形式，适用于动态的、临时会变的 ip 上。

*   DNAT：目标地址转换。

*   REDIRECT：在本机做端口映射。

*   LOG：在/var/log/messages 文件中记录日志信息，然后将数据包传递给下一条规则，也就是说除了记录以外不对数据包做任何其他操作，仍然让下一条规则去匹配。

DROP 和 REJECT 的区别：DROP 是直接把匹配到的报文丢弃，REJECT 除了把报文丢弃还会给该报文中的源 IP 发一个 ICMP 报文说明目的不可达（直接回复不可达，更强硬）。前者报文发送方只能等超时，而后者发送方因为收到了 ICMP 不可达所以马上就给出了提示。

## 参考

*   [https://www.zsythink.net/archives/1199](https://www.zsythink.net/archives/1199 "https://www.zsythink.net/archives/1199")

*   [https://borosan.gitbook.io/lpic2-exam-guide/2121-configuring-a-router](https://borosan.gitbook.io/lpic2-exam-guide/2121-configuring-a-router "https://borosan.gitbook.io/lpic2-exam-guide/2121-configuring-a-router")

*   [http://cn.linux.vbird.org/linux\_server/0250simple\_firewall\_3.php](http://cn.linux.vbird.org/linux_server/0250simple_firewall_3.php "http://cn.linux.vbird.org/linux_server/0250simple_firewall_3.php")

*   [https://www.xiebruce.top/1071.html](https://www.xiebruce.top/1071.html "https://www.xiebruce.top/1071.html")

*   [http://kuring.me/post/iptables/](http://kuring.me/post/iptables/ "http://kuring.me/post/iptables/")
