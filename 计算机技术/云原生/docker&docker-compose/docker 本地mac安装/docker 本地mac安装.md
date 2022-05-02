# docker 本地mac安装

### 不安装  docker-desktop&#x20;

执行 docker ps -a 报错，

```纯文本
ERROR: Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
```

参考：[https://stackoverflow.com/questions/44084846/cannot-connect-to-the-docker-daemon-on-macos](https://stackoverflow.com/questions/44084846/cannot-connect-to-the-docker-daemon-on-macos "https://stackoverflow.com/questions/44084846/cannot-connect-to-the-docker-daemon-on-macos")

在 macOS 上，Docker 二进制文件只是一个客户端，您无法使用它来运行 docker 守护程序，因为 Docker 守护程序使用特定于 Linux 的内核功能，因此您无法在 OS X 中本机运行 Docker。因此，您必须安装 Docker-machine 才能创建 VM 并附加到它。

如果没安装 docker-machine，先安装它

```纯文本
brew install docker-machine docker
```

虚拟机如果装了就不用再装了，如果没装，也安装它：

```纯文本
 brew cask install virtualbox
```

配置 docker-machine

Create a default machine (if you don't have one, see: docker-machine ls):

```纯文本
docker-machine create --driver virtualbox default 
```

Then set-up the environment for the Docker client:

```纯文本
eval "$(docker-machine env default)"

```

Then double-check by listing containers:

```纯文本
docker ps

```

### 安装 Docker的图形化管理工具Portainer

```纯文本
# 下载镜像
docker pull docker.io/portainer/portainer

```

如果仅有一个docker宿主机，则可使用单机版运行，Portainer单机版运行十分简单，只需要一条语句即可启动容器，来管理该机器上的docker镜像、容器等数据。

```纯文本
docker run -d -p 9000:9000 \
    --restart=always \
    -v /var/run/docker.sock:/var/run/docker.sock \
    --name prtainer-test \
    docker.io/portainer/portainer
```

该语句用宿主机9000端口关联容器中的9000端口，并给容器起名为portainer-test。执行完该命令之后，使用该机器IP:PORT即可访问Portainer。
访问方式：[http://IP:9000](http://IP:9000 "http://IP:9000")
首次登陆需要注册用户，给admin用户设置密码：

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h1rllr0qgmj20tx0gf0t4.jpg)

单机版这里选择local即可，选择完毕，点击Connect即可连接到本地docker：

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h1rlm2qhauj21100knwfj.jpg)

**注意：该页面上有提示需要挂载本地 /var/run/docker.socker与容器内的/var/run/docker.socker连接。因此，在启动时必须指定该挂载文件。**
