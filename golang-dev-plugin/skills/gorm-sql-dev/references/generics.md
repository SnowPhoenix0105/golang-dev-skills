# Generic API — Complete Reference

> GORM 1.26+ provides a type-safe generic API `gorm.G[T]`. Compile-time model type checking; finisher methods (First/Find/Create, etc.) return concrete types instead of `interface{}`.

## Table of Contents

- [Entry Point](#entry-point)
- [Query (ExecInterface)](#query-execinterface)
- [Chain Methods (ChainInterface)](#chain-methods-chaininterface)
- [Create (CreateInterface)](#create-createinterface)
- [Set Assignments](#set-assignments)
- [Joins Builder](#joins-builder)
- [Preload Builder](#preload-builder)
- [Generic Interface Map](#generic-interface-map)

## Entry Point

```go
import "gorm.io/gorm"

// G[T] creates a generic query builder
// T must be a database model struct type
query := gorm.G[User](db)

// Optional: pass clause expressions
query := gorm.G[User](db, clause.Locking{Strength: "UPDATE"})
```

## Query (ExecInterface)

All finisher methods accept `context.Context`:

```go
// Single record — returns concrete type
user, err := gorm.G[User](db).First(ctx)
user, err := gorm.G[User](db).Take(ctx)
user, err := gorm.G[User](db).Last(ctx)

// Multiple records — returns []User
users, err := gorm.G[User](db).Where("age > ?", 18).Find(ctx)

// FindInBatches — fc directly receives []User
err := gorm.G[User](db).FindInBatches(ctx, 100, func(data []User, batch int) error {
    for _, u := range data {
        // process u
    }
    return nil
})

// Scan to custom struct
type UserDTO struct { Name string; Age int }
var dtos []UserDTO
err := gorm.G[User](db).Select("name", "age").Scan(ctx, &dtos)

// Row / Rows
row := gorm.G[User](db).Row(ctx)
rows, err := gorm.G[User](db).Rows(ctx)
```

## Chain Methods (ChainInterface)

```go
gorm.G[User](db).
    Where("age > ?", 18).
    Not("name = ?", "admin").
    Or("name = ?", "John").
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
        // custom scope
    }).
    Find(ctx)
```

## Create (CreateInterface)

`CreateInterface` is available at chain start (`Select`/`Omit` preserve `CreateInterface`):

```go
// Create single record
err := gorm.G[User](db).Create(ctx, &User{Name: "John", Age: 30})

// Batch create
err := gorm.G[User](db).CreateInBatches(ctx, &[]User{{Name: "John"}, {Name: "Jane"}}, 100)

// Select specific fields for create
err := gorm.G[User](db).Select("Name", "Age").Create(ctx, &user)

// Omit fields from create
err := gorm.G[User](db).Omit("Password").Create(ctx, &user)
```

## Set Assignments

`Set` is the bulk assignment entry point in the generic API. Its return type constrains which subsequent methods are available:

```go
// At start: Set returns SetCreateOrUpdateInterface[T]
// Both Update and Create are available
rows, err := gorm.G[User](db).
    Set(
        clause.Assignment{Column: clause.Column{Name: "name"}, Value: "Jane"},
        clause.Assignment{Column: clause.Column{Name: "age"}, Value: 31},
    ).
    Where("id = ?", 1).
    Update(ctx) // or .Create(ctx)

// After chaining: Set returns SetUpdateOnlyInterface[T]
// Only Update is available
rows, err := gorm.G[User](db).
    Where("id = ?", 1).
    Set(
        clause.Assignment{Column: clause.Column{Name: "name"}, Value: "Jane"},
    ).
    Update(ctx)
```

## Finisher Methods After Chaining

```go
// Delete
rows, err := gorm.G[User](db).Where("age < ?", 18).Delete(ctx)

// Update single column
rows, err := gorm.G[User](db).Where("id = ?", 1).Update(ctx, "Name", "Jane")

// Updates multiple columns (struct — zeros skipped)
rows, err := gorm.G[User](db).Where("id = ?", 1).Updates(ctx, User{Name: "Jane", Age: 31})

// Count
count, err := gorm.G[User](db).Where("age > ?", 18).Count(ctx, "id")

// Raw / Exec
users, err := gorm.G[User](db).Raw("SELECT * FROM users WHERE name = ?", "John").Find(ctx)
err := gorm.G[User](db).Exec(ctx, "UPDATE users SET active = 1 WHERE id = ?", 1)
```

## Joins Builder

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

## Preload Builder

```go
gorm.G[User](db).Preload("Orders", func(pb PreloadBuilder) error {
    pb.Where("state = ?", "paid").
       Select("id", "amount", "user_id").
       Order("created_at DESC").
       Limit(5).
       LimitPerRecord(3) // max 3 associated records per parent
    return nil
}).Find(ctx)
```

`LimitPerRecord` uses `ROW_NUMBER() OVER (PARTITION BY ...)`. Only works for non-Many2Many associations.

## Generic Interface Map

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

**Interface level control**:
- `Select`/`Omit` at chain start → `CreateInterface` (can `Create` afterward)
- `Select`/`Omit` mid-chain → `ChainInterface` (cannot `Create`)
- Scopes receive `func(db *Statement)` (unlike traditional `func(*DB)*DB`)
