# nginx 性能优化

## gzip&#x20;

```nginx
#开启 gzip 压缩功能
gzip on;
# 设置系统获取几个单位的缓存用于存储 gzip 的压缩结果数据流。16 8k 代表以 8k 为单位，安装原始数据大小以 8k 为单位的 16 倍申请内存
gzip_buffers 16 8k; 
# 设置压缩比率，最小为 1，处理速度快，传输速度慢；9 为最大压缩比，处理速度慢，传输速度快；这里表示压缩级别，可以是 0 到 9 中的任一个，级别越高，压缩就越小，节省了带宽资源，但同时也消耗 CPU 资源，所以一般折中为 6
gzip_comp_level 6;
# 设置 gzip 压缩针对的 HTTP 协议版本
gzip_http_version 1.1;
# 设置允许压缩的页面最小字节数；这里表示如果文件小于 10 个字节，就不用压缩，因为没有意义，本来就很小。
gzip_min_length 10;
# 匹配 mime 类型进行压缩，无论是否指定，”text/html”类型总是会被压缩的。
gzip_types text/xml application/xml application/atom+xml application/rss+xml application/xhtml+xml image/svg+xml text/javascript application/javascript application/x-javascript text/x-json application/json application/x-web-app-manifest+json text/css text/plain text/x-component font/opentype application/x-font-ttf application/vnd.ms-fontobject image/x-icon; 
```

```nginx
gzip on;
   gzip_min_length   1k;
   gzip_buffers      4 16k;
   gzip_http_version 1.0;
   gzip_comp_level 2;
   gzip_types    text/plain text/xml text/css application/x-javascript application/xml application/json text/javascript application/javascript;
   gzip_vary     on;
```

重新加载配置

`openresty -s reload `
