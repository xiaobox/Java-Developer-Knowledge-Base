# 利用Rancher搭建K8s

# 一 卸载旧docker，安装新docker

## 卸载

```bash
sudo yum remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-selinux docker-engine-selinux docker-engine

yum list installed | grep docker*
# 一般情况会有一个
yum remove -y docker-ce
# 18.09新增了cli工具
yum remove -y docker-ce-cli
rm -rf /var/lib/docker
rm -rf /var/lib/docker*
```

如遇到如下问题：rm: cannot remove ‘/var/lib/docker/aufs’: Device or resource busy
解决方法

*   查找挂载的目录cat /proc/mounts | grep "docker"

*   卸载umount /var/lib/docker/aufs

*   rm -rf /var/lib/docke

## 安装

注意查看centos 版本：cat /etc/redhat-release

```bash
#安装需要的工具
yum install -y yum-utils   device-mapper-persistent-data   lvm2
#设置源
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
#查看有哪些docker版本
yum list docker-ce --showduplicates | sort -r
#安装特定的版本
yum makecache fast && yum install -y docker-ce-18.09.8-3.el7 docker-ce-cli-18.09.8-3.el7 containerd.io-1.2.0-3.el7
#启动docker 
systemctl daemon-reload && systemctl restart docker
#设置为开机启动
systemctl enable docker.service

```

## 修改Docker默认存储位置

```bash
systemctl stop docker 或者 service docker stop

#然后移动整个/var/lib/docker目录到目的路径：
mv /var/lib/docker /home/data/docker
ln -s /home/data/docker /var/lib/docker
#reload配置文件
systemctl daemon-reload
#重启docker
systemctl restart docker.service
#设置docker 开机启动
systemctl enable docker

//当然你也可以通过修改配置文件的方式 
vim /etc/docker/daemon.json 

{"registry-mirrors": ["http://7e61f7f9.m.daocloud.io"],"graph": "/new-path/docker"}
```

参考：[https://blog.51cto.com/forangela/1949947](https://blog.51cto.com/forangela/1949947 "https://blog.51cto.com/forangela/1949947")

## 阿里云镜像加速

```bash
#访问：https://cr.console.aliyun.com/cn-beijing/instances/mirrors
#找到加速方法，如：

sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://se35r65b.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

# 二 安装Rancher&#x20;

在所有机器安装完Docker后，就可以安装Rancher了，需要注意Ranchert和docker的版本要一致
比如 ：

![image.png](<https://cdn.nlark.com/yuque/0/2020/png/1069608/1600003844897-267a82a9-d701-4e7e-bd21-5e7f34fc3269.png#align=left\&display=inline\&height=602\&margin=\[object Object]\&name=image.png\&originHeight=1204\&originWidth=1974\&size=214961\&status=done\&style=none\&width=987> "image.png")

根据Rancher的文档（[https://www.rancher.cn/quick-start/](https://www.rancher.cn/quick-start/ "https://www.rancher.cn/quick-start/")）

在主机上执行以下Docker命令，完成Rancher的安装与运行:

```bash
sudo docker run -d --restart=unless-stopped -p 80:80 -p 443:443 -v <主机路径>:/var/lib/rancher/ rancher/rancher:stable
```

之后，打开浏览器，输入https\://<安装容器的主机名或IP地址>，您即可以访问Rancher Server的UI了。跟随用户界面给您的引导，即可设置完成您的第一个Rancher集群。

我们使用的是Rancher的单点安装，Rancher还提供了高可用安装、离线安装等方法，可在对应版本的Rancher文档中查找具体操作。

## 登录 Rancher 界面并配置初始设置

需要先登录 Rancher，然后再开始使用 Rancher。登录以后，您需要完成一些一次性的配置。

1.  打开浏览器，输入主机的 IP 地址：`https://<SERVER_IP>`

请使用真实的主机 IP 地址替换 `<SERVER_IP>` 。

1.  首次登录时，请按照页面提示设置登录密码。

2.  设置 **Rancher Server URL**。URL 既可以是一个 IP 地址，也可以是一个主机名称。请确保您在集群内添加的每个节点都可以连接到这个 URL。如果您使用的是主机名称，请保证主机名称可以被节点的 DNS 服务器成功解析。

**结果：** 完成 Rancher 管理员用户的密码设置和访问地址设置。下次使用 Rancher 时，可以输入 IP 地址或主机地址访问 Rancher 界面，然后输入管理员用户名`admin`和您设置的密码登录 Rancher 界面。

## 创建业务集群

完成安装和登录 Rancher 的步骤之后，您现在可以参考以下步骤，在 Rancher 中创建第一个 Kubernetes 集群。
在这个任务中，您可以使用**自定义集群**选项，使用的**任意** Linux 主机（云主机、虚拟机或裸金属服务器）创建集群。

1.  访问**集群**页面，单击**添加集群**。

2.  选择**自定义**选项。

3.  输入**集群名称**。

4.  跳过**集群角色**和**集群选项**。

5.  单击**下一步**。

6.  勾选**主机选项 - 角色选择**中的所有角色： **Etcd**、 **Control** 和 **Worker**。

7.  **可选：** Rancher 会自动探查用于 Rancher 通信和集群通信的 IP 地址。您可以通过**主机选项 > 显示高级选项**中的`公网地址`和`内网地址`指定 IP 地址。

8.  跳过**主机标签**参数，因为对快速入门来说，这部分的参数不太重要。

9.  复制代码框中的命令。

10. 登录您的 Linux 主机，打开命令行工具，粘贴命令，单击回车键运命令。

11. 运行完成后，回到 Rancher 界面，单击**完成**。

**结果：** 在 Rancher 中创建了一个 Kubernetes 集群。

## 添加主机

在主界面点击 集群-->编辑，拉到页面最下方如图

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h1rlxmz40wj21wi0u0dk0.jpg)

复制最后的命令在你需要添加的主机上执行。

执行后就稍等一会儿，添加成功后在仪表盘上会有如下机器资源的概况：


![](https://tva1.sinaimg.cn/large/e6c9d24ely1h1rlyf0pjyj21ag0u077d.jpg)

如上图我添加包括安装了rancher的maser在内的三台主机。

## 添加项目和命名空间
