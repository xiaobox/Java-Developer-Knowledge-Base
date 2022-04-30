# ShardingSphere 实现数据加密（脱敏）第一篇

## 背景

Apache ShardingSphere 通过对用户输入的 SQL 进行解析，并依据用户提供的加密规则对 SQL 进行改写，从而实现对原文数据进行加密，并将原文数据（可选）及密文数据同时存储到底层数据库。 在用户查询数据时，它仅从数据库中取出密文数据，并对其解密，最终将解密后的原始数据返回给用户。 Apache ShardingSphere 自动化 & 透明化了数据加密过程，让用户无需关注数据加密的实现细节，像使用普通数据那样使用加密数据。 此外，无论是已在线业务进行加密改造，还是新上线业务使用加密功能，Apache ShardingSphere 都可以提供一套相对完善的解决方案。

![](https://tva1.sinaimg.cn/large/008i3skNly1gup8hn30ckj60zo0ko0uf02.jpg)

## 解决方案

参考官方文档分两种场景

*   新上线业务

*   已上线业务

本文首先介绍新上线业务的新操，它相对已上线业务简单许多

## 新上线业务

![](https://tva1.sinaimg.cn/large/008i3skNly1gup8i2yt3wj60r109nmxm02.jpg)

### 版本信息

*   SpringBoot 2

*   ShardingSphere 5

*   MySQL 8

### 引入依赖

```xml
<dependency>
   <groupId>org.apache.shardingsphere</groupId>
   <artifactId>shardingsphere-jdbc-core-spring-boot-starter</artifactId>
   <version>5.0.0-beta</version>
</dependency>
```

### 配置文件：

```yaml
spring:
  profiles:
    include: common-local
  shardingsphere:
    datasource:
      names: write-ds,read-ds-0
      write-ds:
        jdbcUrl: jdbc:mysql://mysql.local.test.myallapp.com:23306/test?allowPublicKeyRetrieval=true&useSSL=false&allowMultiQueries=true&serverTimezone=Asia/Shanghai&useSSL=false&autoReconnect=true&useUnicode=true&characterEncoding=utf-8&zeroDateTimeBehavior=convertToNull
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        username: root
        password: nicai
        connectionTimeoutMilliseconds: 3000
        idleTimeoutMilliseconds: 60000
        maxLifetimeMilliseconds: 1800000
        maxPoolSize: 50
        minPoolSize: 1
        maintenanceIntervalMilliseconds: 30000
      read-ds-0:
        jdbcUrl: jdbc:mysql://mysql.local.test.read1.myallapp.com:23306/test?allowPublicKeyRetrieval=true&useSSL=false&allowMultiQueries=true&serverTimezone=Asia/Shanghai&useSSL=false&autoReconnect=true&useUnicode=true&characterEncoding=utf-8&zeroDateTimeBehavior=convertToNull
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        username: root
        password: nicai
        connectionTimeoutMilliseconds: 3000
        idleTimeoutMilliseconds: 60000
        maxLifetimeMilliseconds: 1800000
        maxPoolSize: 50
        minPoolSize: 1
        maintenanceIntervalMilliseconds: 30000
    rules:
      readwrite-splitting:
        data-sources:
          glapp:
            write-data-source-name: write-ds
            read-data-source-names:
              - read-ds-0
            load-balancer-name: roundRobin # 负载均衡算法名称
        load-balancers:
          roundRobin:
            type: ROUND_ROBIN # 一共两种一种是 RANDOM（随机），一种是 ROUND_ROBIN（轮询）
      encrypt:
        encryptors:
          mobile-encryptor:
            props:
              aes-key-value: 123456abc
            type: AES
        tables:
          t_cipher_new:
            columns:
              mobile:
                cipher-column: mobile # 加密列名称
                encryptor-name: mobile-encryptor # 加密算法名称（名称不能有下划线）
                # plain-column: mobile # 原文列名称
        queryWithCipherColumn: true # 是否使用加密列进行查询。在有原文列的情况下，可以使用原文列进行查询

```

我是将读写分离和加密的配置混合在一块儿了，实际上本文只需要关注 `encrypt` 节点部分

配置上有几个注意的点：

*   key 名称中间不要带下划线，比如`mobile-encryptor` 这个自定义的名字，之前找了半天原因，忘了规则了。

*   由于这部分演示的是新业务，所以将加密列和原文列用一列表示（mobile）, 而逻辑列也是这列。

### 建表语句：

```sql
CREATE TABLE `t_cipher_new` (
  `id` bigint(20) NOT NULL,
  `name` varchar(255) DEFAULT NULL,
  `create_time` datetime DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `pwd` varchar(100) DEFAULT NULL,
  `mobile` varchar(100) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

### 测试

先通过业务接口插入一条数据：

![](https://tva1.sinaimg.cn/large/008i3skNly1guqdx5fzs6j60qi03qjs802.jpg)

可以看到 `mobile` 字段已经自动加密了，明文是`1234567`密文是 `FGPDcFbE1uWPwPUOeRpKbw==`

我们再通过业务接口查询一下这条数据

```json
{
    "code": 100000,
    "msg": "",
    "data": {
        "id": 1440855970338058241,
        "name": "hello",
        "pwd": "123",
        "mobile": "1234567",
        "createTime": "2021-09-23 09:50:09",
        "updateTime": "2021-09-23 09:50:09"
    }
}
```

可以看到，查询出来的 `mobile` 字段数据是解密以后的 `1234567`

## 参考

*   [https://shardingsphere.apache.org/document/5.0.0-beta/cn/features/encrypt/principle/](https://shardingsphere.apache.org/document/5.0.0-beta/cn/features/encrypt/principle/ "https://shardingsphere.apache.org/document/5.0.0-beta/cn/features/encrypt/principle/")
