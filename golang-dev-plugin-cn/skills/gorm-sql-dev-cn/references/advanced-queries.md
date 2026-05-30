# 高级查询完整参考

## 目录

- [子查询](#子查询)
- [命名参数](#命名参数)
- [Group By / Having](#group-by--having)
- [Distinct](#distinct)
- [Scopes](#scopes)
- [Clauses 构建器](#clauses-构建器)
- [行级锁](#行级锁)
- [索引与优化提示](#索引与优化提示)
- [DryRun 与 ToSQL](#dryrun-与-tosql)
- [Raw SQL](#raw-sql)
- [FindInBatches](#findinbatches)
- [Count](#count)

## 子查询

```go
// 在 WHERE 中使用子查询
subQuery := db.Model(&Order{}).Select("user_id").Where("amount > ?", 100)
db.Where("id IN (?)", subQuery).Find(&users)

// 在 FROM 中使用子查询
db.Table("(?) AS u", subQuery).Find(&users)

// 在 SELECT 中使用子查询
db.Model(&User{}).Select(
    "users.*",
    "(SELECT AVG(age) FROM users) AS avg_age",
).Find(&users)

// 在 Updates 中使用子查询
db.Model(&user).Update(
    "company_name",
    db.Model(&Company{}).Select("name").Where("id = users.company_id"),
)
```

## 命名参数

```go
// 使用 @name 语法
db.Where("name = @name AND age = @age", sql.Named("name", "张三"), sql.Named("age", 18)).Find(&users)

// 使用 map
db.Where("name = @name AND age = @age", map[string]interface{}{"name": "张三", "age": 18}).Find(&users)
```

## Group By / Having

```go
type Result struct {
    Name  string
    Total int64
}

// 分组统计
db.Model(&User{}).Select("name, SUM(age) as total").Group("name").Find(&results)

// 带 Having 的分组
db.Model(&User{}).Select("name, SUM(age) as total").
    Group("name").Having("SUM(age) > ?", 50).Find(&results)

// 多个聚合字段
db.Model(&User{}).Select(
    "name, SUM(age) as total, COUNT(*) as count",
).Group("name").Find(&results)
```

## Distinct

```go
// 去重查询
db.Distinct("name").Find(&users)

// 多列去重
db.Distinct("name", "age").Find(&users)

// 搭配 Pluck
var names []string
db.Model(&User{}).Distinct("name").Pluck("name", &names)
```

## Scopes

Scopes 是将常用查询逻辑封装为可复用函数的机制：

```go
// 定义 Scope
func AmountGreaterThan(minAmount float64) func(db *gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        return db.Where("amount > ?", minAmount)
    }
}

func PaidOrders(db *gorm.DB) *gorm.DB {
    return db.Where("state = ?", "paid")
}

func Paginate(page, pageSize int) func(db *gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        offset := (page - 1) * pageSize
        return db.Offset(offset).Limit(pageSize)
    }
}

// 使用
db.Scopes(PaidOrders, AmountGreaterThan(100), Paginate(1, 20)).Find(&orders)
```

## Clauses 构建器

`Clauses()` 允许直接注入 `clause.Expression`，用于 GORM API 不直接支持的高级特性：

```go
// ON CONFLICT (PostgreSQL / SQLite)
db.Clauses(clause.OnConflict{
    Columns:   []clause.Column{{Name: "id"}},
    DoUpdates: clause.AssignmentColumns([]string{"name", "age"}),
}).Create(&users)

// ON CONFLICT DO NOTHING
db.Clauses(clause.OnConflict{DoNothing: true}).Create(&user)

// RETURNING (PostgreSQL)
db.Clauses(clause.Returning{
    Columns: []clause.Column{{Name: "id"}, {Name: "created_at"}},
}).Create(&user)

// SET 表达式
db.Clauses(clause.Set{
    {Column: clause.Column{Name: "counter"}, Value: gorm.Expr("counter + ?", 1)},
}).UpdateColumn("name", "张三")

// 直接设置 Clause 表达式
db.Clauses(clause.Where{Exprs: []clause.Expression{
    clause.Eq{Column: "name", Value: "张三"},
}}).Find(&users)
```

## 行级锁

```go
// SELECT ... FOR UPDATE
db.Clauses(clause.Locking{Strength: "UPDATE"}).Find(&users)

// SELECT ... FOR SHARE（PostgreSQL）
db.Clauses(clause.Locking{Strength: "SHARE"}).Find(&users)

// NOWAIT
db.Clauses(clause.Locking{Strength: "UPDATE", Options: "NOWAIT"}).Find(&users)

// SKIP LOCKED
db.Clauses(clause.Locking{Strength: "UPDATE", Options: "SKIP LOCKED"}).Find(&users)
```

## 索引与优化提示

```go
// 使用指定索引（MySQL）
db.Clauses(hints.UseIndex("idx_user_name")).Find(&users)

// 强制使用索引
db.Clauses(hints.ForceIndex("idx_user_name")).Find(&users)

// 忽略索引
db.Clauses(hints.IgnoreIndex("idx_user_name")).Find(&users)

// 注释提示
db.Clauses(hints.Comment("select", "controller", "user_list")).Find(&users)

// 多个 hints
db.Clauses(
    hints.UseIndex("idx_user_name"),
    hints.Comment("select", "batch"),
).Find(&users)
```

## DryRun 与 ToSQL

### DryRun 模式

```go
// Session 级别
stmt := db.Session(&gorm.Session{DryRun: true}).First(&user).Statement
sql := stmt.SQL.String()        // 生成的 SQL
vars := stmt.Vars               // 参数值

// Config 级别（全局）
db, _ := gorm.Open(sqlite.Open("test.db"), &gorm.Config{DryRun: true})
```

### ToSQL

```go
sql := db.ToSQL(func(tx *gorm.DB) *gorm.DB {
    return tx.Model(&User{}).Where("age > ?", 18).
        Limit(10).Offset(5).
        Order("name ASC").
        Find(&users)
})
// 返回完整的 SQL 语句（含参数值替换）
```

## Raw SQL

```go
// 原始 SQL 查询
var users []User
db.Raw("SELECT * FROM users WHERE name = ?", "张三").Scan(&users)

// 原始 SQL 执行
db.Exec("UPDATE users SET name = ? WHERE id = ?", "张三", 1)
db.Exec("DELETE FROM users WHERE id = ?", 1)

// 命名参数
db.Raw("SELECT * FROM users WHERE name = @name", sql.Named("name", "张三")).Scan(&users)

// Raw + 链式 API
db.Raw("SELECT * FROM users WHERE age > ?", 18).Order("name").Scan(&users)
```

## FindInBatches

```go
batchSize := 100
result := db.FindInBatches(&users, batchSize, func(tx *gorm.DB, batch int) error {
    // 处理当前批次的数据
    for _, user := range users {
        // process user
    }

    // 可以保存当前批次的处理进度
    tx.Save(&users)

    return nil
})
```

`FindInBatches` 依赖主键进行分页，模型必须有主键字段。

## Count

```go
// 基础计数
var count int64
db.Model(&User{}).Where("age > ?", 18).Count(&count)

// DISTINCT 计数
db.Model(&User{}).Distinct("name").Count(&count)

// Count 与 Group By
// 当使用 Group By 时，Count 返回 RowsAffected（即 group 数量）
var result []struct {
    Name  string
    Total int64
}
db.Model(&User{}).Select("name, COUNT(age) as total").Group("name").Find(&result)

// Count 会忽略 ORDER BY 提升性能
// Count 会忽略 Limit 和 Offset
```

### gorm.Expr

```go
// 在条件、赋值、SELECT 中使用 SQL 表达式
db.Where("created_at > ?", gorm.Expr("NOW() - INTERVAL '1 day'")).Find(&users)
db.Model(&user).Updates(map[string]interface{}{
    "counter": gorm.Expr("counter + ?", 1),
})
db.Model(&User{}).Select("name", gorm.Expr("YEAR(birthday) as birth_year")).Find(&results)
```
