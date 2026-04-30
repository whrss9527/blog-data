---
title: "一个有趣且实用的开源项目，将网页打包成应用"
status: 1
created_at: 2023-04-04T16:19:46+08:00
updated_at: 2026-04-30T14:18:04+08:00
category_id: 2
is_top: 0
tag_ids: [54, 55]
description: "最近在GitHub上面发现了一个[很有趣的开源应用](https://github.com/tw93/Pake)，可以将网页打包成应用

底层使用了rust进行开发，支持 Mac、Windows 和 Linux。使用命令将网站直接打包成PC应用。"
word_count: 831
---

最近在GitHub上面发现了一个[很有趣的开源应用](https://github.com/tw93/Pake)，可以将网页打包成应用：

这里是[它的使用文档](https://github.com/tw93/Pake/blob/master/bin/README_EN.md)

底层使用了rust进行开发，支持 Mac、Windows 和 Linux。使用命令将网站直接打包成应用。

这里我以最近部署的一个私人 ChatGPT Next为例， 我的地址为 https://gpt.whrss.com

需要本地先有一个node环境，能使用npm命令。

然后npm全局安装这个工具
```shell
npm install -g pake-cli
```

安装完之后，就可以使用
```shell
pake url [OPTIONS]...
```
来生成桌面应用了，参数有很多，我先用了这些：
这是我的命令：
```shell
pake https://gpt.whrss.com --name GPT  --icon /Users/wu/Downloads/gpt.icns --height 700 --width 900
```
> 这里我指定了网站的url : https://gpt.whrss.com
> 指定了生成的应用包的名字：GPT
> 指定了应用的 icon 
> 指定了应用打开初始化的高度和宽度
> 
	mac icon生成和获取的地方 ： https://macosicons.com/#/

然后我进行了运行，因为我的icon指定的目录是Downloads下面，所以它直接生成在了下面，默认是用户目录。
![image.png](https://raw.githubusercontent.com/whrsss/pic-sync/master/img/20230404161314.png)

如上图，生成的包体非常的小和轻量，只有不到3M。

= 更新 =

```sh
# 查看npm安装的插件的版本
npm -g list
/usr/local/lib
├── xxx
├── npm@9.3.1
├── pake-cli@2.0.0-alpha7
├── xxxxxxx
└── xxxxxxx

# 更新指定插件
npm -g update pake-cli 
# 再次查看 
npm -g list
```


---

关于这个软件的应用，我想到的最多就是那些常用的，但没有pc客户端的应用，比如 Twitter、WeRead、Spotify 等等，我也是接触这个项目后第一时间就使用上了，体验很不错，比之前方便了不少。
![image.png](https://raw.githubusercontent.com/whrsss/pic-sync/master/img/20230404161605.png)

