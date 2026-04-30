---
id: 9bd7ca48dab7420aae1889ec0c98b228
title: "探索服务端通信技术：短轮询、WebSocket、SSE 与长轮询的深度比较"
status: 1
created_at: 2023-11-19T13:51:46+08:00
updated_at: 2026-04-30T15:03:32+08:00
category_id: 1
is_top: 0
tag_ids: [94, 95, 96, 97, 98, 99]
description: "在现代 Web 应用中，服务端与客户端之间的高效通信至关重要。本文探讨了四种主流的服务端通信方法：短轮询、WebSocket、SSE（Server-Sent Events）和长轮询，分析它们的工作原理、适用场景及优缺点。"
word_count: 2254
identity: server-communication-methods
---




在现代 Web 应用中，服务端与客户端之间的高效通信至关重要。本文探讨了四种主流的服务端通信方法：短轮询、WebSocket、SSE（Server-Sent Events）和长轮询，分析它们的工作原理、适用场景及优缺点。

### 一、短轮询：高兼容性的传统选择

短轮询是服务端通信的一种基本方法，客户端通过定期发送 HTTP 请求来检查服务器上的更新。
![image.png](https://pic.whrss.com/2023/11/1700400293.png)

- **实际应用案例：** 适用于新闻网站或博客的评论更新，用户可以在较短的时间内看到新的评论。
- **优点：** 高兼容性，适用于所有支持 HTTP 的客户端。
- **缺点：** 高资源消耗，频繁建立和关闭 TCP 连接。
- **使用场景：** 最适合不频繁更新且对实时性要求不高的应用。

最普通的一个场景：客户端定期向服务器请求最新消息

```go
package main

import (
    "fmt"
    "net/http"
    "time"
)

func main() {
    http.HandleFunc("/poll", func(w http.ResponseWriter, r *http.Request) {
        // 假设这里是从数据库或某个存储检索最新消息的逻辑
        message := "Hello, world! - " + time.Now().Format(time.RFC1123)
        fmt.Fprint(w, message)
    })

    http.ListenAndServe(":8080", nil)
}
```

### 二、WebSocket：实时双向通信的理想选择

WebSocket 是一种在单个 TCP 连接上进行全双工通信的协议，适用于需要实时双向通信的应用。
![image.png](https://pic.whrss.com/2023/11/1700400421.png)

- **实际应用案例：** 在线游戏、股票交易平台，这些应用需要实时的数据交互。
- **优点：** 低延迟，适用于实时通信。
- **缺点：** 在某些网络环境下可能受限。
- **使用场景：** 高效的实时通信，特别是需要频繁数据交换的场景。

最常用的一个场景：实时聊天应用

服务端和客户端通过 WebSocket 保持连接，实现实时通讯。

```go
package main

import (
    "net/http"
    "github.com/gorilla/websocket"
)

var upgrader = websocket.Upgrader{
    ReadBufferSize:  1024,
    WriteBufferSize: 1024,
}

func main() {
    http.HandleFunc("/ws", func(w http.ResponseWriter, r *http.Request) {
        conn, _ := upgrader.Upgrade(w, r, nil) // 错误处理省略
        for {
            messageType, p, _ := conn.ReadMessage() // 错误处理省略
            conn.WriteMessage(messageType, p)       // 错误处理省略
        }
    })

    http.ListenAndServe(":8080", nil)
}

```

### 三、 SSE：简单的单向服务器推送技术

SSE 允许服务器向客户端单向发送更新，是一种基于 HTTP 的技术。
![image.png](https://pic.whrss.com/2023/11/1700400501.png)

- **实际应用案例：** 实时新闻更新、股票市场的价格更新。
- **优点：** 实现简单，支持自动重连。
- **缺点：** 浏览器兼容性问题，无法实现双向通信。
- **使用场景：** 适用于单向数据流的应用，如实时新闻或股价更新。

一个典型的应用场景：股票价格更新

```go
package main

import (
    "fmt"
    "net/http"
    "time"
)

func main() {
    http.HandleFunc("/sse", func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "text/event-stream")
        w.Header().Set("Cache-Control", "no-cache")
        w.Header().Set("Connection", "keep-alive")

        for i := 0; ; i++ {
            fmt.Fprintf(w, "data: Price update %d\n\n", i)
            w.(http.Flusher).Flush()
            time.Sleep(time.Second * 2) // 模拟数据更新
        }
    })

    http.ListenAndServe(":8080", nil)
}

```

### 四、长轮询：短轮询的高效替代

长轮询在服务器有数据更新时才响应客户端请求，是短轮询的改进版。
![](https://pic.whrss.com/2023/11/1700400647.png)
- **实际应用案例：** 社交媒体通知更新，如 Facebook 或 Twitter 的新消息提示。
- **优点：** 减少不必要的轮询请求，相对于短轮询更高效。
- **缺点：** 仍需频繁建立和关闭连接。
- **使用场景：** 适用于对实时性要求不是非常高，但更新频率较高的场景。

我最近使用到的一个应用场景：AI 生图以后，通知给客户端

```go
package main

import (
    "fmt"
    "net/http"
    "time"
)

func main() {
    messages := make(chan string) // 消息通道

    go func() {
        for {
            time.Sleep(time.Second * 5) // 每5秒生成一条消息
            messages <- fmt.Sprintf("New message at %v", time.Now())
        }
    }()

    http.HandleFunc("/long-poll", func(w http.ResponseWriter, r *http.Request) {
        message := <-messages // 等待新消息
        fmt.Fprint(w, message)
    })

    http.ListenAndServe(":8080", nil)
}

```

### 结论

在选择服务端通信技术时，应考虑应用的具体需求和场景。WebSocket 适合需要高实时性和双向通信的应用；SSE 适用于简单的单向数据推送；而短轮询和长轮询则适用于更新频率不高的场景。
选择合适的技术可以显著提高用户体验和应用性能。
我也曾遇到 WebSocket 一把梭的，在一些性能不敏感的地方，确实没有什么区别，但是，这不应该是一个技术人员的追求。


