---
title: "将本地服务通过 SSH 代理给外部访问"
status: 1
created_at: 2023-03-09T16:08:49+08:00
updated_at: 2026-04-30T14:45:27+08:00
category_id: 1
is_top: 0
tag_ids: [43, 44, 45]
description: "如何使用 ssh 将本地服务代理给外部访问并保持 SSH 会话的连接性"
word_count: 660
---

如何使用 ssh 将本地服务代理给外部访问并保持 SSH 会话的连接性

### 1. 外部服务器 nginx 配置

```shell
server {
    listen     localhost:80;
    server_name  _;
    root         /usr/share/nginx/html;

    # 重要：将请求转发到本地服务
    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
        proxy_pass http://127.0.0.1:10412;
        proxy_set_header Host $host:80;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Via "nginx";
    }
}

```

### 2. 权限认证

1.  在外网服务器上运行以下命令以生成公钥：`ssh-keygen -o`
2.  将公钥复制到内网服务器上，并添加到 `~/.ssh/authorized_keys`

### 3. 目标内网服务器 ssh 连接

1.  在本地启动服务并将其监听在端口 8088
2.  将外网访问的端口 10412 转发到本地端口 8088

```shell
nohup ssh -N -v -R 10412:127.0.0.1:8088 root@{外部服务器的外网IP} 2>&1 &
```

### 4. 保持会话

1.  在保持 SSH 会话中，加入以下命令来保持连接
2.  **ServerAliveInterval** 是指定服务器发送保持连接的数据包的时间（单位：秒）
3.  **ServerAliveCountMax** 是指定尝试与服务器保持连接的最大次数

```shell
nohup ssh -N -v -o ServerAliveInterval=10 -o ServerAliveCountMax=1000 -R 10412:127.0.0.1:8088 root@{外部服务器的外网IP} 2>&1 &
```
