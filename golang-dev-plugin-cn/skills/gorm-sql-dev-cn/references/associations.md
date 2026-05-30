# 关联关系完整参考

## 目录

- [关联类型](#关联类型)
  - [Belongs To](#belongs-to)
  - [Has One](#has-one)
  - [Has Many](#has-many)
  - [Many To Many](#many-to-many)
- [Preload（预加载）](#preload预加载)
- [Joins（连接查询）](#joins连接查询)
- [关联模式（Association Mode）](#关联模式association-mode)
- [关联操作](#关联操作)
- [外键与引用约定](#外键与引用约定)

## 关联类型

### Belongs To

```go
type User struct {
    gorm.Model
    Name      string
    CompanyID int      // 外键
    Company   Company  // Belongs To
}

type Company struct {
    ID   int
    Name string
}
```

**约定**：外键名 = `{关联字段名}ID`（如 `CompanyID`）；引用键 = 关联模型的 `ID`。

自定义外键：

```go
type User struct {
    gorm.Model
    Name       string
    CompanyRef string
    Company    Company `gorm:"foreignKey:CompanyRef;references:Code"`
}

type Company struct {
    ID   int
    Code string // 被引用的键
}
```

### Has One

```go
type User struct {
    gorm.Model
    CreditCard CreditCard // Has One
}

type CreditCard struct {
    gorm.Model
    Number string
    UserID uint   // 外键
}
```

**约定**：外键名 = `{所有者模型名}ID`（如 `UserID`）。

自定义外键：

```go
type User struct {
    gorm.Model
    CreditCard CreditCard `gorm:"foreignKey:OwnerID;references:Code"`
}

type CreditCard struct {
    gorm.Model
    Number  string
    OwnerID string // 自定义外键
}
```

### Has Many

```go
type User struct {
    gorm.Model
    Orders []Order // Has Many
}

type Order struct {
    gorm.Model
    UserID int    // 外键
    Amount float64
}
```

自定义外键：

```go
type User struct {
    gorm.Model
    Orders []Order `gorm:"foreignKey:OwnerID;references:Code"`
}
```

### Many To Many

```go
type User struct {
    gorm.Model
    Languages []Language `gorm:"many2many:user_languages"` // 关联表名
}

type Language struct {
    gorm.Model
    Name string
}
```

自定义外键：

```go
type User struct {
    gorm.Model
    Languages []Language `gorm:"many2many:user_languages;foreignKey:UserID;joinForeignKey:UserRefer;references:Code;joinReferences:LangCode"`
}
```

其中：
- `foreignKey`：当前模型在关联表中的外键名
- `references`：关联模型被引用的字段名
- `joinForeignKey`：关联表中连向当前模型的外键名
- `joinReferences`：关联表中连向关联模型的外键名

## Preload（预加载）

GORM 不会自动加载关联。必须显式调用 `Preload` 或 `Joins`。

### 基础用法

```go
// 单层
db.Preload("Orders").Find(&users)

// 嵌套
db.Preload("Orders.Products").Find(&users)

// 多个关联
db.Preload("Orders").Preload("CreditCard").Find(&users)
```

### 带条件的 Preload

```go
// 条件字符串
db.Preload("Orders", "state = ?", "paid").Find(&users)

// 自定义查询
db.Preload("Orders", func(db *gorm.DB) *gorm.DB {
    return db.Order("created_at DESC").Limit(3)
}).Find(&users)

// 嵌套 + 条件
db.Preload("Orders.Products", "active = ?", true).Find(&users)
```

### Preload All

```go
// 预加载所有关联
db.Preload(clause.Associations).Find(&users)
```

## Joins（连接查询）

`Joins` 使用 SQL JOIN，可以在查询时按关联条件过滤，且关联数据只加载到嵌套字段中（仅对 Has One 和 Belongs To 有效）。

```go
// 用关联名 JOIN
db.Joins("Company").Find(&users)

// 用 JOIN 条件过滤 + 加载关联数据
db.Joins("Company", db.Where(&Company{Name: "ACME"})).Find(&users)

// INNER JOIN
db.InnerJoins("Company").Find(&users)

// 原始 SQL JOIN
db.Joins("LEFT JOIN companies ON companies.id = users.company_id").Find(&users)

// 原始 SQL JOIN + 条件
db.Joins("JOIN orders ON orders.user_id = users.id AND orders.state = ?", "paid").Find(&users)
```

### Preload 与 Joins 对比

| | Preload | Joins |
| :--- | :--- | :--- |
| 加载关联数据 | 是 | 仅 Has One / Belongs To |
| 用关联条件过滤 | 否（只能在 Preload 条件中限制关联数据的范围） | 是 |
| SQL 执行 | 多条查询 | 单个 JOIN 查询 |
| 适用关联类型 | 全部 | Has One, Belongs To |
| N+1 问题 | 避免（用批量查询） | 无 |

## 关联模式（Association Mode）

获取 `*gorm.Association` 进行关联操作：

```go
var user User
db.First(&user)
db.Model(&user).Association("Orders")
```

### 查询关联

```go
var orders []Order
db.Model(&user).Association("Orders").Find(&orders)
```

### 追加关联（Append）

```go
db.Model(&user).Association("Orders").Append(&Order{Amount: 100})
db.Model(&user).Association("Orders").Append(&orders) // 批量追加
```

### 替换关联（Replace）

```go
// 用新关联替换所有现有关联
db.Model(&user).Association("Orders").Replace([]Order{newOrder1, newOrder2})
```

### 删除关联（Delete）

```go
// 删除关联的 records（非软删除则为物理删除）
db.Model(&user).Association("Orders").Delete(&orders)

// Many2Many：删除关联表记录，不删除关联 model
// Has Many / Has One：将外键设为 NULL
```

### 清空关联（Clear）

```go
// 清空所有关联
db.Model(&user).Association("Orders").Clear()
// Has Many / Has One：外键设为 NULL
// Many2Many：删除关联表记录
```

### 计数

```go
count := db.Model(&user).Association("Orders").Count()
```

## 关联操作

### 创建时包含关联

```go
// 创建用户同时创建订单
db.Create(&User{
    Name: "张三",
    Orders: []Order{{Amount: 100}, {Amount: 200}},
})

// 跳过关联创建
db.Session(&gorm.Session{FullSaveAssociations: false}).Create(&user)
// 或设置 db.Config.FullSaveAssociations = false（默认值）
```

### Select / Omit 控制关联

```go
// 创建时跳过 Orders 关联
db.Omit("Orders").Create(&user)

// 创建时选择特定关联
db.Select("Orders").Create(&user)
```

### 更新关联

```go
// 替换整个关联（先清空再追加）
db.Model(&user).Association("Orders").Replace([]Order{{Amount: 300}})

// Append 追加（保留已有关联）
db.Model(&user).Association("Orders").Append(Order{Amount: 400})
```

### 多对多关联表操作

```go
// 在关联表中追加记录
db.Model(&user).Association("Languages").Append([]Language{{Name: "Go"}, {Name: "Python"}})

// 从关联表删除记录（不删除语言本身）
db.Model(&user).Association("Languages").Delete(&language)

// 清空关联表中的所有关联记录
db.Model(&user).Association("Languages").Clear()
```

## 外键与引用约定

GORM 的默认外键约定：

| 关联类型 | 默认外键名 | 默认外键所在表 |
| :--- | :--- | :--- |
| Belongs To | `{field}ID` | 当前表 |
| Has One | `{owner}ID` | 关联表 |
| Has Many | `{owner}ID` | 关联表 |
| Many2Many | 关联表 `{table1}_{table2}` | 关联表 |

引用键默认为关联模型的第一个主键字段。

### 外键约束

```go
type User struct {
    gorm.Model
    Orders []Order `gorm:"constraint:OnUpdate:CASCADE,OnDelete:SET NULL"`
}
```

AutoMigrate 时默认不创建外键约束。设置 `DisableForeignKeyConstraintWhenMigrating: false` 可以启用。

### Polymorphism（多态关联）

```go
type Cat struct {
    ID    int
    Name  string
    Toy   Toy `gorm:"polymorphic:Owner"`
}

type Dog struct {
    ID   int
    Name string
    Toy  Toy `gorm:"polymorphic:Owner"`
}

type Toy struct {
    ID        int
    Name      string
    OwnerID   int
    OwnerType string // "cats" 或 "dogs"
}
```
