---
title: "从零开始搭建家庭软路由系统（安装OpenWrt，并做为旁路由接入家庭网络）"
status: 1
created_at: 2023-03-21T14:09:57+08:00
updated_at: 2026-04-30T14:18:09+08:00
category_id: 4
is_top: 0
tag_ids: [49, 50, 48]
description: "之前刷推看到了不少人发软路由，最近又看到a姐发了一句：全推入手软路由了 
开始我还觉得软路由对我的作用应该不大吧，随着从众心理的影响，我觉得我应该试试。
刚好我还有从笔记本上拆下的内存和固态，岂不是严丝合缝？"
word_count: 1987
---


之前刷推看到了不少人发软路由，最近又看到a姐发了一句：全推入手软路由了 
开始我还觉得软路由对我的作用应该不大吧，随着从众心理的影响，我觉得我应该试试。
刚好我还有从笔记本上拆下的内存和固态，岂不是严丝合缝？

然后我就下单了一个![50091679362219_.pic.jpg](https://raw.githubusercontent.com/whrsss/pic-sync/master/img/50091679362219_.pic.jpg)


外观差不多是这个样子：

![50101679362338_.pic.jpg](https://raw.githubusercontent.com/whrsss/pic-sync/master/img/50101679362338_.pic.jpg)
![50111679362342_.pic.jpg](https://raw.githubusercontent.com/whrsss/pic-sync/master/img/50111679362342_.pic.jpg)

内部结构大概是这个样子：

![50121679362346_.pic.jpg](https://raw.githubusercontent.com/whrsss/pic-sync/master/img/50121679362346_.pic.jpg)


拥有5个网口、两个 USB 2.0 和两个 USB 3.0 口、HDMI、DP、存储卡位以及两个笔记本内存条槽位、一根 m.2 插槽和一个 SATA 接口，可以说五脏俱全。

我买的是只带电源的，内存条和磁盘需要自己加。

### 一、第一步 组装机器
-   打开机器背部的固定螺丝以及后盖，将内存条插入相应槽位
-   接入 SATA 固态硬盘或 M.2 硬盘，不要装上后盖，以防有问题还需要再拆开。
-   连接电源，此时路由器将自动开机
-   开机后如果听到一声响，表示内存识别功能正常。如果听到三声响连续作响就表示出现了问题。我使用的两个内存条，分别在两个槽位上尝试，只有一根（金士顿的骇客神条2400）能在其中一个槽位上被识别。如果两个槽位都无法识别，就只能买同店的内存条了。

![50131679363062_.pic.jpg](https://raw.githubusercontent.com/whrsss/pic-sync/master/img/50131679363062_.pic.jpg)


-   接下来就是准备安装系统。

### 二、 制作启动盘
我选择的系统是 [OpenWRT/LEDE](https://openwrt.org/) 在国内的家庭软路由中有着非常高的占有率，拥有海量的软件，和非常强大的生态。同时，OpenWRT 的教程也很丰富详实。
这里我使用的是 [KoolShare 固件](https://fw.koolcenter.com/)，内置了非常强大的插件市场。
##### 1. 下载efi

![image.png](https://raw.githubusercontent.com/whrsss/pic-sync/master/img/20230321095218.png)
![image.png](https://raw.githubusercontent.com/whrsss/pic-sync/master/img/20230321095327.png)

找到最新版本进行下载。
![image.png](https://raw.githubusercontent.com/whrsss/pic-sync/master/img/20230321095417.png)

##### 2. 写盘
	下载完成后，使用balenaetcher写盘工具进行写盘，
	- 如果没有安装balenaEtcher，首先需要下载并安装。
	- 插入你的U盘，并打开balenaEtcher。
	- 点击“选择镜像”按钮，选择刚才下载的.gz文件。
	- 点击“选择驱动器”按钮，选择你插入的U盘。
	- 点击“写入”按钮，balenaEtcher将开始写入镜像文件到你的U盘。
	- 等待写入过程完成，并在完成后安全地弹出。

### 三、 安装和设置
##### 1. 安装
	- 将写好的盘插入软路由，接一个显示器和一个键盘给软路由，接上电源，开机。按f11(不同机器不同快捷键)进入快速启动。一切自动执行到终端提示完成，回车，提示出OpenWrt图标。
	- 输入quickstart，1 设置Lan口IP，2 安装OpenWrt，3 重置 
	- 选择2安装操作系统。这时候会提示安装位置，如果识别磁盘没有问题，就会让你选择你内部的固态或者sata硬盘。如果提示没有找到硬盘，就是硬盘有问题或者接口接错了（我在使用sata协议的m.2接口固态硬盘时，提示了这个问题，固态硬盘需要选择nvme接口协议的） 
	- 几秒后会提示你拔掉U盘，这时候就会自动重启进入系统。 
	- 同样的 输入quickstart  选择1 设置lan口IP。可以设置为 192.168.2.6， 子网掩码设置为255.255.255.0（等下会说明用途）
	- 接一根网线从软路由 lan 口到 PC 上，pc打开浏览器，访问 192.168.2.6 ，这时候会进入软路由后台，用户名root 默认密码 password 
	到这里，安装基本完成，openWrt可以进入后台访问。

##### 2. 设置旁路由
	旁路由就是将软路由接入在普通硬路由下，做数据处理。
![image.png](https://raw.githubusercontent.com/whrsss/pic-sync/master/img/20230321101330.png)

-   在硬路由器控制台上禁用 DHCP 功能，然后记录下其 LAN 口 IP 地址（例如，我的是 192.168.2.5）
-   在软路由面板的“网络”下，选择“接口”并设置 LAN 口，将 LAN 口 IP 设置为硬路由器 IP 同一网段（也就是192.168.2.6），将网关地址和 DNS 解析地址都设置为硬路由器 LAN 口地址 192.168.2.5 点击保存并应用。

拔下PC网线，将软路由Lan口与硬路由Lan口相连。
电脑连接原先的硬路由wifi网络， 浏览器输入192.168.2.6，正确访问到软路由后台。再登录软路由后台系统，检查网络状况，已连通。
