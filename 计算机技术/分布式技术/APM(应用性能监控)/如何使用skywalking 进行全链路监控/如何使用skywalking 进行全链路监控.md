# 如何使用skywalking 进行全链路监控

## 本文涉及内容

*   skywalking 全链路监控

*   skywalking 的参数配置

*   skywalking UI 监控视角与指标介绍

*   一些很有用的点

## skywalking 全链路监控

下图是我从网上找到的一个比较常见的微服务架构，看的出来使用的是 spring cloud 框架组件，后端服务是 java。我所谓的全链路监控是 从 Nginx 到数据库 这个链路的监控。

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ef68e12296f94a6eb008019d4139f9c1\~tplv-k3u1fbpfcp-zoom-1.image "图片")

我们知道 skywalking 可以通过 agent 比较方便的监控到后端的 java 应用。有关 skywalking 的安装请参考官方文档\\\[1\\]

以下是几个界面截图：通过 skywalking , 我们可以从服务入口开始一直监控到数据库，甚至是数据库的 sql 以及参数都可以一览无余（*sql 参数显示需要单独配置，后面会讲*）。

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6106dcaa0a1e487bbc5813289fcdf3e9\~tplv-k3u1fbpfcp-zoom-1.image "图片")

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8d33fc0cfb0e4c308aeca6eff19c1396\~tplv-k3u1fbpfcp-zoom-1.image "图片")

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4169c3777f6941beac4560f4e31c142f\~tplv-k3u1fbpfcp-zoom-1.image "图片")

然而我们并没有监控到请求的上游源头，即 Nginx 入口，如果我们将从 Nginx 入口来的并且经由 java 服务最终到数据库的请求全部监控起来，就完成了请求的全链路监控。上面我们处理了下半段，现在我们来处理上半段。

skywalking-nginx-lua\\\[2\\] 这是 skywalking 的另一个项目，可以通过它来对nginx进行监控。skywalking-nginx-lua 是使用lua来织入 agent 的。所以要求你的 nginx 要么有 lua 模块，要么用 openResty 这样的自带 Lua 功能模块的软件。

我使用的是openResty，只需要加以下配置就可以实现监控（注意中文注释部分）：

```bash
http {
    lua_package_path "/Path/to/.../skywalking-nginx-lua/lib/skywalking/?.lua;;";

    # Buffer represents the register inform and the queue of the finished segment
    lua_shared_dict tracing_buffer 100m;

    # Init is the timer setter and keeper
    # Setup an infinite loop timer to do register and trace report.
    init_worker_by_lua_block {
        local metadata_buffer = ngx.shared.tracing_buffer

        -- Set service name
        metadata_buffer:set('serviceName', 'User Service Name')
        -- Instance means the number of Nginx deployment, does not mean the worker instances
        metadata_buffer:set('serviceInstanceName', 'User Service Instance Name')
        #这是你的skywalking server地址
        require("client"):startBackendTimer("http://127.0.0.1:12800")
    }

    server {
        listen 8080;

        location /ingress {
            default_type text/html;

            rewrite_by_lua_block {
                ------------------------------------------------------
                -- NOTICE, this should be changed manually
                -- This variable represents the upstream logic address
                -- Please set them as service logic name or DNS name
                --
                -- Currently, we can not have the upstream real network address
                ------------------------------------------------------
                require("tracer"):start("upstream service")
                -- If you want correlation custom data to the downstream service
                -- require("tracer"):start("upstream service", {custom = "custom_value"})
            }

            # 这是你的目标下游服务，比如java的微服务网关
            proxy_pass http://127.0.0.1:8080/backend;

            body_filter_by_lua_block {
                if ngx.arg[2] then
                    require("tracer"):finish()
                end
            }

            log_by_lua_block {
                require("tracer"):prepareForReport()
            }
        }
    }
}

```

下面是几个监控到的nginx数据的截图

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/44141839afa24a3196203e4614d3a07c\~tplv-k3u1fbpfcp-zoom-1.image "图片")

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e1ccbbf8e8f44cf9b55fd199d4441d34\~tplv-k3u1fbpfcp-zoom-1.image "图片")

至此我们就完成了整个链路的监控。 &#x20;

## skywalking 的参数配置

### 一些中文文档

*   agent的文档\\\[3\\]

*   ui的文档\\\[4\\]

### 通过修改agent/config/agenet.config 文件得到的能力

根据文档 `https://github.com/apache/skywalking/blob/v8.0.0/docs/en/setup/service-agent/java-agent/README.md` 得知

*   1 可以获取 sql中的参数，默认是获取不到的。当然还要设置参数最大长度。但获取参数有可能引起性能问题。

| property key                               | Description                                                                                                                                                      | Default |
| ------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------- |
| plugin.mysql.trace\_sql\_ parameters       | If set to true, the parameters of the sql (typically java.sql.PreparedStatement) would be collected.                                                             | false   |
| plugin.mysql.sql\_parameters\_ max\_length | If set to positive number, the db.sql.parameters would be truncated to this length, otherwise it would be completely saved, which may cause performance problem. | 512     |

*   2 收集http参数

```bash
#收集SpringMVC plugin插件请求参，在tomcat上时这俩设置一个即可plugin.tomcat.collect_http_params or   plugin.springmvc.collect_http_params
 plugin.springmvc.collect_http_params=true
 #请求参数收集的最大字符长度, 配置过大会影响性能.
 plugin.http.http_params_length_threshold=1024

```

*   3 skywalking-oap 的配置文件中关于数据存储时长的配置

```bash
core:
  selector: ${SW_CORE:default}
  default:
    # Mixed: Receive agent data, Level 1 aggregate, Level 2 aggregate
    # Receiver: Receive agent data, Level 1 aggregate
    # Aggregator: Level 2 aggregate
    role: ${SW_CORE_ROLE:Mixed} # Mixed/Receiver/Aggregator
    restHost: ${SW_CORE_REST_HOST:0.0.0.0}
    restPort: ${SW_CORE_REST_PORT:12800}
    restContextPath: ${SW_CORE_REST_CONTEXT_PATH:/}
    gRPCHost: ${SW_CORE_GRPC_HOST:0.0.0.0}
    gRPCPort: ${SW_CORE_GRPC_PORT:11800}
    gRPCSslEnabled: ${SW_CORE_GRPC_SSL_ENABLED:false}
    gRPCSslKeyPath: ${SW_CORE_GRPC_SSL_KEY_PATH:""}
    gRPCSslCertChainPath: ${SW_CORE_GRPC_SSL_CERT_CHAIN_PATH:""}
    gRPCSslTrustedCAPath: ${SW_CORE_GRPC_SSL_TRUSTED_CA_PATH:""}
    downsampling:
      - Hour
      - Day
      - Month
    # Set a timeout on metrics data. After the timeout has expired, the metrics data will automatically be deleted.
    enableDataKeeperExecutor: ${SW_CORE_ENABLE_DATA_KEEPER_EXECUTOR:true} # Turn it off then automatically metrics data delete will be close.
    dataKeeperExecutePeriod: ${SW_CORE_DATA_KEEPER_EXECUTE_PERIOD:5} # How often the data keeper executor runs periodically, unit is minute
    recordDataTTL: ${SW_CORE_RECORD_DATA_TTL:3} # Unit is day
    metricsDataTTL: ${SW_CORE_RECORD_DATA_TTL:7} # Unit is day

```

主要是这四行

```bash
enableDataKeeperExecutor: ${SW_CORE_ENABLE_DATA_KEEPER_EXECUTOR:true} # Turn it off then automatically metrics data delete will be close.
dataKeeperExecutePeriod: ${SW_CORE_DATA_KEEPER_EXECUTE_PERIOD:5} # How often the data keeper executor runs periodically, unit is minute
recordDataTTL: ${SW_CORE_RECORD_DATA_TTL:3} # Unit is day
metricsDataTTL: ${SW_CORE_RECORD_DATA_TTL:7} # Unit is day

```

## skywalking UI 监控视角与指标介绍

### cpm 每分钟请求数

cpm 全称 call per minutes，是吞吐量(Throughput)指标。下图是拼接的全局、服务、实例和接口的吞吐量及平均吞吐量。

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e60c6b64ecd246d791a6f25f1dcfe269\~tplv-k3u1fbpfcp-zoom-1.image "图片")

第一条185cpm=185/60=3.08个请求/秒。

### SLA 服务等级协议

SLA 全称 Service-Level Agreement，直译为 “服务等级协议”，用来表示提供服务的水平。在IT中，SLA可以衡量平台的可用性，下面是N个9的计算：

1.  1年 = 365天 = 8760小时

2.  99     = 8760 \\\* 1%     => 3.65天

3.  99.9   = 8760 \\\* 0.1%   => 8.76小时

4.  99.99  = 8760 \\\* 0.01%  => 52.6分钟

5.  99.999 = 8760 \\\* 0.001% => 5.26分钟

因此，全年只要发生一次较大规模宕机事故，4个9肯定没戏，一般平台3个9差不多。但2个9就基本不可用了，相当于全年有87.6小时不可用，每周(一个月按4周算)有1.825小时不可用。下图是服务、实例、接口的SLA，一般看年度、月度即可。

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5f1ccea5b7ca4bd28f2a63ba00734d56\~tplv-k3u1fbpfcp-zoom-1.image "图片")

### Percent Response 百分位数统计

表示采集样本中某些值的占比，Skywalking 有 `p50、p75、p90、p95、p99` 一些列值。其中的 “p99:390” 表示 99% 请求的响应时间在390ms以内。而99%一般用于抛掉一些极端值，表示绝大多数请求。

### Slow Endpoint 慢端点

Endpoint 表示具体的服务，例如一个接口。下面是全局Top N的数据，通过这个可以观测平台性能情况。

### Heatmap 热力图

Heapmap 可译为热力图、热度图都可以，其中颜色越深，表示请求数越多，这和GitHub Contributions很像，commit越多，颜色越深。横坐标是响应时间，鼠标放上去，可以看到具体的数量。通过热力图，一方面可以直观感受平台的整体流量，另一方面也可以感受整体性能。

### apdex

是一个衡量服务器性能的标准。apdex有三个指标：

*   满意：请求响应时间小于等于T。

*   可容忍：请求响应时间大于T，小于等于4T。

*   失望：请求响应时间大于4T。

T：自定义的一个时间值，比如：500ms。apdex = （满意数 + 可容忍数/2）/ 总数。例如：服务A定义T=200ms，在100个采样中，有20个请求小于200ms，有60个请求在200ms到800ms之间，有20个请求大于800ms。计算apdex = (20 + 60/2)/100 = 0.5。

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c0f1a7cdc208486c978410d020937a5d\~tplv-k3u1fbpfcp-zoom-1.image "图片")

一些很有用的点

\-------

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/de3152d711e442c9aa9c898ae1c348c8\~tplv-k3u1fbpfcp-zoom-1.image "图片")

在拓扑图中

红色代表当前节点的请求有一段时间内是响应异常的。当节点全部变红的时候证明服务现阶段内就彻底不可用了。我们可以通过Topology迅速发现某一个服务潜在的问题，并进行下一步的排查并做到预防。

仔细看线是有流向的，有单向和双向的，单向有从左至右的或从右至左的，这样你就知道你的服务是谁依赖了谁。双向的就证明你的服务有循环引用依赖问题。

在最新版本8.1中有endpoint端口依赖的分析，可以分析出接口级别的依赖关系，可以知道某接口是被谁调用，它又调用了谁。

![图片](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8656a50cfd9d4f4e98e428db104f31a5\~tplv-k3u1fbpfcp-zoom-1.image "图片")

### 参考资料

*   skywalking官方文档: [*https://github.com/apache/skywalking/blob/master/docs/en/setup/README.md*](https://github.com/apache/skywalking/blob/master/docs/en/setup/README.md "https://github.com/apache/skywalking/blob/master/docs/en/setup/README.md")

*   skywalking-nginx-lua项目地址: [*https://github.com/apache/skywalking-nginx-lua/*](https://github.com/apache/skywalking-nginx-lua/ "https://github.com/apache/skywalking-nginx-lua/")

*   skywalking-agent文档: [*https://skyapm.github.io/document-cn-translation-of-skywalking/zh/8.0.0/setup/service-agent/java-agent/*](https://skyapm.github.io/document-cn-translation-of-skywalking/zh/8.0.0/setup/service-agent/java-agent/ "https://skyapm.github.io/document-cn-translation-of-skywalking/zh/8.0.0/setup/service-agent/java-agent/")

*   skywalking-ui 文档: [*https://skyapm.github.io/document-cn-translation-of-skywalking/zh/8.0.0/ui/*](https://skyapm.github.io/document-cn-translation-of-skywalking/zh/8.0.0/ui/ "https://skyapm.github.io/document-cn-translation-of-skywalking/zh/8.0.0/ui/")
