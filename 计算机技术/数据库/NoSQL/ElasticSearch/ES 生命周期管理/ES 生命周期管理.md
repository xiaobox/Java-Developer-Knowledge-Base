# ES 生命周期管理

对于 **日志** 或 **指标（metric）类时序性强** 的ES索引，因为数据量大，并且写入和查询大多都是近期时间内的数据。我们可以采用

**hot-warm-cold**

架构将索引数据切分成hot/warm/cold的索引。

`hot索引`

负责最新数据的读写，可使用内存存储；

`warm索引`

负责较旧数据的读取，可使用内存或SSD存储；

`cold索引`

很少被读取，可使用大容量磁盘存储。随着时间的推移，数据不断从hot索引->warm索引->cold索引迁移。针对不同阶段的索引我们还可以调整索引的主分片数，副本数，单分片的segment数等等，更好的利用机器资源。

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h1rkbwdbntj20r505a0sy.jpg)

**这一切ES都帮我们实现了。ES从6.7版本推出了索引生命周期管理（Index Lifecycle Management ，简称ILM**)机制，能帮我们自动管理一个`索引策略（Policy）`下索引集群的生命周期。索引策略将一个索引的生命周期定义为四个**阶段**：

*   **Hot**：索引可写入，也可查询。

*   **Warm**：索引不可写入，但可查询。

*   **Cold**：索引不可写入，但很少被查询，查询的慢点也可接受。

*   **Delete**：索引可被安全的删除。

索引策略控制这一个索引的生命从Hot -> Warm -> Cold -> Delete 阶段，每个阶段都可以配置不同的`转化行为（Action）`。下面我们看下几个常用的Action:

*   **Rollover** 当写入索引达到了一定的大小，文档数量或创建时间时，Rollover可创建一个新的写入索引，将旧的写入索引的别名去掉，并把别名赋给新的写入索引。所以便可以通过切换`别名`控制写入的索引是谁。它可用于`Hot`阶段。

*   **Shrink** 减少一个索引的主分片数，可用于`Warm`阶段。需要注意的是当shink完成后索引名会由原来的`<origin-index-name>`变为`shrink-<origin-index-name>`.

*   **Force merge** 可触发一个索引分片的segment merge，同时释放掉被删除文档的占用空间。用于`Warm`阶段。

*   **Allocate** 可指定一个索引的副本数，用于`warm, cold`阶段。

好了现在我们知道一个索引策略是由配置不同的阶段和每个阶段对应的Action组成，那怎么设置一个索引的索引策略呢？把冰箱装进大象分为三部：

**设置一个索引的索引策略**`1.创建一个索引策略`

```纯文本
PUT _ilm/policy/my_policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
//rollover前距离索引的创建时间最大为7天
            "max_age": "7d",
//rollover前索引的最大大小不超过50G
            "max_size": "50G",
//rollover前索引的最大文档数不超过1个（测试用）
            "max_docs": 1,
          }
        }
      },
      "warm": {
//rollover之后进入warm阶段的时间不小于30天
        "min_age": "30d",
        "actions": {
          "forcemerge": {
//强制分片merge到segment为1
            "max_num_segments": 1
          },
          "shrink": {
//收缩分片数为1
            "number_of_shards": 1
          },
          "allocate": {
//副本数为2
            "number_of_replicas": 2
          }
        }
      },
      "cold": {
//rollover之后进入cold阶段的时间不小于60天
        "min_age": "60d",
        "actions": {
          "allocate": {
            "require": {
//分配到cold 节点，ES可根据机器资源配置不同类型的节点
              "type": "cold"
            }
          }
        }
      },
      "delete": {
//rollover之后进入cold阶段的时间不小于60天
        "min_age": "90d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

`2.创建一个索引模版，指定使用的索引策略`

```纯文本
PUT _template/my_template
{
//模版匹配的索引名以"index-"开头
  "index_patterns": ["myindex-*"],                 
  "settings": {
//索引分片数为2
    "number_of_shards":2 ,
//索引副本数为1 
    "number_of_replicas": 1,
//索引使用的索引策略为my_policy
    "index.lifecycle.name": "full_policy",    
//索引rollover后切换的索引别名为  test-alias
    "index.lifecycle.rollover_alias": "myindex"    
  }
}
```

`3.创建一个符合上述索引模版的索引`

```纯文本
PUT index-000001
{
  "aliases": {
    "myindex":{ //别名为 myindex
//允许索引被写入数据
      "is_write_index": true 
    }
  }
}
```

当发生rollover时，老索引的别名myindex将被去掉，新创建的索引别名为myidex，同时索引名自动在索引名上自增，变为myindex-0002。此外对应的配置信息我已注释上了，大家慢慢看吧。

`小贴士`：部署ES集群节点的版本要统一，不然ILM可能出现意想不到的错误。

这里为啥要用索引模版来关联索引和索引策略呢？因为如果在创建索引时不通过模版指定索引策略，当发生rollover时，新的索引并不会继承原来索引的索引策略。

小伙伴将尝试了之后发现不对啊，我插入里两条数据并没有自动rollover啊。不慌，小姐姐是不会骗人的。ES检测索引的索引策略是否该生效的时间默认为10min，可通过修改以下配置：

```纯文本
PUT _cluster/settings
{
  "transient": {
    "indices.lifecycle.poll_interval": "3s" 
  }
}
```

3秒中检测一下是否可执行索引策略，应该够了。

**Logstash使用ILM**

问题来了，当我们使用ELK搭建索引日志系统时，咋让Logstash和ES的ILM无缝连接呢？ Logstash的`Elasticsearch output plugin`插件自从9.3.1版本之后就支持ILM了，我们只需要在Logstash的配置文件中简单配置下就可以全部托管给ES ILM了。

```纯文本
output {
      elasticsearch {
//发生rollover时的写入索引的别名
        ilm_rollover_alias => "myindex"
//将会附在ilm_rollover_alias的值后面共同构成索引名，myindex-00001
        ilm_pattern => "00001"
//使用的索引策略
        ilm_policy => "my_policy"
//使用的索引模版名称
        template_name => "my_template"
      }
    }
```

如果我们一直愉快的使用一个索引策略，当然很好。但是总有意外发生。。索引策略执行失败了怎么办，中途想改变索引策略换车怎么办？这都是问题。

**索引策略执行失败**
首先我们先看一下失败的原因是什么，可以用API查看一下：

```纯文本
GET /myindex/_ilm/explain
```

返回信息中`step_info`就是失败原因，假设是索引策略设置的有问题，比如说Shrink的主分片数设置的比模版的都大，我们只需要更新索引策略，解决问题。然后在重试让ILM继续执行下一步就好。

```纯文本
POST /myindex/_ilm/retry
```

**索引策略的更新**
我们可使用以下API更新索引策略，

```纯文本
PUT _ilm/policy/my_policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "25GB"
          }
        }
      },
      "delete": {
        "min_age": "30d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

查看索引策略发现，每次更新**索引策略的版本**都会增加。
对于还没有开始创建的索引，更新索引策略显然能够生效。

对于**已经存在的策略生效的索引**，当前阶段是不会按照最新版本的策略执行的，**必须等到变为下一个阶段了，才会按照最新版本的策略执行。**
如果想切换索引使用的索引策略，可以使用API进行修改：

```纯文本
PUT myindex/_settings
{
  "lifecycle.name": "my_other_policy"
}
```

此外，在老版本使用ILM机制时，可能还涉及到将原来的索引纳入索引策略管理中，将原来ES的curator索引滚动方案升级到ILM等问题。本文主要结合官方文档介绍了**ILM的开箱使用，Logstash使用ILM，索引策略执行失败和索引策略的更新的使用**。更多问题还请阅读官方文档，获得更好的体验。

## 参考

*   [https://zhuanlan.zhihu.com/p/137810661](https://zhuanlan.zhihu.com/p/137810661 "https://zhuanlan.zhihu.com/p/137810661")
