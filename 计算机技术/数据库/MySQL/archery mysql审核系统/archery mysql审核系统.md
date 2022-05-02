# archery mysql审核系统

## 一、 部署

### 1.1 准备运行配置

具体可参考：[https://github.com/hhyo/Archery/tree/master/src/docker-compose](https://github.com/hhyo/Archery/tree/master/src/docker-compose "https://github.com/hhyo/Archery/tree/master/src/docker-compose")

### 1.2 安装archery

下载 [Releases](https://github.com/hhyo/archery/releases/ "Releases")文件，解压后进入docker-compose文件夹 如果网络受限可访问码云地址: [gitee](https://gitee.com/rtttte/Archery/releases "gitee")

1.2.1 启动

docker-compose -f docker-compose.yml up -d

1.2.2 表结构初始化

docker exec -ti archery /bin/bashcd /opt/archerysource /opt/venv4archery/bin/activate

python3 [manage.py](http://manage.py "manage.py") makemigrations sql &#x20;

python3 [manage.py](http://manage.py "manage.py") migrate

1.2.3 数据初始化

python3 [manage.py](http://manage.py "manage.py") dbshell\<sql/fixtures/auth\_group.sql

python3 [manage.py](http://manage.py "manage.py") dbshell\<src/init\_sql/mysql\_slow\_query\_review\.sql

1.2.4 创建管理用户

python3 [manage.py](http://manage.py "manage.py") createsuperuser

1.2.5 重启服务

docker restart archery

1.2.6 日志查看和问题排查

docker logs archery -f --tail=10

logs/archery.log

## 二、 基础设置：

### 2.1 添加实例：

*   实例类型分为主库/从库，支持的数据库类型为MySQL/MsSQL/Redis/PostgreSQL/Oracle/MongoDB/Phoenix，功能支持明细可查看功能清单

*   资源组：实例都需要关联资源组，才能被关联资源组的用户访问

*   实例标签：通过支持上线、支持查询的标签来控制实例是否在SQL上线/查询中显示，要使用上线和查询的实例需要关联标签

![](https://hewppr0rxd.feishu.cn/space/api/box/stream/download/asynccode/?code=OWRjNGIwNDJjMDZlYjljMGFlMGE5NDkxNTg0MjZjMWRfc2lKeU9rZEkwZHpBaGViOWNWbUJBSmxGWUMzbjBVc3JfVG9rZW46Ym94Y25IRUZnNm1jU3ZrandScnJFQVFzZkVnXzE2NDgyMDAzMTY6MTY0ODIwMzkxNl9WNA)

![](https://hewppr0rxd.feishu.cn/space/api/box/stream/download/asynccode/?code=N2YwZDVlMTA0ZmNhMGZjYTliMjExYzE2MzA5ZDNjNDlfVjNueHMyS1lVVFFZR1J5Rmd0aGNZSEZTMTlPS0RPRUdfVG9rZW46Ym94Y25uVHlmV25MbXVRRUV1YzFYNUdYZHdkXzE2NDgyMDAzMTY6MTY0ODIwMzkxNl9WNA)

### 2.2 添加资源组：

用户必须关联资源组才能访问资源组内的实例资源 - 关联对象管理可以批量关联实例和用户 - 在添加用户和实例的时候也可以批量关联资源组。

![](https://hewppr0rxd.feishu.cn/space/api/box/stream/download/asynccode/?code=Yzc5NTNlMTgxYTQ2ZjAxYTZlNTI0OTBmYWYxZjg0MWNfcXZpM2Ztdk85Y0FFYk5uRHVxcVZMSGNpTWs0MnRDeHBfVG9rZW46Ym94Y25Bd21HRXVhdWxBMXhjbm1raUZVbGNoXzE2NDgyMDAzMTY6MTY0ODIwMzkxNl9WNA)

![](https://hewppr0rxd.feishu.cn/space/api/box/stream/download/asynccode/?code=MGQyZmUzYTBlMWIzM2IyZDRkMGUwMjBmY2E1ZDZiNDdfQjR4VDFId3lCalJHOERTSkZkeGM1RXNLYXhEbVl5c3JfVG9rZW46Ym94Y253YVYyZm9wUk1SZ3J3RWZGeHZPeEFoXzE2NDgyMDAzMTY6MTY0ODIwMzkxNl9WNA)

### 2.3 添加权限组

权限组是一堆权限的集合，类似于角色的概念，工作流的审批配置就是配置的权限组 - 权限组可以按照角色来创建，比如DBA、工程师、项目经理，目前系统初始化数据中会提供五个默认权限组，也可自由分配权限 - 仅\[sql|permission]开头的权限是控制业务操作的权限，其他都是控制Django管理后台的权限，与业务无关，可不分配。

![](https://hewppr0rxd.feishu.cn/space/api/box/stream/download/asynccode/?code=NGFhNGE3MmIxYWEwZDcyOGY2ZTI4YTUzOWNjMzNkODVfN0M4THhlUGExb3JkNnJEZjNEWUlRY1NvdDVEdHdDc01fVG9rZW46Ym94Y25XOXJhUG13Y1ZEeEtXSmw4eG9Ud05lXzE2NDgyMDAzMTY6MTY0ODIwMzkxNl9WNA)

![](https://hewppr0rxd.feishu.cn/space/api/box/stream/download/asynccode/?code=MmE5OGI4NTBlMzJjY2IzOGQ2OTJhNjI1ZGMzMWMyNDNfRzVHMmk3Mml5eU5wUjVYTzM2QkhaaDdmOVpxZVJSVUFfVG9rZW46Ym94Y25XdU1Sbk82ak1kRm5oNFlwV0dOSWhoXzE2NDgyMDAzMTY6MTY0ODIwMzkxNl9WNA)

### 2.4 添加用户：

用户所拥有的权限=用户所在权限组的权限+给用户单独分配的权限

![](https://hewppr0rxd.feishu.cn/space/api/box/stream/download/asynccode/?code=MjEzZjlmNWRlOTY0NDMzNGI5NDEzMzNiMThkODdlMTNfeUxTMTNia0tnZUI5aXhMTk9TSzVKVXZkNDZzV252SDVfVG9rZW46Ym94Y243MTR5VWtpa3ZFSkJDRkpaTVVjWWxkXzE2NDgyMDAzMTY6MTY0ODIwMzkxNl9WNA)

### 2.5 设置工单上线和查询的审批流程

![](https://hewppr0rxd.feishu.cn/space/api/box/stream/download/asynccode/?code=ZWQ2Mzc4NzA4MDk0NDQ0ZWY3OTYwMzY0YzQ5MDY2YzhfYUN0aVZHZlVwRlM4TXNzM096VXpaYkhaS3lFTDNzNWxfVG9rZW46Ym94Y25FQWlUWWpuT2FVQ0poTDNvb0FVTHRoXzE2NDgyMDAzMTY6MTY0ODIwMzkxNl9WNA)
