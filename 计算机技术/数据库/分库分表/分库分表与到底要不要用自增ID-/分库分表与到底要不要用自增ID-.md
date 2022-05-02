# 分库分表与到底要不要用自增ID?

抛开具体业务需求和场景谈论技术方案，无异于纸上谈兵。没有哪一项技术或解决方案有绝对的好坏、优劣之分。都是相对意义上的区分，否则这些项技术或方案是怎么产生的？一定也是为了解决某类具体场景的问题而产生的，在彼时彼刻都可谓 “先进”技术。

### 从0到1

在系统新生的时候，预估业务量和数据量都不大。后来，业务越做越好，数据量越来越大，发现单库单表已经不能满足需求了，需要分库分表。你数据库的发展方向大概是这样的： &#x20;

![](https://mmbiz.qpic.cn/mmbiz_png/YZibCWq4rxD8GGYaJhW6EoHSaKL4hwRKtF6TB3WrETNz4uzQ9puhE2JTIuJkn5GajYFnibiaVDgJQ0AupJBOlO3IQ/640?wx_fmt=png\&wxfrom=5\&wx_lazy=1\&wx_co=1)

很少会有业务一开始就会设计为分库分表，虽说这样会减少后续的坑，但部分公司刚开始都是以业务为主。

### 水平 VS 垂直

如果是单个库太大，这时我们要看是因为表多而导致数据多，还是因为单张表里面的数据多。 如果是因为表多而数据多，使用垂直切分，根据业务切分成不同的库。

如果是因为单张表的数据量太大，这时要用水平切分，即把表的数据按某种规则切分成多张表，甚至多个库上的多张表。 分库分表的顺序应该是先垂直分，后水平分。 因为垂直分更简单，更符合我们处理现实世界问题的方式。

*   **垂直分表**

    也就是“大表拆小表”，基于列字段进行的。一般是表中的字段较多，将不常用的， 数据较大，长度较长（比如text类型字段）的拆分到“扩展表“。一般是针对那种几百列的大表，也避免查询时，数据量太大造成的“跨页”问题。

*   **垂直分库**

    垂直分库针对的是一个系统中的不同业务进行拆分，比如用户User一个库，商品Producet一个库，订单Order一个库。

*   **水平分表**

    针对数据量巨大的单张表（比如订单表），按照某种规则（RANGE,HASH取模等），切分到多张表里面去。

*   **水平分库分表**

    将单张表的数据切分到多个服务器上去，每个服务器具有相应的库与表，只是表中数据集合不同

    于是你开始分表了：也就是将一张大表数据通过某种路由算法将数据尽可能的均匀分配到 N 张小表中。

    ![](https://mmbiz.qpic.cn/mmbiz_png/YZibCWq4rxD8GGYaJhW6EoHSaKL4hwRKtia4reDgCHGwcWOricAjMenOk4ic18KQIv5NtSZWwpsyicACibzhTqxWdCibA/640?wx_fmt=png\&wxfrom=5\&wx_lazy=1\&wx_co=1)

    而分表策略也有好几种，分别适用不同的场景。比如通过创建时间、主键ID，或流行的 hash+mod 的组合（类似HashMap的策略）

    ![](https://mmbiz.qpic.cn/mmbiz_png/YZibCWq4rxD8GGYaJhW6EoHSaKL4hwRKtQDQueKXSMeof1qnPcIqT5hsyWn3XY2mw19sUU8OW3YMxSa5kxQx9qg/640?wx_fmt=png\&wxfrom=5\&wx_lazy=1\&wx_co=1)

    这里的 hash 便是将我们需要分表的字段进行一次散列运算，使得经过散列的数据尽可能的均匀并且不重复。当然如果本身这个字段就是一个整形并且不重复也可以省略这个步骤，直接进行 Mod 得到分表下标即可。

    **当然分表只是第一步，还要做数据迁移，除非是一开始就做好了分表。后面还有分页查询、groupby、多表join和事务等坑需要填。**

    目前市面上的分库分表中间件相对较多，可能拿来解决上述提到的问题。比如Atlas（360）、MySQL-Proxy、DBProxy（美团在Atlas上做的改进）、Mycat( 是 Cobar 的进化版本 ,cobar,是 Amoeba 的进化版本,都是阿里的 )

### 记录标识

我们回到表的记录标识上来，我们对这个标识有两个主要需求：

*   全局唯一

*   趋势有序

于是有了以下方法： &#x20;

1.  使用数据库自增 auto\_increment 来生成全局唯一递增ID。这本没什么问题，如果你使用MySQL，这也是官方建议的。优点不多说了，说下缺点：

    1\) 可用性难以保证：数据库常见架构是一主多从+读写分离；

    2\) 生成自增ID是写请求，主库挂了就玩不转了

    3\) 扩展性差，性能有上限：因为写入是单点，数据库主库的写性能决定ID的生成性能上限，并且难以扩展

2.  改进的方法：如果我们采用了分库分表的设计，可以每个写库设置不同的auto\_increment初始值，以及相同的增长步长。以保证每个数据库生成的ID是不同的。但缺点是丧失了ID生成的“绝对递增性”，数据库的写压力依然很大，每次生成ID都要访问数据库。这时候我们可以采用单点批量ID生成服务，用一个ID生成服务，先批量生成多个ID。这样应用访问ID生成服务索要ID，ID生成服务不需要每次访问数据库，这样好多了，但问题还有，就是服务还是单点。

3.  **UUID**保证在同一时空中的所有机器都是唯一的。且可以在本地生成，扩展性好，基本可以认为没有性能上限。是个方案，但UUID的也有缺点：无法保证趋势递增、uuid过长，往往用字符串表示，作为主键建立索引查询效率低（B+树为了维持平衡，会引起B+树的节点页分裂和碎片问题）。

4.  Twitter的**SnowFlake**是一个非常优秀的ID生成方案，实现也非常简单，8Byte 是一个Long,8Byte等于64bit，核心代码就是毫秒级时间41位+10位机器ID+毫秒内序列12位，也可以调整机器位数和毫秒内序列位数比例。借鉴snowflake的思想，结合各公司的业务逻辑和并发量，可以实现自己的分布式ID生成算法。比如：

    ![](https://mmbiz.qpic.cn/mmbiz/YrezxckhYOydMj2Lichnic5csTOdqI2a1mC7utPZR12icoHP8UzEibcVetkkZHxWg3ZNhuItMhiboT2AvBWWKvnDlzA/640?wx_fmt=png\&wxfrom=5\&wx_lazy=1\&wx_co=1)

    缺点是“没有一个全局时钟”，每台服务器分配的ID是绝对递增的，但从全局看，生成的ID只是趋势递增的（有些服务器的时间早，有些服务器的时间晚）美团的Leaf就是一个类SnowFlake的解决方案。

5.  甚至还可以利用redis的incr原子性操作自增，但安全性不足。

6.  使用**TiDB** 如果你大胆调整数据库，不使用MySQL了，而使用TiDB，自动扩容，业务不关心分库分表。TiDB 是由 PingCAP 研发的一款定位于在线事务处理 / 在线分析处理（HTAP）的开源融合型数据库产品，实现了一键水平伸缩，强一致性的多副本数据安全，分布式事务，实时 OLAP 等重要特性，目前已广泛应用于金融服务、互联网、制造等行业。

    但问题是**TiDB** 的自增 ID (AUTO\_INCREMENT) 只保证自增且唯一，并不保证连续分配。TiDB 目前采用批量分配的方式，所以如果在多台 TiDB 上同时插入数据，分配的自增 ID 会不连续。当多个线程并发往不同的 tidb-server 插入数据的时候，有可能会出现后插入的数据自增 ID 小的情况。此外，TiDB允许给整型类型的字段指定 AUTO\_INCREMENT，且一个表只允许一个属性为 AUTO\_INCREMENT 的字段。不过虽然有问题也有解决方案，感兴趣的小伙伴可以查找下相关资料。

## 参考：

*   [https://www.infoq.cn/article/key-steps-and-likely-problems-of-split-table](https://www.infoq.cn/article/key-steps-and-likely-problems-of-split-table "https://www.infoq.cn/article/key-steps-and-likely-problems-of-split-table")

*   [https://tech.meituan.com/2017/04/21/mt-leaf.html](https://tech.meituan.com/2017/04/21/mt-leaf.html "https://tech.meituan.com/2017/04/21/mt-leaf.html")

*   [https://www.cnblogs.com/jajian/p/11101213.html](https://www.cnblogs.com/jajian/p/11101213.html "https://www.cnblogs.com/jajian/p/11101213.html")

*   [https://www.cnblogs.com/butterfly100/p/9034281.html](https://www.cnblogs.com/butterfly100/p/9034281.html "https://www.cnblogs.com/butterfly100/p/9034281.html")
