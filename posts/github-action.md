---
title: "GitHub Action 自动化部署简单尝试"
status: 1
created_at: 2023-03-31T18:38:20+08:00
updated_at: 2026-04-30T14:45:27+08:00
category_id: 1
is_top: 0
tag_ids: [51, 52, 53]
description: "之前看到很多大佬的 Blog 是部署在 Github 上面的，但因为自己目前的博客是带后端的，所以就没有考虑。很久之前看到 @yihong 的心跳和跑步，感觉挺不错的，但因为自己没有跑步的习惯，就感觉不是很感冒 ???? 直到最近在听零机一动的时候，又听到了 yihong 的跑步， 我突然想到，我应该也可以把我的游泳 骑车 有氧也像 @yihong 的跑步数据一样上传过来。那么第一件事，我需要了解一下 GitHub Action 的机制，小小地尝试一下。"
word_count: 1521
---


之前看到很多大佬的 Blog 是部署在 Github 上面的，但因为自己目前的博客是带后端的，所以就没有考虑。很久之前看到 @yihong 的心跳和跑步，感觉挺不错的，但因为自己没有跑步的习惯，就感觉不是很感冒 ???? 直到最近在听零机一动的时候，又听到了 yihong 的跑步， 我突然想到，我应该也可以把我的游泳 骑车 有氧也像 @yihong 的跑步数据一样上传过来。那么第一件事，我需要了解一下 GitHub Action 的机制，小小地尝试一下。

---

首先，先看下 github action 的基本功能，我是通过[阮一峰老师的文章](https://www.ruanyifeng.com/blog/2019/09/getting-started-with-github-actions.html)了解的。

GitHub Actions 是一种自动化工具，2019 年开始测试，同年 11 月上线。可以在 GitHub 上构建、测试和部署软件项目。它允许 Github 用户通过预定义或自定义的操作序列来自动化整个软件开发生命周期中的流程（过程很像 Dockerfile 或 Jenkinsfile 的构建），并且完全与 Github 集成。

使用 GitHub Actions，你可以创建一个由事件触发或定期运行的工作流（workflow），其中包含一个或多个步骤（steps）。每个步骤都是一个特定任务的命令，例如编译代码、运行测试、打包发布版本等。你可以选择将这些步骤放在同一个工作流内，也可以将它们拆分为不同的工作流文件。

此外，Github Action 还提供了各种操作 (actions)，可以让您轻松地执行常见的任务，如 Shell 脚本、推送 Docker 映像、调用 API 等等。 也有一些和捷径市场一样的分享社区，把自己编写的 action yml 分享给他人使用。

GitHub Actions 使得整个开发过程变得更加高效方便，能够提升团队的交付速度。同时，近年来越来越多的开源社区也开始尝试在其上面构建 CI/CD 流水线。

这里可能没有接触过的朋友可能会想，我是否可以在上面部署应用呢？
这里的 Action 的作用类似于 Jenkins，可以帮你打包编译，如果你是一个静态项目，那么作为静态仓库的 Github 仓库是可以直接部署的； 但是如果是动态的后端服务，需要 Cpu 的，就无法进行了。

---

#### 第一步，创建 GitHub Token

头像 -> settings -> 左侧 Developer settings -> 左侧 Personal access tokens -> Tokens(classic)
-> Generate new token (classic)
赋予 workflow 的权限

#### 第二步： 在目标仓库中创建  `.github/workflows`  目录
目录中创建一个 yml 文件 workflow-name.yml
按照 GitHub 文档进行 yml 编写，我这里先了解机制，所以直接 fork @yihong 的项目，

#### 第三步 Action 设置

在项目的 settings 中找到 actions 下的 general，向下划动
![image.png](https://raw.githubusercontent.com/whrsss/pic-sync/master/img/20230331160949.png)

![image.png](https://raw.githubusercontent.com/whrsss/pic-sync/master/img/20230331161110.png)
找到工作流的读写权限，进行勾选。这样我们在触发 action 后，项目的 push 就有权限进行了。

#### 第四步，触发这个 Action

在手机上部署[这个捷径](https://www.icloud.com/shortcuts/6ab6047b459c41ad822ad6b94b1c03d4)
Token 填写第一步获取到的Token
GitHub_Name 填写Github用户名
Action_ID 填写yml名字（workflow-name.yml就全部填上去）

修改仓库名字和分支（图中的whrsss是我的仓库名字，master是我的分支名）

![image.png](https://raw.githubusercontent.com/whrsss/pic-sync/master/img/20230331162133.png)

修改完成后，点击快捷指令的运行。然后打开Github 仓库，点开 Action 栏，项目正在构建。点开可以查看构建日志。

![image.png](https://raw.githubusercontent.com/whrsss/pic-sync/master/img/20230331162634.png)
