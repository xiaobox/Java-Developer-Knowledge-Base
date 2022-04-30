# skywalking的安装

我们用docker-compose进行的安装

```yaml
version: '3.3'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.5.0
    container_name: elasticsearch
    restart: always
    ports:
      - 9201:9200
    environment:
      - discovery.type=single-node
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
  oap:
    image: apache/skywalking-oap-server:8.0.0-es7
    container_name: oap
    depends_on:
      - elasticsearch
    links:
      - elasticsearch
    restart: always
    ports:
      - 11800:11800
      - 12800:12800
    environment:
      SW_STORAGE: elasticsearch7
      SW_STORAGE_ES_CLUSTER_NODES: elasticsearch:9200
  ui:
    image: apache/skywalking-ui:8.0.0
    container_name: ui
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

简单解释一下，主要有三个组成部分 skywalking使用的版本是8.0.0

*   ES 7.5.0 :Tracing 数据存储。目前支持 ES、MySQL、Sharding Sphere、TiDB、H2 多种存储器

*   skywalking-oap :负责接收 Agent 发送的 Tracing 数据信息，然后进行分析(Analysis Core) ，存储到外部存储器( Storage )，最终提供查询( Query )功能。

*   skywalking-ui :负责提供控台，查看链路等等

**docker启动起来以后，服务这边就完成了，接着就需要把探针放到各个微服务中了。**

1 根据github 官方文档的描述，[https://github.com/apache/skywalking/blob/v8.0.0/docs/en/setup/service-agent/java-agent/README.md](https://github.com/apache/skywalking/blob/v8.0.0/docs/en/setup/service-agent/java-agent/README.md "https://github.com/apache/skywalking/blob/v8.0.0/docs/en/setup/service-agent/java-agent/README.md")
2 找到探针下载位置：[https://www.apache.org/dyn/closer.cgi/skywalking/8.0.0/apache-skywalking-apm-8.0.0.tar.gz](https://www.apache.org/dyn/closer.cgi/skywalking/8.0.0/apache-skywalking-apm-8.0.0.tar.gz "https://www.apache.org/dyn/closer.cgi/skywalking/8.0.0/apache-skywalking-apm-8.0.0.tar.gz")

3 将包下载到微服务所在服务器本地并解压。

4 解压后将agent文件夹copy到指定位置待用，其余文件就没有用了。

5 修改agent文件夹中config文件夹里的config文件 路径：`agent/config/agent.config`

6 只需要修改一个地方，将前面跑起来的skywalking的地址和端口暴露给它

```yaml
collector.backend_service=${SW_AGENT_COLLECTOR_BACKEND_SERVICES:39.107.158.161:11800}
```

7 修改微服务docker-compose配置,比如：

```yaml
version: '2'
services:
  oasis-order:
    image: oasis/java:1.2 
    ports:
      - "10006:10006"
      - "8006:8006"
    container_name : oasis-order
    hostname: oasis-order
    mem_limit: 1000m
    stdin_open: true
    tty: true
    working_dir: /opt/app
    command: java -javaagent:/opt/agent/skywalking-agent.jar -Dskywalking.agent.service_name=oasis-order  -Dfile.encoding=UTF-8 -Duser.timezone=GMT+08 -Dspring.profiles.active=test -jar -Xmx256M -Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=n,address=8006 oasis-ct-order-server.jar
    volumes:
      - /data1/auto_upline/oasis-order:/opt/app
      - /opt/agent:/opt/agent
      - /etc/localtime:/etc/localtime:ro
```

只需要将探针加上去，并映射好探针路径，改动只有两点

```yaml
command: java -javaagent:/opt/agent/skywalking-agent.jar -Dskywalking.agent.service_name=oasis-order
```

```yaml
/opt/agent:/opt/agent
```

8 重新load docker容器 docker-compose -f docker-compose-oasis-order-test.yml up -d

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h1rjcjbc7gj21mw0u0wjq.jpg)

**安装到这里就完成了。**
