# GORM 最佳实践

本文汇总 GORM 使用中的核心模式与推荐做法。详细 API 签名见 `references/model-crud.md` 和 `references/advanced-queries.md`。

## 目录

- [N+1 查询预防](#n1-查询预防)
- [零值更新策略](#零值更新策略)
- [连接池配置](#连接池配置)
- [软删除设计](#软删除设计)
- [事务使用模式](#事务使用模式)
- [迁移策略](#迁移策略)
- [错误处理](#错误处理)
- [批量操作与性能](#批量操作与性能)
- [Logger 配置](#logger-配置)

---

## N+1 查询预防

N+1 是最常见的 GORM 性能陷阱。循环中逐条查询关联数据，产生指数级 SQL 调用。

### 使用 Preload（推荐）

```go
// ❌ N+1：每个 user 一次 Orders 查询
var users []User
db.Find(&users)
for _, u := range users {
    db.Where("user_id = ?", u.ID).Find(&u.Orders) // N 次额外查询
}

// ✅ Preload：一次查询所有关联
db.Preload("Orders").Find(&users)

// 条件 Preload
db.Preload("Orders", "status = ?", "active").Find(&users)

// 嵌套 Preload
db.Preload("Orders.Items").Find(&users)
```

### Preload vs Joins 选择

| 场景 | 推荐 |
| :--- | :--- |
| 加载完整关联对象 | `Preload` |
| 用关联字段做筛选，不需要返回关联数据 | `Joins` + `Select` |
| 关联数据量大（数千条） | `Joins` + 分页 |
| 多层嵌套关联 | `Preload` 链式 |

```go
// Preload：加载关联数据到内存
db.Preload("Orders").Find(&users)

// Joins：用关联字段筛选，不加载 Orders
db.Joins("JOIN orders ON orders.user_id = users.id").
    Where("orders.amount > ?", 100).Find(&users)
```

---

## 零值更新策略

`Updates(struct)` 默认跳过零值字段（`0`, `""`, `false`, `nil`）。需要更新零值时有三种策略：

### 方案一：用 map（简单场景）

```go
db.Model(&user).Updates(map[string]interface{}{"age": 0, "active": false})
```

### 方案二：用 Select 显式指定字段（推荐）

```go
// 指定字段
db.Model(&user).Select("Age", "Active").Updates(User{Age: 0, Active: false})

// 所有字段（包括零值）
db.Model(&user).Select("*").Updates(User{Age: 0})
```

### 方案三：模型中使用指针类型（大型项目）

```go
type User struct {
    Age    *int   // 用指针区分"零值"和"未设置"
    Active *bool
}

// nil = 不更新，非 nil = 更新为该值
db.Model(&user).Updates(User{Age: ptr(0), Active: ptr(false)})
```

### 策略选择

| 场景 | 推荐方案 |
| :--- | :--- |
| 偶尔需要更新零值 | map 或 Select |
| 业务上"未设置"和"零值"语义不同 | 指针类型 |
| 批量更新零值字段 | Select("*") |

---

## 连接池配置

默认连接池参数不一定适合生产环境，建议显式配置：

```go
sqlDB, err := db.DB()
sqlDB.SetMaxOpenConns(100)            // 最大连接数（默认无限制）
sqlDB.SetMaxIdleConns(10)             // 最大空闲连接数（默认 2）
sqlDB.SetConnMaxLifetime(time.Hour)   // 连接最大生命周期
sqlDB.SetConnMaxIdleTime(10 * time.Minute) // 空闲连接最大存活时间
```

### 推荐值

| 参数 | 开发环境 | 生产环境 | 说明 |
| :--- | :--- | :--- | :--- |
| MaxOpenConns | 10-25 | 50-100 | 取决于数据库最大连接数 |
| MaxIdleConns | 5-10 | 10-25 | 一般为 MaxOpenConns 的 25% |
| ConnMaxLifetime | 1h | 30min-1h | 小于数据库 wait_timeout |

> 建议用 `db.Session()` 创建独立 Session 来调整单次查询的连接参数，而非修改全局 DB 实例。

---

## 软删除设计

GORM 通过 `gorm.DeletedAt` 字段支持软删除。

### 基本模型

```go
type User struct {
    ID        uint
    Name      string
    DeletedAt gorm.DeletedAt // 自动启用软删除
}
```

### 关键行为

```go
// 删除：设置 DeletedAt 时间戳，不真正删除
db.Delete(&user)

// 查询：自动过滤已删除记录（WHERE deleted_at IS NULL）
db.Find(&users)

// 查询包含已删除的记录
db.Unscoped().Find(&users)

// 仅查已删除
db.Unscoped().Where("deleted_at IS NOT NULL").Find(&users)

// 真正删除
db.Unscoped().Delete(&user)
```

### 注意事项

- **唯一索引**：软删除记录仍占用索引，使用 `gorm.DeletedAt` 类型（非 `time.Time`），未删除时该字段为 NULL，MySQL 中 NULL 不参与唯一约束判定。
- **关联软删除**：关联记录的删除行为需手动处理或使用 Hook。
- **恢复**：`db.Unscoped().Model(&user).Update("deleted_at", nil)`。

---

## 事务使用模式

### 闭包事务（推荐）

```go
// 自动 commit / rollback，自动处理 panic
err := db.Transaction(func(tx *gorm.DB) error {
    if err := tx.Create(&user).Error; err != nil {
        return err // 自动回滚
    }
    if err := tx.Create(&order).Error; err != nil {
        return err // 自动回滚
    }
    return nil // 自动提交
})
```

### 手动事务（需要细粒度控制）

```go
tx := db.Begin()
defer func() {
    if r := recover(); r != nil {
        tx.Rollback()
    }
}()

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

### 嵌套事务

默认嵌套事务使用 SavePoint：

```go
db.Transaction(func(tx *gorm.DB) error {
    tx.Create(&user)
    tx.Transaction(func(tx2 *gorm.DB) error {
        tx2.Create(&order) // 内层
        return errors.New("rollback") // 仅回滚到 SavePoint
    })
    return nil // user 仍被提交！
})
```

如需禁用嵌套 SavePoint（整个事务一起回滚）：

```go
db, _ := gorm.Open(mysql.Open(dsn), &gorm.Config{
    DisableNestedTransaction: true,
})
```

---

## 迁移策略

### AutoMigrate 的局限性

`db.AutoMigrate(&User{})` **只能做**：创建不存在的表、添加新列、创建索引。

**不能做**：修改列类型、删除列、重命名列、创建外键约束（默认不创建）、修改约束。

### 渐进式迁移

```go
// 开发阶段：AutoMigrate
db.AutoMigrate(&User{}, &Order{})

// 生产变更：使用 Migrator 手动处理
m := db.Migrator()

// 添加列
if !m.HasColumn(&User{}, "Phone") {
    m.AddColumn(&User{}, "Phone")
}

// 修改列类型（需数据库支持）
m.AlterColumn(&User{}, "Phone")

// 创建索引
m.CreateIndex(&User{}, "idx_name")

// 重命名列
m.RenameColumn(&User{}, "OldName", "NewName")
```

### 外键约束

AutoMigrate 默认**不创建**外键约束。如需创建：

```go
db, _ := gorm.Open(mysql.Open(dsn), &gorm.Config{
    DisableForeignKeyConstraintWhenMigrating: false,
})
```

---

## 错误处理

### ErrRecordNotFound

`First`/`Last`/`Take` 找不到记录时返回 `gorm.ErrRecordNotFound`。`Find` 不返回此错误。

```go
// 区分"未找到"和"其他错误"
if errors.Is(err, gorm.ErrRecordNotFound) {
    // 记录不存在
}

// 不需要区分的场景直接用 Find
var users []User
db.Where("id = ?", 999).Find(&users)
if len(users) == 0 {
    // 无记录
}
```

### TranslateError

开启后 GORM 将数据库错误翻译为 `ErrDuplicatedKey`、`ErrForeignKeyViolated` 等：

```go
db, _ := gorm.Open(mysql.Open(dsn), &gorm.Config{
    TranslateError: true,
})

// 判断唯一约束冲突
if errors.Is(err, gorm.ErrDuplicatedKey) {
    // 处理重复键
}
```

---

## 批量操作与性能

### 批量插入

```go
// ✅ 批量插入（一条 SQL）
db.Create(&users) // users 为 []User

// ✅ 分批插入（大量数据）
db.CreateInBatches(users, 100) // 每批 100 条

// ❌ 循环单条插入
for _, u := range users {
    db.Create(&u) // N 条 SQL
}
```

### 大批量查询

```go
// ✅ 分批处理，避免 OOM
db.FindInBatches(&results, 100, func(tx *gorm.DB, batch int) error {
    for _, r := range results {
        // 每批最多 100 条
    }
    return nil
})
```

### PrepareStmt

PrepareStmt 缓存预编译语句，减少 SQL 解析开销。但对大量动态 SQL（条件多变）的场景，缓存频繁失效反而降低性能：

```go
// 全局开启
db, _ := gorm.Open(mysql.Open(dsn), &gorm.Config{
    PrepareStmt: true,
})

// 单次关闭
db.Session(&gorm.Session{PrepareStmt: false}).Find(&users)
```

### 跳过 Hook

批量操作时跳过不必要的 Hook 可显著提速：

```go
db.Session(&gorm.Session{SkipHooks: true}).CreateInBatches(users, 100)
```

---

## Logger 配置

### 开发环境

```go
newLogger := logger.New(
    log.New(os.Stdout, "\r\n", log.LstdFlags),
    logger.Config{
        SlowThreshold:             200 * time.Millisecond,
        LogLevel:                  logger.Info, // 打印所有 SQL
        IgnoreRecordNotFoundError: true,        // 不打印 ErrRecordNotFound
        ParameterizedQueries:      false,       // 显示参数值
        Colorful:                  true,
    },
)
```

### 生产环境

```go
newLogger := logger.New(
    log.New(os.Stdout, "\r\n", log.LstdFlags),
    logger.Config{
        SlowThreshold:             200 * time.Millisecond,
        LogLevel:                  logger.Warn,  // 只打印慢查询和错误
        IgnoreRecordNotFoundError: true,
        ParameterizedQueries:      true,         // 隐藏参数值
    },
)
```

### 开发调试技巧

```go
// 单次查询 Debug
db.Debug().Where("name = ?", "张三").Find(&users)
// 输出：[2024-01-01 12:00:00] [row:2] SELECT * FROM `users` WHERE name = '张三'

// 查看 SQL 不执行
stmt := db.Session(&gorm.Session{DryRun: true}).Find(&users).Statement
fmt.Println(stmt.SQL.String())
```
