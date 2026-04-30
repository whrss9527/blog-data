---
title: "Go 单元测试高效实践"
status: 1
created_at: 2023-04-21T18:27:10+08:00
updated_at: 2026-04-30T14:18:04+08:00
category_id: 1
is_top: 0
tag_ids: [56, 57, 58, 59]
description: "敏捷开发中有一个广为人知的开发方法就是 XP（极限编程），XP 提倡测试先行，为了将以后出现 bug 的几率降到最低，这一点与近些年流行的 TDD（测试驱动开发）有异曲同工之处。

在最开始做编程时，我总是忽略单元测试在代码中的作用，觉得编写单元测试的功夫都赶上甚至超越业务程序了。到后来，业务量越来越复杂，慢慢地，浮现一个问题，就是系统对于测试人员是一个黑盒，简单的测试无法保证系统所设计的东西都可以测试到。"
word_count: 3811
---

敏捷开发中有一个广为人知的开发方法就是 XP（极限编程），XP 提倡测试先行，为了将以后出现 bug 的几率降到最低，这一点与近些年流行的 TDD（测试驱动开发）有异曲同工之处。

在最开始做编程时，我总是忽略单元测试在代码中的作用，觉得编写单元测试的功夫都赶上甚至超越业务程序了。到后来，业务量越来越复杂，慢慢地，浮现一个问题，就是系统对于测试人员是一个黑盒，简单的测试无法保证系统所设计的东西都可以测试到⬇️
	举两个最简单的例子：
	系统设计的数据打点，是无法从功能业务上测试出来的，而对于测试人员，可能由于版本差异，用例未覆盖。
	如果一个表中有两个字段，新用户过来更新一个字段之后，测另一个字段的功能时就不再以一个新用户的身份操作了。
	在这样的情况下，如果开发人员没有对系统做完全的检查，就很可能出现问题。
就以上情况看，需要从开发人员的维度，对功能做一个“预期”测试，一个功能走过，应该输入什么，输出什么，哪些数据变动了，变动是否符合预期等等。

最近，公司业务基本都转入了 Go 做开发，在 Go 的整个业务处理上也日渐完善，而 Go 的单元测试用起来也十分顺手，所以做个小的总结。

## 一. Mock DB

在单元测试中，很重要的一项就是数据库的 Mock，数据库要在每次单元测试时作为一个干净的初始状态，并且每次运行速度不能太慢。

#### 1. Mysql 的 Mock

这里使用到的是 github.com/dolthub/go-mysql-server 借鉴了这位大哥的方法 [如何针对 MySQL 进行 Fake 测试](https://juejin.cn/post/7131661977310461965)

- ###### DB 的初始化
  在 db 目录下

```go
type Config struct {
   DSN             string // write data source name.
   MaxOpenConn     int    // open pool
   MaxIdleConn     int    // idle pool
   ConnMaxLifeTime int
}

var DB *gorm.DB

// InitDbConfig 初始化Db
func InitDbConfig(c *conf.Data) {
   log.Info("Initializing Mysql")
   var err error
   dsn := c.Database.Dsn
   maxIdleConns := c.Database.MaxIdleConn
   maxOpenConns := c.Database.MaxOpenConn
   connMaxLifetime := c.Database.ConnMaxLifeTime
   if DB, err = gorm.Open(mysql.Open(dsn), &gorm.Config{
      QueryFields: true,
      NamingStrategy: schema.NamingStrategy{
         //TablePrefix:   "",   // 表名前缀
         SingularTable: true, // 使用单数表名
      },
   }); err != nil {
      panic(fmt.Errorf("初始化数据库失败: %s \n", err))
   }
   sqlDB, err := DB.DB()
   if sqlDB != nil {
      sqlDB.SetMaxIdleConns(int(maxIdleConns))                               // 空闲连接数
      sqlDB.SetMaxOpenConns(int(maxOpenConns))                               // 最大连接数
      sqlDB.SetConnMaxLifetime(time.Second * time.Duration(connMaxLifetime)) // 单位：秒
   }
   log.Info("Mysql: initialization completed")
}
```

- ###### fake-mysql 的初始化和注入
  在 fake_mysql 目录下

```go
var (
   dbName    = "mydb"
   tableName = "mytable"
   address   = "localhost"
   port      = 3380
)

func InitFakeDb() {
   go func() {
      Start()
   }()
   db.InitDbConfig(&conf.Data{
      Database: &conf.Data_Database{
         Dsn:             "no_user:@tcp(localhost:3380)/mydb?timeout=2s&readTimeout=5s&writeTimeout=5s&parseTime=true&loc=Local&charset=utf8,utf8mb4",
         ShowLog:         true,
         MaxIdleConn:     10,
         MaxOpenConn:     60,
         ConnMaxLifeTime: 4000,
      },
   })
   migrateTable()
}

func Start() {
   engine := sqle.NewDefault(
      memory.NewMemoryDBProvider(
         createTestDatabase(),
         information_schema.NewInformationSchemaDatabase(),
      ))

   config := server.Config{
      Protocol: "tcp",
      Address:  fmt.Sprintf("%s:%d", address, port),
   }

   s, err := server.NewDefaultServer(config, engine)
   if err != nil {
      panic(err)
   }

   if err = s.Start(); err != nil {
      panic(err)
   }

}

func createTestDatabase() *memory.Database {
   db := memory.NewDatabase(dbName)
   db.EnablePrimaryKeyIndexes()
   return db
}

func migrateTable() {
// 生成一个user表到fake mysql中
   err := db.DB.AutoMigrate(&model.User{})
   if err != nil {
      panic(err)
   }
}
```

在单元测试开始，调用  ```InitFakeDb()``` 即可

```go
func setup() {
   fake_mysql.InitFakeDb()
}
```
---
#### 2. Redis 的 Mock
这里用到的是 [miniredis](https://github.com/alicebob/miniredis) , 与之配套的Redis Client 是  ```go-redis/redis/v8``` ，在这里调用 InitTestRedis() 注入即可

```go
// RedisClient redis 客户端  
var RedisClient *redis.Client  
  
// ErrRedisNotFound not exist in redisconst ErrRedisNotFound = redis.Nil  
  
// Config redis config
type Config struct {  
   Addr         string  
   Password     string  
   DB           int  
   MinIdleConn  int  
   DialTimeout  time.Duration  
   ReadTimeout  time.Duration  
   WriteTimeout time.Duration  
   PoolSize     int  
   PoolTimeout  time.Duration  
   // tracing switch  
   EnableTrace bool  
}  
  
// Init 实例化一个redis client  
func Init(c *conf.Data) *redis.Client {  
   RedisClient = redis.NewClient(&redis.Options{  
      Addr:         c.Redis.Addr,  
      Password:     c.Redis.Password,  
      DB:           int(c.Redis.DB),  
      MinIdleConns: int(c.Redis.MinIdleConn),  
      DialTimeout:  c.Redis.DialTimeout.AsDuration(),  
      ReadTimeout:  c.Redis.ReadTimeout.AsDuration(),  
      WriteTimeout: c.Redis.WriteTimeout.AsDuration(),  
      PoolSize:     int(c.Redis.PoolSize),  
      PoolTimeout:  c.Redis.PoolTimeout.AsDuration(),  
   })  
  
   _, err := RedisClient.Ping(context.Background()).Result()  
   if err != nil {  
      panic(err)  
   }  
  
   // hook tracing (using open telemetry)  
   if c.Redis.IsTrace {  
      RedisClient.AddHook(redisotel.NewTracingHook())  
   }  
  
   return RedisClient  
}  
  
// InitTestRedis 实例化一个可以用于单元测试的redis  
func InitTestRedis() {  
   mr, err := miniredis.Run()  
   if err != nil {  
      panic(err)  
   }  
   // 打开下面命令可以测试链接关闭的情况  
   // defer mr.Close()  
  
   RedisClient = redis.NewClient(&redis.Options{  
      Addr: mr.Addr(),  
   })  
   fmt.Println("mini redis addr:", mr.Addr())  
}
```

## 二. 单元测试

经过对比，我选择了 [goconvey](https://github.com/smartystreets/goconvey/wiki/Documentation) 这个单元测试框架
它比原生的go testing 好用很多。goconvey还提供了很多好用的功能：

-   多层级嵌套单测
-   丰富的断言
-   清晰的单测结果
-   支持原生go test

使用
```cmd
go get github.com/smartystreets/goconvey
```


```go
func TestLoverUsecase_DailyVisit(t *testing.T) {  
   Convey("Test TestLoverUsecase_DailyVisit", t, func() {  
      // clean  
      uc := NewLoverUsecase(log.DefaultLogger, &UsecaseManager{})  
  
      Convey("ok", func() {  
         // execute  
         res1, err1 := uc.DailyVisit("user1", 3)  
         So(err1, ShouldBeNil)  
         So(res1, ShouldNotBeNil)  
         // 第 n (>=2)次拜访，不应该有奖励，也不应该报错  
         res2, err2 := uc.DailyVisit("user1", 3)  
         So(err2, ShouldBeNil)  
         So(res2, ShouldBeNil)  
      })  
   })  
}
```
	可以看到，函数签名和 go 原生的 test 是一致的
	测试中嵌套了两层 Convey，外层new了内层Convey所需的参数 
	内层调用了函数，对返回值进行了断言

这里的断言也可以像这样对返回值进行比较 `So(x, ShouldEqual, 2)`
或者判断长度等等 `So(len(resMap),ShouldEqual, 2)`

Convey的嵌套也可以灵活多层，可以像一棵多叉树一样扩展，足够满足业务模拟。

---

## 三. TestMain
为所有的 case 加上一个 TestMain 作为统一入口

```go
import (  
"os"  
"testing"  
  
. "github.com/smartystreets/goconvey/convey"  
)  
  
func TestMain(m *testing.M) {  
   setup()  
   code := m.Run()  
   teardown()  
   os.Exit(code)
}
// 初始化fake db
func setup() {  
   fake_mysql.InitFakeDb()  
   redis.InitTestRedis()
}
```