# ES常用配置

```yaml
#设置这个集群,有多少个节点有master候选资格,如果集群较大官方建议为2-4个
# es7 无须这个配置，会自己选择形成仲裁的节点
#discovery.zen.minimum_master_nodes: 2


#  集群名称 默认是   elasticsearch
cluster.name: esp

# 节点名称 
node.name: node-1


#指定该节点是否有资格被选举成为node（注意这里只是设置成有资格， 不代表该node一定就是master），
#默认是true，es是默认集群中的第一台机器为master，如果这台机挂了就会重新选举master。
node.master: true



# 指定该节点是否存储索引数据，默认为true。
node.data: true



#控制集群在达到多少个节点之后才会开始数据恢复,通过这个设置可以避免集群自动相互发现的初期,shard分片不全的问题,
#假如es集群内一共有5个节点,就可以设置为5,那么这个集群必须有5个节点启动后才会开始数据分片,如果设置为3,就有可能另外两个节点没存储数据分片
gateway.recover_after_nodes: 2
 
#为es 锁住内存, 有可能会启动报错 可以去百度配置，然后重启系统， 不然就执行  sudo swapoff -a
#bootstrap.memory_lock: true

# 避免发生OOM，发生OOM对集群影响很大的
indices.breaker.total.limit: 60%

# 有了这个设置，最久未使用（LRU）的 fielddata 会被回收为新数据腾出空间   
indices.fielddata.cache.size: 25%

# fielddata 断路器默认设置堆的  作为 fielddata 大小的上限。
indices.breaker.fielddata.limit: 40%

# request 断路器估算需要完成其他请求部分的结构大小，例如创建一个聚合桶，默认限制是堆内存
indices.breaker.request.limit: 40%



# 集群内部通信端口，默认是 9300 。 如果需要在 同一堆机器里面配置2套 ES集群就需要修改这个端口
transport.port: 9800


# 提高写入性能和稳定性
indices.memory.index_buffer_size: 15%
thread_pool.write.queue_size: 1024




# JVM
# Xmx 不要超过物理内存的 50% 
# 关闭 JVM的 Swapping  
# https://www.elastic.co/guide/en/elasticsearch/reference/7.x/setup-configuration-memory.html
# 节点出现高内存占用。可以执行清除缓存的操作。会影响查询性能，但是可以避免集群出现OOM
# curl --user admin:pwd  -XPOST 'http://localhost:9200/_cache/clear'
# indices.breaker.total.limit　　所有breaker使用的内存值，默认值为 JVM 堆内存的70%，当内存达到最高值时会触发内存回收。
# 设置各种 indices.breaker.total.limit 避免发生OOM，发生OOM对集群影响很大的


# API 设置
# Transient 在集群重启后会丢失
# Persistent 在集群中重启后不会丢失


# 设置 关闭 动态索引，即 直接插入数据的时候，没有索引自己创建索引
# 设置 action.auto_create_index : false


# 了解集群的使用状况
# Support Diagnostics Tool  或者 eBay Diagnostic Tool  或者 阿里云的EYOU 
# https://github.com/elastic/support-diagnostics

# Master 节点、数据节点宕机 -- 负载过高，导致节点失联
# 副本丢失，导致数据可靠性 
# 集群压力过大，数据写入失败


# 分片健康
# 红 ： 至少一个主分片没有分配
# 黄： 至少一个副本没有 分配
# 绿： 主副本分配全部正常分配
# 索引健康： 最差的分片的状态
# 集群健康： 最差的索引的状态


# get _cluster/allocation/explain 返回第一个未分片Shard的原因

# 查看索引级别，找到红色的索引
# get /_cluster/health?level=indices 
# 更多可以参考 https://www.elastic.co/guide/en/elasticsearch/reference/7.x/cat-shards.html
# 创建索引导致，会有短暂的Red ,不一定代表有问题
# 集群重启， 会有red 问题
# open 一个 之前close 的索引
# 一个节点离开集群期间，有索引被删除。 这个节点重新返回是，会导致Dangling 问题，这时候需要再执行一次删除即可


# 分片的管理
# 控制分片总数在10W以内
# 搜索类应用： 单个分片不要超过20GB
# 日志类应用，单个分片不要大于50GB
```

以上是我 一般配置 es 的 设置， 以及  使用注意说明 ，仅供参考。
更多调优参考 ：  [https://blog.csdn.net/andy\_only/article/details/98172044](https://blog.csdn.net/andy_only/article/details/98172044 "https://blog.csdn.net/andy_only/article/details/98172044")
参考  [https://www.cnblogs.com/cutd/p/5800795.html](https://www.cnblogs.com/cutd/p/5800795.html "https://www.cnblogs.com/cutd/p/5800795.html")

# indices.breaker.total.limit  配置说明

```yaml
7.x 说明：

Parent circuit breaker
The parent-level breaker can be configured with the following settings:

indices.breaker.total.use_real_memory
Static setting determining whether the parent breaker should take real memory usage into account (true) or only consider the amount that is reserved by child circuit breakers (false). Defaults to true.
indices.breaker.total.limit
Starting limit for overall parent breaker, defaults to 70% of JVM heap if indices.breaker.total.use_real_memory is false. If indices.breaker.total.use_real_memory is true, defaults to 95% of the JVM heap.



官方的翻译：

可用的断路器（Available Circuit Breakers）

Elasticsearch 有一系列的断路器，它们都能保证内存不会超出限制：

indices.breaker.fielddata.limit
fielddata 断路器默认设置堆的 60% 作为 fielddata 大小的上限。
indices.breaker.request.limit
request 断路器估算需要完成其他请求部分的结构大小，例如创建一个聚合桶，默认限制是堆内存的 40%。
indices.breaker.total.limit
total 揉合 request 和 fielddata 断路器保证两者组合起来不会使用超过堆内存的 70%。
```

通过英文，不是说高版本的 es 中
parent  break  即总的 父类断路器 默认是 95% 配置的jvm 内存， 但是 翻译的是  默认是 70% ， 且是 柔和了 request 和fielddata 的
但是实际 在 我使用的 7.1 版本中，我发现 默认的 parent break 对应的配置  indices.breaker.total.limit 并不起作用，默认还是 70%
而且 同时也会限制  request 和 fielddata 断路器 不能超过 parent break

```yaml
配置：
indices.breaker.total.use_real_memory: true

# 避免发生OOM，发生OOM对集群影响很大的
indices.breaker.total.limit: 90%

# 有了这个设置，最久未使用（LRU）的 fielddata 会被回收为新数据腾出空间   
indices.fielddata.cache.size: 5%

# fielddata 断路器默认设置堆的  作为 fielddata 大小的上限。
indices.breaker.fielddata.limit: 90%

# request 断路器估算需要完成其他请求部分的结构大小，例如创建一个聚合桶，默认限制是堆内存
indices.breaker.request.limit: 90%



http://120.24.36.51:9500/_nodes/stats/breaker?pretty 
通过查看断路器 限制信息：

{
  "_nodes" : {
    "total" : 3,
    "successful" : 3,
    "failed" : 0
  },
  "cluster_name" : "esp",
  "nodes" : {
    "mbWLWlWFTC-OfTDWzYukog" : {
      "timestamp" : 1588671497199,
      "name" : "node-31",
      "transport_address" : "172.18.0.184:9800",
      "host" : "172.18.0.184",
      "ip" : "172.18.0.184:9800",
      "roles" : [
        "master",
        "data",
        "ingest"
      ],
      "attributes" : {
        "ml.machine_memory" : "16656793600",
        "ml.max_open_jobs" : "20",
        "xpack.installed" : "true"
      },
      "breakers" : {
        "request" : {
          "limit_size_in_bytes" : 3844551475,
          "limit_size" : "3.5gb",
          "estimated_size_in_bytes" : 0,
          "estimated_size" : "0b",
          "overhead" : 1.0,
          "tripped" : 0
        },
        "fielddata" : {
          "limit_size_in_bytes" : 3844551475,
          "limit_size" : "3.5gb",
          "estimated_size_in_bytes" : 0,
          "estimated_size" : "0b",
          "overhead" : 1.03,
          "tripped" : 0
        },
        "in_flight_requests" : {
          "limit_size_in_bytes" : 6407585792,
          "limit_size" : "5.9gb",
          "estimated_size_in_bytes" : 912,
          "estimated_size" : "912b",
          "overhead" : 2.0,
          "tripped" : 0
        },
        "accounting" : {
          "limit_size_in_bytes" : 6407585792,
          "limit_size" : "5.9gb",
          "estimated_size_in_bytes" : 103206,
          "estimated_size" : "100.7kb",
          "overhead" : 1.0,
          "tripped" : 0
        },
        "parent" : {
          "limit_size_in_bytes" : 4485310054,
          "limit_size" : "4.1gb",
          "estimated_size_in_bytes" : 453680968,
          "estimated_size" : "432.6mb",
          "overhead" : 1.0,
          "tripped" : 0
        }
      }
    },
    "hnJ9k8zRRQKRA90rIZegLg" : {
      "timestamp" : 1588671497198,
      "name" : "node-21",
      "transport_address" : "172.18.0.182:9800",
      "host" : "172.18.0.182",
      "ip" : "172.18.0.182:9800",
      "roles" : [
        "master",
        "data",
        "ingest"
      ],
      "attributes" : {
        "ml.machine_memory" : "16656793600",
        "xpack.installed" : "true",
        "ml.max_open_jobs" : "20"
      },
      "breakers" : {
        "request" : {
          "limit_size_in_bytes" : 3844551475,
          "limit_size" : "3.5gb",
          "estimated_size_in_bytes" : 0,
          "estimated_size" : "0b",
          "overhead" : 1.0,
          "tripped" : 0
        },
        "fielddata" : {
          "limit_size_in_bytes" : 3844551475,
          "limit_size" : "3.5gb",
          "estimated_size_in_bytes" : 0,
          "estimated_size" : "0b",
          "overhead" : 1.03,
          "tripped" : 0
        },
        "in_flight_requests" : {
          "limit_size_in_bytes" : 6407585792,
          "limit_size" : "5.9gb",
          "estimated_size_in_bytes" : 16440,
          "estimated_size" : "16kb",
          "overhead" : 2.0,
          "tripped" : 0
        },
        "accounting" : {
          "limit_size_in_bytes" : 6407585792,
          "limit_size" : "5.9gb",
          "estimated_size_in_bytes" : 1050037,
          "estimated_size" : "1mb",
          "overhead" : 1.0,
          "tripped" : 0
        },
        "parent" : {
          "limit_size_in_bytes" : 4485310054,
          "limit_size" : "4.1gb",
          "estimated_size_in_bytes" : 488192520,
          "estimated_size" : "465.5mb",
          "overhead" : 1.0,
          "tripped" : 0
        }
      }
    },
    "BwLynNyKSOyvingwxVedmA" : {
      "timestamp" : 1588671497200,
      "name" : "node-11",
      "transport_address" : "172.18.0.183:9800",
      "host" : "172.18.0.183",
      "ip" : "172.18.0.183:9800",
      "roles" : [
        "master",
        "data",
        "ingest"
      ],
      "attributes" : {
        "ml.machine_memory" : "16656793600",
        "ml.max_open_jobs" : "20",
        "xpack.installed" : "true"
      },
      "breakers" : {
        "request" : {
          "limit_size_in_bytes" : 3844551475,
          "limit_size" : "3.5gb",
          "estimated_size_in_bytes" : 0,
          "estimated_size" : "0b",
          "overhead" : 1.0,
          "tripped" : 0
        },
        "fielddata" : {
          "limit_size_in_bytes" : 3844551475,
          "limit_size" : "3.5gb",
          "estimated_size_in_bytes" : 0,
          "estimated_size" : "0b",
          "overhead" : 1.03,
          "tripped" : 0
        },
        "in_flight_requests" : {
          "limit_size_in_bytes" : 6407585792,
          "limit_size" : "5.9gb",
          "estimated_size_in_bytes" : 912,
          "estimated_size" : "912b",
          "overhead" : 2.0,
          "tripped" : 0
        },
        "accounting" : {
          "limit_size_in_bytes" : 6407585792,
          "limit_size" : "5.9gb",
          "estimated_size_in_bytes" : 1856287,
          "estimated_size" : "1.7mb",
          "overhead" : 1.0,
          "tripped" : 0
        },
        "parent" : {
          "limit_size_in_bytes" : 4485310054,
          "limit_size" : "4.1gb",
          "estimated_size_in_bytes" : 309683168,
          "estimated_size" : "295.3mb",
          "overhead" : 1.0,
          "tripped" : 0
        }
      }
    }
  }
}
```

上面我每台es 配置了es  6G内存，可以看到，哪怕我都配置了 90% ，  parent 依然 limit\_size 还是 70% 一下，而 equest 和 fielddata 断路器 限制的内存不能超过  parent 最大的限制内存。
