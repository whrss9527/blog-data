---
title: "gorm 中 MySQL 错误码映射与主键冲突错误处理"
status: 1
created_at: 2024-03-27T11:43:05+08:00
updated_at: 2026-04-30T15:03:32+08:00
category_id: 1
is_top: 0
tag_ids: [61, 62]
description: "在使用 gorm 处理数据库操作时，尤其是针对 MySQL，有时我们会遇到 golang 标准库`errors.Is`函数无法直接识别特定的 gorm 错误类型的情况，如主键冲突错误。尽管 gorm 提供了`gorm.ErrDuplicatedKey`来表示此类错误，但在原始错误返回中并不能直接通过`errors.Is(err, gorm.ErrDuplicatedKey)`来进行判断。本文深入探究 gorm.io/driver/mysql 包中的错误转换机制，揭示了如何借助`error_translator`模块将 MySQL 的错误码映射为 gorm 的错误类型。
"
word_count: 1565
---



处理 `gorm` 错误返回时，有一些错误是没有办法直接使用 `errors.Is` 来进行判断的，比如主键冲突的错误，直接使用 `errors.Is(err, gorm.ErrDuplicatedKey)` 是无法判断出主键冲突的错误返回的。

如果没有办法进行判断，为什么 `gorm` 要给这样一个 `error` ，但又不能使用呢？

`gorm.io/driver/mysql` 包中有一个 `error_translator` 的 go 文件

```go
package mysql

import (
    "github.com/go-sql-driver/mysql"

    "gorm.io/gorm")

// The error codes to map mysql errors to gorm errors, here is the mysql error codes reference https://dev.mysql.com/doc/mysql-errors/8.0/en/server-error-reference.html.
var errCodes = map[uint16]error{
    1062: gorm.ErrDuplicatedKey,
    1451: gorm.ErrForeignKeyViolated,
    1452: gorm.ErrForeignKeyViolated,
}

func (dialector Dialector) Translate(err error) error {
    if mysqlErr, ok := err.(*mysql.MySQLError); ok {
       if translatedErr, found := errCodes[mysqlErr.Number]; found {
          return translatedErr
       }
       return mysqlErr
    }

    return err
}
```

我们可以看到这个文件将 mysql 的几种错误码进行了枚举，使用 `Translate` 函数会将对应的 `mysql error` 转化为 `gorm error`

那这里的 `Translate` 函数，是谁进行使用了呢？在什么时候进行使用了呢？

主键冲突的错误一定是出现在插入的时候，我们顺着 `gorm` 的 `Create` 方法向下找，可以发现它调用了一个 `AddError` 的函数，如下

```go
// AddError add error to dbfunc (db *DB) AddError(err error) error {
    if err != nil {
       if db.Config.TranslateError {
          if errTranslator, ok := db.Dialector.(ErrorTranslator); ok {
             err = errTranslator.Translate(err)
          }
       }

       if db.Error == nil {
          db.Error = err
       } else {
          db.Error = fmt.Errorf("%v; %w", db.Error, err)
       }
    }
    return db.Error
}
```

这里有一行很关键，`db.Dialector.(ErrorTranslator)` 对 `Dialector` 接口进行了断言，断言成功，就调用对应 `Dialector` 的 `Translate` 函数，而当这里的 `Dialector` 是上面 `gorm.io/driver/mysql` 中的 `Dialector` 时，就可以运行上面的翻译逻辑，将 mysql 的 error 转换为 gorm 的 error。

那么，我们下一步就是找到方法，把这里使用的 `Dialector` 替换成 `gorm.io/driver/mysql` 包下的这个 `Dialector`，这样我们就可以使用 `errors.Is(err, gorm.ErrDuplicatedKey)` 对插入冲突进行判断了。

gorm 中 DB 对象的结构是这样的

```go
// DB GORM DB definition
type DB struct {
    *Config
    Error        error
    RowsAffected int64
    Statement    *Statement
    clone        int
}
```

这里的 Config 中就包含了 `Dialector` 接口，我们只需要在创建 `gorm.DB` 的时候，将接口的实例（`gorm.io/driver/mysql` 包下的这个）传入进去，就可以让 `gorm` 在之后的 error 判断时，对 mysql 的 error 进行翻译。

到这儿，原理部分就明明白白了，接下来简单改写一下 `gorm.DB` 的 init 过程即可！⬇️

```go
import "gorm.io/driver/mysql"
import "gorm.io/gorm"

connUrl := "数据库连接地址"
db, err := gorm.Open(
	mysql.Open(connUrl).(*mysql.Dialector),
	TranslateError: true, // 开启 mysql 方言翻译 (开启后 duplicatedKey err 判断才能生效)
)
```

以上！

参考链接：https://gorm.io/docs/error_handling.html#Dialect-Translated-Errors
