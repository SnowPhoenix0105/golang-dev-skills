# Code Generation Tools — Complete Reference

## Table of Contents

- [gen Code Generator](#gen-code-generator)
  - [Generator Configuration](#generator-configuration)
  - [Generating Models](#generating-models)
  - [Field Type System](#field-type-system)
  - [DO Data Object](#do-data-object)
  - [Query Examples](#query-examples)
  - [Advanced Features](#advanced-features)
- [cli CLI Tool](#cli-cli-tool)
- [gen vs cli Comparison](#gen-vs-cli-comparison)

## gen Code Generator

`gorm.io/gen` generates type-safe DAO code from database tables. Generated code includes:
- **Model structs**: Go structs mapped to database tables
- **Query types**: Type-safe query builders using the `field` package

### Generator Configuration

```go
package main

import (
    "gorm.io/driver/mysql"
    "gorm.io/gen"
    "gorm.io/gorm"
)

func main() {
    db, _ := gorm.Open(mysql.Open(dsn))

    g := gen.NewGenerator(gen.Config{
        OutPath:           "./dal/query",
        ModelPkgPath:      "./dal/model",
        OutFile:           "gen.go",
        WithUnitTest:      false,
        FieldNullable:     false,
        FieldCoverable:    true,
        FieldSignable:     true,
        FieldWithIndexTag: true,
        FieldWithTypeTag:  true,
        FieldWithDefaultTag: true,
        Mode: gen.WithDefaultQuery | gen.WithQueryInterface | gen.WithoutContext,
    })

    g.UseDB(db)

    g.ApplyBasic(
        g.GenerateModel("users"),
        g.GenerateModel("orders",
            gen.FieldType("amount", "decimal.Decimal"),
            gen.FieldIgnore("internal_notes"),
        ),
        g.GenerateModelAs("products", "Product"),
    )

    g.Execute()
}
```

### Generate Modes

```go
const (
    WithDefaultQuery   // generate default query with built-in CRUD
    WithoutContext     // don't require context
    WithQueryInterface // generate exported query interface
    WithGeneric        // use generics
)
```

### Generating Models

```go
// Basic model generation
g.ApplyBasic(
    g.GenerateModel("users"),
    g.GenerateModelAs("users", "User"),                // custom model name
    g.GenerateModel("users",
        gen.FieldType("amount", "decimal.Decimal"),    // custom Go type
        gen.FieldIgnore("password_hash"),              // ignore field
        gen.FieldNewTag("mobile", `gorm:"uniqueIndex"`),
        gen.FieldRename("old_name", "new_name"),       // rename field
        gen.FieldGORMTag("status", `gorm:"-"`),        // override gorm tag
    ),
)

// Generate all tables
g.ApplyBasic(
    g.GenerateAllTable(
        gen.FieldType("amount", "decimal.Decimal"),
    ),
)

// Filter specific tables with options
g.ApplyBasic(
    g.GenerateAllTable(
        g.GenerateModelAs("users", "User"),
        g.GenerateModel("orders"),
    ),
)
```

### Field Type System

```go
// Generated field types
type Field interface {
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
    Value(val interface{}) AssignExpr
    ColumnName() string
}
```

Generated field types: `field.String`, `field.Int64`, `field.Int`, `field.Uint64`, `field.Uint`, `field.Float64`, `field.Float32`, `field.Bool`, `field.Time`, `field.Bytes`.

### DO Data Object

`DO` (Data Object) is the base type for gen-generated query builders, embedding `*gorm.DB`:

```go
u := query.User
o := query.Order

u.UseDB(db)
u.ReplaceConnPool(slaveDB.ConnPool) // for read/write splitting
```

### Query Examples

```go
q := query.Use(db)
u := q.User

// Create
user := model.User{Name: "John", Age: 30}
err := u.Create(&user)

// Query
user, err := u.Where(u.Name.Eq("John")).First()
users, err := u.Where(u.Age.Gt(18)).Order(u.Age.Desc()).Find()

// Compound conditions
users, err := u.Where(
    u.Age.Gt(18),
    u.Name.Like("%John%"),
).Find()

// OR
users, err := u.Where(u.Age.Lt(18)).Or(u.Age.Gt(60)).Find()

// IN
users, err := u.Where(u.Name.In("John", "Jane")).Find()

// Update
info, err := u.Where(u.ID.Eq(1)).Update(u.Name, "Jane")
info, err := u.Where(u.ID.Eq(1)).UpdateSimple(
    u.Name.Value("Jane"),
    u.Age.Value(31),
)

// Update skip hooks
info, err := u.Where(u.ID.Eq(1)).UpdateColumn(u.Name, "Jane")
info, err := u.Where(u.ID.Eq(1)).UpdateColumnSimple(u.Name.Value("Jane"))

// Delete
info, err := u.Where(u.ID.Eq(1)).Delete()
info, err := u.Where(u.ID.In(1, 2, 3)).Delete()

// Count
count, err := u.Where(u.Age.Gt(18)).Count()

// Subquery
subQuery := u.Select(u.ID).Where(u.Age.Gt(18))
orders, err := o.Where(o.Columns(o.UserID).In(subQuery)).Find()
```

### Advanced Features

#### Association Operations

```go
u := query.User
err := u.Roles.Model().Append(&Role{Name: "admin"})
err := u.Roles.Model().Replace(&roles)
err := u.Roles.Model().Clear()
```

#### Smart Field Select

gen auto-selects fields being updated:

```go
u := query.User
info, err := u.Where(u.ID.Eq(1)).UpdateSimple(
    u.Name.Value("Jane"),
    u.Age.Value(31),
)
// Automatically generates: UPDATE users SET name='Jane', age=31 WHERE id=1
```

#### Transactions

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

## cli CLI Tool

`gorm.io/cli` is the next-gen CLI tool centered on the `gorm gen` command.

### Install & Usage

```bash
go install gorm.io/cli@latest

# Generate code
gorm gen

# View version
gorm version
```

### cli vs gen

cli is built on gen but leverages Go generics for a simpler CLI experience.

## gen vs cli Comparison

| Feature | gen | cli |
| :--- | :--- | :--- |
| API Style | Code gen + `field` package | Go generics |
| Usage | Go code calling Generator API | CLI command + Go config |
| Type Safety | Compile-time (field expressions) | Compile-time (generics) |
| Learning Curve | Medium (field type system) | Lower (closer to raw GORM) |
| Maturity | Mature, well-documented | Relatively new |
| Recommended For | Large projects needing fine-grained control | New projects preferring generics |

> **Guidance**: try cli first for new projects (generics + simple CLI); keep gen for existing large projects (mature and stable).
