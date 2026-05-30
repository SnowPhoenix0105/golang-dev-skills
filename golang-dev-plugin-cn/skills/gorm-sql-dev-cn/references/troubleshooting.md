# 排障指南

## 目录

- [查询问题](#查询问题)
- [写入问题](#写入问题)
- [关联问题](#关联问题)
- [迁移问题](#迁移问题)
- [事务问题](#事务问题)
- [性能问题](#性能问题)
- [连接问题](#连接问题)
- [错误信息速查](#错误信息速查)

## 查询问题

### `First`/`Last`/`Take` 返回 `ErrRecordNotFound`

这是正常行为，这三个方法找不到记录时返回此错误。如果不需要区分「未找到」和「空结果」：

```go
// 使用 Find 替代（不返回 ErrRecordNotFound）
var users []User
db.Where("id = ?", 999).Find(&users)
// len(users) == 0，无错误

// 或显式判断
if errors.Is(err, gorm.ErrRecordNotFound) {
    // 处理未找到的情况
}
```

### 软删除记录查不出来

```go
// 默认所有查询自动添加 WHERE deleted_at IS NULL
// 查询包含已删除的记录
db.Unscoped().Find(&users)

// 只查已删除的记录
db.Unscoped().Where("deleted_at IS NOT NULL").Find(&users)
```

### Preload 不生效

GORM **不会**自动加载关联。必须在查询链中显式调用 `Preload`：

```go
// ❌ 关联不会被加载
db.Find(&users)
fmt.Println(users[0].Orders) // 空

// ✅ 显式预加载
db.Preload("Orders").Find(&users)
```

### 查询结果与预期不符

开启 Debug 查看生成的 SQL：

```go
db.Debug().Where("name = ?", "张三").Find(&users)
// 输出：[2024-01-01 12:00:00] [2.00ms] SELECT * FROM `users` WHERE name = '张三'
```

### `Pluck` 结果类型不匹配

```go
// ❌ Pluck 的目标切片元素类型必须与列类型兼容
var names []int
db.Model(&User{}).Pluck("name", &names) // 错误：name 是 string

// ✅
var names []string
db.Model(&User{}).Pluck("name", &names)
```

## 写入问题

### 零值字段未被更新

`Updates(struct)` 默认跳过零值。解决方法：

```go
// 方案一：用 map
db.Model(&user).Updates(map[string]interface{}{"age": 0, "active": false})

// 方案二：用 Select 显式指定
db.Model(&user).Select("Age", "Active").Updates(User{Age: 0, Active: false})
db.Model(&user).Select("*").Updates(User{Age: 0}) // 所有字段

// 方案三：模型中用指针类型
type User struct {
    Age    *int
    Active *bool
}
```

### 批量更新/删除报错 `WHERE conditions required`

GORM 的安全机制，防止意外的全表更新/删除：

```go
// ❌ 无 WHERE 条件
db.Model(&User{}).Update("name", "张三") // ErrMissingWhereClause

// ✅ 添加条件
db.Model(&User{}).Where("id = ?", 1).Update("name", "张三")

// ✅ 允许全局更新
db.Session(&gorm.Session{AllowGlobalUpdate: true}).Model(&User{}).Update("name", "张三")
```

### `Save` 没有更新而是插入

`Save` 根据主键零值判断是更新还是插入。如果主键为零值会执行 INSERT：

```go
user := User{Name: "张三"} // ID 为零值
db.Save(&user)            // INSERT，不是 UPDATE

// ✅ 正确用法
user := User{ID: 1, Name: "李四"}
db.Save(&user)            // UPDATE（主键非零值）
```

### 创建后主键未被回填

确保传入 `Create` 的是指针：

```go
user := User{Name: "张三"}
db.Create(&user) // ✅ user.ID 被回填
// ❌ db.Create(user) — 不会回填
```

## 关联问题

### Many2Many 关联创建失败

确保关联表名符合约定。默认按两个模型名的字母序拼接：

```go
// 关联表默认：user_languages（User 字母序 < Language）
db.AutoMigrate(&User{}, &Language{})
```

手动指定关联表：

```go
type User struct {
    Languages []Language `gorm:"many2many:custom_table"`
}
```

### 外键约束创建失败

AutoMigrate 默认**不创建**外键约束。如需创建：

```go
db, _ := gorm.Open(mysql.Open(dsn), &gorm.Config{
    DisableForeignKeyConstraintWhenMigrating: false,
})
```

### `FullSaveAssociations` 导致意外的关联更新

`Create`/`Updates` 默认不处理关联。如果开启了 `FullSaveAssociations`，关联对象也会被创建/更新：

```go
// 可能意外创建/更新关联记录
db.Session(&gorm.Session{FullSaveAssociations: true}).Create(&user)
```

## 迁移问题

### AutoMigrate 不修改列类型

AutoMigrate 只添加新列，不修改已有列的类型、大小、约束。对于需要修改列的场景：

```go
// 使用 Migrator 手动迁移
if !db.Migrator().HasColumn(&User{}, "Phone") {
    db.Migrator().AddColumn(&User{}, "Phone")
} else {
    db.Migrator().AlterColumn(&User{}, "Phone") // 修改列类型
}
```

### AutoMigrate 没有创建索引

确保模型中定义了正确的 index tag：

```go
type User struct {
    ID   uint   `gorm:"primaryKey"`
    Name string `gorm:"index"`           // 普通索引
    Code string `gorm:"uniqueIndex"`     // 唯一索引
    Age  int    `gorm:"index:idx_name_age,priority:2"` // 组合索引
}
```

## 事务问题

### 嵌套事务回滚行为不符合预期

默认情况下嵌套事务使用 SavePoint。内层 `Rollback` 只回滚到 SavePoint，外层事务仍可提交：

```go
db.Transaction(func(tx *gorm.DB) error {
    tx.Create(&user) // 会被提交
    tx.Transaction(func(tx2 *gorm.DB) error {
        tx2.Create(&order)
        return errors.New("rollback") // 只回滚 order
    })
    return nil // user 仍被提交
})
```

如需禁用嵌套 SavePoint：

```go
db, _ := gorm.Open(mysql.Open(dsn), &gorm.Config{
    DisableNestedTransaction: true,
})
```

### 事务中 panic 后数据未回滚

闭包事务 (`db.Transaction`) 会自动处理 panic 并回滚。手动事务则不会：

```go
// ✅ 闭包事务自动处理 panic
db.Transaction(func(tx *gorm.DB) error {
    panic("oops") // 自动回滚
})

// ❌ 手动事务不自动处理
tx := db.Begin()
panic("oops") // 连接未释放，未回滚
// 用 defer + recover 处理
```

## 性能问题

### N+1 查询

循环中逐条查询：

```go
// ❌ N+1 问题
var users []User
db.Find(&users)
for _, u := range users {
    db.Where("user_id = ?", u.ID).Find(&u.Orders)
}

// ✅ 使用 Preload
db.Preload("Orders").Find(&users)
```

### 大量数据查询 OOM

使用 `FindInBatches` 分批处理：

```go
db.FindInBatches(&users, 100, func(tx *gorm.DB, batch int) error {
    // 每次处理最多 100 条
    return nil
})
```

### 慢查询排查

```go
// 开启慢查询日志
newLogger := logger.New(
    log.New(os.Stdout, "\r\n", log.LstdFlags),
    logger.Config{
        SlowThreshold: 200 * time.Millisecond,
        LogLevel:      logger.Warn,
    },
)
```

### PrepareStmt 缓存失效

`PrepareStmt` 在大量动态 SQL 时可能导致缓存频繁失效：

```go
// 频繁变动的条件禁用 PrepareStmt
db.Session(&gorm.Session{PrepareStmt: false}).Find(&users)
```

## 连接问题

### 连接池耗尽

```go
sqlDB, err := db.DB()
sqlDB.SetMaxOpenConns(100)     // 最大连接数
sqlDB.SetMaxIdleConns(10)      // 最大空闲连接数
sqlDB.SetConnMaxLifetime(time.Hour) // 连接最大生命周期
```

### Context 超时不生效

确保使用 `WithContext` 传递 context：

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

db.WithContext(ctx).Find(&users)
```

> GORM 的 `Session.Context` 不会自动传播到所有操作。每个新的 `Session` 或链式调用可能丢失 Context。

## 错误信息速查

| 错误 | 含义 | 常见原因 |
| :--- | :--- | :--- |
| `ErrRecordNotFound` | 记录未找到 | `First`/`Last`/`Take` 匹配不到记录 |
| `ErrMissingWhereClause` | 缺少 WHERE 条件 | `Update`/`Delete` 未设条件，未启用 `AllowGlobalUpdate` |
| `ErrInvalidTransaction` | 无效事务 | 对非事务连接调用 `Commit`/`Rollback` |
| `ErrPrimaryKeyRequired` | 需要主键 | `FindInBatches` 需要主键字段 |
| `ErrDuplicatedKey` | 重复键 | 违反唯一约束（需 `TranslateError: true`） |
| `ErrForeignKeyViolated` | 外键约束 | 违反外键约束（需 `TranslateError: true`） |
| `ErrInvalidValue` | 无效值 | `dest` 不是指针或非 struct/slice |
| `ErrEmptySlice` | 空切片 | `Create` 传入非空切片指针 |
| `ErrDryRunModeUnsupported` | DryRun 不支持 | `Row`/`Rows` 在 DryRun 模式下调用 |
| `ErrUnsupportedDriver` | 驱动不支持 | `SavePoint`/`RollbackTo` 驱动未实现 |
| `ErrUnsupportedRelation` | 不支持的关联 | `Association("xxx")` 中关联名不存在 |
| `ErrPreloadNotAllowed` | 不允许 Preload | `Count` 与 `Preload` 混用 |
| `ErrInvalidDB` | 无效数据库连接 | `db.DB()` 获取不到 `*sql.DB` |
