---
id: fa24be8d123a408eb15ef4317d1c20f0
title: "CloudFlare Tunnel 免费内网穿透的简明教程"
status: 1
created_at: 2024-02-27T13:01:35+08:00
updated_at: 2026-04-30T15:03:32+08:00
category_id: 4
is_top: 0
tag_ids: [101]
description: "Tunnel 可以做什么?

将本地网络的服务暴露到公网，可以理解为内网穿透。
将非常规端口服务转发到 80/443 常规端口。
自动为你的域名提供 HTTPS 认证。
为你的服务提供额外保护认证。
跳过国内服务器备案域名。
最重要的是——免费。"
word_count: 2472
identity: cloudflare-tunnel 
---



## Tunnel 可以做什么

- **将本地网络的服务暴露到公网，可以理解为内网穿透。** 例如我们在本地服务器 `localhost:8091` 搭建了一个 博客网站，我们只能在内网环境才能访问这个服务，但通过内网穿透技术，我们可以在任何广域网环境下访问该服务。相比 NPS 之类传统穿透服务，Tunnel 不需要公网云服务器，同时自带域名解析，无需 DDNS 和公网 IP。
- **将非常规端口服务转发到 80/443 常规端口。** 无论是使用公网 IP + DDNS 还是传统内网穿透服务，都免不了使用非常规端口进行访问，如果某些服务使用了复杂的重定向可能会导致 URL 中端口号丢失而引起不可控的问题，同时也不够优雅。
- **自动为你的域名提供 HTTPS 认证。**
- **为你的服务提供额外保护认证。**
- **最重要的是——免费。**

## Tunnel 工作原理

Tunnel 通过在本地网络运行的一个 Cloudflare 守护程序，与 Cloudflare 云端通信，将云端请求数据转发到本地网络的 IP + 端口。

## 前置条件

- 持有一个域名
- 将域名 DNS 解析托管到 CloudFlare (我目前都直接转到了 CloudFlare )
- 有一台服务器（本地非本地都可以，有没有公网 IP 都 OK），用于运行本地与 cloudflare 通信的 cloudflared 程序
- 一张境内双币信用卡（仅用于添加付款方式，服务是免费的）

## 开始

#### 1. 打开 [Cloudflare Zero Trust](https://one.dash.cloudflare.com/) 工作台面板

#### 2. 创建 **Cloudflare Zero Trust** ，选择免费计划。需要提供付款方式，使用境内的双币卡即可

![image.png](https://pic.whrss.com/2024/02/1708419784.png)

填写 team name，随意填写

![image.png](https://pic.whrss.com/2024/02/1708419807.png)

选择免费计划

![image.png](https://pic.whrss.com/2024/02/1708419823.png)

添加付款方式

![image.png](https://pic.whrss.com/2024/02/1708419835.png)

填写信用卡信息（仅验证，不会扣款），完成配置

#### 3\. 完成后，在 Access Tunnels 中，创建一个 Tunnel。

![image.png](https://pic.whrss.com/2024/02/1708419913.png)

创建 Tunnel
![image.png](https://pic.whrss.com/2024/02/1708419932.png)

#### 4\. 选择 Cloudflared 部署方式。

Tunnel 需要通过 Cloudflared 来建立云端与本地网络的通道，这里推荐选择 Docker 部署 Cloudflared 守护进程以使用 Tunnel 功能。

![image.png](https://pic.whrss.com/2024/02/1708420329.png)

获取 Cloudflared 启动命令及 Token

在本地网络主机上运行命令。我们还可以加上`--name cloudflared -d --restart unless-stop`为 Docker 容器增加名称和后台运行。你可以使用下方我修改好命令来创建 Docker，注意替换你为自己的 Token（就是网页中`—-token` 之后的长串字符）

```shell
docker run --name cloudflared -d --restart unless-stop cloudflare/cloudflared:latest tunnel --no-autoupdate run --token <YourToken>
```

#### 5\. 配置域名和转发 URL

为你的域名配置一个子域名（Subdomain），Path 留空，URL 处填写内网服务的 IP 加端口号。注意 Type 处建议使用 HTTP，因为 Cloudflare 会自动为你提供 HTTPS，因此此处的转发目标可以是 HTTP 服务端口。

![image.png](https://pic.whrss.com/2024/02/1708420312.png)

配置内网目标 IP+端口

这里要注意，配置的 ip 如果是 127.0.0.1 或者是 localhost，是不行的
对于 linux 可以创建一个桥接网络  
下面的 localNet 是网络名字,可自行修改;关于 192.168.0.0 这个子网,也可以自行定义.  
默认按照下面的命令,执行后将可以通过 192.168.1.100 访问宿主机.

```sh
# 使用192.168.1.100替换127.0.0.1,如mongodb://192.168.1.100:27017
docker network create -d bridge --subnet 192.168.0.0/24 --gateway 192.168.1.100 localNet
```

## 完成

接着访问刚刚配置的三级域名，例如 [https://app.yourdomain.com](https://app.yourdomain.com/)（是的，你没看错，是 https，cloudflare 已经自动为域名提供了 https 证书）就可以访问到内网的非公端口号服务了。一个 Tunnel 中可以添加多条三级域名来跳转到不同的内网服务，在 Tunnel 页面的 Public Hostname 中新增即可。

## 为你的服务添加额外验证

如果你觉得这种直接暴露内网服务的方式有较高的安全风险，我们还可以使用 Application 功能为服务添加额外的安全验证。

1\. 点击 Application - Get started。

![image.png](https://pic.whrss.com/2024/02/1708420288.png)

创建 Application

2\. 选择 Self-hosted。

![image.png](https://pic.whrss.com/2024/02/1708420275.png)

选择类型

3\. 填写配置，**注意 Subdomain 和 Domain 需要使用刚刚创建的 Tunnel 服务相同的 Domain 配置**。

![image.png](https://pic.whrss.com/2024/02/1708420257.png)

配置三级域名

4\. 选择验证方式。填写 Policy name（任意）。在 Include 区域选择验证方式，示例图片中使用的是 Email 域名的方式，用户在访问该网络时需要使用指定的邮箱域名（如@gmail.com）验证，这种方式比较适合自定义域名的企业邮箱用户。另外你还可以指定特定完整邮箱地址、IP 地址范围等方式。

![image.png](https://pic.whrss.com/2024/02/1708420237.png)

选择验证方式

5\. 完成添加

![image.png](https://pic.whrss.com/2024/02/1708420205.png)

此时，访问 https://app.yourdomain.com 可以看到网站多了一个验证页面，使用刚刚设置的域名邮箱，接收验证码来访问。

![image.png](https://pic.whrss.com/2024/02/1708420185.png)

## 评价

除了上述直接转发 http 服务之外，Tunnel 还支持 RDP、SSH 等协议的转发，玩法丰富。

作为一款免费的服务，简单的配置，低门槛使用条件，适合简单部署尝试。

通过 ssh 隧道部署，还可以跳过国内服务器域名必须备案的条件。

不过要注意的是 Tunnel 在国内访问速度不快，并且有断流的情况，请酌情使用。
