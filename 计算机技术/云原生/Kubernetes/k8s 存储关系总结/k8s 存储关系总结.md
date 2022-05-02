# k8s 存储关系总结

# Docker

当我们使用 Docker 时，设置数据卷（Volume）还是比较简单的，只需要在容器映射指定卷的路径，然后在容器中使用该路径即可。

比如这种：

```yaml
# tomcat
  tomcat01:
    hostname: tomcat01
    restart: always
    image: jdk-tomcat:v8
    container_name: tomcat8-1
    links:
      -  mysql:mysql
    volumes:
      - /home/soft/docker/tomcat/webapps:/usr/local/apache-tomcat-8.5.39/webapps
      - /home/soft/docker/tomcat/logs:/usr/local/apache-tomcat-8.5.39/logs
      - /etc/localtime:/etc/localtime
    environment:
      JAVA_OPTS: -Dspring.profiles.active=prod
      TZ: Asia/Shanghai
      LANG: C.UTF-8
      LC_ALL: zh_CN.UTF-8
    env_file:
      - /home/soft/docker/env/tomcat.env
```

为什么要设置 Volume？ 当然是因为我们要持久化数据，要把数据存储到硬盘上。

## k8s

到了 k8s 这儿，你会发现事情没那么简单了，涌现出了一堆概念：

*   Pv

*   Pvc

*   StorageClass

*   Provisioner

*   ...

先不管这些复杂的概念，我只想存个文件，有没有简单的方式？

有，我们先回顾下基本概念。

我们知道，Container 中的文件在磁盘上是临时存放的，当容器崩溃时文件丢失。kubelet 会重新启动容器， 但容器会以干净的状态重启。所以我们要使用 Volume 来持久化数据。

> Docker 也有 卷（Volume） 的概念，但对它只有少量且松散的管理。 Docker 卷是磁盘上或者另外一个容器内的一个目录  Docker 提供卷驱动程序，但是其功能非常有限。 &#x20;

> Kubernetes 支持很多类型的卷。 Pod 可以同时使用任意数目的卷类型。

> 临时卷类型的生命周期与 Pod 相同，但持久卷可以比 Pod 的存活期长。 当 Pod 不再存在时，Kubernetes 也会销毁临时卷；不过 Kubernetes 不会销毁 持久卷。对于给定 Pod 中任何类型的卷，在容器重启期间数据都不会丢失。

> 卷的核心是一个目录，其中可能存有数据，Pod 中的容器可以访问该目录中的数据。 所采用的特定的卷类型将决定该目录如何形成的、使用何种介质保存数据以及目录中存放 的内容。

> 使用卷时，在 .spec.volumes 字段中设置为 Pod 提供的卷，并在 .spec.containers\[\*].volumeMounts 字段中声明卷在容器中的挂载位置。 各个卷则挂载在镜像内的指定路径上。 卷不能挂载到其他卷之上，也不能与其他卷有硬链接。Pod 配置中的每个容器必须独立指定各个卷的挂载位置。

通过上面的概念我们知道 Volume 有不同的类型，有临时的，也有持久的，那么我们先说说简单的，即解决“我只想存个文件，有没有简单的方式”的需求。

### hostPath

hostPath 卷能将主机节点文件系统上的文件或目录挂载到你的 Pod 中。看个示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-webserver
spec:
  containers:
  - name: test-webserver
    image: k8s.gcr.io/test-webserver:latest
    volumeMounts:
    - mountPath: /var/local/aaa
      name: mydir
    - mountPath: /var/local/aaa/1.txt
      name: myfile
  volumes:
  - name: mydir
    hostPath:
      # 确保文件所在目录成功创建。
      path: /var/local/aaa
      type: DirectoryOrCreate
  - name: myfile
    hostPath:
      path: /var/local/aaa/1.txt
      type: FileOrCreate
```

通过 hostPath 能够简单解决文件在宿主机上存储的问题。

不过需要注意的是：

**HostPath 卷存在许多安全风险，最佳做法是尽可能避免使用 HostPath。 当必须使用 HostPath 卷时，它的范围应仅限于所需的文件或目录，并以只读方式挂载。**

使用 hostPath 还有一个局限性就是，我们的 Pod 不能随便漂移，需要固定到一个节点上，因为一旦漂移到其他节点上去了宿主机上面就没有对应的数据了，所以我们在使用 hostPath 的时候都会搭配 nodeSelector 来进行使用。

### emptyDir

emptyDir 也是比较常见的一种存储类型。

上面的 hostPath 显示的定义了宿主机的目录。emptyDir 类似隐式的指定。

Kubernetes 会在宿主机上创建一个临时目录，这个目录将来就会被绑定挂载到容器所声明的 Volume 目录上。而 Pod 中的容器，使用的是 volumeMounts 字段来声明自己要挂载哪个 Volume，并通过 mountPath 字段来定义容器内的 Volume 目录

> 当 Pod 分派到某个 Node 上时，emptyDir 卷会被创建，并且在 Pod 在该节点上运行期间，卷一直存在。 就像其名称表示的那样，卷最初是空的。 尽管 Pod 中的容器挂载 emptyDir 卷的路径可能相同也可能不同，这些容器都可以读写 emptyDir 卷中相同的文件。 当 Pod 因为某些原因被从节点上删除时，emptyDir 卷中的数据也会被永久删除。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir: {}
```

如果执行 `kubectl describe` 命令查看 pod 信息的话，可以验证前面我们说的内容：
"EmptyDir (a temporary directory that shares a pod's lifetime)"

```text
...
Containers:
  nginx:
    Container ID:   docker://07b4f89248791c2aa47787e3da3cc94b48576cd173018356a6ec8db2b6041343
    Image:          nginx:1.8
    ...
    Environment:    <none>
    Mounts:
      /usr/share/nginx/html from nginx-vol (rw)
...
Volumes:
  nginx-vol:
    Type:    EmptyDir (a temporary directory that shares a pod's lifetime)
```

### PV 和 PVC

*   PV(PersistentVolume): 持久化卷

*   PVC(PersistentVolumeClaim): 持久化卷声明

PV 和 PVC 的关系就像 java 中接口和实现的关系类似。

PVC 是用户存储的一种声明，PVC 和 Pod 比较类似，Pod 消耗的是节点，PVC 消耗的是 PV 资源，Pod 可以请求 CPU 和内存，而 PVC 可以请求特定的存储空间和访问模式。对于真正使用存储的用户不需要关心底层的存储实现细节，只需要直接使用 PVC 即可。

PV 是对底层共享存储的一种抽象，由管理员进行创建和配置，它和具体的底层的共享存储技术的实现方式有关，比如 Ceph、GlusterFS、NFS、hostPath 等，都是通过插件机制完成与共享存储的对接。

我们来看一个例子：

比如，运维人员可以定义这样一个 NFS 类型的 PV

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: 10.244.1.4
    path: "/"
```

PVC 描述的，则是 Pod 所希望使用的持久化存储的属性。比如，Volume 存储的大小、可读写权限等等。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: manual
  resources:
    requests:
      storage: 1Gi

```

用户创建的 PVC 要真正被容器使用起来，就必须先和某个符合条件的 PV 进行绑定。

*   第一个条件是 PV 和 PVC 的 spec 字段。比如，PV 的存储（storage）大小，就必须满足 PVC 的要求。

*   第二个条件，则是 PV 和 PVC 的 storageClassName 字段必须一样

在成功地将 PVC 和 PV 进行绑定之后，Pod 就能够像使用 hostPath 等常规类型的 Volume 一样，在自己的 YAML 文件里声明使用这个 PVC 了

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    role: web-frontend
spec:
  containers:
  - name: web
    image: nginx
    ports:
      - name: web
        containerPort: 80
    volumeMounts:
        - name: nfs
          mountPath: "/usr/share/nginx/html"
  volumes:
  - name: nfs
    persistentVolumeClaim:
      claimName: nfs
```

我们前面使用的 hostPath 和 emptyDir 类型的 Volume 并不具备“持久化”特征，既有可能被 kubelet 清理掉，也不能被“迁移”到其他节点上。所以，大多数情况下，持久化 Volume 的实现，往往依赖于一个远程存储服务，比如：远程文件存储（比如，NFS、GlusterFS）、远程块存储（比如，公有云提供的远程磁盘）等等。

### StorageClass

前面我们人工管理 PV 的方式就叫作 Static Provisioning。

一个大规模的 Kubernetes 集群里很可能有成千上万个 PVC，这就意味着运维人员必须得事先创建出成千上万个 PV。更麻烦的是，随着新的 PVC 不断被提交，运维人员就不得不继续添加新的、能满足条件的 PV，否则新的 Pod 就会因为 PVC 绑定不到 PV 而失败。在实际操作中，这几乎没办法靠人工做到。所以，Kubernetes 为我们提供了一套可以自动创建 PV 的机制，即：Dynamic Provisioning。

Dynamic Provisioning 机制工作的核心，在于一个名叫 StorageClass 的 API 对象。而 StorageClass 对象的作用，其实就是创建 PV 的模板。

具体地说，StorageClass 对象会定义如下两个部分内容：

*   第一，PV 的属性。比如，存储类型、Volume 的大小等等。

*   第二，创建这种 PV 需要用到的存储插件。比如，Ceph 等等。

有了这样两个信息之后，Kubernetes 就能够根据用户提交的 PVC，找到一个对应的 StorageClass 了。然后，Kubernetes 就会调用该 StorageClass 声明的存储插件，创建出需要的 PV。

在下面的例子中，PV 是被自动创建出来的。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: claim1
spec:
  accessModes:
    - ReadWriteOnce
# 指定所使用的存储类，此存储类将会自动创建符合要求的 PV
 storageClassName: fast
 resources:
    requests:
      storage: 30Gi
```

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
```

**StorageClass 的作用，则是充当 PV 的模板。并且，只有同属于一个 StorageClass 的 PV 和 PVC，才可以绑定在一起。StorageClass 的另一个重要作用，是指定 PV 的 Provisioner（存储插件）。这时候，如果你的存储插件支持 Dynamic Provisioning 的话，Kubernetes 就可以自动为你创建 PV 了。**

### Local PV

Kubernetes 依靠 PV、PVC 实现了一个新的特性，这个特性的名字叫作：Local Persistent Volume，也就是 Local PV。

Local PV 实现的功能就非常类似于 hostPath 加上 nodeAffinity，比如，一个 Pod 可以声明使用类型为 Local 的 PV，而这个 PV 其实就是一个 hostPath 类型的 Volume。如果这个 hostPath 对应的目录，已经在节点 A 上被事先创建好了，那么，我只需要再给这个 Pod 加上一个 nodeAffinity=nodeA，不就可以使用这个 Volume 了吗？理论上确实是可行的，但是事实上，我们绝不应该把一个宿主机上的目录当作 PV 来使用，因为本地目录的存储行为是完全不可控，它所在的磁盘随时都可能被应用写满，甚至造成整个宿主机宕机。所以，**一般来说 Local PV 对应的存储介质是一块额外挂载在宿主机的磁盘或者块设备，我们可以认为就是“一个 PV 一块盘”**。

Local PV 和普通的 PV 有一个很大的不同在于 Local PV 可以保证 Pod 始终能够被正确地调度到它所请求的 Local PV 所在的节点上面，对于普通的 PV 来说，Kubernetes 都是先调度 Pod 到某个节点上，然后再持久化节点上的 Volume 目录，进而完成 Volume 目录与容器的绑定挂载，但是对于 Local PV 来说，节点上可供使用的磁盘必须是提前准备好的，因为它们在不同节点上的挂载情况可能完全不同，甚至有的节点可以没这种磁盘，所以，这时候，调度器就必须能够知道所有节点与 Local PV 对应的磁盘的关联关系，然后根据这个信息来调度 Pod，实际上就是在调度的时候考虑 Volume 的分布。

例子：

先创建本地磁盘对应的 pv

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/disks/vol1 
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node-1
```

其中：

*   lcal.path 写对应的磁盘路径

*   必须指定对应的 node , 用 .spec.nodeAffinity 来对应的 node

*   .spec.volumeMode 可以是 FileSystem（Default）和 Block

*   确保先运行了 StorageClass （即下面写的文件）

再写对于的 StorageClass 文件

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer

```

其中：

*   provisioner 是 [kubernetes.io/no-provisioner](http://kubernetes.io/no-provisioner "kubernetes.io/no-provisioner") , 这是因为 local pv 不支持 Dynamic Provisioning, 所以它没有办法在创建出 pvc 的时候，自动创建对应 pv

*   volumeBindingMode 是 WaitForFirstConsumer , WaitForFirstConsumer 即延迟绑定 , 这样可以既保证推迟到调度的时候再进行绑定 , 又可以保证调度到指定的 pod 上 , 其实 WaitForFirstConsumer 又 2 种：一种是 WaitForFirstConsumer , 一种是 Immediate , 这里必须用延迟绑定模式。

再创建一个 pvc

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: example-local-claim
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: local-storage

```

这里需要注意的地方就是 storageClassName 要写出我们之前自己创建的 storageClassName 的名字：local-storage

之后应用这个文件 , 使用命令 kubectl get pvc 可以看到他的状态是 Pending , 这个时候虽然有了匹配的 pv , 但是也不会进行绑定 , 依然在等待。

之后我们写个 pod 应用这个 pvc

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: example-pv-pod
spec:
  volumes:
    - name: example-pv-storage
      persistentVolumeClaim:
       claimName: example-local-claim
  containers:
    - name: example-pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: example-pv-storage
```

这样就部署好了一个 local pv 在 pod 上 , 这样即使 pod 没有了 , 再次重新在这个 node 上创建，写入的文件也能持久化的存储在特定位置。

**如何删除这个 pv 一定要按照流程来 , 要不然会删除失败**

*   删除使用这个 pv 的 pod

*   从 node 上移除这个磁盘（按照一个 pv 一块盘）

*   删除 pvc

*   删除 pv

## 总结&#x20;

本文我们讨论了 kubernetes 存储的几种类型，有临时存储如：hostPath、emptyDir,也有真正的持久化存储，还讨论了相关的概念，如：PVC、PV、StorageClass等,下图是对这些概念的一个概括：

![](https://tva1.sinaimg.cn/large/008i3skNly1gwsowy98dhj30oz0jrgnn.jpg)

、

![](https://tva1.sinaimg.cn/large/008i3skNly1gwsp3h9q3rj30lt0gzwg6.jpg)

## 参考

*   极客时间：深入剖析 Kubernetes 课程

*   [https://kubernetes.io/zh/docs/concepts/storage/volumes/#emptydir](https://kubernetes.io/zh/docs/concepts/storage/volumes/#emptydir "https://kubernetes.io/zh/docs/concepts/storage/volumes/#emptydir")

*   [https://www.qikqiak.com/k8strain/storage/local/](https://www.qikqiak.com/k8strain/storage/local/ "https://www.qikqiak.com/k8strain/storage/local/")

*   [https://www.kubernetes.org.cn/4078.html](https://www.kubernetes.org.cn/4078.html "https://www.kubernetes.org.cn/4078.html")

*   [https://haojianxun.github.io/2019/01/10/kubernetes的本地持久化存储--Local Persistent Volume解析/](<https://haojianxun.github.io/2019/01/10/kubernetes的本地持久化存储--Local Persistent Volume解析/> "https://haojianxun.github.io/2019/01/10/kubernetes的本地持久化存储--Local Persistent Volume解析/")
