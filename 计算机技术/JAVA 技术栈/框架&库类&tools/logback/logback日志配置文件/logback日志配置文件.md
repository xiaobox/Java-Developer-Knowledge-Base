# logback日志配置文件

**不重启服务，动态修改日志级别**

根节点，包含下面三个属性：

*   scan: 当此属性设置为true时，配置文件如果发生改变，将会被重新加载，默认值为true。

*   scanPeriod: 设置监测配置文件是否有修改的时间间隔，如果没有给出时间单位，默认单位是毫秒。当scan为true时，此属性生效。默认的时间间隔为1分钟。

*   debug: 当此属性设置为true时，将打印出logback内部日志信息，实时查看logback运行状态。默认值为false。

比如我们可以对外暴露接口动态设置：

```java
LoggerContext loggerContext = (LoggerContext) LoggerFactory.getILoggerFactory();
loggerContext.getLogger("com.hsuns").setLevel(Level.DEBUG);
```

子节点`<contextName>`：用来设置上下文名称，每个logger都关联到logger上下文，默认上下文名称为default。但可以使用设置成其他名字，用于区分不同应用程序的记录。一旦设置，不能修改-

```xml
<contextName>oasis-ct-interrogation</contextName>
```

子节点`<property>` ：用来定义变量值，它有两个属性name和value，通过`<property>`定义的值会被插入到logger上下文中，可以使“\${}”来使用变量。

```xml
<configuration scan="true" scanPeriod="60 seconds" debug="false"> 
　　　<property name="APP_NAME" value="oasis-ct-interrogation" /> 
　　　<contextName>${APP_NAME}</contextName> 
　　　<!--其他配置省略--> 
</configuration>
```

通过springProperty标签直接读取application.yml中数据库的配置

```xml
<springProperty scope="context" name="tags" source="spring.profiles"/>
<springProperty scope="context" name="app_port" source="server.port"/>
```

## 加入sql日志配置

1 加入依赖

```xml
<dependency>
    <groupId>com.googlecode.log4jdbc</groupId>
    <artifactId>log4jdbc</artifactId>
    <version>1.2</version>
    <scope>runtime</scope>
</dependency>
```

2 修改数据源配置

```yaml
# 修改前
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost:3306/springboot?useUnicode=true&characterEncoding=UTF-8

# 修改后
spring.datasource.driver-class-name=net.sf.log4jdbc.DriverSpy
spring.datasource.url=jdbc:log4jdbc:mysql://localhost:3306/springboot?useUnicode=true&characterEncoding=UTF-8
```

3 修改logback.xml

```xml
 <appender name="SQL_LOG" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <!-- 每天一归档 -->
            <fileNamePattern>${LOG_DIR}/sql.%d{yyyy-MM-dd}-%i.log</fileNamePattern>
            <!-- 单个日志文件最多 150MB, 30天的日志周期，最大不能超过30GB -->
            <maxFileSize>${MAX_FILE_SIZE}</maxFileSize>
            <maxHistory>${MAX_HISTORY}</maxHistory>
            <totalSizeCap>${TOTAL_SIZE_CAP}</totalSizeCap>
        </rollingPolicy>
        <append>true</append>
        <encoder>
            <pattern>${ENCODER_PATTERN}</pattern>
            <charset>utf-8</charset>
        </encoder>

    </appender>

    <appender name ="ASYNC_SQL" class= "ch.qos.logback.classic.AsyncAppender">
        <!-- 不丢失日志.默认的,如果队列的80%已满,则会丢弃TRACT、DEBUG、INFO级别的日志 -->
        <discardingThreshold>0</discardingThreshold>
        <!-- 更改默认的队列的深度,该值会影响性能.默认值为256 -->
        <queueSize>512</queueSize>
        <!-- 添加附加的appender,最多只能添加一个 -->
        <appender-ref ref ="SQL_LOG"/>
    </appender>


    <!--将sql日志打印到文件的同时传递到root logger-->
    <logger name="jdbc.connection"  level="ERROR" additivity="true">
        <appender-ref ref="ASYNC_SQL" />
    </logger>
    <logger name="jdbc.resultset"  level="OFF">
        <appender-ref ref="ASYNC_SQL" />
    </logger>
    <logger name="jdbc.audit"  level="OFF">
        <appender-ref ref="ASYNC_SQL" />
    </logger>
    <logger name="jdbc.sqlonly"  level="OFF">
        <appender-ref ref="ASYNC_SQL" />
    </logger>
    <logger name="jdbc.sqltiming"  level="INFO">
        <appender-ref ref="ASYNC_SQL" />
    </logger>

    <!-- 过滤日志 level=OFF可以关闭-->
    <logger name="javax.activation" level="WARN"  />
    <logger name="javax.mail" level="WARN"  />
    <logger name="javax.xml.bind" level="WARN"  />
    <logger name="ch.qos.logback" level="WARN"  />
    <logger name="com.codahale.metrics" level="WARN"  />
    <logger name="com.ryantenney" level="WARN"  />
    <logger name="com.sun" level="WARN"  />
    <logger name="com.zaxxer" level="WARN"  />
    <logger name="io.undertow" level="WARN"  />
    <logger name="net.sf.ehcache" level="ERROR"  />
    <logger name="org.apache" level="WARN"  />
    <logger name="org.apache.catalina.startup.DigesterFactory" level="OFF"/>
    <logger name="org.bson" level="WARN"  />
    <logger name="org.hibernate.validator" level="WARN"  />
    <logger name="org.hibernate" level="WARN"  />
    <logger name="org.hibernate.ejb.HibernatePersistence" level="OFF"/>
    <logger name="org.springframework" level="WARN"  />
    <logger name="org.springframework.web" level="WARN"  />
    <logger name="org.springframework.security" level="WARN"  />
    <logger name="org.springframework.cache" level="WARN"  />
    <logger name="org.springframework.boot" level="WARN"  />
    <logger name="org.thymeleaf" level="WARN"  />
    <logger name="org.xnio" level="WARN"  />
    <logger name="springfox" level="WARN"  />
    <logger name="sun.rmi" level="WARN"  />
    <logger name="liquibase" level="WARN" />
    <logger name="sun.rmi.transport" level="WARN"  />

    <!--为了测试，打开更多日志-->
    <logger name="com.zaxxer.hikari.pool.HikariPool" level="INFO" />
    <!--打印配置文件加载顺序,有利于问题排查 https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-external-config.html -->
    <logger name="org.springframework.boot.context.config.ConfigFileApplicationListener"
            level="INFO"  />

```

日志模版

```xml
<?xml version="1.0" encoding="UTF-8"?>
 
<!--
-debug:当此属性设置为true时，将打印出logback内部日志信息，实时查看logback运行状态。默认值为false。
-scan:当此属性设置为true时，配置文件如果发生改变，将会被重新加载，默认值为true
-scanPeriod:设置监测配置文件是否有修改的时间间隔，如果没有给出时间单位，默认单位是毫秒。当scan为true时，此属性生效。默认的时间间隔为1分钟
-->
<configuration debug="false" scan="true" scanPeriod="60 seconds">
 
 
  <!--通过springProperty标签直接读取application.yml中数据库的配置-->
  <springProperty scope="context" name="APP_NAME" source="spring.application.name" />
  <springProperty scope="context" name="TAGS" source="spring.profiles.active"/>
  <springProperty scope="context" name="APP_PORT" source="server.port"/>
 
 
  <property name="LOG_DIR" value="./logs"/>
  <property name="ENCODER_PATTERN" value="%d{yyyy-MM-dd HH:mm:ss.SSS} | %thread | %-5level | %logger{50} : %msg%n"/>
  <!-- 活动文件的大小 -->
  <property name="MAX_FILE_SIZE" value="150MB"/>
  <!-- 保留的归档文件的最大数量 -->
  <property name="MAX_HISTORY" value="30"/>
  <!-- 控制所有归档日志文件的总大小 -->
  <property name="TOTAL_SIZE_CAP" value="30GB"/>
  <!--logstash配置-->
  <property name="LOGSTASH_REMOTE_HOST" value="39.107.158.161" />
  <property name="LOGSTASH_PORT" value="5001" />
 
  <!--引入logback默认的console appender 配置 -->
  <include resource="org/springframework/boot/logging/logback/defaults.xml"/>
  <include resource="org/springframework/boot/logging/logback/console-appender.xml"/>
 
 
  <!--用来设置上下文名称，每个logger都关联到logger上下文，默认上下文名称为default。但可以使用设置成其他名字，用于区分不同应用程序的记录。一旦设置，不能修改-->
  <contextName>${APP_NAME}</contextName>
 
 
  <!--输出debug级别日志到文件-->
  <appender name="DEBUG_LOG" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
      <!-- 每天一归档 -->
      <fileNamePattern>${LOG_DIR}/debug.%d{yyyy-MM-dd}-%i.log</fileNamePattern>
      <!-- 单个日志文件最多 150MB, 30天的日志周期，最大不能超过30GB -->
      <maxFileSize>${MAX_FILE_SIZE}</maxFileSize>
      <maxHistory>${MAX_HISTORY}</maxHistory>
      <totalSizeCap>${TOTAL_SIZE_CAP}</totalSizeCap>
    </rollingPolicy>
    <encoder>
      <pattern>${ENCODER_PATTERN}</pattern>
      <charset>utf-8</charset>
    </encoder>
    <!-- 只打印DEBUG日志 -->
    <filter class="ch.qos.logback.classic.filter.LevelFilter">
      <level>DEBUG</level>
      <onMatch>ACCEPT</onMatch>
      <onMismatch>DENY</onMismatch>
    </filter>
  </appender>
 
  <!--输出warn级别日志到文件-->
  <appender name="INFO_LOG" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
      <!-- 每天一归档 -->
      <fileNamePattern>${LOG_DIR}/info.%d{yyyy-MM-dd}-%i.log</fileNamePattern>
      <!-- 单个日志文件最多 150MB, 30天的日志周期，最大不能超过30GB -->
      <maxFileSize>${MAX_FILE_SIZE}</maxFileSize>
      <maxHistory>${MAX_HISTORY}</maxHistory>
      <totalSizeCap>${TOTAL_SIZE_CAP}</totalSizeCap>
    </rollingPolicy>
    <encoder>
      <pattern>${ENCODER_PATTERN}</pattern>
      <charset>utf-8</charset>
    </encoder>
    <!-- 只打印INFO日志 -->
    <filter class="ch.qos.logback.classic.filter.LevelFilter">
      <level>INFO</level>
      <onMatch>ACCEPT</onMatch>
      <onMismatch>DENY</onMismatch>
    </filter>
  </appender>
 
  <!--输出warn级别日志到文件-->
  <appender name="WARN_LOG" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
      <!-- 每天一归档 -->
      <fileNamePattern>${LOG_DIR}/warn.%d{yyyy-MM-dd}-%i.log</fileNamePattern>
      <!-- 单个日志文件最多 150MB, 30天的日志周期，最大不能超过30GB -->
      <maxFileSize>${MAX_FILE_SIZE}</maxFileSize>
      <maxHistory>${MAX_HISTORY}</maxHistory>
      <totalSizeCap>${TOTAL_SIZE_CAP}</totalSizeCap>
    </rollingPolicy>
    <encoder>
      <pattern>${ENCODER_PATTERN}</pattern>
      <charset>utf-8</charset>
    </encoder>
    <!-- 只打印WARN日志 -->
    <filter class="ch.qos.logback.classic.filter.LevelFilter">
      <level>WARN</level>
      <onMatch>ACCEPT</onMatch>
      <onMismatch>DENY</onMismatch>
    </filter>
  </appender>
 
  <!--输出error级别日志到文件-->
  <appender name="ERROR_LOG" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
      <!-- 每天一归档 -->
      <fileNamePattern>${LOG_DIR}/error.%d{yyyy-MM-dd}-%i.log</fileNamePattern>
      <!-- 单个日志文件最多 150MB, 30天的日志周期，最大不能超过30GB -->
      <maxFileSize>${MAX_FILE_SIZE}</maxFileSize>
      <maxHistory>${MAX_HISTORY}</maxHistory>
      <totalSizeCap>${TOTAL_SIZE_CAP}</totalSizeCap>
    </rollingPolicy>
    <append>true</append>
    <encoder>
      <pattern>${ENCODER_PATTERN}</pattern>
      <charset>utf-8</charset>
    </encoder>
    <!-- 只打印ERROR日志 -->
    <filter class="ch.qos.logback.classic.filter.LevelFilter">
      <level>ERROR</level>
      <onMatch>ACCEPT</onMatch>
      <onMismatch>DENY</onMismatch>
    </filter>
  </appender>
 
  <appender name="SQL_LOG" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <!-- 每天一归档 -->
            <fileNamePattern>${LOG_DIR}/sql.%d{yyyy-MM-dd}-%i.log</fileNamePattern>
            <!-- 单个日志文件最多 150MB, 30天的日志周期，最大不能超过30GB -->
            <maxFileSize>${MAX_FILE_SIZE}</maxFileSize>
            <maxHistory>${MAX_HISTORY}</maxHistory>
            <totalSizeCap>${TOTAL_SIZE_CAP}</totalSizeCap>
        </rollingPolicy>
        <append>true</append>
        <encoder>
            <pattern>${ENCODER_PATTERN}</pattern>
            <charset>utf-8</charset>
        </encoder>
 
   </appender>
 
   <appender name ="ASYNC_SQL" class= "ch.qos.logback.classic.AsyncAppender">
        <!-- 不丢失日志.默认的,如果队列的80%已满,则会丢弃TRACT、DEBUG、INFO级别的日志 -->
        <discardingThreshold>0</discardingThreshold>
        <!-- 更改默认的队列的深度,该值会影响性能.默认值为256 -->
        <queueSize>512</queueSize>
        <!-- 添加附加的appender,最多只能添加一个 -->
        <appender-ref ref ="SQL_LOG"/>
    </appender>
 
  <!--将日志写到远程logstash里-->
  <appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
    <!--配置1M的buffer-->
    
    <writeBufferSize>1048576</writeBufferSize>
    <ringBufferSize>1048576</ringBufferSize>
    <param name="Encoding" value="UTF-8"/>
    <!-- logstas的地址  -->
    <remoteHost>${LOGSTASH_REMOTE_HOST}</remoteHost>
    <!-- logstas的端口号  -->
    <port>${LOGSTASH_PORT}</port>
    <!-- encoder is required -->
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
      <includeContext>false</includeContext>
      <!-- 记录skywalking的stanceId -->
      <provider class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.logstash.TraceIdJsonProvider">
      </provider>
      <!-- 可以用type来区环境  app_name来区分服务-->
      <customFields>{"type":"microservice","app_name":"${APP_NAME}","tags":"${TAGS}","app_port":"${APP_PORT}"}</customFields>
    </encoder>
  </appender>
 
 
  <!-- 异步输出日志 -->
  <appender name ="ASYNC_ERROR" class= "ch.qos.logback.classic.AsyncAppender">
    <!-- 不丢失日志.默认的,如果队列的80%已满,则会丢弃TRACT、DEBUG、INFO级别的日志 -->
    <discardingThreshold>0</discardingThreshold>
    <!-- 更改默认的队列的深度,该值会影响性能.默认值为256 -->
    <queueSize>512</queueSize>
    <!-- 添加附加的appender,最多只能添加一个 -->
    <appender-ref ref ="ERROR_LOG"/>
  </appender>
 
  <appender name ="ASYNC_WARN" class= "ch.qos.logback.classic.AsyncAppender">
    <!-- 不丢失日志.默认的,如果队列的80%已满,则会丢弃TRACT、DEBUG、INFO级别的日志 -->
    <discardingThreshold>0</discardingThreshold>
    <!-- 更改默认的队列的深度,该值会影响性能.默认值为256 -->
    <queueSize>512</queueSize>
    <!-- 添加附加的appender,最多只能添加一个 -->
    <appender-ref ref ="WARN_LOG"/>
  </appender>
 
  <appender name ="ASYNC_INFO" class= "ch.qos.logback.classic.AsyncAppender">
    <!-- 不丢失日志.默认的,如果队列的80%已满,则会丢弃TRACT、DEBUG、INFO级别的日志 -->
    <discardingThreshold>0</discardingThreshold>
    <!-- 更改默认的队列的深度,该值会影响性能.默认值为256 -->
    <queueSize>512</queueSize>
    <!-- 添加附加的appender,最多只能添加一个 -->
    <appender-ref ref ="INFO_LOG"/>
  </appender>
 
  <appender name ="ASYNC_DEBUG" class= "ch.qos.logback.classic.AsyncAppender">
    <!-- 不丢失日志.默认的,如果队列的80%已满,则会丢弃TRACT、DEBUG、INFO级别的日志 -->
    <discardingThreshold>0</discardingThreshold>
    <!-- 更改默认的队列的深度,该值会影响性能.默认值为256 -->
    <queueSize>512</queueSize>
    <!-- 添加附加的appender,最多只能添加一个 -->
    <appender-ref ref ="DEBUG_LOG"/>
  </appender>
 
 
 
    <!--将sql日志打印到文件的同时传递到root logger-->
    <logger name="jdbc.connection"  level="ERROR" >
        <appender-ref ref="ASYNC_SQL" />
    </logger>
    <logger name="jdbc.resultset"  level="OFF">
        <appender-ref ref="ASYNC_SQL" />
    </logger>
    <logger name="jdbc.audit"  level="OFF">
        <appender-ref ref="ASYNC_SQL" />
    </logger>
    <logger name="jdbc.sqlonly"  level="OFF">
        <appender-ref ref="ASYNC_SQL" />
    </logger>
    <logger name="jdbc.sqltiming"  level="INFO">
        <appender-ref ref="ASYNC_SQL" />
    </logger>
    <logger name="org.springframework.orm.ibatis" level="INFO">
        <appender-ref ref="ASYNC_SQL" />
    </logger>
    <logger name="org.springframework.transaction" level="INFO"  >
        <appender-ref ref="ASYNC_SQL" />
    </logger>
    <logger name="org.springframework.jpa" level="INFO"  >
        <appender-ref ref="ASYNC_SQL" />
    </logger>
    <logger name="org.springframework.jdbc" level="INFO"  >
        <appender-ref ref="ASYNC_SQL" />
    </logger>
    <logger name="org.springframework.jdbc.datasource.DataSourceTransactionManager" level="INFO"  >
        <appender-ref ref="ASYNC_SQL" />
    </logger>
 
 
 
    <!-- 过滤日志 level=OFF可以关闭-->
    <logger name="javax.activation" level="WARN"  />
    <logger name="javax.mail" level="WARN"  />
    <logger name="javax.xml.bind" level="WARN"  />
    <logger name="ch.qos.logback" level="WARN"  />
    <logger name="com.codahale.metrics" level="WARN"  />
    <logger name="com.ryantenney" level="WARN"  />
    <logger name="com.sun" level="WARN"  />
    <logger name="com.zaxxer" level="WARN"  />
    <logger name="io.undertow" level="WARN"  />
    <logger name="net.sf.ehcache" level="ERROR"  />
    <logger name="org.apache" level="WARN"  />
    <logger name="org.apache.catalina.startup.DigesterFactory" level="OFF"/>
    <logger name="org.bson" level="WARN"  />
    <logger name="org.hibernate.validator" level="WARN"  />
    <logger name="org.hibernate" level="WARN"  />
    <logger name="org.hibernate.ejb.HibernatePersistence" level="OFF"/>
    <logger name="org.springframework" level="WARN"  />
    <logger name="org.springframework.web" level="WARN"  />
    <logger name="org.springframework.security" level="WARN"  />
    <logger name="org.springframework.cache" level="WARN"  />
    <logger name="org.springframework.boot" level="WARN"  />
    <logger name="org.thymeleaf" level="WARN"  />
    <logger name="org.xnio" level="WARN"  />
    <logger name="springfox" level="WARN"  />
    <logger name="sun.rmi" level="WARN"  />
    <logger name="liquibase" level="WARN" />
    <logger name="sun.rmi.transport" level="WARN"  />
 
    <!--为了测试，打开更多日志-->
    <logger name="com.zaxxer.hikari.pool.HikariPool" level="INFO" />
    <!--打印配置文件加载顺序,有利于问题排查 https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-external-config.html -->
    <logger name="org.springframework.boot.context.config.ConfigFileApplicationListener"
            level="INFO"  />
 
    <!-- LoggerContextListener 接口的实例能监听 logger context 上发生的事件，比如说日志级别的变化，可做日志级别变更穿透-->
    <contextListener class="ch.qos.logback.classic.jul.LevelChangePropagator">
        <resetJUL>true</resetJUL>
    </contextListener>
 
  
 
  <springProfile name="prod">
    <root level="INFO">
      <appender-ref ref="LOGSTASH"/>
      <appender-ref ref="ASYNC_INFO" />
      <appender-ref ref="ASYNC_WARN" />
      <appender-ref ref="ASYNC_ERROR" />
 
    </root>
  </springProfile>
 
  <springProfile name="test">
    <root level="INFO">
      <appender-ref ref="CONSOLE"/>     
      <appender-ref ref="LOGSTASH"/>
      <appender-ref ref="ASYNC_INFO" />
      <appender-ref ref="ASYNC_WARN" />
      <appender-ref ref="ASYNC_ERROR" />
    </root>
  </springProfile>
 
  <springProfile name="dev">
    <root level="DEBUG">
      <appender-ref ref="CONSOLE"/>
    </root>
  </springProfile>
 
 
</configuration>
```

## 参考

*   &#x20;[https://www.cnblogs.com/xrq730/p/8628945.html](https://www.cnblogs.com/xrq730/p/8628945.html "https://www.cnblogs.com/xrq730/p/8628945.html")
