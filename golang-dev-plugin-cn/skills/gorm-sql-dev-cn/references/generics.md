# 泛型 API 完整参考

> GORM 1.26+ 提供类型安全的泛型 API `gorm.G[T]`。编译时检查模型类型，终结方法 (First/Find/Create 等) 直接返回具体类型而非 `interface{}`。

## 目录

- [入口函数](#入口函数)
- [查询 (ExecInterface)](#查询-execinterface)
- [链式方法 (ChainInterface)](#链式方法-chaininterface)
- [创建 (CreateInterface)](#创建-createinterface)
- [Set 赋值](#set-赋值)
- [Joins 构建器](#joins-构建器)
- [Preload 构建器](#preload-构建器)
- [链式后可用的终结方法](#链式后可用的终结方法)
- [泛型接口对照表](#泛型接口对照表)

## 入口函数

```go
import "gorm.io/gorm"

// G[T] 创建泛型查询构建器
// T 必须是数据库模型 struct 类型
query := gorm.G[User](db)

// 可选：传入 clause 表达式
query := gorm.G[User](db, clause.Locking{Strength: "UPDATE"})
```

## 查询 (ExecInterface)

所有终结方法接受 `context.Context` 参数：

```go
// 单条查询 — 返回具体类型
user, err := gorm.G[User](db).First(ctx)
user, err := gorm.G[User](db).Take(ctx)
user, err := gorm.G[User](db).Last(ctx)

// 多条查询 — 返回 []User
users, err := gorm.G[User](db).Where("age > ?", 18).Find(ctx)

// 分批查询 — fc 直接接收 []User
err := gorm.G[User](db).FindInBatches(ctx, 100, func(data []User, batch int) error {
    for _, u := range data {
        // process u
    }
    return nil
})

// Scan 扫描到自定义结构
type UserDTO struct { Name string; Age int }
var dtos []UserDTO
err := gorm.G[User](db).Select("name", "age").Scan(ctx, &dtos)

// Row / Rows
row := gorm.G[User](db).Row(ctx)
rows, err := gorm.G[User](db).Rows(ctx)
```

## 链式方法 (ChainInterface)

```go
gorm.G[User](db).
    Where("age > ?", 18).
    Not("name = ?", "admin").
    Or("name = ?", "张三").
    Order("age DESC").
    Limit(10).
    Offset(20).
    Select("name", "age").
    Omit("password").
    Distinct("name").
    Group("age").
    Having("count(*) > ?", 1).
    Joins(clause.JoinTarget{Association: "Company"}, func(jb JoinBuilder, joinTable, curTable clause.Table) error {
        jb.Where("name = ?", "ACME")
        return nil
    }).
    Preload("Orders", func(pb PreloadBuilder) error {
        pb.Where("state = ?", "paid").Order("created_at DESC")
        return nil
    }).
    Scopes(func(stmt *gorm.Statement) {
        // 自定义作用域
    }).
    Find(ctx)
```

## 创建 (CreateInterface)

`CreateInterface` 在链式开头可用（`Select`/`Omit` 保持 `CreateInterface` 以允许后续 `Create`）：

```go
// 创建单条
err := gorm.G[User](db).Create(ctx, &User{Name: "张三", Age: 30})

// 批量创建
err := gorm.G[User](db).CreateInBatches(ctx, &[]User{{Name: "张三"}, {Name: "李四"}}, 100)

// 指定字段创建
err := gorm.G[User](db).Select("Name", "Age").Create(ctx, &user)

// 排除字段创建
err := gorm.G[User](db).Omit("Password").Create(ctx, &user)
```

## Set 赋值

`Set` 是泛型 API 中批量赋值的入口，返回类型约束了后续可用方法：

```go
import "gorm.io/gorm/clause"

// 起始状态: Set 返回 SetCreateOrUpdateInterface[T]
// 支持 Update 或 Create
rows, err := gorm.G[User](db).
    Set(
        clause.Assignment{Column: clause.Column{Name: "name"}, Value: "李四"},
        clause.Assignment{Column: clause.Column{Name: "age"}, Value: 31},
    ).
    Where("id = ?", 1).
    Update(ctx) // 或 .Create(ctx)

// 链式后: Set 返回 SetUpdateOnlyInterface[T]
// 只支持 Update
rows, err := gorm.G[User](db).
    Where("id = ?", 1).
    Set(
        clause.Assignment{Column: clause.Column{Name: "name"}, Value: "李四"},
    ).
    Update(ctx)
```

## 链式后可用的终结方法

```go
// Delete
rows, err := gorm.G[User](db).Where("age < ?", 18).Delete(ctx)

// Update 单列
rows, err := gorm.G[User](db).Where("id = ?", 1).Update(ctx, "Name", "李四")

// Updates 多列（struct — 零值被跳过）
rows, err := gorm.G[User](db).Where("id = ?", 1).Updates(ctx, User{Name: "李四", Age: 31})

// Count
count, err := gorm.G[User](db).Where("age > ?", 18).Count(ctx, "id")

// Raw / Exec
rows, err := gorm.G[User](db).Raw("SELECT * FROM users WHERE name = ?", "张三").Find(ctx)
err := gorm.G[User](db).Exec(ctx, "UPDATE users SET active = 1 WHERE id = ?", 1)
```

## Joins 构建器

```go
gorm.G[User](db).Joins(clause.JoinTarget{
    Association: "Company",
    Table:       "c",
}, func(jb JoinBuilder, joinTable, curTable clause.Table) error {
    jb.Where("c.name = ?", "ACME").
       Select("id", "name").
       Omit("deleted_at")
    return nil
}).Find(ctx)
```

## Preload 构建器

```go
gorm.G[User](db).Preload("Orders", func(pb PreloadBuilder) error {
    pb.Where("state = ?", "paid").
       Select("id", "amount", "user_id").
       Order("created_at DESC").
       Limit(5).
       LimitPerRecord(3) // 每条父记录最多 3 条关联记录
    return nil
}).Find(ctx)
```

`LimitPerRecord` 通过 `ROW_NUMBER() OVER (PARTITION BY ...)` 实现。仅对非 Many2Many 关联有效。

## 泛型接口对照表

```
gorm.G[T](db)
└── Interface[T]
    ├── Raw() / Exec()
    └── CreateInterface[T]
        ├── ExecInterface[T]          — First, Last, Take, Find, FindInBatches, Scan, Row, Rows
        ├── ChainInterface[T]         — Where, Not, Or, Order, Limit, Offset, Select, Omit, ...
        │   ├── ExecInterface[T]
        │   └── Set() → SetUpdateOnlyInterface[T] → Update()
        ├── Create() / CreateInBatches()
        └── Set() → SetCreateOrUpdateInterface[T] → Update() / Create()
```

**接口级别控制**：
- `Select`/`Omit` 在链式起始时为 `CreateInterface`（之后可 `Create`）
- `Select`/`Omit` 在链式中为 `ChainInterface`（不可 `Create`）
- 所有链式方法接收 `func(db *Statement)` 作为 Scopes 参数（与传统的 `func(*DB)*DB` 不同）

> 泛型 API 的终结方法返回具体类型和 `error`，而不像传统 API 返回 `*gorm.DB`。如果需要 `RowsAffected`，可以使用传统 API 获取：
>
> ```go
> result := db.Create(&user)
> fmt.Println(result.RowsAffected)
> ```
