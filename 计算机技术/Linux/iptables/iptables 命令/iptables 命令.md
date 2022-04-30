# iptables 命令

详细文档可以查一下 man 手册

```bash
# man iptables
```

*   也可以查看在线的：[https://linux.die.net/man/8/iptables](https://linux.die.net/man/8/iptables "https://linux.die.net/man/8/iptables")

*   也有中文的：[http://linux.51yip.com/search/iptables](http://linux.51yip.com/search/iptables "http://linux.51yip.com/search/iptables")

## 增删改查

### 查看表中的规则

```bash
iptables -t filter -L

#上面的是查看 filter 表规则，也可以查看其他表的
#可以省略-t filter，当没有使用-t 选项指定表时，默认为操作 filter 表，即 iptables -L 表示列出 filter 表中的所有规则。
iptables -t raw -L
iptables -t mangle -L
iptables -t nat -L

#我们只查看 filter 表中 INPUT 链的规则
#查看详细信息 具体含义文档中有，比如 -v 参数就是查看详细信息的意思
iptables --line-numbers  -nvL INPUT
```

### 插入表中的规则

**规则的顺序很重要**

如果报文已经被前面的规则匹配到，iptables 则会对报文执行对应的动作，即使后面的规则也能匹配到当前报文，很有可能也没有机会再对报文执行相应的动作了，

```bash
# 插入规则，比如
iptables -t filter -I INPUT -s 192.168.1.146 -j DROP

# 在尾部追加
iptables -t filter -A INPUT -s 192.168.1.146 -j DROP

# 根据规则编号位置插入
iptables -t filter -A INPUT  2 -s 192.168.1.146 -j DROP

```

上面命令中

*   使用 -t 选项指定了要操作的表，此处指定了操作 filter 表，与之前的查看命令一样，不使用-t 选项指定表时，默认为操作 filter 表。

*   使用-I 选项，指明将”规则”插入至哪个链中，-I 表示 insert，即插入的意思，所以-I INPUT 表示将规则插入于 INPUT 链中，即添加规则之意。

*   使用-s 选项，指明”匹配条件”中的”源地址”，即如果报文的源地址属于-s 对应的地址，那么报文则满足匹配条件，-s 为 source 之意，表示源地址。

*   使用-j 选项，指明当”匹配条件”被满足时，所对应的动作，上例中指定的动作为 DROP，在上例中，当报文的源地址为 192.168.1.146 时，报文则被 DROP（丢弃）

### 删除表中的规则

```bash
# 按编号删除
iptables -t filter -D INPUT 3
# 根据具体的匹配条件与动作去删除规则
iptables -D INPUT -s 192.168.1.146 -j ACCEPT

# 删除指定表中某条链中的所有规则 iptables -t 表名 -F 链名
# -F 选项为 flush 之意，即冲刷指定的链，即删除指定链中的所有规则
iptables -t filter -F INPUT

# 清空整个表中所有链上的规则 iptables -t 表名 -F （慎用）
```

### 修改表中的规则

```bash
# 修改某条规则中的动作
# 注意：-s 选项以及对应的源地址不可省略
# 但是在使用-R 选项修改某个规则时，必须指定规则对应的原本的匹配条件（如果有多个匹配条件，都需要指定）
iptables -t filter -R INPUT 1 -s 192.168.1.146 -j REJECT

```

每张表的每条链中，都有自己的默认策略，我们也可以理解为默认”动作”

当报文没有被链中的任何规则匹配到时，或者，当链中没有任何规则时，防火墙会按照默认动作处理报文，我们可以修改指定链的默认策略，使用如下命令即可。

```bash
# 修改指定链的”默认策略” -P FORWARD DROP 表示将表中 FORWRD 链的默认策略改为 DROP
iptables -t filter -p FORWARD DROP
```

### 保存规则

在默认的情况下，我们对”防火墙”所做出的修改都是”临时的”，换句话说就是，当重启 iptables 服务或者重启服务器以后，我们平常添加的规则或者对规则所做出的修改都将消失，为了防止这种情况的发生，我们需要将规则”保存”。

```bash
#centos6
service iptables save

#centos7

#配置好 yum 源以后安装 iptables-service
yum install -y iptables-services
#停止 firewalld
systemctl stop firewalld
#禁止 firewalld 自动启动
systemctl disable firewalld
#启动 iptables
systemctl start iptables
#将 iptables 设置为开机自动启动，以后即可通过 iptables-service 控制 iptables 服务
systemctl enable iptables

上述配置过程只需一次，以后即可在 centos7 中愉快的使用 service iptables save 命令保存 iptables 规则了
```

## 匹配条件

### 基本匹配条件

*   \-s 用于匹配报文的源地址，可以同时指定多个源地址，每个 IP 之间用逗号隔开，也可以指定为一个网段。

```bash
#示例如下
iptables -t filter -I INPUT -s 192.168.1.111,192.168.1.118 -j DROP
iptables -t filter -I INPUT -s 192.168.1.0/24 -j ACCEPT
iptables -t filter -I INPUT ! -s 192.168.1.0/24 -j ACCEPT

```

*   \-d 用于匹配报文的目标地址，可以同时指定多个目标地址，每个 IP 之间用逗号隔开，也可以指定为一个网段。

```bash
#示例如下
iptables -t filter -I OUTPUT -d 192.168.1.111,192.168.1.118 -j DROP
iptables -t filter -I INPUT -d 192.168.1.0/24 -j ACCEPT
iptables -t filter -I INPUT ! -d 192.168.1.0/24 -j ACCEPT

```

*   \-p 用于匹配报文的协议类型，可以匹配的协议类型 tcp、udp、udplite、icmp、esp、ah、sctp 等（centos7 中还支持 icmpv6、mh）。

```bash
#示例如下
iptables -t filter -I INPUT -p tcp -s 192.168.1.146 -j ACCEPT
iptables -t filter -I INPUT ! -p udp -s 192.168.1.146 -j ACCEPT

```

*   \-i 用于匹配报文是从哪个网卡接口流入本机的，由于匹配条件只是用于匹配报文流入的网卡，所以在 OUTPUT 链与 POSTROUTING 链中不能使用此选项。

```bash
#示例如下
iptables -t filter -I INPUT -p icmp -i eth4 -j DROP
iptables -t filter -I INPUT -p icmp ! -i eth4 -j DROP
```

*   \-o 用于匹配报文将要从哪个网卡接口流出本机，于匹配条件只是用于匹配报文流出的网卡，所以在 INPUT 链与 PREROUTING 链中不能使用此选项。

```bash
#示例如下
iptables -t filter -I OUTPUT -p icmp -o eth4 -j DROP
iptables -t filter -I OUTPUT -p icmp ! -o eth4 -j DROP

```

### 扩展匹配条件

tcp 扩展模块

常用的扩展匹配条件如下：

*   \-p tcp -m tcp –sport 用于匹配 tcp 协议报文的源端口，可以使用冒号指定一个连续的端口范围

*   \-p tcp -m tcp –dport 用于匹配 tcp 协议报文的目标端口，可以使用冒号指定一个连续的端口范围

```bash
#示例如下
iptables -t filter -I OUTPUT -d 192.168.1.146 -p tcp -m tcp --sport 22 -j REJECT
iptables -t filter -I INPUT -s 192.168.1.146 -p tcp -m tcp --dport 22:25 -j REJECT
iptables -t filter -I INPUT -s 192.168.1.146 -p tcp -m tcp --dport :22 -j REJECT
iptables -t filter -I INPUT -s 192.168.1.146 -p tcp -m tcp --dport 80: -j REJECT
iptables -t filter -I OUTPUT -d 192.168.1.146 -p tcp -m tcp ! --sport 22 -j ACCEPT
```

*   –tcp-flags 用于匹配报文的 tcp 头的标志位

*   –syn 用于匹配 tcp 新建连接的请求报文，相当于使用”–tcp-flags SYN,RST,ACK,FIN  SYN”

```bash
#示例
iptables -t filter -I INPUT -p tcp -m tcp --dport 22 --tcp-flags SYN,ACK,FIN,RST,URG,PSH SYN -j REJECT
iptables -t filter -I OUTPUT -p tcp -m tcp --sport 22 --tcp-flags SYN,ACK,FIN,RST,URG,PSH SYN,ACK -j REJECT
iptables -t filter -I INPUT -p tcp -m tcp --dport 22 --tcp-flags ALL SYN -j REJECT
iptables -t filter -I OUTPUT -p tcp -m tcp --sport 22 --tcp-flags ALL SYN,ACK -j REJECT
#示例
iptables -t filter -I INPUT -p tcp -m tcp --dport 22 --syn -j REJECT
```

udp 扩展 常用的扩展匹配条件

*   –sport：匹配 udp 报文的源地址

*   –dport：匹配 udp 报文的目标地址

```bash
#示例
iptables -t filter -I INPUT -p udp -m udp --dport 137 -j ACCEPT
iptables -t filter -I INPUT -p udp -m udp --dport 137:157 -j ACCEPT
#可以结合 multiport 模块指定多个离散的端口
```

icmp 扩展 常用的扩展匹配条件

*   –icmp-type：匹配 icmp 报文的具体类型

```bash
#示例
iptables -t filter -I INPUT -p icmp -m icmp --icmp-type 8/0 -j REJECT
iptables -t filter -I INPUT -p icmp --icmp-type 8 -j REJECT
iptables -t filter -I OUTPUT -p icmp -m icmp --icmp-type 0/0 -j REJECT
iptables -t filter -I OUTPUT -p icmp --icmp-type 0 -j REJECT
iptables -t filter -I INPUT -p icmp --icmp-type "echo-request" -j REJECT
```

multiport 扩展模块 常用的扩展匹配条件如下：

*   \-p tcp -m multiport –sports 用于匹配报文的源端口，可以指定离散的多个端口号，端口之间用”逗号”隔开

*   \-p udp -m multiport –dports 用于匹配报文的目标端口，可以指定离散的多个端口号，端口之间用”逗号”隔开

```bash
#示例如下
iptables -t filter -I OUTPUT -d 192.168.1.146 -p udp -m multiport --sports 137,138 -j REJECT
iptables -t filter -I INPUT -s 192.168.1.146 -p tcp -m multiport --dports 22,80 -j REJECT
iptables -t filter -I INPUT -s 192.168.1.146 -p tcp -m multiport ! --dports 22,80 -j REJECT
iptables -t filter -I INPUT -s 192.168.1.146 -p tcp -m multiport --dports 80:88 -j REJECT
iptables -t filter -I INPUT -s 192.168.1.146 -p tcp -m multiport --dports 22,80:88 -j REJECT

```

### 常用扩展模块

iprange 模块
包含的扩展匹配条件如下

*   \--src-range：指定连续的源地址范围

*   \--dst-range：指定连续的目标地址范围

```bash
#示例
iptables -t filter -I INPUT -m iprange --src-range 192.168.1.127-192.168.1.146 -j DROP
iptables -t filter -I OUTPUT -m iprange --dst-range 192.168.1.127-192.168.1.146 -j DROP
iptables -t filter -I INPUT -m iprange ! --src-range 192.168.1.127-192.168.1.146 -j DROP
```

string 模块
常用扩展匹配条件如下

*   \--algo：指定对应的匹配算法，可用算法为 bm、kmp，此选项为必需选项。

*   \--string：指定需要匹配的字符串

```bash
#示例
iptables -t filter -I INPUT -p tcp --sport 80 -m string --algo bm --string "OOXX" -j REJECT
```

time 模块
常用扩展匹配条件如下

*   –-timestart：用于指定时间范围的开始时间，不可取反

*   –-timestop：用于指定时间范围的结束时间，不可取反

*   –-weekdays：用于指定”星期几”，可取反

*   –-monthdays：用于指定”几号”，可取反

*   –-datestart：用于指定日期范围的开始日期，不可取反

*   –-datestop：用于指定日期范围的结束时间，不可取反

```bash
#示例
iptables -t filter -I OUTPUT -p tcp --dport 80 -m time --timestart 09:00:00 --timestop 19:00:00 -j REJECT
iptables -t filter -I OUTPUT -p tcp --dport 443 -m time --timestart 09:00:00 --timestop 19:00:00 -j REJECT
iptables -t filter -I OUTPUT -p tcp --dport 80  -m time --weekdays 6,7 -j REJECT
iptables -t filter -I OUTPUT -p tcp --dport 80  -m time --monthdays 22,23 -j REJECT
iptables -t filter -I OUTPUT -p tcp --dport 80  -m time ! --monthdays 22,23 -j REJECT
iptables -t filter -I OUTPUT -p tcp --dport 80  -m time --timestart 09:00:00 --timestop 18:00:00 --weekdays 6,7 -j REJECT
iptables -t filter -I OUTPUT -p tcp --dport 80  -m time --weekdays 5 --monthdays 22,23,24,25,26,27,28 -j REJECT
iptables -t filter -I OUTPUT -p tcp --dport 80  -m time --datestart 2017-12-24 --datestop 2017-12-27 -j REJECT
```

connlimit 模块
常用的扩展匹配条件如下

*   –-connlimit-above：单独使用此选项时，表示限制每个 IP 的链接数量。

*   –-connlimit-mask：此选项不能单独使用，在使用–connlimit-above 选项时，配合此选项，则可以针对”某类 IP 段内的一定数量的 IP” 进行连接数量的限。

```bash
#示例
iptables -I INPUT -p tcp --dport 22 -m connlimit --connlimit-above 2 -j REJECT
iptables -I INPUT -p tcp --dport 22 -m connlimit --connlimit-above 20 --connlimit-mask 24 -j REJECT
iptables -I INPUT -p tcp --dport 22 -m connlimit --connlimit-above 10 --connlimit-mask 27 -j REJECT
```

limit 模块 常用的扩展匹配条件如下

*   –-limit-burst：类比”令牌桶”算法，此选项用于指定令牌桶中令牌的最大数量，上文中已经详细的描述了”令牌桶”的概念，方便回顾。

*   –-limit：类比”令牌桶”算法，此选项用于指定令牌桶中生成新令牌的频率，可用时间单位有 second、minute 、hour、day。

```bash
#示例 #注意，如下两条规则需配合使用，具体原因上文已经解释过，忘记了可以回顾。
iptables -t filter -I INPUT -p icmp -m limit --limit-burst 3 --limit 10/minute -j ACCEPT
iptables -t filter -A INPUT -p icmp -j REJECT
```

## 自定义链

**创建自定义链**

```yaml
#示例：在filter表中创建IN_WEB自定义链
iptables -t filter -N IN_WEB
```

**引用自定义链**

```yaml
#示例：在INPUT链中引用刚才创建的自定义链
iptables -t filter -I INPUT -p tcp --dport 80 -j IN_WEB
```

**重命名自定义链**

```yaml
#示例：将IN_WEB自定义链重命名为WEB
iptables -E IN_WEB WEB
```

**删除自定义链**

删除自定义链需要满足两个条件

1.  自定义链没有被引用

2.  自定义链中没有任何规则

```yaml
#示例：删除引用计数为0并且不包含任何规则的WEB链
iptables -X WEB
```

## 参考

*   [https://www.zsythink.net/](https://www.zsythink.net/ "https://www.zsythink.net/")
