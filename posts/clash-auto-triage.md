---
title: "Clash 设置国内国外自动分流访问"
status: 1
created_at: 2023-03-10T17:15:55+08:00
updated_at: 2026-04-30T14:45:27+08:00
category_id: 4
is_top: 0
tag_ids: [46, 47, 48]
description: "Clash 是一款开源的网络代理工具，可以帮助用户实现对网络流量的控制和管理。我使用了很久，但苦于每次访问国内外网络需要手动开关代理，于是我就问了下GPT， 还真就解决了。"
word_count: 1268
---

Clash 是一款开源的网络代理工具，可以帮助用户实现对网络流量的控制和管理。我使用了很久，但苦于每次访问国内外网络需要手动开关代理，于是我就问了下GPT， 还真就解决了。

如果你也需要设置 Clash 区分国内和国外流量，可以按照以下步骤进行操作：

#### 1. 打开配置文件(windows在右下角)：

![image.png](https://raw.githubusercontent.com/whrsss/pic-sync/master/img/20230310170129.png)

![image.png](https://raw.githubusercontent.com/whrsss/pic-sync/master/img/20230310170252.png)

使用 Visual Studio Code 打开 config.yaml
内容是这样的：
```yml
#---------------------------------------------------#
## 配置文件需要放置在 $HOME/.config/clash/*.yaml

## 这份文件是clashX的基础配置文件，请尽量新建配置文件进行修改。
## ！！！只有这份文件的端口设置会随ClashX启动生效

## 如果您不知道如何操作，请参阅 官方Github文档 https://github.com/Dreamacro/clash/blob/dev/README.md
#---------------------------------------------------#

# (HTTP and SOCKS5 in one port)
mixed-port: 7890
# RESTful API for clash
external-controller: 127.0.0.1:9090
allow-lan: false
mode: rule
log-level: warning

proxies: 

proxy-groups:

rules:
- DOMAIN-SUFFIX,google.com,DIRECT
- DOMAIN-KEYWORD,google,DIRECT
- DOMAIN,google.com,DIRECT
- DOMAIN-SUFFIX,ad.com,REJECT
- GEOIP,CN,DIRECT
- MATCH,DIRECT
```

#### 2. 在 Clash 配置文件中添加以下规则：

```yaml
# 添加下面的, 跟在后面
payload:
- DOMAIN-SUFFIX,cn,Direct
- DOMAIN-KEYWORD,geosite,Proxy
- IP-CIDR,10.0.0.0/8,Direct
- IP-CIDR,172.16.0.0/12,Direct
- IP-CIDR,192.168.0.0/16,Direct
- IP-CIDR,127.0.0.0/8,Direct
- IP-CIDR,224.0.0.0/4,Direct
- IP-CIDR,240.0.0.0/4,Direct
- MATCH,Final

```

#### 3. 每个配置段的作用

然后我问了一下 gpt 这些配置的作用：

- DOMAIN-SUFFIX,cn,Direct 表示所有以 ".cn" 结尾的域名都直接连接（不通过代理）。
- DOMAIN-KEYWORD,geosite,Proxy 表示包含关键词 "geosite" 的域名（比如 "[www.geosite.com"）都通过代理连接。](http://www.geosite.xn--com%22%29-7b5ky65m8h8a714d7hau1aj3h./)
- IP-CIDR,10.0.0.0/8,Direct 表示 IP 地址在 10.0.0.0 - 10.255.255.255 范围内的都直接连接。
- IP-CIDR,172.16.0.0/12,Direct 表示 IP 地址在 172.16.0.0 - 172.31.255.255 范围内的都直接连接。
- IP-CIDR,192.168.0.0/16,Direct 表示 IP 地址在 192.168.0.0 - 192.168.255.255 范围内的都直接连接。
- IP-CIDR,127.0.0.0/8,Direct 表示本地回环地址都直接连接（通常是 127.0.0.1）。
- IP-CIDR,224.0.0.0/4,Direct 和 IP-CIDR,240.0.0.0/4,Direct 表示多播地址都直接连接。
- MATCH,Final 表示匹配到这条规则后，后面的规则不会再被匹配，可以理解为中断匹配流程，即该规则为终结规则（Final Rule）。