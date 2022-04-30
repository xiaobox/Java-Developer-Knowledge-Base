# 关于 feign ribbon的重试&#x20;

## Feign

feign默认关闭了重试

```java
@Bean
@ConditionalOnMissingBean
public Retryer feignRetryer() {
  return Retryer.NEVER_RETRY;
}
```

如果想要开启的话，就自己声明一个bean

```java
@Bean
public Retryer feignRetryer() {
    return  new Retryer.Default();
}

```

## Ribbon

```yaml
#如果ribbon和feign的超时时间都配置了，ribbon的配置会被覆盖 ( 待验证！)
##Ribbon超时重试配置
ribbon:
  ConnectTimeout: 20000  #毫秒    连接超时时间,默认连接超时时间是1秒
  ReadTimeout: 20000     #毫秒      逻辑处理超时时间,默认超时时间是1秒
  OkToRetryOnAllOperations: true    # 是否对所有操作都进行重试
  MaxAutoRetries: 1     # 对当前实例的最大重试次数(请求服务超时6s则会再请求一次)
  Ma  

```

## Hystrix

注意Hystrix的超时时间要超过ribbon的重试时间，否则ribbon重试过程中，就会先触发Hystrix的熔断

```yaml
hystrix:
  command:
    default:
      execution:
        timeout:
          enabled: true
        isolation:
          thread:
            timeoutInMilliseconds: 1000
```
