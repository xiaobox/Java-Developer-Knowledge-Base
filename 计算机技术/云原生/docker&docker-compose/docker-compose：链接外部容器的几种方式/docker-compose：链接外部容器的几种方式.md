# docker-compose：链接外部容器的几种方式

在Docker中，容器之间的链接是一种很常见的操作：它提供了访问其中的某个容器的网络服务而不需要将所需的端口暴露给Docker Host主机的功能。Docker Compose中对该特性的支持同样是很方便的。然而，如果需要链接的容器没有定义在同一个`docker-compose.yml`中的时候，这个时候就稍微麻烦复杂了点。

如果使用Docker Compose，那么这个事情就更简单了，还是以上面的`nginx`镜像为例子，编辑`docker-compose.yml`文件为：

```yaml
version: "3"
services:
  test2:
    image: nginx
    depends_on:
      - test1
    links:
      - test1
  test1:
    image: nginx
```

最终效果与使用普通的Docker命令`docker run xxxx`建立的链接并无区别。这只是一种最为理想的情况。

1.  **如果容器没有定义在同一个 docker-compose.yml 文件中，应该如何链接它们呢？**

2.  **又如果定义在 docker-compose.yml 文件中的容器需要与 docker run xxx 启动的容器链接，需要如何处理？**

针对这两种典型的情况，下面给出我个人测试可行的办法：

### 方式一：让需要链接的容器同属一个外部网络

我们还是使用nginx镜像来模拟这样的一个情景：假设我们需要将两个使用Docker Compose管理的nignx容器（`test1`和`test2`）链接起来，使得`test2`能够访问`test1`中提供的服务，这里我们以能ping通为准。
首先，我们定义容器`test1`的`docker-compose.yml`文件内容为：

```yaml
version: "3"
services:
  test2:
    image: nginx
    container_name: test1
    networks:
      - default
      - app_net
networks:
  app_net:
    external: true
```

容器`test2`内容与`test1`基本一样，只是多了一个`external_links`,需要特别说明的是：**最近发布的Docker版本已经不需要使用external\_links来链接容器，容器的DNS服务可以正确的作出判断**，因此如果你你需要兼容较老版本的Docker的话，那么容器`test2`的`docker-compose.yml`文件内容为：

```yaml
version: "3"
services:
  test2:
    image: nginx
    networks:
      - default
      - app_net
    external_links:
      - test1
    container_name: test2
networks:
  app_net:
    external: true
```

否则的话，`test2`的`docker-compose.yml`和`test1`的定义完全一致，不需要额外多指定一个`external_links`。相关的问题请参见stackoverflow上的相关问题：[docker-compose + external container](https://stackoverflow.com/questions/39067295/docker-compose-external-container "docker-compose + external container")
正如你看到的那样，这里两个容器的定义里都使用了同一个外部网络`app_net`,因此，我们需要在启动这两个容器之前通过以下命令再创建外部网络：

| docker network create app\_net |
| ------------------------------ |

之后，通过`docker-compose up -d`命令启动这两个容器，然后执行`docker exec -it test2 ping test1`,你将会看到如下的输出：

```yaml
docker exec -it test2 ping test1
PING test1 (172.18.0.2): 56 data bytes
64 bytes from 172.18.0.2: icmp_seq=0 ttl=64 time=0.091 ms
64 bytes from 172.18.0.2: icmp_seq=1 ttl=64 time=0.146 ms
64 bytes from 172.18.0.2: icmp_seq=2 ttl=64 time=0.150 ms
64 bytes from 172.18.0.2: icmp_seq=3 ttl=64 time=0.145 ms
64 bytes from 172.18.0.2: icmp_seq=4 ttl=64 time=0.126 ms
64 bytes from 172.18.0.2: icmp_seq=5 ttl=64 time=0.147 ms
```

证明这两个容器是成功链接了，反过来在`test1`中ping`test2`也是能够正常ping通的。
如果我们通过`docker run --rm --name test3 -d nginx`这种方式来先启动了一个容器(`test3`)并且没有指定它所属的外部网络，而需要将其与`test1`或者`test2`链接的话，这个时候手动链接外部网络即可：

| docker network connect app\_net test3 |
| ------------------------------------- |

这样，三个容器都可以相互访问了。

### 方式二：更改需要链接的容器的网络模式

通过更改你想要相互链接的容器的网络模式为`bridge`,并指定需要链接的外部容器（`external_links`)即可。与同属外部网络的容器可以相互访问的链接方式一不同，这种方式的访问是单向的。
还是以nginx容器镜像为例子，如果容器实例`nginx1`需要访问容器实例`nginx2`，那么`nginx2`的`doker-compose.yml`定义为：

```yaml
version: "3"
services:
  nginx2:
    image: nginx
    container_name: nginx2
    network_mode: bridge
```

与其对应的，`nginx1`的`docker-compose.yml`定义为：

```yaml
version: "3"
services:
  nginx1:
    image: nginx
    external_links:
      - nginx2
    container_name: nginx1
    network_mode: bridge
```

> 需要特别说明的是，这里的`external_links`是不能省略的，而且`nginx1`的启动必须要在`nginx2`之后，否则可能会报找不到容器`nginx2`的错误。

接着我们使用ping来测试下连通性：

```yaml
$ docker exec -it nginx1 ping nginx2  # nginx1 to nginx2
PING nginx2 (172.17.0.4): 56 data bytes
64 bytes from 172.17.0.4: icmp_seq=0 ttl=64 time=0.141 ms
64 bytes from 172.17.0.4: icmp_seq=1 ttl=64 time=0.139 ms
64 bytes from 172.17.0.4: icmp_seq=2 ttl=64 time=0.145 ms
 
$ docker exec -it nginx2 ping nginx1 #nginx2 to nginx1
ping: unknown host
```

以上也能充分证明这种方式是属于单向联通的。

在实际应用中根据自己的需要灵活的选择这两种链接方式，如果想偷懒的话，大可选择第二种。不过我更推荐第一种，不难看出无论是联通性还是灵活性，较为更改网络模式的第二种都更为友好。
