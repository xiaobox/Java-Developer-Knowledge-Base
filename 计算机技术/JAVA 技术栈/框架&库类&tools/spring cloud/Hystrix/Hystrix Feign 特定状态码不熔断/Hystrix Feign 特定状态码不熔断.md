# Hystrix Feign 特定状态码不熔断

## 背景

Hystrix是个强大的熔断降级框架：收集目标方法的成功、失败等指标信息，触发熔断器。

其中失败信息**通过异常来表示**，交给Hystrix进行统计

**有的时候有些异常我们并不想触发熔断，比如请求参数异常等，那怎么办呢？**

先说个约定：目标方法执行抛出异常时，**除**`HystrixBadRequestException`**之外**，其他异常都会认为是Hystrix命令执行失败并触发服务降级处理逻辑。

## 为何不会触发熔断器？

所有的命令执行，最终在`AbstractCommand` 类的`executeCommandAndObserve()`方法内：

```java
AbstractCommand：

  private Observable<R> executeCommandAndObserve(final AbstractCommand<R> _cmd) {
    ...
        return execution.doOnNext(markEmits)
                .doOnCompleted(markOnCompleted)
                .onErrorResumeNext(handleFallback)
                .doOnEach(setRequestContext);
  }
```

其它部分本文不用关心，仅需关心`onErrorResumeNext(handleFallback)`这个函数，它的触发条件是：发射数据时（目标方法执行时）出现异常便会回调此函数，因此需要看看`handleFallback`的逻辑。

**handleFallback顾名思义，它是用于处理fallback的函数。**

```java
Func1<Throwable, Observable<R>> handleFallback = new Func1<Throwable, Observable<R>>() {

  // 当大声异常时，回调此方法，该异常就是t
    @Override
    public Observable<R> call(Throwable t) {
      // 若t就是Exception类型，那么t和e一样
      // 若不是Exception类型，比如是Error类型。那就用Exception把它包起来
      // new Exception("Throwable caught while executing.", t);
        Exception e = getExceptionFromThrowable(t);
        // 把异常写进结果里
        executionResult = executionResult.setExecutionException(e);

    // ==============针对不同异常的处理==============
        if (e instanceof RejectedExecutionException) {
            return handleThreadPoolRejectionViaFallback(e);
        } else if (t instanceof HystrixTimeoutException) {
            return handleTimeoutViaFallback();
        } else if (t instanceof HystrixBadRequestException) {
            return handleBadRequestByEmittingError(e);
        } else {
            return handleFailureViaFallback(e);
        }
    }
};
```

下面具体看看`handleBadRequestByEmittingError()`对该异常的处理。

`handleBadRequestByEmittingError()`此方法专门用于处理`HystrixBadRequestException`异常类型。

```java
AbstractCommand：

    private Observable<R> handleBadRequestByEmittingError(Exception underlying) {
        Exception toEmit = underlying;

        try {
            long executionLatency = System.currentTimeMillis() - executionResult.getStartTimestamp();
            // 请注意：这里发送的是BAD_REQUEST事件哦~~~~
            eventNotifier.markEvent(HystrixEventType.BAD_REQUEST, commandKey);
            executionResult = executionResult.addEvent((int) executionLatency, HystrixEventType.BAD_REQUEST);
            // 留个钩子：调用者可以对异常类型进行偷天换日
            Exception decorated = executionHook.onError(this, FailureType.BAD_REQUEST_EXCEPTION, underlying);


      // 如果调用者通过hook处理完后还是HystrixBadRequestException类型，那就直接把数据发射出去
      // 若不是，那就不管，还是发射原来的异常类型
            if (decorated instanceof HystrixBadRequestException) {
                toEmit = decorated;
            } else {
                logger.warn("ExecutionHook.onError returned an exception that was not an instance of HystrixBadRequestException so will be ignored.", decorated);
            }
        } catch (Exception hookEx) {
            logger.warn("Error calling HystrixCommandExecutionHook.onError", hookEx);
        }
        return Observable.error(toEmit);
    }
```

这就是`HystrixBadRequestException`的**特殊对待**逻辑，它发出的事件类型是`HystrixEventType.BAD_REQUEST`，而此事件类型是不会被`HealthCounts`作为健康指标所统计的，**因此它并不会触发熔断器**。

## 应用

了解了`HystrixBadRequestException`的这个特性后，使用场景可根据具体业务而定喽。比如我们最为常用的场景便是在Feign上自定义一个错误解码器`ErrorDecoder`，然后针对于错误码是400的响应统一转换为`HystrixBadRequestException`异常抛出，这样是比较优雅的一种实践方案。

### 实现

重写Feign error decoder逻辑

```java
import com.netflix.hystrix.exception.HystrixBadRequestException;
import feign.Response;
import feign.Util;
import feign.codec.ErrorDecoder;

import static java.lang.String.format;

/**
 * @author yugj
 * @date 2020/4/29 9:13 上午.
 */
public class SkipHttpStatusErrorDecoder extends ErrorDecoder.Default {

    public SkipHttpStatusErrorDecoder() {
        super();
    }

    @Override
    public Exception decode(String methodKey, Response response) {

        int status = response.status();
        if (status == 400 || status == 404) {
            String message = statusFormat(methodKey, response);
            return new HystrixBadRequestException(message);
        }

        return super.decode(methodKey, response);
    }

    private String statusFormat(String methodKey, Response response) {

        String message = format("status %s reading %s", response.status(), methodKey);
        if (response.body() != null) {
            try {
                String body = Util.toString(response.body().asReader());
                message += "; content:\n" + body;
            } catch (Exception e) {
                //do nothing
            }
        }

        return message;
    }
}
```

```java
@Configuration
public class SkipHttpStatusConfiguration {
    @Bean
    public SkipHttpStatusErrorDecoder errorDecoder() {
        return new SkipHttpStatusErrorDecoder();
    }

```

原理：

*   `com.netflix.hystrix.AbstractCommand#executeCommandAndObserve`异常控制

*   `HystrixBadRequestException` 没有走到熔断逻辑

## 参考

*   [https://cloud.tencent.com/developer/article/1600699](https://cloud.tencent.com/developer/article/1600699 "https://cloud.tencent.com/developer/article/1600699")

*   [https://my.oschina.net/yugj/blog/4257960](https://my.oschina.net/yugj/blog/4257960 "https://my.oschina.net/yugj/blog/4257960")
