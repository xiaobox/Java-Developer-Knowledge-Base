# docker 容器安装字体

如果容器是alpine系统

需要用alpine的包管理工具 apk 安装软件

1 修改apk的数据源，不然下载缓慢

```bash
vi /etc/apk/repositories
aliyun数据源（建议选这个）
https://mirrors.aliyun.com/alpine/v3.6/main/
https://mirrors.aliyun.com/alpine/v3.6/community/
#更新源
apk update
#安装字体软件
apk add --update font-adobe-100dpi ttf-dejavu fontconfig
```

2 查看已经安装的中文字体

```纯文本
fc-list :lang=zh
```

3 下载我们需要的字体文件放入到字体目录

```纯文本
docker cp /home/long.he/simsun.ttc 5554ed7bb342:/usr/share/fonts
```

4  启用新字体

```纯文本
 fc-cache
```

5 查看新字体

fc-list :lang=zh
