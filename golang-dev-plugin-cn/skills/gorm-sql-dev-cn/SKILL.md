---
name: gorm-sql-dev-cn
description: |
  GORM SQL 数据库 ORM 开发。当用户需要创建、修改、调试 Go 语言的数据库操作代码，使用 GORM 进行 CRUD、
  关联查询、事务、迁移、代码生成时使用此技能。
  触发场景：提到 gorm、数据库、MySQL、PostgreSQL、SQLite、CRUD、ORM、DAO、gen、gorm gen、
  gorm cli、关联查询、Preload、事务、AutoMigrate、gorm.Model、钩子/Hooks 等。
  即使用户没有明确说"gorm"，只要在做 Go 数据库开发就应该考虑此技能。
---

# GORM SQL 数据库开发

GORM（`gorm.io/gorm`）是 Go 语言最流行的全功能 ORM 库，提供声明式模型定义、链式查询 API、关联管理、自动迁移、事务、钩子系统等能力。

## GORM 生态系统

| 仓库 | 用途 | go module |
| :--- | :--- | :--- |
| **gorm** | 核心 ORM 引擎：模型定义、CRUD、关联、迁移、事务、钩子、Logger | `gorm.io/gorm` |
| **gen** | 代码生成工具：从数据库表自动生成 DAO 和 Model，提供类型安全的查询 API | `gorm.io/gen` |
| **cli** | 新一代代码生成工具：基于泛型，提供 `gorm` CLI 命令 | `gorm.io/cli` |

按数据库选择对应的驱动：
- **MySQL**：`gorm.io/driver/mysql`
- **PostgreSQL**：`gorm.io/driver/postgres`
- **SQLite**：`gorm.io/driver/sqlite`
- **SQL Server**：`gorm.io/driver/sqlserver`

本地源码位置：`go env GOMODCACHE` 输出缓存目录 + `gorm.io/gorm@<version>`。
如需查阅官方文档网站源码，仓库位于 `gorm.io`（Hugo 站点）。

## 核心原则

1. **模型驱动** — 用 Go struct 定义数据库表，通过 tag 定义字段约束和行为
2. **链式 API，不可变链** — `db.Where().Order().Limit()` 每次返回新 `*gorm.DB` 实例，安全复用
3. **零值有语义** — `Create`/`Updates` 默认忽略零值字段；用 `Select` 或指针字段覆盖此行为
4. **软删除优先** — 包含 `gorm.DeletedAt` 字段的模型，`Delete` 默认为软删除
5. **显式错误处理** — 查询/操作后检查 `result.Error` 或 `result.RowsAffected`
6. **`gorm.DB` 是连接池封装** — 不要手动关闭，按需通过 `Session()` 创建新会话

## 最小示例

```go
package main

import (
    "gorm.io/driver/sqlite"
    "gorm.io/gorm"
)

type User struct {
    gorm.Model
    Name string
    Age  int
}

func main() {
    db, err := gorm.Open(sqlite.Open("test.db"), &gorm.Config{})
    if err != nil {
        panic("failed to connect database")
    }

    // AutoMigrate
    db.AutoMigrate(&User{})

    // Create
    db.Create(&User{Name: "张三", Age: 30})

    // Read
    var user User
    db.First(&user, 1)                   // 按主键
    db.First(&user, "name = ?", "张三")  // 按条件

    // Update
    db.Model(&user).Update("Age", 31)

    // Delete（软删除）
    db.Delete(&user)
}
```

## API 分类速查

### 链式 API（Chainable）

构建查询条件，返回新 `*gorm.DB` 实例。可安全复用。

| 方法 | 用途 |
| :--- | :--- |
| `Model()` | 指定操作的模型 |
| `Table()` | 指定表名 |
| `Where()` / `Not()` / `Or()` | 添加条件 |
| `Select()` / `Omit()` | 指定/排除字段 |
| `Order()` / `Group()` / `Having()` | 排序、分组、过滤 |
| `Limit()` / `Offset()` | 分页 |
| `Joins()` / `InnerJoins()` | 连接查询 |
| `Preload()` | 预加载关联 |
| `Clauses()` | 直接添加 clause 表达式 |
| `Scopes()` | 复用查询逻辑 |
| `Unscoped()` | 跳过软删除过滤 |
| `Distinct()` | 去重 |
| `Attrs()` / `Assign()` | 配合 FirstOrCreate/FirstOrInit |
| `Debug()` | 开启 Debug 日志 |

完整 API 签名和用法见 `references/model-crud.md`。

### 终结 API（Finisher）

执行查询或操作，返回结果。

| 方法 | 用途 |
| :--- | :--- |
| `Create()` / `CreateInBatches()` | 插入记录 |
| `Save()` | 保存（有主键则更新，否则插入） |
| `First()` / `Take()` / `Last()` | 查询单条 |
| `Find()` / `FindInBatches()` | 查询多条 |
| `Update()` / `Updates()` | 更新（触发钩子） |
| `UpdateColumn()` / `UpdateColumns()` | 更新（跳过钩子） |
| `Delete()` | 删除（默认软删除） |
| `Count()` | 计数 |
| `Row()` / `Rows()` | 返回 `*sql.Row` / `*sql.Rows` |
| `Scan()` / `Pluck()` | 扫描到自定义结构 / 提取单列 |
| `FirstOrInit()` / `FirstOrCreate()` | 查找或初始化/创建 |
| `Transaction()` | 事务 |
| `Exec()` / `Raw()` | 执行原始 SQL |

完整 API 签名和用法见 `references/model-crud.md`。

## 核心概念

### 模型定义

```go
type User struct {
    gorm.Model                        // 内嵌：ID, CreatedAt, UpdatedAt, DeletedAt
    Name     string    `gorm:"size:100;not null;index"`     // varchar(100)，非空，索引
    Email    string    `gorm:"uniqueIndex;default:unknown"` // 唯一索引，默认值
    Age      int       `gorm:"default:18"`                  // 默认值
    Balance  float64   `gorm:"column:account_balance"`      // 自定义列名
}
```

`gorm.Model` 提供：`ID`(uint, primarykey)、`CreatedAt`(time.Time)、`UpdatedAt`(time.Time)、`DeletedAt`(gorm.DeletedAt, index)。

模型定义、字段 Tag、表名约定详见 `references/model-crud.md`。

### 命名策略

默认 `NamingStrategy` 将 Go struct 的 `CamelCase` 转为 `snake_case`，表名自动复数化。完整配置选项见 `references/model-crud.md`。

```go
gorm.Open(sqlite.Open("test.db"), &gorm.Config{
    NamingStrategy: schema.NamingStrategy{
        SingularTable: true,   // 禁用表名复数
        TablePrefix:   "t_",   // 表名前缀
        NoLowerCase:   false,
    },
})
```

### 关联关系

四种关联类型：`HasOne`、`HasMany`、`BelongsTo`、`Many2Many`。

```go
type User struct {
    gorm.Model
    CreditCard CreditCard   // Has One
    Orders     []Order      // Has Many
}

type Order struct {
    gorm.Model
    UserID   int            // Belongs To（默认外键）
    User     User
    Products []Product      `gorm:"many2many:order_products"` // Many2Many
}
```

完整说明见 `references/associations.md`。

### 预加载

```go
db.Preload("Orders").Preload("Orders.Products").Find(&users)
db.Preload("Orders", "state = ?", "paid").Find(&users)   // 带条件
db.Joins("Company").Find(&users)                          // JOIN（单条关联）
```

完整说明见 `references/associations.md`。

### 事务

```go
db.Transaction(func(tx *gorm.DB) error {
    err := tx.Create(&user).Error
    if err != nil {
        return err // 返回 error 自动回滚
    }
    return nil // 返回 nil 自动提交
})

// 手动控制
tx := db.Begin()
// ...
tx.Commit() / tx.Rollback()
```

完整说明见 `references/transactions-migration.md`。

### 迁移

```go
db.AutoMigrate(&User{}, &Order{})           // 自动建表/加列
db.Migrator().CreateTable(&Product{})       // 手动建表
db.Migrator().AddColumn(&User{}, "Phone")   // 加列
db.Migrator().CreateIndex(&User{}, "Name")  // 建索引
```

完整说明见 `references/transactions-migration.md`。

### 钩子

模型实现特定接口方法，在操作前后自动调用：

```go
func (u *User) BeforeCreate(tx *gorm.DB) error {
    u.UUID = uuid.New().String()
    return nil
}
```

完整钩子列表见 `references/hooks-plugins-logger.md`。

### 泛型接口（v1.26+）

GORM 1.26+ 提供类型安全的泛型 API，编译时检查字段名和类型：

```go
// 泛型查询
users, _ := gorm.G[User](db).Where("id > ?", 10).Find(ctx)
user, _ := gorm.G[User](db).First(ctx)

// 泛型更新
rows, _ := gorm.G[User](db).Where("id = ?", 1).Update(ctx, "Name", "李四")
```

完整说明见 `references/generics.md`。

## 代码生成工具

### gen（`gorm.io/gen`）

从数据库表生成类型安全的 DAO 代码（基于代码生成，非泛型）：

```go
// 配置生成器
g := gen.NewGenerator(gen.Config{
    OutPath: "./query",
    Mode:    gen.WithDefaultQuery | gen.WithQueryInterface,
})

// 连接数据库并生成
g.UseDB(db)
g.ApplyBasic(g.GenerateModel("users"), g.GenerateModel("orders"))
g.Execute()
```

生成的代码使用 `field` 包提供类型安全的字段引用和查询构建。

### cli（`gorm.io/cli`）

新一代 CLI 工具，利用 Go 泛型：

```bash
gorm gen                        # 从数据库生成代码
```

完整使用指南见 `references/code-generation.md`。

## 参考文档索引

| 参考文档 | 内容 |
| :--- | :--- |
| `references/model-crud.md` | 模型定义、字段 Tag、CRUD 操作、链式 API、终结 API 完整参考 |
| `references/associations.md` | HasOne、HasMany、BelongsTo、Many2Many、Preload、关联模式 |
| `references/advanced-queries.md` | Joins 子查询、Scopes、Clauses、Group/Having、Locking、DryRun、ToSQL |
| `references/transactions-migration.md` | 事务、SavePoint、AutoMigrate、Migrator 接口 |
| `references/hooks-plugins-logger.md` | 生命周期钩子、插件系统、Logger 配置、错误处理 |
| `references/generics.md` | `gorm.G[T]` 泛型 API 完整使用指南 |
| `references/code-generation.md` | gen（代码生成）和 cli（CLI 工具）的使用指南 |
| `references/troubleshooting.md` | 常见问题与排障指南 |

## 排障速查

| 现象 | 常见原因 |
| :--- | :--- |
| `ErrRecordNotFound` | `First`/`Last`/`Take` 未找到匹配记录（正常行为，用 `Find` 替代不含此错误） |
| `WHERE conditions required` | 未设置 `AllowGlobalUpdate: true` 时执行无条件的 Update/Delete |
| 零值字段未被更新 | `Updates` 默认跳过零值；用 `Select` 显式指定，或用指针类型字段 |
| 软删除记录查不出来 | 使用 `db.Unscoped()` 绕过软删除过滤 |
| 关联未加载 | 需要显式调用 `Preload` 或 `Joins`，GORM 不会自动加载关联 |
| 事务嵌套未回滚 | 确认 `DisableNestedTransaction` 配置；嵌套事务使用 SavePoint |
| `AutoMigrate` 未创建外键 | 默认禁用外键约束创建，需关闭 `DisableForeignKeyConstraintWhenMigrating` |
| 连接池耗尽 | 检查 `sql.DB` 的 `SetMaxOpenConns`/`SetMaxIdleConns` 配置 |

完整排障指南见 `references/troubleshooting.md`。

## 兜底

当本技能和参考资料无法解决问题时，**向用户报告具体情况并寻求帮助**，不得静默猜测。报告内容：
- 使用的 GORM 版本和数据库驱动版本
- 模型定义和出错的代码片段
- 完整的错误信息或 SQL 日志
- 预期的 SQL 和实际执行的 SQL（开启 `Debug()` 获取）
