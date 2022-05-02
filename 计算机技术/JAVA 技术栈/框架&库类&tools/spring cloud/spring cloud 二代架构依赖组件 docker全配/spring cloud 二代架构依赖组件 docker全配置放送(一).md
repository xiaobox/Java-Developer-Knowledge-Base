# spring cloud 二代架构依赖组件 docker全配置放送(一)

# 一 背景介绍

### 先来看一下我们熟悉的第一代 spring cloud 的组件

| 组件名称    | 功能                     |
| ------- | ---------------------- |
| Ribbon  | 客户端负载均衡器               |
| Eureka  | 服务治理（注册、发现......）      |
| Hystrix | 服务之间远程调用时的熔断保护         |
| Feign   | 通过定义接口的方式直接调用其他服务的 API |
| Zuul    | 服务网关                   |
| Config  | 分布式配置组件                |
| Sleuth  | 用于请求链路跟踪               |

spring cloud 现在已经是一种标准了，各公司可以基于它的编程模型编写自己的组件 ，比如Netflix、阿里巴巴都有自己的一套通过spring cloud 编程模型开发的分布式服务组件 。

### Spring Cloud 二代组件

Spring Cloud Alibaba 主要包含 Sentinel、Nacos、RocketMQ、Dubbo、Seata 等组件。

二代引入了 **Spring Cloud Alibaba**

| 第一代组件   | 第二代组件                |
| ------- | -------------------- |
| Eureka  | Nacos                |
| Config  | Apollo               |
| Zuul    | spring cloud gateway |
| Hystrix | Sentinel             |

### 再加上我们常用的组件

| 组件         | 功能        |
| ---------- | --------- |
| XXL-Job    | 分布式定时任务中心 |
| Redis      | 分布式缓存     |
| Rocket-MQ  | 消息队列      |
| Seata      | 分布式事务     |
| ELK        | 日志处理      |
| Skywalking | 调用链监控     |
| Prometheus | metrics监控 |

**这其有中除 spring cloud gateway都需要外部单独部署服务来支持**

## 二 利用docker-compose 进行本地简化部署

### apollo

```bash
version: '2'

services:
  apollo-quick-start:
    image: nobodyiam/apollo-quick-start
    container_name: apollo-quick-start
    depends_on:
      - apollo-db
    ports:
      - "8080:8080"
      - "8070:8070"
    links:
      - apollo-db

  apollo-db:
    image: mysql:5.7
    container_name: apollo-db
    environment:
      TZ: Asia/Shanghai
      MYSQL_ALLOW_EMPTY_PASSWORD: 'yes'
    depends_on:
      - apollo-dbdata
    ports:
      - "13306:3306"
    volumes:
      - ./sql:/docker-entrypoint-initdb.d
    volumes_from:
      - apollo-dbdata

  apollo-dbdata:
    image: alpine:latest
    container_name: apollo-dbdata
    volumes:
      - /var/lib/mysql

```

**注意：** ./sql下面的文件在[这里](https://github.com/ctripcorp/apollo/tree/master/scripts/sql "这里")，是两个初始化的sql文件

### nacos

```bash
version: "2"
services:
  nacos:
    image: nacos/nacos-server:latest
    container_name: nacos-standalone-mysql
    env_file:
      - ./env/nacos-standlone-mysql.env
    volumes:
      - ./standalone-logs/:/home/nacos/logs
      - ./init.d/custom.properties:/home/nacos/init.d/custom.properties
    ports:
      - "8848:8848"
      - "9555:9555"
    depends_on:
      - mysql
    restart: on-failure
  mysql:
    container_name: mysql
    image: nacos/nacos-mysql:5.7
    env_file:
      - ./env/mysql.env
    volumes:
      - ./mysql:/var/lib/mysql
    ports:
      - "3308:3306"
```

### redis

```bash
version: '2'
services:
  #redis容器
  redis:
    #定义主机名
    container_name: redis
    #使用的镜像
    image: redis:6.0.8
    #容器的映射端口
    ports:
      - 6379:6379
    command: redis-server /etc/conf/redis.conf
    #定义挂载点
    volumes:
      - ./data:/data
      - ./conf:/etc/conf
    #环境变量
    privileged: true
    environment:
      - TZ=Asia/Shanghai
      - LANG=en_US.UTF-8
```

**注意：** conf下的redis.conf配置文件可以找个默认的模版文件，然后进行相应修改

### rocket-mq

```bash
version: '2'

services:
  #Service for nameserver
  namesrv:
    image: apacherocketmq/rocketmq-nameserver:4.5.0-alpine-operator-0.3.0
    container_name: rmqnamesrv
    ports:      
      - 9876:9876
    volumes:     
      - ./data/namesrv/logs:/home/rocketmq/logs
    command: sh mqnamesrv
    environment:
      TZ: Asia/Shanghai
      JAVA_OPT_EXT: "-server -Xms512m -Xmx512m -Xmn256m"

  #Service for broker
  broker:
    image: apacherocketmq/rocketmq-broker:4.5.0-alpine-operator-0.3.0
    container_name: rmqbroker-a
    depends_on:     
      - namesrv
    ports:      
      - 10909:10909
      - 10911:10911
      - 10912:10912
    environment:      
      NAMESRV_ADDR: namesrv:9876
      JAVA_OPT_EXT: "-server -Xms512m -Xmx512m -Xmn256m"
    volumes:      
      - ./data/broker/logs:/home/rocketmq/logs     
      - ./data/broker/store:/home/rocketmq/store      
      - ./data/broker/conf/broker.conf:/opt/rocketmq-4.7.1/conf/broker.conf

    command: sh mqbroker -c /opt/rocketmq-4.7.1/conf/broker.conf

 #Service for another broker -- broker1
  broker1:
    image: apacherocketmq/rocketmq-broker:4.5.0-alpine-operator-0.3.0
    container_name: rmqbroker-b
    depends_on:     
      - namesrv
    ports:      
      - 10929:10909
      - 10931:10911
      - 10932:10912
    environment:      
      NAMESRV_ADDR: namesrv:9876
      JAVA_OPT_EXT: "-server -Xms512m -Xmx512m -Xmn256m"
    volumes:      
      - ./data1/broker/logs:/home/rocketmq/logs      
      - ./data1/broker/store:/home/rocketmq/store      
      - ./data1/broker/conf/broker.conf:/opt/rocketmq-4.7.1/conf/broker.conf
    command: sh mqbroker -c /opt/rocketmq-4.7.1/conf/broker.conf

  rmqconsole:
    image: styletang/rocketmq-console-ng
    container_name: rmqconsole
    ports:
      - 8180:8080
    environment:
        TZ: Asia/Shanghai
        JAVA_OPTS: "-Drocketmq.namesrv.addr=namesrv:9876 -Dcom.rocketmq.sendMessageWithVIPChannel=false"
    depends_on:
      - namesrv
```

此外还有两个配置文件

*   ./data/broker/conf/broker.conf

*   ./data1/broker/conf/broker.conf

```bash
## ./data/broker/conf/broker.conf
brokerClusterName = DefaultCluster
brokerName = broker-abroker
Id = 0
deleteWhen = 04
fileReservedTime = 48
brokerRole = ASYNC_MASTER
flushDiskType = ASYNC_FLUSH

### ./data1/broker/conf/broker.conf
brokerClusterName = Default
ClusterbrokerName = broker-bbroker
Id = 0
deleteWhen = 04
fileReservedTime = 48
brokerRole = ASYNC_MASTER
flushDiskType = ASYNC_FLUSH
```

### seata-server

```bash
version: "3.1"
services:
  seata-server:
    image: seataio/seata-server:latest
    hostname: seata-server
    ports:
      - 8091:8091
    environment:
      - SEATA_PORT=8091
    expose:
      - 8091

```

### sentinel&#x20;

*   没有现成的docker镜像，需要自己编写一个

```bash
FROM openjdk:8

#复制上下文目录下的jar包到容器里  使用COPY命令亦可
ADD sentinel-dashboard-1.8.0.jar sentinel-dashboard-1.8.0.jar

EXPOSE 8080

#指定容器启动程序及参数   <ENTRYPOINT> "<CMD>"
ENTRYPOINT ["java","-jar","sentinel-dashboard-1.8.0.jar"]
```

*   利用自己编译的镜像再编写docker-compose配置文件

```bash
version: '3'
services:
  sentinel-dashboard:
    image: sentinel-dashboard:1.8.0
    container_name: sentinel-dashboard
    restart: always
    environment:
      JAVA_OPTS: "-Dserver.port=8080 -Dcsp.sentinel.dashboard.server=localhost:8080 -Dproject.name=sentinel-dashboard -Djava.security.egd=file:/dev/./urandom -Dcsp.sentinel.api.port=8719"
    ports: #避免出现端口映射错误，建议采用字符串格式 8080端口为Dockerfile中EXPOSE端口
      - "58080:8080"
      - "8719:8719"
    volumes:
      - ./root/logs:/root/logs

```

### xxl-job

```bash
version: '3'
services:
  xxl-job-admin:
    image: xuxueli/xxl-job-admin:2.2.0
    restart: always
    container_name: xxl-job-admin
    depends_on:
      - mysql
    environment:
      PARAMS: '--spring.datasource.url=jdbc:mysql://mysql:3306/xxl_job?Unicode=true&characterEncoding=UTF-8 --spring.datasource.username=root --spring.datasource.password=root'
    ports:
      - 8067:8080
    volumes:
      - ./data/applogs:/data/applogs
```

**注意：** 这时引用的数据库是你现有的mysql，找一个现有的，因为为了它再新建一个容器有点儿浪费

### prometheus(altermanager+prometheus+grafana)

```bash
version: '3'
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - /opt/docker_compose/monitor/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - /opt/docker_compose/monitor/prometheus/alertmanager_rules.yml:/etc/prometheus/alertmanager_rules.yml
    ports:
      - 9090:9090
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'

  grafana:
    image: grafana/grafana
    container_name: grafana
    restart: always
    hostname: grafana
    volumes:
      - /opt/docker_compose/monitor/grafana/grafana.ini:/etc/grafana/grafana.ini
    ports:
      - "3000:3000"
    
  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    hostname: alertmanager
    restart: always
    volumes:
      - /opt/docker_compose/monitor/altermanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml
    ports:
      - "9093:9093"  

  prometheus-webhook-alert:
    image: timonwong/prometheus-webhook-dingtalk:v1.3.0
    container_name: prometheus-webhook-alertmanagers
    hostname: webhook-alertmanagers
    restart: always
    volumes:
      - /opt/docker_compose/monitor/prometheus-webhook-dingtalk/config.yml:/etc/prometheus-webhook-dingtalk/config.yml
      - /etc/localtime:/etc/localtime
    ports:
      - "8060:8060"
    entrypoint: /bin/prometheus-webhook-dingtalk   --config.file=/etc/prometheus-webhook-dingtalk/config.yml  --web.enable-ui

```

这里我的alter没有用grafana的，而是结合altermanager和 prometheus-webhook-dingtalk实现的钉钉告警。关于prometheus、altermanager、grafana都是常规配置大家可以找模板然后根据自己的需求修改，**唯一需要说明的就是prometheus-webhook-dingtalk，虽然github上说明可以配置通知模版，但最新版本的，我怎么修改也不成，是个问题。** 需要观察以后版本会不会好，或者直接上手改它的go代码。

### skywalking&#x20;

```bash
version: '3.3'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.5.0
    container_name: elasticsearch
    restart: always
    ports:
      - 9200:9200
      - 9300:9300
    environment:
      - discovery.type=single-node
      - bootstrap.memory_lock=true
    network_mode: bridge
    volumes:
      - /data/docker_compose/skywalking/es/config/jvm.options:/usr/share/elasticsearch/config/jvm.options:rw
      - /data/docker_compose/skywalking/es/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
      - /data/docker/elk/elk_elastic/data:/usr/share/elasticsearch/data:rw
    ulimits:
      memlock:
        soft: -1
        hard: -1
  oap:
    image: apache/skywalking-oap-server:8.1.0-es7
    container_name: oap
    depends_on:
      - elasticsearch
    links:
      - elasticsearch
    network_mode: bridge
    restart: always
    ports:
      - 11800:11800
      - 12800:12800
    environment:
      SW_ES_USER: elastic
      SW_ES_PASSWORD: oasises
      SW_STORAGE: elasticsearch7
      SW_STORAGE_ES_CLUSTER_NODES: elasticsearch:9200
      SW_TRACE_SAMPLE_RATE: 8000
  ui:
    image: apache/skywalking-ui:8.1.0
    container_name: ui
    network_mode: bridge
    depends_on:
      - oap
    links:
      - oap
    restart: always
    ports:
      - 8083:8080
    environment:
      SW_OAP_ADDRESS: oap:12800

```

**注意：** es的详细配置文件需要你自己写哈。

### kibana(ELK)

```bash
version: '2'
services:
  elk-logstash:
    image: docker.elastic.co/logstash/logstash:7.5.0
    container_name : elk_logstash
    hostname: elk_logstash
    stdin_open: true
    tty: true
    ports:
      - "5000:5000/udp"
      -  5001:5001
    command: logstash --path.settings /etc/logstash -f /etc/logstash/conf.d/logstash.conf
    external_links:
      - elasticsearch
    network_mode: bridge
    volumes:
      - /data1/docker/elk/elk_logstash/conf.d:/etc/logstash/conf.d
      - /data1/docker/elk/elk_logstash/heapdump.hprof:/usr/share/logstash/heapdump.hprof -rw
      - /data1/docker/elk/elk_logstash/gc.log:/usr/share/logstash/gc.log -rw

  elk-kibana:
    image: docker.elastic.co/kibana/kibana:7.5.0
    container_name : elk_kibana
    hostname: elk_kibana
    stdin_open: true
    tty: true
    ports:
      - 5601:5601
    external_links:
      - elasticsearch
    network_mode: bridge
    volumes:  
      - /data1/docker/elk/elk_kibana/config/kibana.yml:/usr/share/kibana/config/kibana.yml 
    environment:
      - ELASTICSEARCH_URL=http://elasticsearch:9200

```

*   **由于ES一般我们会建集群，这里忽略ES容器**

*   **logstash和kibana的相关配置也可从官网找到模版进行修改**
