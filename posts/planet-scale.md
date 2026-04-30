---
title: "探索 PlanetScale：划分分支的 MySQL Serverless 平台"
status: 1
created_at: 2023-07-12T03:37:52+08:00
updated_at: 2026-04-30T14:18:03+08:00
category_id: 2
is_top: 0
tag_ids: [73, 74, 75, 36]
description: "最近我发现了一个非常有趣的国外MySQL Serverless平台，它叫做PlanetScale。这个平台不仅仅是一个数据库，它能像代码一样轻松地创建开发和测试环境。你可以从主库中拉出一个与之完全相同结构的development或staging数据库，并在这个环境中进行开发和测试。所有的数据都是隔离的，不会相互干扰。

当你完成开发后，你可以创建一个deploy request，PlanetScale会自动比对并生成Schema diff，然后你可以仔细审查需要部署的内容。确认没问题，你就可以将这些变更部署到线上库中。整个部署过程不会导致停机时间，非常方便。"
word_count: 1394
---




最近我发现了一个非常有趣的国外MySQL Serverless平台，它叫做[PlanetScale](https://app.planetscale.com)。这个平台不仅仅是一个数据库，它能像代码一样轻松地创建开发和测试环境。你可以从主库中拉出一个与之完全相同结构的development或staging数据库，并在这个环境中进行开发和测试。所有的数据都是隔离的，不会相互干扰。

当你完成开发后，你可以创建一个deploy request，PlanetScale会自动比对并生成Schema diff，然后你可以仔细审查需要部署的内容。确认没问题，你就可以将这些变更部署到线上库中。整个部署过程不会导致停机时间，非常方便。

PlanetScale的入门使用是免费的，他们提供了以下免费套餐：

- 5GB存储空间
- 每月10亿行读取操作
- 每月1000万行写入操作
- 1个生产分支
- 1个开发分支
- 社区支持

如果超出了免费套餐的限制，他们会按照以下价格收费：每GB存储空间每月2.5美元，每10亿行读取操作每月1美元，每100万行写入操作每月1.5美元。对于我这样的个人使用者，真的太不错了。

这个平台运行在云上，提供了一个Web管理界面和一个CLI工具。我试了一下他们的Web管理界面，但发现它并不是很好用，无法进行批量的SQL执行。于是我研究了一下CLI工具的使用，并做了一份小记录，现在和大家分享一下。

以下是在 `Mac` 中使用`PlanetScale CLI`工具的步骤：
其他系统安装可见：[官方文档](https://planetscale.com/docs/concepts/planetscale-environment-setup)

#### 1. 安装pscale工具

`brew install planetscale/tap/pscale`

#### 2. 更新brew和pscale，确保使用的是最新版本

`brew update && brew upgrade pscale`

#### 3. 进行认证

`pscale auth login`

这个命令会在浏览器中打开一个页面
![image.png](https://raw.githubusercontent.com/whrsss/pic-sync/master/img/20230712112110.png)
如果你已经登录了PlanetScale账号，它会直接让你确认验证。验证成功后，你就可以开始使用CLI工具了。

如果你走到这步的时候提示：
```java
Error: error decoding error response: invalid character '<' looking for beginning of value
```
你需要调整一下网络～ 目前是不给大陆用户IP使用的。

#### 4. 连接到相应的数据库分支

`pscale connect [数据库名] [分支名] # 例如： pscale connect blog main`

连接成功后，你就可以通过本地的3306端口代理访问远程数据库了。
```
Secure connection to database whrss and branch main is established!.
Local address to connect your application: 127.0.0.1:3306 (press ctrl-c to quit)
```

![image.png](https://raw.githubusercontent.com/whrsss/pic-sync/master/img/20230712111233.png)

#### 5. 本地连接  
 点击`Get connection strings`，你就可以得到连接数据库所需的账号名和密码，然后可以在本地的数据库连接软件中直接连接数据库了。  
    ![](https://raw.githubusercontent.com/whrsss/pic-sync/master/img/202307121100351.png)
    
6. 选择适合你的编程语言的连接串，这样你就可以在不同的程序中直接使用了。  
    ![image.png](https://raw.githubusercontent.com/whrsss/pic-sync/master/img/20230712110227.png)
    

通过这些简单的步骤，你就可以轻松地使用PlanetScale来管理和部署你的MySQL应用了。快来体验一下吧！


