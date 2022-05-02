# mysql线上DDL 锁表 问题

**总结来说可以使用mysql5.6之后自带的online DDL 解决**

![](https://cdn.nlark.com/yuque/0/2020/png/1069608/1605060340639-58b31b4b-69d8-4848-90aa-6180e0c836c5.png)

## 如果是大表在线 DDL

### 可以使用主备切换的方法

**在主库新建表然后从老表同步数据的方式，这种方式可以使用工具同步数据**

*   pt-online-schema-change（但使用这个工具有条件，不能有触发器和外键）

*   gh-ost [https://github.com/github/gh-ost](https://github.com/github/gh-ost "https://github.com/github/gh-ost")

pt-online-schema-change 是percona公司开发的一个工具，在percona-toolkit包里面可以找到这个功能，它可以在线修改表结构 [https://www.percona.com/doc/percona-toolkit/3.0/pt-online-schema-change.html](https://www.percona.com/doc/percona-toolkit/3.0/pt-online-schema-change.html "https://www.percona.com/doc/percona-toolkit/3.0/pt-online-schema-change.html")

### 利用云产品的数据同步服务

## 参考

*   [https://help.aliyun.com/knowledge\_detail/41733.html](https://help.aliyun.com/knowledge_detail/41733.html "https://help.aliyun.com/knowledge_detail/41733.html")

*   [https://www.alibabacloud.com/help/zh/doc-detail/98373.htm](https://www.alibabacloud.com/help/zh/doc-detail/98373.htm "https://www.alibabacloud.com/help/zh/doc-detail/98373.htm")
