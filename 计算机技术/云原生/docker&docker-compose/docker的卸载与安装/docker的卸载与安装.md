# docker的卸载与安装

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
