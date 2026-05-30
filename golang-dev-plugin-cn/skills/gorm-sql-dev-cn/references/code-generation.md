# 代码生成工具完整参考

## 目录

- [gen 代码生成器](#gen-代码生成器)
  - [Generator 配置](#generator-配置)
  - [生成模型](#生成模型)
  - [字段类型系统](#字段类型系统)
  - [DO 数据对象](#do-数据对象)
  - [查询示例](#查询示例)
  - [高级特性](#高级特性)
- [cli CLI 工具](#cli-cli-工具)
- [gen vs cli 对比](#gen-vs-cli-对比)

## gen 代码生成器

`gorm.io/gen` 从数据库表生成类型安全的 DAO 层代码。生成的代码包含：
- **Model struct**：数据库表映射的 Go 结构体
- **Query 类型**：类型安全的查询构建器，使用 `field` 包定义字段表达式

### Generator 配置

```go
package main

import (
    "gorm.io/driver/mysql"
    "gorm.io/gen"
    "gorm.io/gorm"
)

func main() {
    // 连接数据库
    db, _ := gorm.Open(mysql.Open(dsn))

    // 创建生成器
    g := gen.NewGenerator(gen.Config{
        OutPath:           "./dal/query",   // 查询代码输出路径
        ModelPkgPath:      "./dal/model",   // 模型代码的包路径
        OutFile:           "gen.go",        // 输出文件名
        WithUnitTest:      false,           // 是否生成单元测试
        FieldNullable:     false,           // 可空字段用指针类型
        FieldCoverable:    true,            // 有默认值的字段用指针
        FieldSignable:     true,            // 检测 unsigned 类型
        FieldWithIndexTag: true,            // 生成 index tag
        FieldWithTypeTag:  true,            // 生成 type tag
        FieldWithDefaultTag: true,          // 生成 default tag
        Mode: gen.WithDefaultQuery | gen.WithQueryInterface | gen.WithoutContext,
        // WithoutContext: 生成的代码不强制传 context
        // WithQueryInterface: 生成导出的查询接口
        // WithDefaultQuery: 生成默认查询
    })

    // 设置数据库连接
    g.UseDB(db)

    // 生成模型
    g.ApplyBasic(
        g.GenerateModel("users"),
        g.GenerateModel("orders",
            gen.FieldType("amount", "decimal.Decimal"), // 自定义字段类型
            gen.FieldIgnore("internal_notes"),           // 忽略字段
        ),
        g.GenerateModelAs("products", "Product"),       // 自定义类型名
    )

    // 执行生成
    g.Execute()
}
```

### 生成模式 (GenerateMode)

```go
const (
    WithDefaultQuery  // 生成默认查询（包含内置 CRUD 方法）
    WithoutContext    // 不强制要求 context
    WithQueryInterface // 生成导出的查询接口
    WithGeneric       // 使用泛型
)
```

### 生成模型

```go
// 基础模型生成
g.ApplyBasic(
    g.GenerateModel("users"),                              // 表名 → 驼峰模型名
    g.GenerateModelAs("users", "User"),                    // 自定义模型名
    g.GenerateModel("users",
        gen.FieldType("amount", "decimal.Decimal"),        // 自定义字段 Go 类型
        gen.FieldIgnore("password_hash"),                  // 忽略字段
        gen.FieldNewTag("mobile", `gorm:"uniqueIndex"`),   // 自定义 tag
        gen.FieldNewTagWithNS("name", func(columnName string) string {
            return `gorm:"column:` + columnName + `;type:varchar(100)"`
        }),
        gen.FieldRename("old_name", "new_name"),           // 重命名字段
        gen.FieldGORMTag("status", `gorm:"-"`),            // 覆盖 gorm tag
    ),
)

// 按表名过滤（支持正则）
g.ApplyBasic(
    g.GenerateAllTable(
        gen.FieldType("amount", "decimal.Decimal"),
    ), // 生成所有表
    g.GenerateAllTable(
        gen.FieldGORMTag("created_at", ""),
        gen.FieldGORMTag("updated_at", ""),
        gen.FieldGORMTag("deleted_at", ""),
    ), // 覆盖可多次调用
)

// 指定特定表
g.ApplyBasic(
    g.GenerateAllTable(
        g.GenerateModelAs("users", "User"),
        g.GenerateModel("orders"),
    ),
)
```

### 字段类型系统

```go
import "gorm.io/gen/field"

// gen 生成的字段类型映射
type Field interface {
    // 比较方法
    Eq(col interface{}) Expr
    Neq(col interface{}) Expr
    Gt(col interface{}) Expr
    Gte(col interface{}) Expr
    Lt(col interface{}) Expr
    Lte(col interface{}) Expr
    In(cols ...interface{}) Expr
    NotIn(cols ...interface{}) Expr
    Between(min, max interface{}) Expr
    Like(col interface{}) Expr

    // 赋值
    Value(val interface{}) AssignExpr

    // 列信息
    ColumnName() string
}
```

生成的字段类型：
- `field.String` — 字符串列
- `field.Int64` / `field.Int` — 整数列
- `field.Uint64` / `field.Uint` — 无符号整型
- `field.Float64` / `field.Float32` — 浮点列
- `field.Bool` — 布尔列
- `field.Time` — 时间列
- `field.Bytes` — 字节列

### DO 数据对象

`DO` (Data Object) 是 gen 生成的查询构建器的基础类型，嵌入 `*gorm.DB` 并提供类型安全的查询 API：

```go
// 使用生成的 DO
u := query.User
o := query.Order

// 设置数据库连接
u.UseDB(db)

// 替换连接池（用于读写分离等）
u.ReplaceConnPool(slaveDB.ConnPool)
```

### 查询示例

```go
import "your_project/dal/query"

// 初始化
q := query.Use(db)  // 或用 query.SetDefault(db)

// 基础 CRUD
u := q.User

// Create
user := model.User{Name: "张三", Age: 30}
err := u.Create(&user) // 同 u.WithContext(ctx).Create(&user)

// 查询
user, err := u.Where(u.Name.Eq("张三")).First()
users, err := u.Where(u.Age.Gt(18)).Order(u.Age.Desc()).Find()

// 条件查询
users, err := u.Where(
    u.Age.Gt(18),
    u.Name.Like("%张%"),
).Find()

// OR 条件
users, err := u.Where(u.Age.Lt(18)).Or(u.Age.Gt(60)).Find()

// IN 查询
users, err := u.Where(u.Name.In("张三", "李四")).Find()

// 复杂条件
users, err := u.Where(
    u.WithContext(ctx).Where(u.Age.Gt(18)).Or(u.Name.Eq("张三")),
).Find()

// Update
info, err := u.Where(u.ID.Eq(1)).Update(u.Name, "李四")
info, err := u.Where(u.ID.Eq(1)).UpdateSimple(
    u.Name.Value("李四"),
    u.Age.Value(31),
)
info, err := u.Where(u.ID.Eq(1)).Updates(&model.User{Name: "李四"})

// Update 跳过钩子
info, err := u.Where(u.ID.Eq(1)).UpdateColumn(u.Name, "李四")
info, err := u.Where(u.ID.Eq(1)).UpdateColumnSimple(
    u.Name.Value("李四"),
)

// Delete
info, err := u.Where(u.ID.Eq(1)).Delete()
info, err := u.Where(u.ID.In(1, 2, 3)).Delete()

// Count
count, err := u.Where(u.Age.Gt(18)).Count()

// 子查询
subQuery := u.Select(u.ID).Where(u.Age.Gt(18))
orders, err := o.Where(o.Columns(o.UserID).In(subQuery)).Find()
```

### 高级特性

#### 关联操作

```go
// 多对多关联
u := query.User
roles := u.Roles.Model() // 获取关联模型

// 追加关联
err := u.Roles.Model().Append(&Role{Name: "admin"})

// 替换关联
err := u.Roles.Model().Replace(&roles)

// 清空关联
err := u.Roles.Model().Clear()
```

#### 智能选择字段

```go
// gen 会分析赋值表达式，自动 Select 被更新的字段
u := query.User
info, err := u.Where(u.ID.Eq(1)).UpdateSimple(
    u.Name.Value("李四"),
    u.Age.Value(31),
)
// 自动生成: UPDATE users SET name='李四', age=31 WHERE id=1
```

#### 事务

```go
q := query.Use(db)
q.Transaction(func(tx *query.Query) error {
    err := tx.User.Create(&user1)
    if err != nil {
        return err
    }
    return tx.User.Create(&user2)
})
```

#### 读写分离

```go
g.ApplyInterface(func(Querier) {}, g.GenerateModel("users"))

// 生成的代码中
u.ReplaceDB(writeDB)
u.ReplaceDB(readDB)
```

## cli CLI 工具

`gorm.io/cli` 是新一代 CLI 工具，核心是 `gorm gen` 命令。

### 安装与使用

```bash
go install gorm.io/cli@latest

# 生成代码
gorm gen

# 查看版本
gorm version
```

### 配置 (genconfig)

针对 `gorm gen` 需要 `genconfig` 配置。

### 与 gen 的区别

cli 基于 gen 并利用了 Go 泛型特性，提供更简洁的命令行体验：

```go
// cli 生成的代码使用泛型
// 提供更简洁的字段表达式
```

## gen vs cli 对比

| 特性 | gen | cli |
| :--- | :--- | :--- |
| 接口类型 | 代码生成 + `field` 包 | 利用 Go 泛型 |
| 使用方式 | Go 代码调用 Generator API | CLI 命令 + Go 配置文件 |
| 类型安全 | 编译时（字段表达式） | 编译时（泛型） |
| 学习曲线 | 中等（需理解 field 类型系统） | 较低（更接近原生 GORM） |
| 成熟度 | 成熟，文档完善 | 相对较新 |
| 推荐场景 | 大型项目、需要精细控制 | 新项目、偏好泛型 |

> **选择建议**：新项目优先试 cli（泛型 + CLI 简单）；已使用 gen 的大型项目继续使用 gen（成熟稳定）。
