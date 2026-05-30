# Advanced Queries — Complete Reference

## Table of Contents

- [Subqueries](#subqueries)
- [Named Arguments](#named-arguments)
- [Group By / Having](#group-by--having)
- [Distinct](#distinct)
- [Scopes](#scopes)
- [Clauses Builder](#clauses-builder)
- [Row-Level Locking](#row-level-locking)
- [Index & Optimizer Hints](#index--optimizer-hints)
- [DryRun & ToSQL](#dryrun--tosql)
- [Raw SQL](#raw-sql)
- [FindInBatches](#findinbatches)
- [Count](#count)

## Subqueries

```go
// Subquery in WHERE
subQuery := db.Model(&Order{}).Select("user_id").Where("amount > ?", 100)
db.Where("id IN (?)", subQuery).Find(&users)

// Subquery in FROM
db.Table("(?) AS u", subQuery).Find(&users)

// Subquery in SELECT
db.Model(&User{}).Select(
    "users.*",
    "(SELECT AVG(age) FROM users) AS avg_age",
).Find(&users)

// Subquery in Updates
db.Model(&user).Update(
    "company_name",
    db.Model(&Company{}).Select("name").Where("id = users.company_id"),
)
```

## Named Arguments

```go
// @name syntax
db.Where("name = @name AND age = @age", sql.Named("name", "John"), sql.Named("age", 18)).Find(&users)

// Map
db.Where("name = @name AND age = @age", map[string]any{"name": "John", "age": 18}).Find(&users)
```

## Group By / Having

```go
type Result struct {
    Name  string
    Total int64
}

db.Model(&User{}).Select("name, SUM(age) as total").Group("name").Find(&results)

db.Model(&User{}).Select("name, SUM(age) as total").
    Group("name").Having("SUM(age) > ?", 50).Find(&results)
```

## Distinct

```go
db.Distinct("name").Find(&users)
db.Distinct("name", "age").Find(&users)
```

## Scopes

Reusable query logic:

```go
func AmountGreaterThan(minAmount float64) func(db *gorm.DB) *gorm.DB {
    return func(db *gorm.DB) *gorm.DB {
        return db.Where("amount > ?", minAmount)
    }
}

func PaidOrders(db *gorm.DB) *gorm.DB {
    return db.Where("state = ?", "paid")
}

db.Scopes(PaidOrders, AmountGreaterThan(100), Paginate(1, 20)).Find(&orders)
```

## Clauses Builder

`Clauses()` injects `clause.Expression` for advanced features not directly supported by GORM's API:

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

// SET expression
db.Clauses(clause.Set{
    {Column: clause.Column{Name: "counter"}, Value: gorm.Expr("counter + ?", 1)},
}).UpdateColumn("name", "John")
```

## Row-Level Locking

```go
// SELECT ... FOR UPDATE
db.Clauses(clause.Locking{Strength: "UPDATE"}).Find(&users)

// SELECT ... FOR SHARE (PostgreSQL)
db.Clauses(clause.Locking{Strength: "SHARE"}).Find(&users)

// NOWAIT
db.Clauses(clause.Locking{Strength: "UPDATE", Options: "NOWAIT"}).Find(&users)

// SKIP LOCKED
db.Clauses(clause.Locking{Strength: "UPDATE", Options: "SKIP LOCKED"}).Find(&users)
```

## Index & Optimizer Hints

```go
// Use index (MySQL)
db.Clauses(hints.UseIndex("idx_user_name")).Find(&users)

// Force index
db.Clauses(hints.ForceIndex("idx_user_name")).Find(&users)

// Ignore index
db.Clauses(hints.IgnoreIndex("idx_user_name")).Find(&users)

// Comment hint
db.Clauses(hints.Comment("select", "controller", "user_list")).Find(&users)
```

## DryRun & ToSQL

### DryRun Mode

```go
// Session level
stmt := db.Session(&gorm.Session{DryRun: true}).First(&user).Statement
sql := stmt.SQL.String()
vars := stmt.Vars

// Config level (global)
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
// Returns complete SQL with parameter values substituted
```

## Raw SQL

```go
var users []User
db.Raw("SELECT * FROM users WHERE name = ?", "John").Scan(&users)

db.Exec("UPDATE users SET name = ? WHERE id = ?", "John", 1)
db.Exec("DELETE FROM users WHERE id = ?", 1)

// Named arguments
db.Raw("SELECT * FROM users WHERE name = @name", sql.Named("name", "John")).Scan(&users)

// Raw + chainable API
db.Raw("SELECT * FROM users WHERE age > ?", 18).Order("name").Scan(&users)
```

## FindInBatches

```go
result := db.FindInBatches(&users, 100, func(tx *gorm.DB, batch int) error {
    for _, user := range users {
        // process
    }
    return nil
})
```

`FindInBatches` relies on the primary key for pagination. Model must have a primary key field.

## Count

```go
var count int64
db.Model(&User{}).Where("age > ?", 18).Count(&count)

// DISTINCT count
db.Model(&User{}).Distinct("name").Count(&count)

// Count ignores ORDER BY and Limit/Offset
```

### gorm.Expr

```go
// SQL expressions in conditions, assignments, SELECT
db.Where("created_at > ?", gorm.Expr("NOW() - INTERVAL '1 day'")).Find(&users)
db.Model(&user).Updates(map[string]interface{}{
    "counter": gorm.Expr("counter + ?", 1),
})
db.Model(&User{}).Select("name", gorm.Expr("YEAR(birthday) as birth_year")).Find(&results)
```
