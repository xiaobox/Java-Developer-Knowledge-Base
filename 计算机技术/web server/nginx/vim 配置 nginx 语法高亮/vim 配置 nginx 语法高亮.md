# vim 配置 nginx 语法高亮

*   当你使用 vim 编辑器编辑 nginx 的配置文件时，vim 编辑器是无法自动识别出 nginx 的相关语法的。

*   所以，使用 vim 编辑器编辑 nginx 配置文件时，无法实现”语法高亮”功能，也就是说，默认情况下，使用 vim 编辑 nginx 配置文件时，没有彩色的语法着色。

*   对于使用者来说，这样体验不好，nginx 官方很贴心，在源码包中为我们提供了 vim 针对 nginx 的语法高亮配置文件，我们只要把这些文件拷贝到 vim 的对应目录中即可直接使用，方法很简单

如下：

```bash
# wget http://nginx.org/download/nginx-1.14.2.tar.gz
# tar -xf nginx-1.14.2.tar.gz

进入到源码包解压目录
# cd nginx-1.14.2/
将相应的语法文件拷贝到对应的目录中，即可完成
# cp -r contrib/vim/* /usr/share/vim/vimfiles/

```

![](https://mmbiz.qpic.cn/mmbiz_jpg/YZibCWq4rxD9tem5CqZKQPnZcdd51GkwgIm9ulnfTPp8RGdK1A7vavgiaxXIrKFgqJfyuTnCwOQiaR8rJsZ3ZpupQ/640?wx_fmt=jpeg\&wxfrom=5\&wx_lazy=1\&wx_co=1)

## 参考：

*   [https://www.zsythink.net/archives/3091](https://www.zsythink.net/archives/3091 "https://www.zsythink.net/archives/3091")
