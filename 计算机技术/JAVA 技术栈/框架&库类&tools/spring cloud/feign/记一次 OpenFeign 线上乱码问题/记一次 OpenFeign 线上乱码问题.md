# 记一次 OpenFeign 线上乱码问题

## 问题描述

前两天，开发同学发现线上某服务往第三方 API 发出的请求（这个请求是用 openFeign 包装过的），其响应有时为乱码

后来经过测试能够稳定复现问题，开发同学通过分析发现，只要请求的 header 中有 “Accept-Encoding” 且值为 “gzip, deflate, br”，那么响应回来的数据必是乱码。

通过这个现象我们得出结论，即给请求头加了压缩标识，数据也响应回来了，但是并没有解压缩。

知道了原因，那么解决思路无非有二：

*   不压缩了

*   加上解压缩实现

## 解决方案

### 第一种思路实现

第一个思路的解决方案即不压缩了，不管 openFeign 之前谁给加了什么 header 参数，我们只要把“Accept-Encoding” 重置就可以了。这里顺便介绍下这个参数详情：

Accept-Encoding 和 Content-Encoding 是 HTTP 中用来对采用何种压缩格式传输正文进行协定的一对 header。工作原理如下：

*   浏览器发送请求，通过 Accept-Encoding 带上自己支持的内容编码格式列表

*   服务端从中挑选一个用来对正文进行编码，并通过 Content-Encoding 响应头指明响应编码格式。

*   浏览器拿到响应正文后，根据 Content-Encoding 进行解压缩。服务端若响应未压缩的正文，则不允许返回 Content-Encoding。

压缩类型：

*   gzip：表示采用 Lempel-Ziv coding (LZ77) 压缩算法，以及 32 位 CRC 校验的编码方式

*   Compress：采用 Lempel-Ziv-Welch (LZW) 压缩算法。

*   deflate：表示采用 zlib 结构 （在 RFC 1950 中规定），和 deflate 压缩算法（在 RFC 1951 中规定）。

*   identity：用于指代自身（未经过压缩和修改）。除非特别指明，这个标记始终可以被接受。

*   Br：表示采用 Brotli 算法的编码方式。

内容编码：

1.  内容编码针对的只是传输正文。HTTP/1 中，header 始终是以 ASCII 文本传输，没有经过任何压缩；HTTP/2 中引入 header 压缩技术。

所以我们下 2 种方法都是基于设置  “identity：用于指代自身（未经过压缩和修改）”，告诉请求不用压缩了，自然也就不用解压了。

**httpclient**

在原始 feign 配置下，仍然利用 httpclient 作为 http 代理，不用 okhttp

![](https://tva1.sinaimg.cn/large/008i3skNly1gvo2379lpyj609v0870sv02.jpg)

```java
package com.my.fedex.kuaidi100.rest;

import com.my.fedex.common.constants.FedexConstants;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestParam;

@FeignClient(name = "kuaidi100", url = FedexConstants.KUAIDI100_QUERY_URL, fallbackFactory = KuaiDi100FeignFallBack.class)
//@RequestMapping(value = "/", headers = {"Accept-Encoding=identity"})
public interface KuaiDi100Feign {

  
    @PostMapping(headers = {"Accept-Encoding=identity"})
    String findKuaiDi100(@RequestParam("customer") String customer,
                         @RequestParam("sign") String sign,
                         @RequestParam("param") String param);
}
```

只需将方法上的 postMapping 添加入一个新的 headers 即可，这个方法之前是这样声明的：

```java
@PostMapping
String findKuaiDi100(@RequestParam("customer") String customer,
                         @RequestParam("sign") String sign,
                         @RequestParam("param") String param);
```

或者也可以用原来的方法和方法上的 @PostMapping 声明，把接口上的注释打开即可。

**okhttp**

在我的测试中，如果客户端代理用 okhttp , 那么会报一个错，主要信息为

```javascript
java.io.EOFException: \n not found: limit=0 content=…

```

报这个的原因是 response 响应回来的内容为空 也就是 content-size 是 0 。那又是为什么呢？

原因是：请求的 host 不对，我本地请求的 host 居然变成了 “localhost:8080”，显然我们请求对方接口的 host 应该为：“[poll.kuaidi100.com](http://poll.kuaidi100.com "poll.kuaidi100.com")”。这个现象只有在用 okhttp 时会这样，想来应该是透传了请求客户端的 host ，而 okhttp 没有计算对最终目标的 host。

解决方法也很简单，是基于上面的方案再多加一个 header，最终为：

```java
@PostMapping(headers = {"Accept-Encoding=identity","host=poll.kuaidi100.com"})

```

**总结：无论是利用 httpclient 还是 okhttp 都可以通过添加 headers 解决乱码的问题。但 okhttp 比较特殊要多加一个 host 。**

### 第二种思路实现

第二种思路即加上压缩，根据 spring 官方文档得知，httpclient 是可以直接配置 response 的解压实现的（人家 feign 都实现好了）, 配置方法也很简单，如下图所示：

![](https://tva1.sinaimg.cn/large/008i3skNly1gvo2aa1be0j60zk0560t702.jpg)

需要注意的是，也是文档中所写的，它不支持 okhttp，也就是说如果我们用  okhttp 代理不能这么干。

那么问题来了，如果用 okhttp 怎么办，也是有办法的，但是是相对最麻烦的一种，目前想到的是自己手动实现一个解码器了，比如：

```java
import feign.Response;
import feign.Util;
import feign.codec.Decoder;
import org.springframework.cloud.openfeign.encoding.HttpEncoding;

import java.io.BufferedReader;
import java.io.ByteArrayInputStream;
import java.io.IOException;
import java.io.InputStreamReader;
import java.lang.reflect.Type;
import java.nio.charset.StandardCharsets;
import java.util.Collection;
import java.util.Objects;
import java.util.zip.GZIPInputStream;

public class CustomGZIPResponseDecoder implements Decoder {

    final Decoder delegate;

    public CustomGZIPResponseDecoder(Decoder delegate) {
        Objects.requireNonNull(delegate, "Decoder must not be null. ");
        this.delegate = delegate;
    }

    @Override
    public Object decode(Response response, Type type) throws IOException {
        Collection<String> values = response.headers().get(HttpEncoding.CONTENT_ENCODING_HEADER);
        if(Objects.nonNull(values) && !values.isEmpty() && values.contains(HttpEncoding.GZIP_ENCODING)){
            byte[] compressed = Util.toByteArray(response.body().asInputStream());
            if ((compressed == null) || (compressed.length == 0)) {
               return delegate.decode(response, type);
            }
            //decompression part
            //after decompress we are delegating the decompressed response to default 
            //decoder
            if (isCompressed(compressed)) {
                final StringBuilder output = new StringBuilder();
                final GZIPInputStream gis = new GZIPInputStream(new ByteArrayInputStream(compressed));
                final BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(gis, StandardCharsets.UTF_8));
                String line;
                while ((line = bufferedReader.readLine()) != null) {
                    output.append(line);
                }
                Response uncompressedResponse = response.toBuilder().body(output.toString().getBytes()).build();
                return delegate.decode(uncompressedResponse, type);
            }else{
                return delegate.decode(response, type);
            }
        }else{
            return delegate.decode(response, type);
        }
    }

    private static boolean isCompressed(final byte[] compressed) {
        return (compressed[0] == (byte) (GZIPInputStream.GZIP_MAGIC)) && (compressed[1] == (byte) (GZIPInputStream.GZIP_MAGIC >> 8));
    }
}

```

根据官方文档的描述还是比较容易设置的。

另外，下面这位网友给出了实践操作：
[https://stackoverflow.com/questions/51901333/okhttp-3-how-to-decompress-gzip-deflate-response-manually-using-java-android](https://stackoverflow.com/questions/51901333/okhttp-3-how-to-decompress-gzip-deflate-response-manually-using-java-android "https://stackoverflow.com/questions/51901333/okhttp-3-how-to-decompress-gzip-deflate-response-manually-using-java-android")
也就是自己 new okHttpClient 然后设置一个 interceptor

```java
OkHttpClient.Builder clientBuilder = new OkHttpClient.Builder().addInterceptor(new UnzippingInterceptor());
OkHttpClient client = clientBuilder.build();

```

```java
private class UnzippingInterceptor implements Interceptor {
    @Override
    public Response intercept(Chain chain) throws IOException {
        Response response = chain.proceed(chain.request());
        return unzip(response);
    }
  

// copied from okhttp3.internal.http.HttpEngine (because is private)
private Response unzip(final Response response) throws IOException {
    if (response.body() == null)
    {
        return response;
    }
    
    //check if we have gzip response
    String contentEncoding = response.headers().get("Content-Encoding");
    
    //this is used to decompress gzipped responses
    if (contentEncoding != null && contentEncoding.equals("gzip"))
    {
        Long contentLength = response.body().contentLength();
        GzipSource responseBody = new GzipSource(response.body().source());
        Headers strippedHeaders = response.headers().newBuilder().build();
        return response.newBuilder().headers(strippedHeaders)
                .body(new RealResponseBody(response.body().contentType().toString(), contentLength, Okio.buffer(responseBody)))
                .build();
    }
    else
    {
        return response;
    }
}

```

基于上面这位网友的，给出我自己的代码实现：

首先，自己生成 client, 加入自己的 interceptor。

```java
package com.my.fedex.kuaidi100.config;

import feign.Client;
import feign.Feign;
import okhttp3.ConnectionPool;
import okhttp3.OkHttpClient;
import org.springframework.boot.autoconfigure.AutoConfigureAfter;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.cloud.commons.httpclient.OkHttpClientConnectionPoolFactory;
import org.springframework.cloud.commons.httpclient.OkHttpClientFactory;
import org.springframework.cloud.openfeign.FeignAutoConfiguration;
import org.springframework.cloud.openfeign.support.FeignHttpClientProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.concurrent.TimeUnit;

/**
 * @author helong
 * @since 2021-10-21 23:22
 */
@Configuration
@ConditionalOnClass(Feign.class)
@AutoConfigureAfter(FeignAutoConfiguration.class)
public class OkHttpFeignLoadBalancedConfiguration {

    @Bean
    @ConditionalOnMissingBean({Client.class})
    public Client feignClient(okhttp3.OkHttpClient client) {
        return new feign.okhttp.OkHttpClient(client);
    }

    @Bean
    @ConditionalOnMissingBean({ConnectionPool.class})
    public ConnectionPool httpClientConnectionPool(FeignHttpClientProperties httpClientProperties, OkHttpClientConnectionPoolFactory connectionPoolFactory) {
        Integer maxTotalConnections = httpClientProperties.getMaxConnections();
        Long timeToLive = httpClientProperties.getTimeToLive();
        TimeUnit ttlUnit = httpClientProperties.getTimeToLiveUnit();
        return connectionPoolFactory.create(maxTotalConnections, timeToLive, ttlUnit);
    }

    @Bean
    public OkHttpClient client(OkHttpClientFactory httpClientFactory, ConnectionPool connectionPool, FeignHttpClientProperties httpClientProperties) {
        Boolean followRedirects = httpClientProperties.isFollowRedirects();
        Integer connectTimeout = httpClientProperties.getConnectionTimeout();
        Boolean disableSslValidation = httpClientProperties.isDisableSslValidation();
        return httpClientFactory.createBuilder(disableSslValidation)
            .connectTimeout((long) connectTimeout, TimeUnit.MILLISECONDS)
            .followRedirects(followRedirects)
            .connectionPool(connectionPool)
            .retryOnConnectionFailure(true)
            .addInterceptor(new UnzippingInterceptor())
            .build();
    }
}

```

自定义 interceptor, 用于解压数据。

```java
package com.my.fedex.kuaidi100.config;

import okhttp3.Headers;
import okhttp3.Interceptor;
import okhttp3.Request;
import okhttp3.Response;
import okhttp3.internal.http.RealResponseBody;
import okio.GzipSource;
import okio.Okio;

import java.io.IOException;

/**
 * @author helong
 * @since 2021-10-21 23:23
 */
public class UnzippingInterceptor implements Interceptor {
    @Override
    public Response intercept(Chain chain) throws IOException {
        Request build = chain.request().newBuilder().build();
        Response response = chain.proceed(build);
        return unzip(response);
    }

    // copied from okhttp3.internal.http.HttpEngine (because is private)
    private Response unzip(final Response response) throws IOException {
        if (response.body() == null) {
            return response;
        }

        //check if we have gzip response
        String contentEncoding = response.headers().get("Content-Encoding");

        //this is used to decompress gzipped responses
        if (contentEncoding != null && contentEncoding.equals("gzip")) {
            Long contentLength = response.body().contentLength();
            GzipSource responseBody = new GzipSource(response.body().source());
            Headers strippedHeaders = response.headers().newBuilder().build();
            return response.newBuilder().headers(strippedHeaders)
                .body(new RealResponseBody(response.body().contentType().toString(), contentLength, Okio.buffer(responseBody)))
                .build();
        } else {
            return response;
        }
    }
}

```

最后 feign 请求这里还是不要忘了加 host

```java
@PostMapping(headers = {"host=poll.kuaidi100.com"})
String findKuaiDi100(@RequestParam("customer") String customer,
                     @RequestParam("sign") String sign,
                     @RequestParam("param") String param);

```

## 参考

*   HTTP 中的 Accept-Encoding、Content-Encoding、Transfer-Encoding、Content-Type - AmyZYX - 博客园 [https://www.codeleading.com/article/84154165372/](https://www.codeleading.com/article/84154165372/ "https://www.codeleading.com/article/84154165372/")

*   [https://www.codeleading.com/article/84154165372/](https://www.codeleading.com/article/84154165372/ "https://www.codeleading.com/article/84154165372/")

*   [https://github.com/square/okhttp/issues/3590](https://github.com/square/okhttp/issues/3590 "https://github.com/square/okhttp/issues/3590")

*   [https://stackoverflow.com/questions/57831707/spring-feign-not-compressing-response](https://stackoverflow.com/questions/57831707/spring-feign-not-compressing-response "https://stackoverflow.com/questions/57831707/spring-feign-not-compressing-response")

*   [https://cloud.spring.io/spring-cloud-static/spring-cloud-openfeign/2.2.1.RELEASE/reference/html/](https://cloud.spring.io/spring-cloud-static/spring-cloud-openfeign/2.2.1.RELEASE/reference/html/ "https://cloud.spring.io/spring-cloud-static/spring-cloud-openfeign/2.2.1.RELEASE/reference/html/")
