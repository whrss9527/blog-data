---
title: "最新版 Let’s Encrypt 免费证书申请步骤，保姆级教程"
status: 1
created_at: 2022-12-02T19:17:50+08:00
updated_at: 2026-04-30T14:45:27+08:00
category_id: 1
is_top: 0
tag_ids: [29]
description: "最近将域名迁到了google domain，就研究了一下Let’s Encrypt的域名证书配置。发现网上找到的教程在官方说明中已经废弃，所以自己写一个流程记录一下。"
word_count: 2940
---

最近将域名迁到了google domain，就研究了一下Let’s Encrypt的域名证书配置。发现网上找到的教程在官方说明中已经废弃，所以自己写一个流程记录一下。

步骤方法官方文档见：https://eff-certbot.readthedocs.io/en/stable/install.html#installation

snapd官方文档见：https://certbot.eff.org/instructions

#### 1. 安装snapd（这里我使用的是centos系统）

   ````shell
   sudo yum install snapd
   sudo systemctl enable --now snapd.socket
   sudo ln -s /var/lib/snapd/snap /snap
   ````


#### 2. 使用snapd安装certbot

   ````shell
   sudo snap install --classic certbot
   sudo ln -s /snap/bin/certbot /usr/bin/certbot
   ````

####3. 生成证书（需要指定nginx）
:fa-chevron-circle-right: [手动安装nginx](https://blog.csdn.net/w_monster/article/details/126352529 "手动安装nginx")
   ````shell
   certbot certonly --nginx --nginx-ctl /usr/local/nginx/sbin/nginx --nginx-server-root /usr/local/nginx/conf
   ````
这里的certonly就是只下载对应文件，不进行配置nginx，适用于自己配置或者更新使用。去掉则会帮你进行配置nginx（我没有试用）。
   > 可能出现的问题：
   >
   > The error was: PluginError('Nginx build is missing SSL module (--with-http_ssl_module).')
   >
   > 提示这个错误是因为目前nginx缺少–with-http_ssl_module这个模块，我们要添加这个模块。重新编译nginx
   >
   > 进入nginx下载的目录
   >
   > ./configure --prefix=/usr/local/nginx --with-http_ssl_module
   >
   > 编译完成后
   >
   > make
   >
   > make install
   >
   > /usr/local/nginx/sbin/nginx -s stop
   >
   > /usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf

   使用 /usr/local/nginx/sbin/nginx -V 查看是否生效

   ````shell
   [root]# /usr/local/nginx/sbin/nginx -V
   nginx version: nginx/1.23.2
   built by gcc 10.2.1 20200825 (Alibaba 10.2.1-3 2.32) (GCC) 
   built with OpenSSL 1.1.1k  FIPS 25 Mar 2021
   TLS SNI support enabled
   configure arguments: --prefix=/usr/local/nginx --with-http_ssl_module
   ````

   > 生效，重新执行上面的命令即可。

#### 4. 生成证书

   执行上面的命令后，程序会让你确认你的邮箱和你的域名，确认完成后会将证书文件生成在指定目录中。

   ````shell
   certbot certonly --nginx --nginx-ctl /usr/local/nginx/sbin/nginx --nginx-server-root /usr/local/nginx/conf
   Saving debug log to /var/log/letsencrypt/letsencrypt.log
   Enter email address (used for urgent renewal and security notices)
    (Enter 'c' to cancel): [这里输入你的邮箱]
   
   - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
   Please read the Terms of Service at
   https://letsencrypt.org/documents/LE-SA-v1.3-September-21-2022.pdf. You must
   agree in order to register with the ACME server. Do you agree?
   - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
   (Y)es/(N)o: Y
   
   - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
   Would you be willing, once your first certificate is successfully issued, to
   share your email address with the Electronic Frontier Foundation, a founding
   partner of the Let's Encrypt project and the non-profit organization that
   develops Certbot? We'd like to send you email about our work encrypting the web,
   EFF news, campaigns, and ways to support digital freedom.
   - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
   (Y)es/(N)o: Y [选Y 继续]
   Account registered.
   
   Which names would you like to activate HTTPS for?
   We recommend selecting either all domains, or all domains in a VirtualHost/server block.
   - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
   1: whrss.com
   - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
   Select the appropriate numbers separated by commas and/or spaces, or leave input
   blank to select all options shown (Enter 'c' to cancel):  [这里不需要输入，回车选所有]
   Requesting a certificate for whrss.com
   
   Successfully received certificate.
   Certificate is saved at: 
   # [这里告诉我们生成的文件路径和有效期]
   /etc/letsencrypt/live/whrss.com/fullchain.pem
   Key is saved at:         /etc/letsencrypt/live/whrss.com/privkey.pem
   This certificate expires on 2023-03-02.
   These files will be updated when the certificate renews.
   Certbot has set up a scheduled task to automatically renew this certificate in the background.
   
   - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
   If you like Certbot, please consider supporting our work by:
    * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
    * Donating to EFF:                    https://eff.org/donate-le
   - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
   
   ````

#### 5. Nginx.conf的配置

   ````
   server{
           #监听443端口
           listen 443 ssl;
           #对应的域名，空格分隔域名就可以了
           server_name whrss.com; 
           #第一个域名的文件
           ssl_certificate /etc/letsencrypt/live/whrss.com/fullchain.pem;
           ssl_certificate_key /etc/letsencrypt/live/whrss.com/privkey.pem;
           # 其他配置
           ssl_session_timeout 5m;
           ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
           ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
           ssl_prefer_server_ciphers on;
           #这是我的主页访问地址，因为使用的是静态的html网页，所以直接使用location就可以完成了。
           location / {
               root   /;
               index  /;
               proxy_pass http://127.0.0.1:9091;
               proxy_set_header Host $host:443;
               proxy_set_header X-Real-IP $remote_addr;
               proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
               proxy_set_header Via "nginx";
           }
   }
   ````

以上！！

