# spring boot 微服务-动态调整日志级别

## springboot 借助 actuator  组件是可以实现动态调整日志级别，不用重启的。

## 具体做法 

### 1  确认项目中包括了 actuator 的依赖

```xml
<dependency>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### 2 修改application.xml配置文件，将端点暴露出去

```yaml
#监控端点暴露配置
management:
  security:
    enabled: false
 
endpoints:
  shutdown:
    enabled: false
  sensitive: false
 
#spirngboot Actuator 配置
info:
  app:
    name: oasis-ct-order-v2-server
```

### 3 查看目前的日志级别，比如 http://ip:port/loggers/root，可以看到类似这样的内容：

```yaml
{
    "configuredLevel": "INFO",
    "effectiveLevel": "INFO"
}
```

 这里只调了root,当然你也可以调整其他logger的，需要先请求 [http://ip:port/loggers/](http://60.205.151.68:10026/loggers/root "http://ip:port/loggers/") 看看有哪些，然后调整你想调整的。

### 4 修改日志级别，通过postman或其他工具发起POST请求 

请求地址：[http://ip:port/loggers/root](http://60.205.151.68:10026/loggers/root "http://ip:port/loggers/root")
请求报文：

```yaml
{
    "configuredLevel": "DEBUG",
    "effectiveLevel": "DEBUG"
}

```

### 5 继续第3步的操作，就看到日志级别被改过来了，这时可以打印DEBUG级别的日志到ELK了。

## 参考

[https://ricstudio.top/archives/spring\_boot\_actuator\_learn](https://ricstudio.top/archives/spring_boot_actuator_learn "https://ricstudio.top/archives/spring_boot_actuator_learn")
