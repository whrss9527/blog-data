---
title: "os.signal golang 中的信号处理"
status: 1
created_at: 2023-07-29T05:56:55+08:00
updated_at: 2026-04-30T14:47:03+08:00
category_id: 1
is_top: 0
tag_ids: [61, 76, 77]
description: "在程序进行重启等操作时，我们需要让程序完成一些重要的任务之后，优雅地退出，Golang为我们提供了signal包，实现信号处理机制，允许Go 程序与传入的信号进行交互。"
word_count: 1142
---

在程序进行重启等操作时，我们需要让程序完成一些重要的任务之后，优雅地退出，Golang为我们提供了signal包，实现信号处理机制，允许Go 程序与传入的信号进行交互。

Go语言标准库中signal包的核心功能主要包含以下几个方面:

##### 1. signal处理的全局状态管理

通过handlers结构体跟踪每个signal的处理状态，包含信号与channel的映射关系,以及每个信号的引用计数。

##### 2. 信号处理的注册与注销

Notify函数用于向指定的channel注册信号处理，会更新handlers的状态。

Stop函数用于注销指定channel的信号处理，将其从handlers中移除。

Reset函数用于重置指定信号的处理为默认行为。

##### 3. 信号的抓取与分发

process函数在收到signal时，会把它分发给所有注册了该信号的channel。

##### 4. signal处理的恢复

通过cancel函数，可以恢复signal的默认行为或忽略。

##### 5. Context信号通知支持

NotifyContext函数会创建一个Context，在Context结束时自动注销signal处理。

##### 6. 处理signal并发访问的同步

通过handlers的锁保证对全局状态的线程安全访问。

##### 7. 一些工具函数

如handler的mask操作,判断signal是否在ignore列表中等。

总的来说,该实现通过handlers跟踪signal与channel的关系,在收到signal时分发给感兴趣的channel,提供了flexible和高效的signal处理机制。




在实际地使用中，我们需要创建一个接收信号量的channel，使用Notify将这个channel注册进去，当信号发生时，channel就可以接收到信号，后续的业务就可以针对性地进行处理。如下：
```go
package main

import (
	"fmt"
	"os"
	"os/signal"
	"syscall"
)

func main() {

	// 创建一个channel来接收SIGINT信号
	c := make(chan os.Signal)

	// 监听SIGINT信号并发送到c
	signal.Notify(c, syscall.SIGINT)

	// 使用一个无限循环来响应SIGINT信号
	for {
		fmt.Println("Waiting for SIGINT")
		<-c
		fmt.Println("Got SIGINT. Breaking...")
		break
	}
}
```


*共有32个信号量，相对应的枚举在syscall包下*

常用的信号值包括：

	 SIGHUP 1 终端控制进程结束(终端连接断开)
	 SIGINT 2 用户发送INTR字符(Ctrl+C)触发
	 SIGQUIT 3 用户发送OUIT字符(Ctrl+/触发
	 SIGKILL 9 无条件结束程序(不能被捕获、阻塞或忽略)
	 SIGUSR1 10 用户保留
	 SIGUSR2 12 用户保留
	 SIGPIPE 13 消息管道损坏(FIFO/Socket通信时，管道未打开而进行写操作)
	 SIGALRM 14 时钟定时信号
	 SIGTERM 15 结束程序(可以被捕获、阻塞或忽略)

在go框架中，项目中实际使用到signal进行优雅退出见：[如何在go中实现程序的优雅退出，go-kratos源码解析](https://whrss.com/post/go-grace-stop)