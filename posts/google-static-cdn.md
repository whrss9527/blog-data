---
title: "在 Google 设置静态页面 CDN 加速"
status: 1
created_at: 2023-02-15T16:52:35+08:00
updated_at: 2026-04-30T14:18:09+08:00
category_id: 1
is_top: 0
tag_ids: [34, 29, 35]
description: "在google设置静态页面 CDN加速"
word_count: 597
---

在google设置静态页面 CDN加速

### 一、 创建bucket，设置bucket
https://console.cloud.google.com/storage/browser 
- #####  创建bucket 
![image.png](https://raw.githubusercontent.com/whrsss/pic-sync/master/img/20230214111414.png)

- ##### 设置bucket公开访问

1.  在bucket列表中，进入刚创建的bucket。
    
2.  选择页面顶部附近的**权限**标签。
    
3.  在**权限**部分中，点击 person_add **授予访问权限**按钮。
    
    此时将显示“授予访问权限”对话框。
    
4.  在**新的主帐号**字段中，输入 `allUsers`。
    
5.  在**选择角色**下拉列表的过滤条件框中输入 **Storage Object Viewer**，然后从过滤后的结果中选择 **Storage Object Viewer**。
    
6.  点击**保存**。
    
7.  点击**允许公开访问**。

### 二、设置CDN
https://console.cloud.google.com/net-services/cdn/list
添加来源，创建时可能会附带创建好负载均衡，用来做DNS解析使用
![image.png](https://raw.githubusercontent.com/whrsss/pic-sync/master/img/20230214111757.png)

指向刚才新创建的bucket

在host and path rules设置地址规则。


### 三、DNS解析
进入负载均衡控制台
https://console.cloud.google.com/net-services/loadbalancing/list/loadBalancers

选择创建好的负载均衡，点进去，拿到公网IP，到DNS控制台进行解析即可。