# 配置 arthas 实现远程在线 debug

## 本地配置

arthas 有多种启动方式：

*   java agent 像 skywalking 一样

*   [as.sh](http://as.sh "as.sh") 利用 arthas 的 shell 启动 或者  java -jar 启动

*   sprintboot starter 集成到应用中启动

我们采用最方便的把 arthas 集成到 springboot-starter 的应用中启动

## 加入相关依赖

```xml
<dependency>
      <groupId>com.taobao.arthas</groupId>
      <artifactId>arthas-spring-boot-starter</artifactId>
      <version>3.4.8</version>
      <scope>runtime</scope>
  </dependency>

  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-actuator</artifactId>
  </dependency>
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-test</artifactId>
      <scope>test</scope>
      <exclusions>
          <exclusion>
              <groupId>org.junit.vintage</groupId>
              <artifactId>junit-vintage-engine</artifactId>
          </exclusion>
      </exclusions>
  </dependency>

```

## 修改 application.yml 配置文件

```yaml
# arthas tunnel server配置
arthas:
  agent-id: arthasDemo
  tunnel-server: ws://47.75.156.201:7777/ws

# 监控配置
management:
  endpoints:
    web:
      exposure:
        include: '*'
  endpoint:
    health:
      show-details: always
```

## 启动

本地访问 [http://localhost:8080/actuator/arthas](http://localhost:8080/actuator/arthas "http://localhost:8080/actuator/arthas") 查看 arthas 配置信息

![](https://tva1.sinaimg.cn/large/008i3skNly1gt2jz7ng1oj30ec04f74b.jpg)

其他配置可以参考：[https://arthas.aliyun.com/doc/arthas-properties.html](https://arthas.aliyun.com/doc/arthas-properties.html "https://arthas.aliyun.com/doc/arthas-properties.html")

启动项目后，然后在浏览器中输入 [http://localhost:3658](http://localhost:3658 "http://localhost:3658") 地址(web console)。显示如下界面，就代表已经设置成功了。

![](https://tva1.sinaimg.cn/large/008i3skNly1gt2jzr1bclj31cs0cidh2.jpg)

## Arthas Tunnel Server

通过 Arthas Tunnel Server/Client 来远程管理/连接多个 Agent。

### 部署  Tunnel Server

下载 jar 包  [https://github.com/alibaba/arthas/releases](https://github.com/alibaba/arthas/releases "https://github.com/alibaba/arthas/releases")

```bash
## Arthas tunnel server 是一个 spring boot fat jar应用
## 直接java -jar启动：
java -jar  arthas-tunnel-server.jar
```

默认情况下，arthas tunnel server 的 web 端口是 8080，arthas agent 连接的端口是 7777

也可以修改端口，比如 `java-jar arthas-tunnel-server.jar --server.port=8082`

### 远程连接管理多个 Agent

部署起来后，agent 的配置就可以生效了，比如

```yaml
arthas:
#  telnetPort: -1
#  httpPort: -1
  tunnel-server: ws://127.0.0.1:7777/ws
  app-name: arthasDemo
```

此时打开 Tunnel Server    [http://127.0.0.1:8082/](http://127.0.0.1:8082/ "http://127.0.0.1:8082/")   是空白的

![](https://tva1.sinaimg.cn/large/008i3skNly1gt2k1vxim9j31co04wq38.jpg)

![](https://tva1.sinaimg.cn/large/008i3skNly1gt2k271sr6j30bp04m3yd.jpg)

![](https://tva1.sinaimg.cn/large/008i3skNly1gt2k2bm5m5j317t03omx7.jpg)

点击按钮，或输入 AgentId 便可连接上指定的 agent 了&#x20;

![](https://tva1.sinaimg.cn/large/008i3skNly1gt2k42m037j31d40asjst.jpg)

## 参考：

*   [https://arthas.aliyun.com/](https://arthas.aliyun.com/ "https://arthas.aliyun.com/")
