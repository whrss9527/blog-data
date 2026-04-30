---
id: 04a832ec2abd4b90acb868f54e324649
title: "Nginx 代理静态网站 CSS 解析异常"
status: 1
created_at: 2023-01-31T21:55:01+08:00
updated_at: 2026-04-30T14:41:37+08:00
category_id: 1
is_top: 0
tag_ids: [32]
description: "今天在使用ecs进行部署网页时，出现了一个问题。使用nginx代理到页面index.html路径下，同路径的资源都可以加载到，但是却无法正确加载到页面样式。打开f12，网络和控制台都没有资源异常，但页面乱成了一锅粥。
本地打开是正常的，上到服务器却不行？"
word_count: 492
identity: nginx-website
---

今天在使用ecs进行部署网页时，出现了一个问题。使用nginx代理到页面index.html路径下，同路径的资源都可以加载到，但是却无法正确加载到页面样式。打开f12，网络和控制台都没有资源异常，但页面乱成了一锅粥。
本地打开是正常的，上到服务器却不行？
之前使用nginx时，并没有这个问题，于是我猜测是不是nginx新的版本对配置参数进行了修改？
但我翻看了nginx的文档，却没有找到。于是我跟着症状开始在网上翻文，终于：

## 解决方法

若不对于css文件解析进行配置，nginx默认文件都是[`text/plain`](https://developer.mozilla.org/en-US/docs/Learn/Server-side/Configuring_server_MIME_types)类型进行解析，为此我们需要对此进行简单配置。将下面这段代码放入到`location`模块下面，然后重启nginx。**记住一定要清理浏览器的缓存**。

```sh
include mime.types;  
default_type application/octet-stream;
```

完整如下：
```sh
location / {
              include mime.types;
              default_type application/octet-stream;
              alias /opt/website/;
              autoindex on;
         }

```

