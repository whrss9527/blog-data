---
title: "如何在 Go 中实现程序的优雅退出，go-kratos 源码解析"
status: 1
created_at: 2023-07-29T05:58:45+08:00
updated_at: 2026-04-30T14:47:03+08:00
category_id: 1
is_top: 0
tag_ids: [61, 78, 79]
description: "使用kratos这个框架有近一年了，最近了解了一下kratos关于程序优雅退出的具体实现。
"
word_count: 2118
---

使用kratos这个框架有近一年了，最近了解了一下kratos关于程序优雅退出的具体实现。

这部分逻辑在app.go文件中，在main中，找到app.Run方法，点进入就可以了

它包含以下几个部分:

1. App结构体:包含应用程序的配置选项和运行时状态。
2. New函数:创建一个App实例。
3. Run方法:启动应用程序。主要步骤包括:

	- 构建ServiceInstance注册实例
	- 启动Server
	- 注册实例到服务发现
	- 监听停止信号

4. Stop方法:优雅停止应用程序。主要步骤包括:

	- 从服务发现中注销实例
	- 取消应用程序上下文
	- 停止Server

5. buildInstance方法:构建用于服务发现注册的实例。
6. NewContext和FromContext函数:给Context添加AppInfo,便于后续从Context获取。

核心的逻辑流程是:

1. 创建App实例
2. 在App.Run()里面启动Server,注册实例,监听信号
3. 接收到停止信号后会调用App.Stop()停止应用

我们先对Run方法进行一个源码进行查看

```go
// Run executes all OnStart hooks registered with the application's Lifecycle.
func (a *App) Run() error {

  // 构建服务发现注册实例
  instance, err := a.buildInstance() 
  if err != nil {
    return err
  }

  // 保存实例  
  a.mu.Lock()
  a.instance = instance
  a.mu.Unlock()

  // 创建错误组
  eg, ctx := errgroup.WithContext(NewContext(a.ctx, a))

  // 等待组,用于等待Server启动完成
  wg := sync.WaitGroup{}

  // 启动每个Server
  for _, srv := range a.opts.servers {
    srv := srv 
    eg.Go(func() error {
      // 等待停止信号
      <-ctx.Done()  
      // 停止Server
      stopCtx, cancel := context.WithTimeout(a.opts.ctx, a.opts.stopTimeout)
      defer cancel()
      return srv.Stop(stopCtx)
    })

    wg.Add(1)
    eg.Go(func() error {
      // Server启动完成
      wg.Done() 
      // 启动Server  
      return srv.Start(NewContext(a.opts.ctx, a)) 
    })
  }

  // 等待所有Server启动完成
  wg.Wait()

  // 注册服务实例
  if a.opts.registrar != nil {
    rctx, rcancel := context.WithTimeout(ctx, a.opts.registrarTimeout)
    defer rcancel()
    if err := a.opts.registrar.Register(rctx, instance); err != nil {
      return err
    }
  }
  
  // 监听停止信号
  c := make(chan os.Signal, 1)
  signal.Notify(c, a.opts.sigs...)
  eg.Go(func() error {
    select {
    case <-ctx.Done():
      return nil
    case <-c:
      // 收到停止信号,停止应用------------- ⬅️注意此时
      return a.Stop() 
    }
  })

  // 等待错误组执行完成
  if err := eg.Wait(); err != nil && !errors.Is(err, context.Canceled) {
    return err
  }

  return nil
}
```

核心逻辑就是这里⬇️，使用signal.Notify去监听操作系统给出的停止信号。

```go
  // 监听停止信号
  c := make(chan os.Signal, 1)
  signal.Notify(c, a.opts.sigs...)
  eg.Go(func() error {
    select {
    case <-ctx.Done():
      return nil
    case <-c:
      // 收到停止信号,停止应用
      return a.Stop() 
    }
  })
```

然后调用了Stop方法，我们再看下Stop的源码

```go
// Stop gracefully stops the application.
func (a *App) Stop() error {

  // 获取服务实例 
  a.mu.Lock()
  instance := a.instance
  a.mu.Unlock()

  // 从服务发现注销实例
  if a.opts.registrar != nil && instance != nil {
    ctx, cancel := context.WithTimeout(NewContext(a.ctx, a), a.opts.registrarTimeout)
    defer cancel()
    if err := a.opts.registrar.Deregister(ctx, instance); err != nil {
      return err
    }
  }

  // 取消应用上下文
  if a.cancel != nil {
    a.cancel() 
  }

  return nil
}
```

主要步骤是:

	1. 获取已经保存的服务实例
	2. 如果配置了服务发现,则从服务发现中注销该实例
	3. 取消应用上下文来通知应用停止
	
	在Run方法中,我们通过context.WithCancel创建的可取消的上下文Context，在这里通过调用cancel函数来取消该上下文，以通知应用停止。
	
	取消上下文会导致在Run方法中启动的协程全部退出，从而优雅停止应用。
	
	所以Stop方法比较简单，关键是利用了Context来控制应用生命周期。

我们可以注意到，在Run方法中，我们使用到了一个signal包下的Notify方法来对操作系统的关闭事件进行监听，这个是我们动作的核心，我把这部分单独整理在了[另一篇文章](https://whrss.com/post/go-signal)中。

通过对操作系统事件的监听，我们就可以对一些必须完成的任务进行优雅地停止，如果有一些任务必须完成，我们可以在任务开始使用 wg := sync.WaitGroup{} 来对任务进行一个Add操作，当所有任务完成，监听到操作系统的关闭动作，我们需要使用wg.wait() 等待任务完成再进行退出。以实现一个优雅地启停。