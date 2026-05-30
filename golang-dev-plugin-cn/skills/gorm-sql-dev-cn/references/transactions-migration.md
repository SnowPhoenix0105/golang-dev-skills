# 事务与迁移完整参考

## 目录

- [事务](#事务)
  - [闭包事务](#闭包事务)
  - [手动事务](#手动事务)
  - [嵌套事务与 SavePoint](#嵌套事务与-savepoint)
  - [事务配置](#事务配置)
- [迁移](#迁移)
  - [AutoMigrate](#automigrate)
  - [Migrator 接口](#migrator-接口)
  - [表操作](#表操作)
  - [列操作](#列操作)
  - [索引操作](#索引操作)
  - [约束操作](#约束操作)
  - [视图操作](#视图操作)

## 事务

### 闭包事务

推荐写法，自动处理 Commit/Rollback：

```go
err := db.Transaction(func(tx *gorm.DB) error {
    // 所有使用 tx 的操作都在事务中

    if err := tx.Create(&user).Error; err != nil {
        return err // 返回任何 error 都会触发 Rollback
    }

    if err := tx.Create(&order).Error; err != nil {
        return err
    }

    // 返回 nil 则自动 Commit
    return nil
})

if err != nil {
    // 事务失败
}
```

**panic 安全**：闭包内 panic 会自动 Rollback。

### 手动事务

```go
tx := db.Begin()

if tx.Error != nil {
    return tx.Error
}

if err := tx.Create(&user).Error; err != nil {
    tx.Rollback()
    return err
}

if err := tx.Create(&order).Error; err != nil {
    tx.Rollback()
    return err
}

tx.Commit()
```

### 事务选项

```go
// 设置隔离级别
tx := db.Begin(&sql.TxOptions{
    Isolation: sql.LevelSerializable,
    ReadOnly:  true,
})

// 闭包事务 + 选项
err := db.Transaction(func(tx *gorm.DB) error {
    // ...
}, &sql.TxOptions{Isolation: sql.LevelReadCommitted})
```

### 嵌套事务与 SavePoint

```go
// 内层事务使用 SavePoint，而非真正的嵌套事务
db.Transaction(func(tx *gorm.DB) error {
    tx.Create(&user)

    // 嵌套事务
    tx.Transaction(func(tx2 *gorm.DB) error {
        tx2.Create(&order)
        return nil // SavePoint 释放（commit）
    })

    // 内层回滚不影响外层
    tx.Transaction(func(tx2 *gorm.DB) error {
        tx2.Create(&badOrder)
        return errors.New("rollback this savepoint only")
    })

    return nil // 外层 commit
})
```

**禁用嵌套事务**：

```go
db, _ := gorm.Open(mysql.Open(dsn), &gorm.Config{
    DisableNestedTransaction: true,
})
// 或 Session 级别
db.Session(&gorm.Session{DisableNestedTransaction: true})
```

禁用后，嵌套的 `Transaction` 调用会直接复用同一个事务，不再创建 SavePoint。

### SavePoint 手动操作

```go
tx := db.Begin()
tx.Create(&user)
tx.SavePoint("sp1")    // 设置保存点
tx.Create(&order)
tx.RollbackTo("sp1")   // 回滚到保存点（order 被撤销）
tx.Commit()            // user 被提交
```

### 事务配置

```go
db, _ := gorm.Open(mysql.Open(dsn), &gorm.Config{
    SkipDefaultTransaction:    false, // 默认 true 表示单条 Create/Update/Delete 自动包在事务中
    DefaultTransactionTimeout: 10 * time.Second, // 事务默认超时
})
```

### 事务中的上下文

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
    // tx 中的操作受 context 超时控制
    return tx.Create(&user).Error
})
```

## 迁移

### AutoMigrate

`AutoMigrate` 自动化地迁移 schema：创建不存在的表、添加缺失的列和索引。**不会**删除现有列或修改列类型。

```go
// 自动迁移
db.AutoMigrate(&User{}, &Order{}, &Product{})

// 一次迁移多个模型
db.AutoMigrate(&User{}, &Order{})

// AutoMigrate 是 Migrator 的快捷方式
db.Migrator().AutoMigrate(&User{})
```

**AutoMigrate 的限制**：
- 不会删除未使用的列
- 不会修改列类型
- 默认不创建外键约束（可通过 config 启用）

### Migrator 接口

完整的 Migrator 接口可通过 `db.Migrator()` 获取。所有方法均由具体数据库驱动实现。

### 表操作

```go
// 建表
db.Migrator().CreateTable(&User{})

// 删表
db.Migrator().DropTable(&User{})
db.Migrator().DropTable("users")

// 检查表是否存在
db.Migrator().HasTable(&User{})    // true/false
db.Migrator().HasTable("users")    // true/false

// 重命名
db.Migrator().RenameTable("users", "t_users")

// 获取所有表名
tableList, _ := db.Migrator().GetTables()
```

### 列操作

```go
// 添加列
db.Migrator().AddColumn(&User{}, "Phone")

// 删除列
db.Migrator().DropColumn(&User{}, "Phone")

// 修改列
db.Migrator().AlterColumn(&User{}, "Phone")

// 重命名列
db.Migrator().RenameColumn(&User{}, "Phone", "Tel")

// 检查列是否存在
db.Migrator().HasColumn(&User{}, "Phone")

// 获取列类型
columns, _ := db.Migrator().ColumnTypes(&User{})
for _, col := range columns {
    name := col.Name()
    dbType := col.DatabaseTypeName()
    nullable, _ := col.Nullable()
}
```

### 索引操作

```go
// 创建索引
db.Migrator().CreateIndex(&User{}, "Name")
db.Migrator().CreateIndex(&User{}, "idx_name_age")

// 删除索引
db.Migrator().DropIndex(&User{}, "Name")
db.Migrator().DropIndex(&User{}, "idx_name_age")

// 检查索引
db.Migrator().HasIndex(&User{}, "Name")

// 重命名索引
db.Migrator().RenameIndex(&User{}, "old_name", "new_name")

// 获取所有索引
indexes, _ := db.Migrator().GetIndexes(&User{})
```

### 约束操作

```go
// 创建约束
db.Migrator().CreateConstraint(&User{}, "chk_user_age")

// 删除约束
db.Migrator().DropConstraint(&User{}, "chk_user_age")

// 检查约束
db.Migrator().HasConstraint(&User{}, "chk_user_age")
```

### 视图操作

```go
// 创建视图
db.Migrator().CreateView("active_users", gorm.ViewOption{
    Query: db.Model(&User{}).Where("active = ?", true),
})

// 替换视图
db.Migrator().CreateView("active_users", gorm.ViewOption{
    Replace: true,
    Query:   db.Model(&User{}).Where("active = ? AND age > ?", true, 18),
})

// 删除视图
db.Migrator().DropView("active_users")
```

### 自定义迁移逻辑

对于 AutoMigrate 无法处理的复杂变更，可以在迁移前后编写自定义逻辑：

```go
// 使用 Migrator() 配合迁移
m := db.Migrator()

// 检测是否需要迁移
if m.HasTable(&User{}) {
    // 已有表，执行增量迁移
    if !m.HasColumn(&User{}, "Phone") {
        m.AddColumn(&User{}, "Phone")
    }
} else {
    // 新建表
    m.CreateTable(&User{})
}

// 数据迁移
if m.HasColumn(&User{}, "OldField") {
    db.Exec("UPDATE users SET new_field = old_field")
    m.DropColumn(&User{}, "OldField")
}
```

### 外键约束配置

```go
// 启用外键约束创建
db, _ := gorm.Open(mysql.Open(dsn), &gorm.Config{
    DisableForeignKeyConstraintWhenMigrating: false,
})

// 忽略关联关系（不创建关联表的外键列）
db, _ := gorm.Open(mysql.Open(dsn), &gorm.Config{
    IgnoreRelationshipsWhenMigrating: true,
})
```
