# Models & CRUD — Complete Reference

## Table of Contents

- [Model Definition](#model-definition)
  - [gorm.Model](#gormmodel)
  - [Field Tags](#field-tags)
  - [Embedded Structs](#embedded-structs)
  - [Table Name Convention](#table-name-convention)
- [Create](#create)
- [Query](#query)
  - [Single Record](#single-record)
  - [Multiple Records](#multiple-records)
  - [Building Conditions](#building-conditions)
- [Update](#update)
- [Delete](#delete)
- [Chainable API Reference](#chainable-api-reference)
- [Finisher API Reference](#finisher-api-reference)
- [Session Mode](#session-mode)

## Model Definition

GORM models are plain Go structs with basic types, custom types implementing `Scanner`/`Valuer`, and nullable fields via pointers or `sql.Null*`.

```go
type User struct {
    ID           uint           `gorm:"primaryKey"`
    Name         string         `gorm:"size:100;not null"`
    Email        *string        // pointer — nullable
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
// gorm.Model definition
type Model struct {
    ID        uint `gorm:"primarykey"`
    CreatedAt time.Time
    UpdatedAt time.Time
    DeletedAt DeletedAt `gorm:"index"`
}
```

Embedding `gorm.Model` is equivalent to including those fields directly. If you don't need some fields, define your own instead.

### Field Tags

Tag format: `gorm:"key1:value1;key2:value2"`. Common tags:

**Column Definition**

| Tag | Description | Example |
| :--- | :--- | :--- |
| `column` | Column name | `gorm:"column:user_name"` |
| `type` | Column data type | `gorm:"type:varchar(128)"` |
| `size` | Column size | `gorm:"size:256"` |
| `precision` | Precision | `gorm:"precision:10"` |
| `scale` | Decimal scale | `gorm:"scale:2"` |
| `not null` | NOT NULL constraint | `gorm:"not null"` |
| `default` | Default value | `gorm:"default:18"` |
| `comment` | Column comment | `gorm:"comment:username"` |
| `serializer` | Serializer | `gorm:"serializer:json"` |

**Keys & Indexes**

| Tag | Description | Example |
| :--- | :--- | :--- |
| `primaryKey` | Primary key | `gorm:"primaryKey"` |
| `unique` | Unique constraint | `gorm:"unique"` |
| `uniqueIndex` | Unique index | `gorm:"uniqueIndex:idx_name"` |
| `index` | Regular index | `gorm:"index:idx_age"` |
| `uniqueIndex:,sort` | Composite index sort | `gorm:"uniqueIndex:,sort:desc"` |

**Associations**

| Tag | Description | Example |
| :--- | :--- | :--- |
| `foreignKey` | Custom foreign key | `gorm:"foreignKey:CompanyID"` |
| `references` | Referenced key | `gorm:"references:ID"` |
| `many2many` | Many-to-many join table | `gorm:"many2many:user_languages"` |
| `constraint` | Constraint behavior | `gorm:"constraint:OnUpdate:CASCADE,OnDelete:SET NULL"` |

**Behavior Control**

| Tag | Description |
| :--- | :--- |
| `<-` | Write-only |
| `->` | Read-only |
| `->:false;<-:create` | Create-only |
| `->:false;<-:update` | Update-only |
| `-` | Ignore this field |
| `-:migration` | Ignore during migration |
| `-:all` | Ignore in all operations |
| `autoCreateTime` | Auto-set timestamp on create (nano/milli/second) |
| `autoUpdateTime` | Auto-set timestamp on update |
| `embedded` | Embed struct |
| `embeddedPrefix` | Prefix for embedded fields |

### Embedded Structs

```go
type Author struct {
    Name  string
    Email string
}

type Blog struct {
    ID      int
    Author  Author `gorm:"embedded"`           // fields: Name, Email
    Author2 Author `gorm:"embedded;embeddedPrefix:author2_"` // fields: author2_Name, author2_Email
}
```

### Table Name Convention

Default: struct name → `snake_case` plural. Customize:

```go
// Method 1: implement Tabler interface
func (User) TableName() string {
    return "t_users"
}

// Method 2: ad-hoc
db.Table("t_users").First(&user)
```

## Create

```go
user := User{Name: "John", Age: 30}

// Single insert
result := db.Create(&user)
// user.ID is backfilled with database-generated primary key
// result.RowsAffected -> 1
// result.Error -> nil

// Batch insert
var users = []User{{Name: "John"}, {Name: "Jane"}}
db.Create(&users)

// Batch insert in batches
db.CreateInBatches(users, 100) // 100 per batch

// Select specific fields
db.Select("Name", "Age").Create(&user)
db.Omit("Age").Create(&user)

// Skip hooks
db.Session(&gorm.Session{SkipHooks: true}).Create(&user)

// Insert with map
db.Model(&User{}).Create(map[string]interface{}{
    "Name": "John", "Age": 18,
})
```

### Zero Value Problem

GORM skips zero-value fields (`0`, `""`, `false`, etc.) by default. Solutions:

```go
// Option 1: use pointer or sql.Null* types
type User struct {
    Name string
    Age  *int           // int -> *int
    Active sql.NullBool // bool -> sql.NullBool
}

// Option 2: use Select to specify fields explicitly
db.Select("Name", "Age", "Active").Create(&user)

// Option 3: use map
db.Model(&User{}).Create(map[string]interface{}{
    "Name": "John", "Age": 0, "Active": false,
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

## Query

### Single Record

```go
// First by primary key ascending
db.First(&user)              // SELECT * FROM users ORDER BY id LIMIT 1
db.First(&user, 1)           // WHERE id = 1
db.First(&user, "id = ?", 1) // WHERE id = 1

// Take (no ordering)
db.Take(&user)

// Last by primary key descending
db.Last(&user)

// First/Last/Take return ErrRecordNotFound when no record found
// Find does NOT return this error
```

To distinguish "not found" from other errors:

```go
err := db.First(&user, 99).Error
if errors.Is(err, gorm.ErrRecordNotFound) {
    // record not found
}
```

### Multiple Records

```go
// Find all
var users []User
db.Find(&users)

// With conditions
db.Where("age > ?", 18).Find(&users)

// Into map
var results []map[string]interface{}
db.Model(&User{}).Find(&results)

// Extract single column
var ages []int64
db.Model(&User{}).Pluck("age", &ages)

// Find in batches
db.FindInBatches(&users, 100, func(tx *gorm.DB, batch int) error {
    // process current batch
    return nil
})

// Scan to custom struct
type APIUser struct {
    Name string
    Age  int
}
var apiUsers []APIUser
db.Model(&User{}).Select("name", "age").Scan(&apiUsers)
```

### Building Conditions

```go
// String conditions
db.Where("name = ? AND age >= ?", "John", 18).Find(&users)

// Struct conditions (zero-value fields ignored)
db.Where(&User{Name: "John", Age: 18}).Find(&users)

// Map conditions
db.Where(map[string]interface{}{"name": "John", "age": 18}).Find(&users)

// IN
db.Where("name IN ?", []string{"John", "Jane"}).Find(&users)

// LIKE
db.Where("name LIKE ?", "%Jo%").Find(&users)

// BETWEEN
db.Where("age BETWEEN ? AND ?", 18, 30).Find(&users)

// NOT
db.Not("name = ?", "John").Find(&users)

// OR
db.Where("name = ?", "John").Or("name = ?", "Jane").Find(&users)

// Inline conditions (finisher directly)
db.First(&user, "name = ?", "John")
db.Find(&users, "age > ?", 18)
```

**Zero values in Where**: struct conditions skip zero-value fields. Use map or string conditions to include zero values.

## Update

```go
// Update single column (triggers hooks and timestamp tracking)
db.Model(&user).Update("name", "John")

// Update multiple columns (struct — zero values skipped)
db.Model(&user).Updates(User{Name: "John", Age: 0})

// Update multiple columns (map — zero values NOT skipped)
db.Model(&user).Updates(map[string]interface{}{"name": "John", "age": 0})

// Use Select for explicit control
db.Model(&user).Select("Name", "Age").Updates(User{Name: "John", Age: 0})
db.Model(&user).Select("*").Updates(User{Name: "John"}) // all fields including zeros

// Skip hooks and timestamp tracking
db.Model(&user).UpdateColumn("name", "John")
db.Model(&user).UpdateColumns(User{Name: "John", Age: 0})

// Batch update (WHERE condition required)
db.Model(&User{}).Where("age > ?", 18).Update("name", "John")

// Expression update
db.Model(&user).Update("age", gorm.Expr("age + ?", 1))
db.Model(&user).Updates(map[string]interface{}{"age": gorm.Expr("age + ?", 1)})

// Save: PK present → update all fields (including zeros); PK absent → insert
db.Save(&user)
```

> **Key difference**: `Update`/`Updates` trigger `BeforeUpdate`/`AfterUpdate` hooks and auto-update `UpdatedAt`; `UpdateColumn`/`UpdateColumns` skip hooks and timestamp tracking.

> **Safety guard**: unconditional `Update`/`Delete` returns `ErrMissingWhereClause`. Set `AllowGlobalUpdate: true` or use `db.Session(&gorm.Session{AllowGlobalUpdate: true})` for global updates.

## Delete

```go
// Soft delete (sets deleted_at)
db.Delete(&user)                      // by primary key
db.Delete(&user, 1)                   // by PK value
db.Where("age > ?", 80).Delete(&User{}) // batch soft delete

// Hard delete (bypass soft delete logic)
db.Unscoped().Delete(&user)

// Hard delete + skip hooks
db.Unscoped().Session(&gorm.Session{SkipHooks: true}).Delete(&user)
```

After soft delete, all queries automatically add `WHERE deleted_at IS NULL`. Use `Unscoped()` to include soft-deleted records.

## Chainable API Reference

### Model / Table

```go
db.Model(&User{})     // specify model for table inference
db.Table("users")     // specify table directly
db.Table("users AS u") // table alias
```

### Select / Omit

```go
db.Select("name", "age").Find(&users)
db.Select([]string{"name", "age"}).Find(&users)
db.Omit("password").Find(&users)
```

### Where / Not / Or

```go
db.Where("name = ?", "John")
db.Where(&User{Name: "John"})       // struct — zeros skipped
db.Where(map[string]any{"name": "John"}) // map — zeros not skipped
db.Not("age > ?", 20)
db.Or("name = ?", "Jane")
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
db.Limit(-1).Offset(-1) // cancel previous Limit/Offset
```

### Joins / InnerJoins

```go
db.Joins("Company").Find(&users)                // association name
db.Joins("JOIN companies ON companies.id = users.company_id").Find(&users) // raw SQL
db.InnerJoins("Company").Find(&users)
```

### Preload

```go
db.Preload("Orders").Find(&users)
db.Preload("Orders.Products").Find(&users)       // nested preload
db.Preload("Orders", "state = ?", "paid").Find(&users) // conditional
db.Preload("Orders", func(db *gorm.DB) *gorm.DB {
    return db.Order("created_at DESC")
}).Find(&users)
```

### Clauses

```go
// Inject clause expressions directly
db.Clauses(clause.Locking{Strength: "UPDATE"}).Find(&users)
db.Clauses(clause.OnConflict{UpdateAll: true}).Create(&users)
```

### Scopes

```go
// Reusable query logic
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
// Attrs: only set if record not found
db.Where(&User{Name: "John"}).Attrs(&User{Age: 18}).FirstOrCreate(&user)

// Assign: set regardless of whether found (updates if found)
db.Where(&User{Name: "John"}).Assign(&User{Age: 18}).FirstOrCreate(&user)
```

## Finisher API Reference

### Create

```go
db.Create(&value)                     // insert, backfill PK
db.CreateInBatches(&values, 100)      // batch insert
```

### Query

```go
db.First(&dest, conds...)             // PK ASC LIMIT 1, ErrRecordNotFound if none
db.Take(&dest, conds...)              // no ordering LIMIT 1, ErrRecordNotFound if none
db.Last(&dest, conds...)              // PK DESC LIMIT 1, ErrRecordNotFound if none
db.Find(&dest, conds...)              // find all matching
db.FindInBatches(&dest, batch, fc)    // batch query
db.FirstOrInit(&dest, conds...)       // init struct without writing to DB
db.FirstOrCreate(&dest, conds...)     // create if not found
db.Count(&count)                      // count
db.Pluck(column, &dest)               // extract single column
db.Scan(&dest)                        // scan to custom struct
db.ScanRows(rows, &dest)              // scan from *sql.Rows
db.Row()                              // return *sql.Row
db.Rows()                             // return *sql.Rows
```

### Update

```go
db.Save(&value)                       // PK → update all fields; no PK → insert
db.Update(col, val)                   // single column (hooks + timestamp)
db.Updates(values)                    // multiple columns (struct skips zeros, map doesn't)
db.UpdateColumn(col, val)             // single column (skip hooks)
db.UpdateColumns(values)              // multiple columns (skip hooks)
```

### Delete

```go
db.Delete(&value, conds...)           // soft delete
db.Unscoped().Delete(&value)          // hard delete
```

### Transactions

```go
db.Transaction(func(tx *gorm.DB) error { ... })  // closure transaction
tx := db.Begin()                                  // manual transaction
tx.Commit() / tx.Rollback()
```

### Other

```go
db.Exec(sql, vars...)                 // execute raw SQL
db.Raw(sql, vars...)                  // raw SQL query (chainable, pair with Scan/Find)
db.Connection(func(tx *gorm.DB) error { ... }) // use single connection
```

## Session Mode

`db.Session(&gorm.Session{...})` creates a new DB instance, reusing the underlying connection pool but with independent config:

```go
db.Session(&gorm.Session{
    DryRun:                 true,   // generate SQL without executing
    PrepareStmt:            true,   // use prepared statements
    SkipHooks:              true,   // skip hooks
    SkipDefaultTransaction: true,   // skip default transaction wrapping
    AllowGlobalUpdate:      true,   // allow global updates
    QueryFields:            true,   // SELECT all fields
    Context:                ctx,    // set context
    NewDB:                  true,   // fresh DB (no inherited conditions)
})

// Common shortcuts
db.WithContext(ctx)  // equivalent to Session(&Session{Context: ctx})
db.Debug()           // enable debug logging
```
