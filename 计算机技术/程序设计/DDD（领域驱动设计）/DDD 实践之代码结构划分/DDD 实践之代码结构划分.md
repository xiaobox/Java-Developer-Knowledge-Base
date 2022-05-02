# DDD 实践之代码结构划分

# 一 战略建模

*   划分限界上下文

*   确定上下文间的关系&#x20;

## 二 战术建模

*   实体(entity)&#x20;

*   值对象(value object)

*   聚合根(Aggregate)

*   领域服务(service）

## 三 基于充血模型的标准领域模型代码组织方式

*   模块（Module）是DDD中明确提到的一种控制限界上下文的手段，在我们的工程中，一般尽量用一个模块来表示一个领域的限界上下文。

*   一般的工程中包的组织方式为{com.公司名.组织架构.业务.上下文.\*}

*   对于模块内的组织结构，一般情况下我们是按照领域对象、领域服务、领域资源库、防腐层等组织方式定义的。

```java
import com.company.team.bussiness.lottery.domain.valobj.*;//领域对象-值对象
import com.company.team.bussiness.lottery.domain.entity.*;//领域对象-实体
import com.company.team.bussiness.lottery.domain.aggregate.*;//领域对象-聚合根
import com.company.team.bussiness.lottery.service.*;//领域服务
import com.company.team.bussiness.lottery.repo.*;//领域资源库
import com.company.team.bussiness.lottery.facade.*;//领域防腐层
```

## 四 其他组织形式

*   [https://www.infoq.cn/article/s\_LFUlU6ZQODd030RbH9](https://www.infoq.cn/article/s_LFUlU6ZQODd030RbH9 "https://www.infoq.cn/article/s_LFUlU6ZQODd030RbH9")

*   [https://zhuanlan.zhihu.com/p/91525839](https://zhuanlan.zhihu.com/p/91525839 "https://zhuanlan.zhihu.com/p/91525839")

*   利用领域模型框架COLA [https://github.com/alibaba/COLA](https://github.com/alibaba/COLA "https://github.com/alibaba/COLA")

*   [https://start.aliyun.com/bootstrap.html](https://start.aliyun.com/bootstrap.html "https://start.aliyun.com/bootstrap.html")

## 五 按传统MVC分层

### 源码包

*   api（模块）

    *   enum

    *   dto

    *   api（接口）

*   server（模块）

    *   common

        *   config

        *   enum

        *   constant

        *   util

    *   controller&#x20;

        *   api

        *   内部controller  &#x20;

    *   facade

        *   按业务分包

    *   service

        *   按业务提供服务

        *   mq

        *   annotation

        *   aop

        *   breaker

    *   repository

        *   mapper

        *   cache

    *   domain

        *   do（数据对象 xxxDO，xxx即为数据表名）

        *   dto (数据传输对象：xxxDTO，xxx为业务领域相关的名称)

### resource 包&#x20;

*   bootstrap.xml

*   application.xml

*   mapper

*   static

*   dev

    *   application-dev.xml

    *   logback-dev.xml&#x20;

*   test

    *   application-test.xml

    *   logback-test.xml

*   pre

    *   application-pre.xml

    *   logback-pre.xml

*   prod

    *   application-prod.xml

    *   logback-prod.xml

## 说明

*   server 模块里的 controller包中的 api api模块中接口的实现&#x20;

*   controller 即可以调 facade 层也可以调用 service 层

*   事务注解可以加在 facade 层也可以加在 service 层。 &#x20;

*   dto 到 do 转换在 service层
