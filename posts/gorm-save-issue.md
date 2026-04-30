---
title: "【Gorm】Save 方法更新踩坑记录"
status: 1
created_at: 2023-06-03T16:53:55+08:00
updated_at: 2026-04-30T14:18:04+08:00
category_id: 1
is_top: 0
tag_ids: [61, 62]
description: "在我最近使用Gorm进行字段更新的过程中，我遇到了一个问题。当我尝试更新status字段时，即使该字段的值没有发生变化，Gorm还是提示我“Duplicate entry 'xxxx' for key 'PRIMARY'”。"
word_count: 2476
---

在我最近使用Gorm进行字段更新的过程中，我遇到了一个问题。当我尝试更新status字段时，即使该字段的值没有发生变化，Gorm还是提示我“Duplicate entry 'xxxx' for key 'PRIMARY'”。

首先，让我们看看Gorm的官方文档对`Save`方法的描述：

`Save`方法会保存所有的字段，即使字段是零值。

```go
db.First(&user)  

user.Name = "jinzhu 2"  
user.Age = 100  
db.Save(&user)  
// UPDATE users SET name='jinzhu 2', age=100, birthday='2016-01-01', updated_at = '2013-11-17 21:34:10' WHERE id=111;  

```  

`Save`方法是一个复合函数。如果保存的数据不包含主键，它将执行`Create`。反之，如果保存的数据包含主键，它将执行`Update`（带有所有字段）。

```go
db.Save(&User{Name: "jinzhu", Age: 100})  
// INSERT INTO `users` (`name`,`age`,`birthday`,`update_at`) VALUES ("jinzhu",100,"0000-00-00 00:00:00","0000-00-00 00:00:00")  

db.Save(&User{ID: 1, Name: "jinzhu", Age: 100})  
// UPDATE `users` SET `name`="jinzhu",`age`=100,`birthday`="0000-00-00 00:00:00",`update_at`="0000-00-00 00:00:00" WHERE `id` = 1  

```



根据这个描述，我预期的行为应该是更新操作，因为我提供了ID字段。然而，实际发生的却是插入操作。这让我感到困惑。

为了理解这个问题，我深入阅读了Gorm的源码：

```go
// Save updates value in database. If value doesn't contain a matching primary key, value is inserted.func (db *DB) Save(value interface{}) (tx *DB) {  
tx = db.getInstance()  
tx.Statement.Dest = value  
  
reflectValue := reflect.Indirect(reflect.ValueOf(value))  
for reflectValue.Kind() == reflect.Ptr || reflectValue.Kind() == reflect.Interface {  
reflectValue = reflect.Indirect(reflectValue)  
}  
  
switch reflectValue.Kind() {  
case reflect.Slice, reflect.Array:  
if _, ok := tx.Statement.Clauses["ON CONFLICT"]; !ok {  
tx = tx.Clauses(clause.OnConflict{UpdateAll: true})  
}  
tx = tx.callbacks.Create().Execute(tx.Set("gorm:update_track_time", true))  
case reflect.Struct:  
if err := tx.Statement.Parse(value); err == nil && tx.Statement.Schema != nil {  
for _, pf := range tx.Statement.Schema.PrimaryFields {  
if _, isZero := pf.ValueOf(tx.Statement.Context, reflectValue); isZero {  
return tx.callbacks.Create().Execute(tx)  
}  
}  
}  
  
fallthrough  
default:  
selectedUpdate := len(tx.Statement.Selects) != 0  
// when updating, use all fields including those zero-value fields  
if !selectedUpdate {  
tx.Statement.Selects = append(tx.Statement.Selects, "*")  
}  
  
updateTx := tx.callbacks.Update().Execute(tx.Session(&Session{Initialized: true}))  
  
if updateTx.Error == nil && updateTx.RowsAffected == 0 && !updateTx.DryRun && !selectedUpdate {  
return tx.Create(value)  
}  
  
return updateTx  
}  
  
return  
}
```

源码的主要逻辑如下：

1.  获取数据库实例并准备执行SQL语句。`value`是要操作的数据。
2.  利用反射机制确定`value`的类型。
3.  如果`value`是Slice或Array，并且没有定义冲突解决策略（"ON CONFLICT"），那么设置更新所有冲突字段的冲突解决策略，并执行插入操作。
4.  如果`value`是一个Struct，那么会尝试解析这个结构体，然后遍历它的主键字段。如果主键字段是零值，则执行插入操作。
5.  对于除Slice、Array、Struct以外的类型，将尝试执行更新操作。如果在更新操作后，没有任何行受到影响，并且没有选择特定的字段进行更新，则执行插入操作。

从这个函数我们可以看出，当传入的`value`对应的数据库记录不存在时（根据主键判断），Gorm会尝试创建一个新的记录。如果更新操作不影响任何行，Gorm同样会尝试创建一个新的记录。

这个行为与我们通常理解的“upsert”（update + insert）逻辑略有不同。在这种情况下，即使更新的数据与数据库中的数据完全相同，Gorm还是会尝试进行插入操作。这就是为什么我会看到`Duplicate entry 'xxxx' for key 'PRIMARY'`的错误，因为这就是主键冲突的错误提示。

对于Gorm的这种行为我感到困惑，同时我也对官方文档的描述感到失望，因为它并没有提供这部分的信息。

如何解决这个问题呢？

我们可以自己实现一个Save方法，利用GORM的Create方法和冲突解决策略：

```go
// Update all columns to new value on conflict except primary keys and those columns having default values from sql func  
db.Clauses(clause.OnConflict{  
  UpdateAll: true,  
}).Create(&users)  
// INSERT INTO "users" *** ON CONFLICT ("id") DO UPDATE SET "name"="excluded"."name", "age"="excluded"."age", ...;  
// INSERT INTO `users` *** ON DUPLICATE KEY UPDATE `name`=VALUES(name),`age`=VALUES(age), ...; MySQL

```

在Gorm的Create方法的文档中，我们可以看到这种用法。如果提供了ID，它会更新其他所有的字段。如果没有提供ID，它会插入新的记录。

---
经热心老哥提醒，当使用 `INSERT ... ON DUPLICATE KEY UPDATE` 语句时，如果插入的行因为唯一索引或主键冲突而失败，MySQL 会执行更新操作。这种情况下，会涉及到锁的问题，如果多个事务同时试图对同一行进行 `INSERT ... ON DUPLICATE KEY UPDATE` 操作，可能会引起死锁，尤其是在复杂的查询和多表操作中。死锁发生时，MySQL 会自动检测并回滚其中一个事务来解决死锁。频繁使用这种语句在高并发场景下可能会对性能造成影响，因为每次冲突都需要进行额外的更新操作，并且涉及到锁的管理。

所以，如果你的业务涉及复杂和并发的业务场景，可以尝试手动在应用层进行检测冲突，避免数据库的冲突和锁的竞争。