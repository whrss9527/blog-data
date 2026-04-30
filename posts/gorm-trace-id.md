---
id: 2ebd3586387c4d4e866d94cf1ff2a7b1
title: "GORM 中 SQL、慢 SQL 打印日志传递 trace ID"
status: 1
created_at: 2023-09-25T11:14:27+08:00
updated_at: 2026-04-30T14:45:27+08:00
category_id: 1
is_top: 0
tag_ids: [61, 62, 81, 78]
description: "GORM 中SQL、慢SQL打印日志传递 trace ID， 重写GORM日志实现，SQL日志、慢日志请求追踪
"
word_count: 1298
identity: gorm-trace-id
---

实现 `gorm.io/gorm/logger` 下的函数⬇️
```go
// gorm 源码
type Interface interface {  
   LogMode(LogLevel) Interface  
   Info(context.Context, string, ...interface{})  
   Warn(context.Context, string, ...interface{})  
   Error(context.Context, string, ...interface{})  
   Trace(ctx context.Context, begin time.Time, fc func() (sql string, rowsAffected int64), err error)  
}
```

以下为自定义的重写实现，主要函数是`Trace`
```go
package log  
  
import (  
   "context"  
   "time"  
   "github.com/go-kratos/kratos/v2/log"
   "gorm.io/gorm/logger"
)  
  
type GormLogger struct {  
   SlowThreshold time.Duration  
}  
  
func NewGormLogger() *GormLogger {  
   return &GormLogger{  
      SlowThreshold: 200 * time.Millisecond, // 一般超过200毫秒就算慢查所以不使用配置进行更改  
   }  
}  
  
var _ logger.Interface = (*GormLogger)(nil)  
  
func (l *GormLogger) LogMode(lev logger.LogLevel) logger.Interface {  
   return &GormLogger{}  
}  
func (l *GormLogger) Info(ctx context.Context, msg string, data ...interface{}) {  
   log.Context(ctx).Infof(msg, data)  
}  
func (l *GormLogger) Warn(ctx context.Context, msg string, data ...interface{}) {  
   log.Context(ctx).Errorf(msg, data)  
}  
func (l *GormLogger) Error(ctx context.Context, msg string, data ...interface{}) {  
   log.Context(ctx).Errorf(msg, data)  
}  
func (l *GormLogger) Trace(ctx context.Context, begin time.Time, fc func() (sql string, rowsAffected int64), err error) {  
   // 获取运行时间  
   elapsed := time.Since(begin)  
   // 获取 SQL 语句和返回条数  
   sql, rows := fc()  
   // Gorm 错误时打印
   if err != nil {  
      log.Context(ctx).Errorf("SQL ERROR, | sql=%v, rows=%v, elapsed=%v", sql, rows, elapsed)  
   }  
   // 慢查询日志  
   if l.SlowThreshold != 0 && elapsed > l.SlowThreshold {  
      log.Context(ctx).Warn("Database Slow Log, | sql=%v, rows=%v, elapsed=%v", sql, rows, elapsed)  
   }  
  
}
```
> 这里我使用的log组件是kratos框架的log组件，设置好zap后注入为全局log


在`GORM`创建连接的地方注入我们重写后的自定义Logger
```go
db, err := gorm.Open(mysql.Open(cfg.Dsn()), &gorm.Config{  
   QueryFields: true,    
   Logger: logdef.NewGormLogger(),  // 注入
})
```

最后，在查询的地方，带上`withContext`即可

```go
func (ud *userDao) AddOne(c context.Context, user *model.User) (userId int64, err error) {  
   err = db.GetConn().WithContext(c).Create(user).Error  
   if err != nil {  
      log.Context(c).Errorf("AddOne fail|err=%+v", err)  
      return  
   }  
   return user.Id, nil  
}
```

