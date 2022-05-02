# skywalking 的参数配置

## 中文文档

*   ageng的文档：[https://skyapm.github.io/document-cn-translation-of-skywalking/zh/8.0.0/setup/service-agent/java-agent/](https://skyapm.github.io/document-cn-translation-of-skywalking/zh/8.0.0/setup/service-agent/java-agent/ "https://skyapm.github.io/document-cn-translation-of-skywalking/zh/8.0.0/setup/service-agent/java-agent/")

*   ui的文档：[https://skyapm.github.io/document-cn-translation-of-skywalking/zh/8.0.0/ui/](https://skyapm.github.io/document-cn-translation-of-skywalking/zh/8.0.0/ui/ "https://skyapm.github.io/document-cn-translation-of-skywalking/zh/8.0.0/ui/")

## 通过修改agent/config/agenet.config 文件得到的能力

根据文档 [https://github.com/apache/skywalking/blob/v8.0.0/docs/en/setup/service-agent/java-agent/README.md](https://github.com/apache/skywalking/blob/v8.0.0/docs/en/setup/service-agent/java-agent/README.md "https://github.com/apache/skywalking/blob/v8.0.0/docs/en/setup/service-agent/java-agent/README.md") 得知

1 可以获取 sql中的参数，默认是获取不到的。当然还要设置参数最大长度。但获取参数有可能引起性能问题。

| `plugin.mysql.trace_sql_parameters`      | If set to true, the parameters of the sql (typically `java.sql.PreparedStatement`) would be collected.                                                             | `false` |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------- |
| `plugin.mysql.sql_parameters_max_length` | If set to positive number, the `db.sql.parameters` would be truncated to this length, otherwise it would be completely saved, which may cause performance problem. | `512`   |

2 收集http参数

```yaml
#收集SpringMVC plugin插件请求参，在tomcat上时这俩设置一个即可plugin.tomcat.collect_http_params or   plugin.springmvc.collect_http_params
 plugin.springmvc.collect_http_params=true
 #请求参数收集的最大字符长度, 配置过大会影响性能.
 plugin.http.http_params_length_threshold=1024
```

## skywalking-oap 的配置文件中关于数据存储时长的配置

```yaml
core:
  selector: ${SW_CORE:default}
  default:
    # Mixed: Receive agent data, Level 1 aggregate, Level 2 aggregate
    # Receiver: Receive agent data, Level 1 aggregate
    # Aggregator: Level 2 aggregate
    role: ${SW_CORE_ROLE:Mixed} # Mixed/Receiver/Aggregator
    restHost: ${SW_CORE_REST_HOST:0.0.0.0}
    restPort: ${SW_CORE_REST_PORT:12800}
    restContextPath: ${SW_CORE_REST_CONTEXT_PATH:/}
    gRPCHost: ${SW_CORE_GRPC_HOST:0.0.0.0}
    gRPCPort: ${SW_CORE_GRPC_PORT:11800}
    gRPCSslEnabled: ${SW_CORE_GRPC_SSL_ENABLED:false}
    gRPCSslKeyPath: ${SW_CORE_GRPC_SSL_KEY_PATH:""}
    gRPCSslCertChainPath: ${SW_CORE_GRPC_SSL_CERT_CHAIN_PATH:""}
    gRPCSslTrustedCAPath: ${SW_CORE_GRPC_SSL_TRUSTED_CA_PATH:""}
    downsampling:
      - Hour
      - Day
      - Month
    # Set a timeout on metrics data. After the timeout has expired, the metrics data will automatically be deleted.
    enableDataKeeperExecutor: ${SW_CORE_ENABLE_DATA_KEEPER_EXECUTOR:true} # Turn it off then automatically metrics data delete will be close.
    dataKeeperExecutePeriod: ${SW_CORE_DATA_KEEPER_EXECUTE_PERIOD:5} # How often the data keeper executor runs periodically, unit is minute
    recordDataTTL: ${SW_CORE_RECORD_DATA_TTL:3} # Unit is day
    metricsDataTTL: ${SW_CORE_RECORD_DATA_TTL:7} # Unit is day
```

主要是这四行

```yaml
enableDataKeeperExecutor: ${SW_CORE_ENABLE_DATA_KEEPER_EXECUTOR:true} # Turn it off then automatically metrics data delete will be close.
dataKeeperExecutePeriod: ${SW_CORE_DATA_KEEPER_EXECUTE_PERIOD:5} # How often the data keeper executor runs periodically, unit is minute
recordDataTTL: ${SW_CORE_RECORD_DATA_TTL:3} # Unit is day
metricsDataTTL: ${SW_CORE_RECORD_DATA_TTL:7} # Unit is day
```
