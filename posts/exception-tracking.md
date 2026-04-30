---
title: "一次线上异常的追踪与处理"
status: 1
created_at: 2023-06-02T19:24:29+08:00
updated_at: 2026-04-30T14:47:03+08:00
category_id: 1
is_top: 0
tag_ids: [60]
description: "5月31日晚，我们接到游戏玩家反馈，经常出现请求超时的提示。在我亲自登录游戏验证后，也出现了相同的错误，但游戏仍然可以正常运行，数据也没有任何问题。

经过客户端的错误检查，我们发现请求出现了`408 Request Timeout`的错误。该响应状态码意味着服务器打算关闭没有在使用的连接，即使客户端没有发送任何请求，一些服务器仍会在空闲连接上发送此信息。服务器决定关闭连接，而不是继续等待。"
word_count: 2613
---

# 一次线上异常的追踪与处理

5月31日晚，我们接到游戏玩家反馈，经常出现请求超时的提示。在我亲自登录游戏验证后，也出现了相同的错误，但游戏仍然可以正常运行，数据也没有任何问题。

经过客户端的错误检查，我们发现请求出现了`408 Request Timeout`的错误。该响应状态码意味着服务器打算关闭没有在使用的连接，即使客户端没有发送任何请求，一些服务器仍会在空闲连接上发送此信息。服务器决定关闭连接，而不是继续等待。

## 1. 日志检查

接下来，我查看了服务器的日志，发现后台的两个服务的日志都在正常运行，没有异常提示。当我进行pod查看时，发现有两个pod显示容器没有日志，这两个pod已经挂掉。

为什么这两个pod会宕机呢？我开始回溯近1小时的日志，发现在晚上10点左右，出现了JDBC连接异常。

```sh
### Error querying database. Cause: org.springframework.jdbc.CannotGetJdbcConnectionException: Failed to obtain JDBC Connection; nested exception is java.sql.SQLTransientConnectionException: HikariPool-1 - Connection is not available, request timed out after 31363ms.
```


通过Google查询，我了解到这种错误是由于Spring Boot的默认连接池HikariPool在连接排队阻塞，无法获取连接，最后导致超时。在数据库错误之后的一段时间内，出现了Java内存异常。

```json
{"@timestamp":"2023-05-31 22:18:24.382","level":"ERROR","source":{"className":"org.apache.juli.logging.DirectJDKLog","methodName":"log","line":175},"message":"Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Handler dispatch failed; nested exception is java.lang.OutOfMemoryError: Java heap space] with root cause","error.type":"java.lang.OutOfMemoryError","error.message":"Java heap space","error.stack_trace":"java.lang.OutOfMemoryError: Java heap space\n"}
```


由于我们没有设置连接池上限（默认最大为10），当获取连接阻塞后，请求排队，最终导致内存溢出。最后，由于内存溢出，pod触发`java.io.IOException: Broken pipe`错误，即管道断开，服务宕机。

```json
{"@timestamp":"2023-05-31 22:18:24.393","level":"WARN","source":{"className":"org.springframework.web.servlet.handler.AbstractHandlerExceptionResolver","methodName":"logException","line":199},"message":"Resolved [org.springframework.web.util.NestedServletException: Handler dispatch failed; nested exception is java.lang.OutOfMemoryError: Java heap space]"}
```


## 2. 增加连接池大小

Hikari是Spring Boot自带的连接池，默认最大只有10个。因此，我的第一步解决方案是增加这个服务的连接池大小。在服务的yaml数据库连接配置中增加了一些参数。 

```yml
  datasource:
    url: 'jdbc:mysql://rm-2xxxxxx'
    username: 'xx'
    password: 'xxx'
    # 下面这些????
    type: com.zaxxer.hikari.HikariDataSource
    driver-class-name: com.mysql.cj.jdbc.Driver
    hikari:
      #连接池名
      pool-name: DateHikariCP
      #最小空闲连接数
      minimum-idle: 10
      # 空闲连接存活最大时间，默认600000（10分钟）
      idle-timeout: 180000
      # 连接池最大连接数，默认是10
      maximum-pool-size: 100
      # 此属性控制从池返回的连接的默认自动提交行为,默认值：true
      auto-commit: true
      # 此属性控制池中连接的最长生命周期，值0表示无限生命周期，默认1800000即30分钟
      max-lifetime: 1800000
      # 数据库连接超时时间,默认30秒，即30000
      connection-timeout: 30000
      connection-test-query: SELECT 1

```

 
## 3. 性能优化

尽管从表面上看，问题是由于连接池数量太少，导致连接请求阻塞。但深层的原因是服务对数据库的请求处理过慢，最后导致阻塞。如果请求数量继续增加，即使扩大了连接池，同样会阻塞连接。这就像滴滴打车，碰到下雨天儿，队一旦开始排，后面就不知道要排多久了。

这个数据库存储了大量的数据，其中聊天记录的存储主要占用了性能。我们在处理聊天记录时做了分表处理。但由于数据量过大，单表依然有近两千万的数据。这张大表有一个联合索引，索引数据量较大。每次更新都需要维护索引空间，每次单个玩家数据量到达限值，就会进行局部清理。

这里的数据插入动作可能消耗时间较长。由于对消息的可靠性要求不高，我们可以使用异步进行，这样在等待插入的过程中可以省去大量的请求连接占用资源。

我们优化了消息保存数量。以前，每个玩家保存900条消息，但一般只查询最近的300条。现在，每500条进行一次清理，清理至300条，以节省空间。

即使数据量节省了很多，但由于业务价值相对成本比例因素，与业务部门进行沟通，将业务的容忍度定为定期3个月。

## 4. 空间优化

查询表空间占用：

````sql
-- 查询库中每个表的空间占用，分项列出 
select   table_schema as '数据库',   table_name as '表名',   table_rows as '记录数',   truncate(data_length/1024/1024, 2) as '数据容量(MB)',   truncate(index_length/1024/1024, 2) as '索引容量(MB)'   
from information_schema.tables   
where table_schema='表名'   
order by data_length desc, index_length desc;
````

对经常有删除操作的数据表进行碎片清理：

````sql
alter table 表名 engine=innodb;
````

经过清理，可以看到表空间占用缩小了40%左右。加上之前的业务修改，数据量又有了明显的缩减，使得数据库到了MySQL的舒适区，单表在500万左右。

## 5. 相关经验

以前我们有一个Go服务有非常大的IO，偶尔会出现崩溃，日志也是提示:"write tcp IP: xxx-> IP:xxx write: broken pipe"。开始以为是服务器在上传到OSS的过程中出现的连接异常，后来和阿里确认了并非OSS的断开错误。经过多次排查，最后发现在上传文件前，对内容进行了json序列化，这个过程非常费性能。当请求过多时，就发生了阻塞，阻塞过多，内存占用过大，溢出，服务就会拒绝服务。此时，连接的管道就会强行断开。

在很多业务场景中，都会出现这种情况：当计算资源不足时，请求就会阻塞堆积，最后最先崩溃的总是内存。