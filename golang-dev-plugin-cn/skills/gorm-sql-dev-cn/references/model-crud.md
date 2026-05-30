# 模型定义与 CRUD 完整参考

## 目录

- [模型定义](#模型定义)
  - [gorm.Model](#gormmodel)
  - [字段 Tag](#字段-tag)
  - [嵌入结构体](#嵌入结构体)
  - [表名约定](#表名约定)
- [创建](#创建)
- [查询](#查询)
  - [单条查询](#单条查询)
  - [多条查询](#多条查询)
  - [条件构建](#条件构建)
- [更新](#更新)
- [删除](#删除)
- [链式 API 参考](#链式-api-参考)
- [终结 API 参考](#终结-api-参考)
- [Session 模式](#session-模式)

## 模型定义

GORM 模型是普通的 Go struct，包含基础类型、实现了 `Scanner`/`Valuer` 接口的自定义类型、以及通过指针或 `sql.Null*` 表示的可空字段。

```go
type User struct {
    ID           uint           `gorm:"primaryKey"`
    Name         string         `gorm:"size:100;not null"`
    Email        *string        // 指针类型 — 可空
    Age          uint8          `gorm:"default:18;index"`
    Birthday     *time.Time
    MemberNumber sql.NullString
    ActivatedAt  sql.NullTime
    CreatedAt    time.Time
    UpdatedAt    time.Time
    DeletedAt    gorm.DeletedAt `gorm:"index"`
}
```

### gorm.Model

```go
// gorm.Model 定义
type Model struct {
    ID        uint `gorm:"primarykey"`
    CreatedAt time.Time
    UpdatedAt time.Time
    DeletedAt DeletedAt `gorm:"index"`
}
```

内嵌 `gorm.Model` 等价于在 struct 中直接包含这些字段。如果不需要其中某些字段，可以直接定义自己的字段代替。

### 字段 Tag

Tag 格式：`gorm:"key1:value1;key2:value2"`。常用 Tag：

**列定义**

| Tag | 说明 | 示例 |
| :--- | :--- | :--- |
| `column` | 指定列名 | `gorm:"column:user_name"` |
| `type` | 列数据类型 | `gorm:"type:varchar(128)"` |
| `size` | 列数据大小 | `gorm:"size:256"` |
| `precision` | 精度 | `gorm:"precision:10"` |
| `scale` | 小数位数 | `gorm:"scale:2"` |
| `not null` | 非空约束 | `gorm:"not null"` |
| `default` | 默认值 | `gorm:"default:18"` |
| `comment` | 列注释 | `gorm:"comment:用户名"` |
| `serializer` | 序列化器 | `gorm:"serializer:json"` |

**键与索引**

| Tag | 说明 | 示例 |
| :--- | :--- | :--- |
| `primaryKey` | 主键 | `gorm:"primaryKey"` |
| `unique` | 唯一约束 | `gorm:"unique"` |
| `uniqueIndex` | 唯一索引 | `gorm:"uniqueIndex:idx_name"` |
| `index` | 普通索引 | `gorm:"index:idx_age"` |
| `uniqueIndex:,sort` | 聚合索引排序 | `gorm:"uniqueIndex:,sort:desc"` |

**关联**

| Tag | 说明 | 示例 |
| :--- | :--- | :--- |
| `foreignKey` | 指定外键 | `gorm:"foreignKey:CompanyID"` |
| `references` | 指定引用键 | `gorm:"references:ID"` |
| `many2many` | 多对多关联表 | `gorm:"many2many:user_languages"` |
| `constraint` | 约束行为 | `gorm:"constraint:OnUpdate:CASCADE,OnDelete:SET NULL"` |

**行为控制**

| Tag | 说明 |
| :--- | :--- |
| `<-` | 只写（禁止读取） |
| `->` | 只读（禁止写入） |
| `->:false;<-:create` | 只允许创建 |
| `->:false;<-:update` | 只允许更新 |
| `-` | 忽略此字段 |
| `-:migration` | 迁移时忽略 |
| `-:all` | 所有操作忽略 |
| `autoCreateTime` | 创建时自动设置时间（nano/milli/second） |
| `autoUpdateTime` | 更新时自动设置时间 |
| `embedded` | 嵌入结构体 |
| `embeddedPrefix` | 嵌入字段前缀 |

### 嵌入结构体

```go
type Author struct {
    Name  string
    Email string
}

type Blog struct {
    ID      int
    Author  Author `gorm:"embedded"`           // 字段名为 Name, Email
    Author2 Author `gorm:"embedded;embeddedPrefix:author2_"` // 字段名为 author2_Name, author2_Email
}
```

### 表名约定

默认将 struct 名转为 `snake_case` 复数。自定义表名：

```go
// 方法一：实现 Tabler 接口
func (User) TableName() string {
    return "t_users"
}

// 方法二：临时指定
db.Table("t_users").First(&user)
```

## 创建

```go
user := User{Name: "张三", Age: 30}

// 插入单条
result := db.Create(&user)
// user.ID 被回填为数据库生成的主键值
// result.RowsAffected -> 1
// result.Error -> nil

// 批量插入
var users = []User{{Name: "张三"}, {Name: "李四"}}
db.Create(&users)

// 分批插入
db.CreateInBatches(users, 100) // 每批 100 条

// 指定字段插入
db.Select("Name", "Age").Create(&user)
db.Omit("Age").Create(&user)

// 忽略钩子
db.Session(&gorm.Session{SkipHooks: true}).Create(&user)

// 使用 map 插入
db.Model(&User{}).Create(map[string]interface{}{
    "Name": "张三", "Age": 18,
})
```

### 零值问题

GORM 默认忽略零值字段（`0`, `""`, `false` 等）。解决办法：

```go
// 方法一：使用指针或 sql.Null* 类型
type User struct {
    Name string
    Age  *int           // int -> *int
    Active sql.NullBool // bool -> sql.NullBool
}

// 方法二：用 Select 显式指定
db.Select("Name", "Age", "Active").Create(&user)

// 方法三：使用 map
db.Model(&User{}).Create(map[string]interface{}{
    "Name": "张三", "Age": 0, "Active": false,
})
```

### Upsert

```go
// ON CONFLICT DO NOTHING
db.Clauses(clause.OnConflict{DoNothing: true}).Create(&user)

// ON CONFLICT DO UPDATE
db.Clauses(clause.OnConflict{
    Columns:   []clause.Column{{Name: "id"}},
    DoUpdates: clause.AssignmentColumns([]string{"name", "age"}),
}).Create(&users)
```

## 查询

### 单条查询

```go
// 按主键正序取第一条
db.First(&user)              // SELECT * FROM users ORDER BY id LIMIT 1
db.First(&user, 1)           // WHERE id = 1
db.First(&user, "id = ?", 1) // WHERE id = 1

// 不排序取第一条
db.Take(&user)

// 按主键倒序取最后一条
db.Last(&user)

// 未找到时 First/Last/Take 返回 ErrRecordNotFound
// Find 不返回此错误
```

`First`/`Last`/`Take` 在查询不到记录时返回 `gorm.ErrRecordNotFound`。如需区分「未找到」和「其他错误」：

```go
err := db.First(&user, 99).Error
if errors.Is(err, gorm.ErrRecordNotFound) {
    // 记录不存在
}
```

### 多条查询

```go
// 查询所有
var users []User
db.Find(&users)

// 带条件查询
db.Where("age > ?", 18).Find(&users)

// 查询到 map
var results []map[string]interface{}
db.Model(&User{}).Find(&results)

// 提取单列
var ages []int64
db.Model(&User{}).Pluck("age", &ages)

// 分批查询
db.FindInBatches(&users, 100, func(tx *gorm.DB, batch int) error {
    // 处理当前批次
    return nil
})

// 扫描到自定义结构
type APIUser struct {
    Name string
    Age  int
}
var apiUsers []APIUser
db.Model(&User{}).Select("name", "age").Scan(&apiUsers)
```

### 条件构建

```go
// String 条件
db.Where("name = ? AND age >= ?", "张三", 18).Find(&users)

// Struct 条件（零值字段被忽略）
db.Where(&User{Name: "张三", Age: 18}).Find(&users)

// Map 条件
db.Where(map[string]interface{}{"name": "张三", "age": 18}).Find(&users)

// IN
db.Where("name IN ?", []string{"张三", "李四"}).Find(&users)

// LIKE
db.Where("name LIKE ?", "%张%").Find(&users)

// BETWEEN
db.Where("age BETWEEN ? AND ?", 18, 30).Find(&users)

// NOT
db.Not("name = ?", "张三").Find(&users)

// OR
db.Where("name = ?", "张三").Or("name = ?", "李四").Find(&users)

// 内联条件（终结 API 直接传参）
db.First(&user, "name = ?", "张三")
db.Find(&users, "age > ?", 18)
```

**Where 条件中的零值**：使用 struct 条件时，零值字段不会被加入 WHERE 子句。如果需要含零值的条件，使用 map 或 string 条件。

## 更新

```go
// 更新单列（触发钩子和时间追踪）
db.Model(&user).Update("name", "张三")

// 更新多列（struct — 零值被跳过）
db.Model(&user).Updates(User{Name: "张三", Age: 0})

// 更新多列（map — 零值不跳过）
db.Model(&user).Updates(map[string]interface{}{"name": "张三", "age": 0})

// 使用 Select 显式控制
db.Model(&user).Select("Name", "Age").Updates(User{Name: "张三", Age: 0})
db.Model(&user).Select("*").Updates(User{Name: "张三"}) // 包含所有字段（含零值）

// 跳过钩子和时间追踪
db.Model(&user).UpdateColumn("name", "张三")
db.Model(&user).UpdateColumns(User{Name: "张三", Age: 0})

// 批量更新（必须有 WHERE 条件）
db.Model(&User{}).Where("age > ?", 18).Update("name", "张三")

// 表达式更新
db.Model(&user).Update("age", gorm.Expr("age + ?", 1))
db.Model(&user).Updates(map[string]interface{}{"age": gorm.Expr("age + ?", 1)})

// Save：有主键则更新所有字段（含零值），无主键则插入
db.Save(&user)
```

> **关键区别**：`Update`/`Updates` 触发 `BeforeUpdate`/`AfterUpdate` 钩子，自动更新 `UpdatedAt`；`UpdateColumn`/`UpdateColumns` 跳过钩子和时间追踪。

> **安全限制**：无 WHERE 条件的 `Update`/`Delete` 默认报错 `ErrMissingWhereClause`。设置 `AllowGlobalUpdate: true` 或使用 `db.Session(&gorm.Session{AllowGlobalUpdate: true})` 可全局更新。

## 删除

```go
// 软删除（设置 deleted_at）
db.Delete(&user)                    // 按主键
db.Delete(&user, 1)                 // 按主键值
db.Where("age > ?", 80).Delete(&User{})  // 批量软删除

// 物理删除（跳过软删除逻辑）
db.Unscoped().Delete(&user)

// 物理删除 + 跳过钩子
db.Unscoped().Session(&gorm.Session{SkipHooks: true}).Delete(&user)
```

软删除后，所有查询自动加 `WHERE deleted_at IS NULL`。使用 `Unscoped()` 可以查询软删除的记录。

## 链式 API 参考

### Model / Table

```go
db.Model(&User{})     // 指定模型，用于推断表名
db.Table("users")     // 直接指定表名
db.Table("users AS u") // 表别名
```

### Select / Omit

```go
db.Select("name", "age").Find(&users)
db.Select([]string{"name", "age"}).Find(&users)
db.Omit("password").Find(&users)
```

### Where / Not / Or

```go
db.Where("name = ?", "张三")
db.Where(&User{Name: "张三"})       // struct — 零值被跳过
db.Where(map[string]interface{}{"name": "张三"}) // map — 零值不跳过
db.Not("age > ?", 20)
db.Or("name = ?", "李四")
```

### Order / Group / Having / Distinct

```go
db.Order("age desc, name")
db.Order(clause.OrderByColumn{Column: clause.Column{Name: "age"}, Desc: true})
db.Group("name")
db.Having("count(*) > ?", 1)
db.Distinct("name")
```

### Limit / Offset

```go
db.Limit(10).Offset(20).Find(&users)
db.Limit(-1).Offset(-1) // 取消之前的 Limit/Offset
```

### Joins / InnerJoins

```go
db.Joins("Company").Find(&users)                // 关联名
db.Joins("JOIN companies ON companies.id = users.company_id").Find(&users) // 原始 SQL
db.InnerJoins("Company").Find(&users)
```

### Preload

```go
db.Preload("Orders").Find(&users)
db.Preload("Orders.Products").Find(&users)       // 嵌套预加载
db.Preload("Orders", "state = ?", "paid").Find(&users) // 条件预加载
db.Preload("Orders", func(db *gorm.DB) *gorm.DB {
    return db.Order("created_at DESC")
}).Find(&users)
```

### Clauses

```go
// 直接注入 clause 表达式
db.Clauses(clause.Locking{Strength: "UPDATE"}).Find(&users)
db.Clauses(hints.UseIndex("idx_user_name")).Find(&users)
db.Clauses(clause.OnConflict{UpdateAll: true}).Create(&users)
```

### Scopes

```go
// 定义可复用的查询逻辑
func ActiveUser(db *gorm.DB) *gorm.DB {
    return db.Where("active = ?", true)
}

func Paginate(page, pageSize int) func(db *gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        return db.Offset((page - 1) * pageSize).Limit(pageSize)
    }
}

db.Scopes(ActiveUser, Paginate(1, 20)).Find(&users)
```

### Attrs / Assign

```go
// Attrs: 仅在未找到记录时设置
db.Where(&User{Name: "张三"}).Attrs(&User{Age: 18}).FirstOrCreate(&user)

// Assign: 无论是否找到都设置（找到则更新）
db.Where(&User{Name: "张三"}).Assign(&User{Age: 18}).FirstOrCreate(&user)
```

## 终结 API 参考

### 创建

```go
db.Create(&value)                     // 插入，回填主键
db.CreateInBatches(&values, 100)      // 分批插入
```

### 查询

```go
db.First(&dest, conds...)             // 按主键正序 LIMIT 1，未找到返回 ErrRecordNotFound
db.Take(&dest, conds...)              // 不排序 LIMIT 1，未找到返回 ErrRecordNotFound
db.Last(&dest, conds...)              // 按主键倒序 LIMIT 1，未找到返回 ErrRecordNotFound
db.Find(&dest, conds...)              // 查询所有匹配记录
db.FindInBatches(&dest, batch, fc)    // 分批查询
db.FirstOrInit(&dest, conds...)       // 查不到则初始化 struct（不写库）
db.FirstOrCreate(&dest, conds...)     // 查不到则创建
db.Count(&count)                      // 计数
db.Pluck(column, &dest)               // 提取单列
db.Scan(&dest)                        // 扫描到自定义结构
db.ScanRows(rows, &dest)              // 从 *sql.Rows 扫描
db.Row()                              // 返回 *sql.Row
db.Rows()                             // 返回 *sql.Rows
```

### 更新

```go
db.Save(&value)                       // 有主键则更新（全字段），无主键则插入
db.Update(col, val)                   // 更新单列（触发钩子和时间追踪）
db.Updates(values)                    // 更新多列（struct 跳过零值，map 不跳过）
db.UpdateColumn(col, val)             // 更新单列（跳过钩子）
db.UpdateColumns(values)              // 更新多列（跳过钩子）
```

### 删除

```go
db.Delete(&value, conds...)           // 软删除
db.Unscoped().Delete(&value)          // 物理删除
```

### 事务

```go
db.Transaction(func(tx *gorm.DB) error { ... })  // 闭包事务
tx := db.Begin()                                  // 手动事务
tx.Commit() / tx.Rollback()
```

### 其他

```go
db.Exec(sql, vars...)                 // 执行原始 SQL
db.Raw(sql, vars...)                  // 原始 SQL 查询（链式 API，需配合 Scan/Find）
db.Connection(func(tx *gorm.DB) error { ... }) // 使用单个连接
```

## Session 模式

`db.Session(&gorm.Session{...})` 创建新的 DB 实例，复用底层连接池但使用独立配置：

```go
db.Session(&gorm.Session{
    DryRun:                 true,   // 仅生成 SQL，不执行
    PrepareStmt:            true,   // 使用预处理语句
    SkipHooks:              true,   // 跳过钩子
    SkipDefaultTransaction: true,   // 跳过默认事务包装
    AllowGlobalUpdate:      true,   // 允许全局更新
    QueryFields:            true,   // SELECT 时使用所有字段
    Context:                ctx,    // 设置 context
    NewDB:                  true,   // 创建全新的 DB（不继承之前的条件）
})

// 常用快捷方式
db.WithContext(ctx)  // 等同于 Session(&Session{Context: ctx})
db.Debug()           // 开启 Debug 日志
```
