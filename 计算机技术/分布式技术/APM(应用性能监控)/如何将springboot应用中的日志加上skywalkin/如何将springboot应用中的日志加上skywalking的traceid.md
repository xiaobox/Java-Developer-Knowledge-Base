# 如何将springboot应用中的日志加上skywalking的traceid

为了能够根据skywalking中的traceId，在kibana中查询到相关日志，需要修改springboot应用中的logback配置文件

由于我们是利用logstash直接打到logstash服务器上的，所以根据官方文档

（[https://github.com/apache/skywalking/blob/v8.0.0/docs/en/setup/service-agent/java-agent/Application-toolkit-logback-1.x.md](https://github.com/apache/skywalking/blob/v8.0.0/docs/en/setup/service-agent/java-agent/Application-toolkit-logback-1.x.md "https://github.com/apache/skywalking/blob/v8.0.0/docs/en/setup/service-agent/java-agent/Application-toolkit-logback-1.x.md")）

这样操作

*   Dependency the toolkit, such as using maven or gradle

    ```xml
    <dependency>
      <groupId>org.apache.skywalking</groupId>
      
      <artifactId>apm-toolkit-logback-1.x</artifactId>
      
      <version>${skywalking.version}</version>

    </dependency>
    ```

*   set `LogstashEncoder` of logback.xml

    ```xml
    <encoder charset="UTF-8" class="net.logstash.logback.encoder.LogstashEncoder">

      <provider class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.logstash.TraceIdJsonProvider">

      </provider>

    </encoder>
    ```

如果你的appender是 ConsoleAppender，则可以这样

*   Dependency the toolkit, such as using maven or gradle

    ```xml
    <dependency>

      <groupId>org.apache.skywalking</groupId>
      
      <artifactId>apm-toolkit-logback-1.x</artifactId>
      
      <version>{project.release.version}</version>

    </dependency>
    ```

*   set `%tid` in `Pattern` section of logback.xml

    ```xml
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">

      <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
      
      <layout class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.TraceIdPatternLogbackLayout">
      
      <Pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%tid] [%thread] %-5level %logger{36} -%msg%n</Pattern>
      
      </layout>
      
      </encoder>

    </appender>
    ```

*   with the MDC, set `%X{tid}` in `Pattern` section of logback.xml

    ```xml
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">

      <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
      
      <layout class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.mdc.TraceIdMDCPatternLogbackLayout">
      
      <Pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%X{tid}] [%thread] %-5level %logger{36} -%msg%n</Pattern>
      
      </layout>
      
      </encoder>

    </appender>
    ```

这样配置以后我们就可以根据skywalking中的traceId找到相关的请求调用链日志了。
