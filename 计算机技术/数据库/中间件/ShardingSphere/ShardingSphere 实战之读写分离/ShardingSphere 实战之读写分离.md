# ShardingSphere 实战之读写分离

## 简述

采用 ShardingShpere 的 Sharding-Porxy（透明化的数据库代理端） 模式在本地实现 mysql 数据库读写分离,并通过 java 应用程序连接.

![](https://tva1.sinaimg.cn/large/008i3skNly1guhgp33asej61ha0u0n0602.jpg)

## sharding-sphere

本地下载并安装 最新 5.0 的 beta 版本：[https://dlcdn.apache.org/shardingsphere/5.0.0-beta/](https://dlcdn.apache.org/shardingsphere/5.0.0-beta/ "https://dlcdn.apache.org/shardingsphere/5.0.0-beta/")

由于我需要连接 mysql 数据库所以，需要下载 mysql-connector-java-5.1.47.jar（[https://repo1.maven.org/maven2/mysql/mysql-connector-java/5.1.47/mysql-connector-java-5.1.47.jar），并将其放入](https://repo1.maven.org/maven2/mysql/mysql-connector-java/5.1.47/mysql-connector-java-5.1.47.jar），并将其放入 "https://repo1.maven.org/maven2/mysql/mysql-connector-java/5.1.47/mysql-connector-java-5.1.47.jar），并将其放入") `%SHARDINGSPHERE_PROXY_HOME%/lib` 目录

### 修改配置文件

*   ShardingSphere-Proxy 支持多逻辑数据源，每个以 `config-` 前缀命名的 YAML 配置文件，即为一个逻辑数据源，比如默认文件 `config-database-discovery.yaml`

*   ShardingSphere-Proxy 默认使用 3307 端口，可以通过启动脚本追加参数作为启动端口号。如：bin/start.sh 3308

*   ShardingSphere-Proxy 使用 conf/server.yaml 配置注册中心、认证信息以及公用属性。

**先来添加并修改 config-myapp.yaml 文件（注意扩展名要写 yaml，写 yml 不能识别）**

```yaml
schemaName: my-app

dataSources:
 write_ds:
   url: jdbc:mysql://mysql.local.test.myapp.com:23306/test?allowPublicKeyRetrieval=true&useSSL=false&allowMultiQueries=true&serverTimezone=Asia/Shanghai&useSSL=false&autoReconnect=true&useUnicode=true&characterEncoding=utf-8&zeroDateTimeBehavior=convertToNull
   username: root
   password: nicai
   connectionTimeoutMilliseconds: 3000
   idleTimeoutMilliseconds: 60000
   maxLifetimeMilliseconds: 1800000
   maxPoolSize: 50
   minPoolSize: 1
   maintenanceIntervalMilliseconds: 30000
 read_ds_0:
   url: jdbc:mysql://mysql.local.test.read1.myapp.com:23306/test?allowPublicKeyRetrieval=true&useSSL=false&allowMultiQueries=true&serverTimezone=Asia/Shanghai&useSSL=false&autoReconnect=true&useUnicode=true&characterEncoding=utf-8&zeroDateTimeBehavior=convertToNull
   username: root
   password: nicai
   connectionTimeoutMilliseconds: 3000
   idleTimeoutMilliseconds: 60000
   maxLifetimeMilliseconds: 1800000
   maxPoolSize: 50
   minPoolSize: 1
   maintenanceIntervalMilliseconds: 30000

rules:
- !READWRITE_SPLITTING # 配置读写分离规则
  dataSources:
    pr_ds: # 读写分离的逻辑数据源名称 `pr_ds` 用于在数据分片中使用
      writeDataSourceName: write_ds #写库数据源名称
      readDataSourceNames: #读库数据源名称
        - read_ds_0
      loadBalancerName: roundRobin # 负载均衡算法名称
  # 负载均衡算法配置
  loadBalancers:
    roundRobin:
      type: ROUND_ROBIN # 一共两种一种是 RANDOM（随机），一种是 ROUND_ROBIN（轮询）

```

如上配置我只添加了一个主库一个只读从库，而数据库之间的主从同步过程由于不是重点本文就省略了，具体我这边比较简单直接用的云创建的只读实例，也就是说主从实例同步让云帮我实现了，当然你也可以用原生的方法，通过 mysql 的 master-salve 等配置来实现。

server.yaml 文件修改如下：

```yaml
rules:
 - !AUTHORITY
   users:
     - root@%:root
     - sharding@:sharding
   provider:
     type: ALL_PRIVILEGES_PERMITTED

props:
 max-connections-size-per-query: 1
 executor-size: 16  # Infinite by default.
 proxy-frontend-flush-threshold: 128  # The default value is 128.
   # LOCAL: Proxy will run with LOCAL transaction.
   # XA: Proxy will run with XA transaction.
   # BASE: Proxy will run with B.A.S.E transaction.
 proxy-transaction-type: LOCAL
 xa-transaction-manager-type: Atomikos
 proxy-opentracing-enabled: false
 proxy-hint-enabled: false
 sql-show: true
 check-table-metadata-enabled: false
 lock-wait-timeout-milliseconds: 50000 # The maximum time to wait for a lock

```

### 启动

1 在 bin 目录下 执行 [start.sh](http://start.sh "start.sh") 启动 ShardingSphere

2 用任意 mysql 客户端连接数据库，就像连接一个往常的数据库一样

*   地址：127.0.0.1

*   端口：3307

*   账号：root/root&#x20;

*   数据库 my-app （这里就用到上面配置的逻辑数据库名了）

3 查看库表中的数据，应该跟主、从库中的一致。

**这里我遇到了一个小问题，就是 select 数据的时候用 库名。表名不行，直接写表名可以。**

### java 应用程序修改配置

```yaml
spring:
  profiles:
    include: common-local
  datasource:
    url: jdbc:mysql://127.0.0.1:3307/my-app?allowPublicKeyRetrieval=true&useSSL=false&allowMultiQueries=true&serverTimezone=Asia/Shanghai&useSSL=false&autoReconnect=true&useUnicode=true&characterEncoding=utf-8&zeroDateTimeBehavior=convertToNull
    username: root
    password: root

```

由于采用 proxy 模式，对应用几乎无感，不需要修改代码，只需要修改数据库部分配置文件。

### 测试

在我测试读写分离时，读库的请求情况：

![](https://tva1.sinaimg.cn/large/008i3skNly1guhg2p2jhaj60qi0hyjs602.jpg)

证明读请求都打到读库上了。

### 遇到过的问题

1 启动报错，需要配置 server.yaml 第一次启动的时候没配置

2 启动报错： `The MySQL server is running with the --read-only option so it cannot execute this statement
`我的从库是设置的只读库，但不知道为什么会报错，没有解决，再次启动就好了。

3 启动成功后，用客户端，无法连接 sharding-proxy 数据库，连接异常报错，解决方法是修改了 server.yaml 文件

```yaml
rules:
 - !AUTHORITY
   users:
     - root@%:root
     - sharding@:sharding
   provider:
     type: ALL_PRIVILEGES_PERMITTED

```

将 provider 的 type  从之前的 `NATIVE` 修改为了 `ALL_PRIVILEGES_PERMITTED` （默认授予所有权限（不鉴权），不会与实际数据库数据库交互。）

## 参考：

*   [https://shardingsphere.apache.org/document/5.0.0-beta/cn/overview/](https://shardingsphere.apache.org/document/5.0.0-beta/cn/overview/ "https://shardingsphere.apache.org/document/5.0.0-beta/cn/overview/")

*   [https://shardingsphere.apache.org/blog/cn/material/database/](https://shardingsphere.apache.org/blog/cn/material/database/ "https://shardingsphere.apache.org/blog/cn/material/database/")
