# Spring Boot 整合 OpenFeign + Nacos

## Nacos

先在本地部署 Nacos server，然后在 springBoot 项目中添加依赖，理想情况下，服务会自动会注册到注册中心。

### 环境

*   SpringBoot 版本：2.3.1.RELEASE

*   Nacos server 版本：2.0.3

*   nacos-discovery-spring-boot-starter 版本：0.2.10

### Nacos Server

之前用过 Nacos 的 V1 版本，V2 版本没用过，这次用最新版本做个试验。

本地部署 Nacos 没什么问题，我的本地环境是 Mac, 下载好最新的版本 ([https://github.com/alibaba/nacos/archive/refs/tags/2.0.3.zip](https://github.com/alibaba/nacos/archive/refs/tags/2.0.3.zip "https://github.com/alibaba/nacos/archive/refs/tags/2.0.3.zip")) 解压后，到 bin 目录执行

```bash
sh startup.sh -m standalone
```

然后浏览器查看 `http://localhost:8848/nacos/` 用户名密码都是：nacos

![](https://tva1.sinaimg.cn/large/008i3skNly1gvsp7dwrcej31ck0dvmya.jpg)

可以看到，我用的版本是 2.0.3

### Spring Boot

出问题的地方是在程序中引入的时候，由于要演示 feign 的远程调用，我们分别创建两个项目，provier 和 consumer.

首先创建 provider, 先看下 pom.xml 文件中的依赖，可以看到跟 nacos 相关的只有一个。

```xml
<!-- BOM 全局管理 starter 版本 -->
   <dependencyManagement>
       <dependencies>
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-dependencies</artifactId>
               <version>${spring-boot-dependencies.version}</version>
               <scope>import</scope>
               <type>pom</type>
           </dependency>
       </dependencies>
   </dependencyManagement>

   <dependencies>

       <!-- openfeign -->
       <dependency>
           <groupId>org.springframework.cloud</groupId>
           <artifactId>spring-cloud-starter-openfeign</artifactId>
           <version>${spring-cloud-starter-openfeign.version}</version>
       </dependency>

       <dependency>
           <groupId>org.projectlombok</groupId>
           <artifactId>lombok</artifactId>
           <version>${lombok.version}</version>
           <scope>provided</scope>
       </dependency>
       <dependency>
           <groupId>org.springframework.boot</groupId>
           <artifactId>spring-boot-starter</artifactId>
       </dependency>
       <dependency>
           <groupId>org.springframework.boot</groupId>
           <artifactId>spring-boot-starter-test</artifactId>
           <scope>test</scope>
       </dependency>

       <dependency>
           <groupId>org.springframework.boot</groupId>
           <artifactId>spring-boot-starter-web</artifactId>
       </dependency>

       <dependency>
           <groupId>com.alibaba.boot</groupId>
           <artifactId>nacos-discovery-spring-boot-starter</artifactId>
           <version>0.2.10</version>
       </dependency>

       <dependency>
           <groupId>com.google.guava</groupId>
           <artifactId>guava</artifactId>
           <version>30.1.1-jre</version>
       </dependency>
   </dependencies>
```

然后再看下 application.yml

```yaml
server:
  port: 8080
  servlet:
    context-path: /provider
spring:
  mvc:
    throw-exception-if-no-handler-found: true # 处理 404 问题
  resources:
    add-mappings: false # 关闭 404 资源映射
  application:
    name: feign-provider

nacos:
  discovery:
    server-addr: 127.0.0.1:8848 # Nacos 服务器地址
   
```

通过官方文档查看，配置比较简单，没什么特殊的。这里强调下，**不需要在入口程序 BootstrapApplication 添加什么注解**

然后就出问题了，服务死活注册不上去，服务列表一直是空的。

翻阅了各种文档和配置，终于在 github 的 wiki 上找到了官方的说明：[https://github.com/nacos-group/nacos-spring-boot-project/wiki](https://github.com/nacos-group/nacos-spring-boot-project/wiki "https://github.com/nacos-group/nacos-spring-boot-project/wiki")

![](https://tva1.sinaimg.cn/large/008i3skNly1gvspl1suxpj31mw0u0q7w.jpg)

我需要的就是这个：

```javascript
# 是否允许服务自动注册（默认为关闭自动注册）
nacos.discovery.auto-register=true
```

由于之前用的是 V1 的版本，2 以后的配置没接触过，看来想要自动注册要把这个从 0.2.3 才有的配置加上（我们试验用的 starter 版本是 0.2.10)

以后跟 nacos-discovery-spring-boot-starter 相关版本配置有关的功能，可以参考官方的 wiki：

![](https://tva1.sinaimg.cn/large/008i3skNly1gvspp9qhf9j30jq0okwgw.jpg)

加上这个配置又丰富了一下后，配置文件变成这样：

```yaml
server:
  port: 8080
  servlet:
    context-path: /provider

spring:
  mvc:
    throw-exception-if-no-handler-found: true # 处理 404 问题
  resources:
    add-mappings: false # 关闭 404 资源映射
  application:
    name: feign-provider

nacos:
  discovery:
    server-addr: 127.0.0.1:8848 # Nacos 服务器地址
    auto-register: true # 是否自动注册到 Nacos 中。默认为 false。
    namespace:  # 使用的 Nacos 的命名空间，默认为 null。
    register:
      service-name: ${spring.application.name} # 注册到 Nacos 的服务名
      group-name: DEFAULT_GROUP # 使用的 Nacos 服务分组，默认为 DEFAULT_GROUP。
    
```

然后就顺利地注册上了：

![](https://tva1.sinaimg.cn/large/008i3skNly1gvspsgrv90j32oe0segp8.jpg)

**but**&#x20;

正当我兴冲冲地去写完 consumer 准备消费的时候，却发现怎么也消费不了，google 了个遍，一直都报这个错：

`Load balancer does not have available server for client`

实在是不想 debug 源码了，消耗完耐心后，决定不用 springBoot 去整合了，还是回到

Spring Cloud + Spring Cloud Alibaba 的路上。

### Spring Cloud + Spring Cloud Alibaba

首先要弄清楚版本间的依赖关系，因为版本众多，稍不注意就容易出问题，可以参考这里：

[https://github.com/alibaba/spring-cloud-alibaba/wiki/版本说明](https://github.com/alibaba/spring-cloud-alibaba/wiki/版本说明 "https://github.com/alibaba/spring-cloud-alibaba/wiki/版本说明")

而 Nacos 的例子，官方有一个直接抄
[https://spring-cloud-alibaba-group.github.io/github-pages/hoxton/zh-cn/index.html#](https://spring-cloud-alibaba-group.github.io/github-pages/hoxton/zh-cn/index.html# "https://spring-cloud-alibaba-group.github.io/github-pages/hoxton/zh-cn/index.html#")*%E4%B8%80%E4%B8%AA%E4%BD%BF%E7%94%A8\_nacos\_discovery*%E8%BF%9B%E8%A1%8C%E6%9C%8D%E5%8A%A1%E6%B3%A8%E5%86%8C%E5%8F%91%E7%8E%B0%E5%B9%B6%E8%B0%83%E7%94%A8%E7%9A%84%E4%BE%8B%E5%AD%90

参考完官方的例子后，我的 provider 的 pom.xml 修改如下：

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.3.2.RELEASE</version>
    <relativePath/>
</parent>

<properties>
    <maven-compiler-plugin.version>3.8.1</maven-compiler-plugin.version>
    <maven-source.version>3.2.1</maven-source.version>
    <lombok.version>1.18.12</lombok.version>

    <spring.boot.version>2.3.2.RELEASE</spring.boot.version>
    <spring.cloud.version>Hoxton.SR9</spring.cloud.version>
    <spring.cloud.alibaba.version>2.2.6.RELEASE</spring.cloud.alibaba.version>

</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring.cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-alibaba-dependencies</artifactId>
            <version>${spring.cloud.alibaba.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>
</dependencies>
```

application.yml 修改如下：

```yaml
server:
  port: 8080
  servlet:
    context-path: /provider

spring:
  mvc:
    throw-exception-if-no-handler-found: true # 处理 404 问题
  resources:
    add-mappings: false # 关闭 404 资源映射
  application:
    name: feign-provider

  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848 # Nacos 服务器地址
        # namespace: 03afd923-0972-48c4-80fc-69098625d8b0

```

controller 的接口该怎么写还怎么写，在启动类需要加入一个注解

```javascript
@EnableDiscoveryClient
```

之后把 provider 启动起来，到 nacos 的 dashboard 看一下注册成功了。

接着是 consumer

**consumer 的 pom.xml、application.yml 与 provider 一样**
入口程序同样要加入：

```javascript
@EnableDiscoveryClient
```

feign 接口：

```java
@FeignClient(name = "feign-provider",path = "/provider")
public interface ProviderClient {

    @GetMapping("test/hello")
    String sayHello();

}

```

controller 消费一下：

```java
@RestController
@RequestMapping("/test")
public class ConsumerController {

    @Autowired
    private ProviderClient remoteClient;

    @GetMapping("/consume")
    public String consume() {

        // ServiceInstance serviceInstance = loadBalancerClient.choose("feign-provider");
        // String url = String.format("http://%s:%s/echo/%s",serviceInstance.getHost(),serviceInstance.getPort(),"feign-provider");
        // return restTemplate.getForObject(url,String.class);

        return remoteClient.sayHello();
    }

}

```

用浏览器请求 consumer 接口测试成功，可以调用到远程 provider 服务。

## 总结&#x20;

整体看技术上没什么难度，但 nacos-spring-boot-projec 这个项目直觉上是有坑的，试了许久还是有问题，也许是我没找到问题的根源，所以如果你有耐心可以试着搞搞，如果着急用，还是 Spring Cloud Alibaba 靠谱。

## 参考

*   [https://github.com/nacos-group/nacos-spring-boot-project/wiki](https://github.com/nacos-group/nacos-spring-boot-project/wiki "https://github.com/nacos-group/nacos-spring-boot-project/wiki")

*   [https://github.com/alibaba/spring-cloud-alibaba/wiki/版本说明](https://github.com/alibaba/spring-cloud-alibaba/wiki/版本说明 "https://github.com/alibaba/spring-cloud-alibaba/wiki/版本说明")

*   [https://spring-cloud-alibaba-group.github.io/github-pages/hoxton/zh-cn/index](https://spring-cloud-alibaba-group.github.io/github-pages/hoxton/zh-cn/index "https://spring-cloud-alibaba-group.github.io/github-pages/hoxton/zh-cn/index")
