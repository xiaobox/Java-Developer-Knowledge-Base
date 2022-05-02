# OpenFeign 如何设置动态 URL？

如果你利用 Spring Cloud OpenFeign 进行服务间调用一般会加入这个注解：

```java
@FeignClient(name = "" ,url = "http://myapp.com",path = "")
```

可以看出其中的 url 参数是一个字符串，上面的配置是把它写“死”在代码中了。

如果我们想根据不同的环境作动态配置，让这个 url 动态的变化应该怎么办呢？

可以这样：

首先修改注解

```java
@FeignClient(name = "" ,url = "${feign.client.url.TestUrl}",path = "")
```

然后添加配置文件，比如

在你的 application-dev.yml 文件中

```yaml
feign:
  client:
    url:
      TestUrl: http://dev:dev
```

在你的 application-pre.yml  文件中

```yaml
feign:
  client:
    url:
      TestUrl: http://pre:pre
```

利用 Spring 的 EL 表达式，我们就可以让`url` 根据不同文件的不同值动态获得了。

另外，还可以给这个表达式指定默认值

![](https://tva1.sinaimg.cn/large/008i3skNly1gvkfunp707j60vb0u0dk202.jpg)

也就是当配置文件没有这个配置的时候给一个默认的配置，这样的话，我们的注解要修改成如下：

```java
@FeignClient(name = "" ,url = "${feign.client.url.TestUrl ?: 'http://myapp.com'}",path = "")
```

最后给出一个我在实际项目中的例子

```javascript
@FeignClient(name = "idGenerateClient",
    path = "/v1/app/internal/test",
    url = "#{" +
        "('${spring.profiles.active}' eq 'local') ?  " +
        "('${feignclient-url." + APPConstant.APPLICATION_NAME + "}' ?: 'http://${env.domain}' ): " +
        "'http://" + APPConstant.APPLICATION_NAME + "'" +
        "}",
    fallbackFactory = XXXClientFallback.class)

```

```yaml
feignclient-url:
  my-app: '127.0.0.1:8805'
```

## 参考

*   [https://stackoverflow.com/questions/43733569/how-can-i-change-the-feign-url-during-the-runtime](https://stackoverflow.com/questions/43733569/how-can-i-change-the-feign-url-during-the-runtime "https://stackoverflow.com/questions/43733569/how-can-i-change-the-feign-url-during-the-runtime")

*   [https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#expressions](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#expressions "https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#expressions")
