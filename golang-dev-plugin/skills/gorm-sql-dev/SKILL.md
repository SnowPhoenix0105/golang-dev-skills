---
name: gorm-sql-dev
description: |
  GORM SQL database ORM development. Use this skill when creating, modifying, or debugging Go database
  operations with GORM for CRUD, associations, transactions, migrations, and code generation.
  Trigger scenarios: mentions of gorm, database, MySQL, PostgreSQL, SQLite, CRUD, ORM, DAO,
  gen, gorm gen, gorm cli, association queries, Preload, transactions, AutoMigrate,
  gorm.Model, Hooks.
  Even without explicit mention of "gorm", consider this skill for any Go database development.
---

# GORM SQL Database Development

GORM (`gorm.io/gorm`) is the most popular full-featured ORM library for Go, offering declarative model definitions, chainable query API, association management, auto-migration, transactions, and hook system.

## GORM Ecosystem

| Repository | Purpose | Go Module |
| :--- | :--- | :--- |
| **gorm** | Core ORM engine: model definitions, CRUD, associations, migrations, transactions, hooks, Logger | `gorm.io/gorm` |
| **gen** | Code generation tool: generates DAO and Model from database tables, providing type-safe query APIs | `gorm.io/gen` |
| **cli** | Next-gen code generation CLI: leverages generics, provides `gorm` CLI command | `gorm.io/cli` |

Database drivers:
- **MySQL**: `gorm.io/driver/mysql`
- **PostgreSQL**: `gorm.io/driver/postgres`
- **SQLite**: `gorm.io/driver/sqlite`
- **SQL Server**: `gorm.io/driver/sqlserver`

Local source: use `go env GOMODCACHE` + `gorm.io/gorm@<version>` to locate cached modules.
Official docs site source is in the `gorm.io` repo (Hugo site).

## Core Principles

1. **Model-Driven** — Define database tables with Go structs; constraints and behaviors via tags
2. **Chainable API, Immutable Chains** — `db.Where().Order().Limit()` returns new `*gorm.DB` instances for safe reuse
3. **Zero Values Have Semantics** — `Create`/`Updates` skip zero-value fields by default; use `Select` or pointer fields to override
4. **Soft Delete by Default** — Models with `gorm.DeletedAt` field get soft-delete behavior from `Delete`
5. **Explicit Error Handling** — Always check `result.Error` or `result.RowsAffected` after operations
6. **`gorm.DB` Is a Connection Pool Wrapper** — Never close it manually; create new sessions with `Session()` as needed

## Minimal Example

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
    db.Create(&User{Name: "John", Age: 30})

    // Read
    var user User
    db.First(&user, 1)                   // by primary key
    db.First(&user, "name = ?", "John")  // by condition

    // Update
    db.Model(&user).Update("Age", 31)

    // Delete (soft delete)
    db.Delete(&user)
}
```

## API Quick Reference

### Chainable API

Build query conditions; each call returns a new `*gorm.DB` instance. Safe to reuse.

| Method | Purpose |
| :--- | :--- |
| `Model()` | Specify model for operation |
| `Table()` | Specify table name |
| `Where()` / `Not()` / `Or()` | Add conditions |
| `Select()` / `Omit()` | Select/exclude fields |
| `Order()` / `Group()` / `Having()` | Sort, group, filter |
| `Limit()` / `Offset()` | Pagination |
| `Joins()` / `InnerJoins()` | Join queries |
| `Preload()` | Preload associations |
| `Clauses()` | Inject raw clause expressions |
| `Scopes()` | Reuse query logic |
| `Unscoped()` | Skip soft-delete filter |
| `Distinct()` | Distinct |
| `Attrs()` / `Assign()` | For FirstOrCreate/FirstOrInit |
| `Debug()` | Enable debug logging |

Full signatures and usage: `references/model-crud.md`.

### Finisher API

Execute queries or operations, returning results.

| Method | Purpose |
| :--- | :--- |
| `Create()` / `CreateInBatches()` | Insert records |
| `Save()` | Save (update if PK present, else insert) |
| `First()` / `Take()` / `Last()` | Query single record |
| `Find()` / `FindInBatches()` | Query multiple records |
| `Update()` / `Updates()` | Update (triggers hooks) |
| `UpdateColumn()` / `UpdateColumns()` | Update (skip hooks) |
| `Delete()` | Delete (soft delete by default) |
| `Count()` | Count records |
| `Row()` / `Rows()` | Return `*sql.Row` / `*sql.Rows` |
| `Scan()` / `Pluck()` | Scan to custom struct / extract column |
| `FirstOrInit()` / `FirstOrCreate()` | Find or initialize/create |
| `Transaction()` | Execute in transaction |
| `Exec()` / `Raw()` | Execute raw SQL |

Full signatures and usage: `references/model-crud.md`.

## Core Concepts

### Model Definition

```go
type User struct {
    gorm.Model                        // Embeds: ID, CreatedAt, UpdatedAt, DeletedAt
    Name     string    `gorm:"size:100;not null;index"`     // varchar(100), NOT NULL, indexed
    Email    string    `gorm:"uniqueIndex;default:unknown"` // unique index, default value
    Age      int       `gorm:"default:18"`                  // default value
    Balance  float64   `gorm:"column:account_balance"`      // custom column name
}
```

`gorm.Model` provides: `ID`(uint, primarykey), `CreatedAt`(time.Time), `UpdatedAt`(time.Time), `DeletedAt`(gorm.DeletedAt, index).

Model definition, field tags, and table naming conventions: `references/model-crud.md`.

### Naming Strategy

Default `NamingStrategy` converts `CamelCase` to `snake_case`, and pluralizes table names. Full config options: `references/model-crud.md`.

```go
gorm.Open(sqlite.Open("test.db"), &gorm.Config{
    NamingStrategy: schema.NamingStrategy{
        SingularTable: true,   // disable table name pluralization
        TablePrefix:   "t_",   // table name prefix
        NoLowerCase:   false,
    },
})
```

### Associations

Four association types: `HasOne`, `HasMany`, `BelongsTo`, `Many2Many`.

```go
type User struct {
    gorm.Model
    CreditCard CreditCard   // Has One
    Orders     []Order      // Has Many
}

type Order struct {
    gorm.Model
    UserID   int            // Belongs To (default foreign key)
    User     User
    Products []Product      `gorm:"many2many:order_products"` // Many2Many
}
```

Full details: `references/associations.md`.

### Preloading

```go
db.Preload("Orders").Preload("Orders.Products").Find(&users)
db.Preload("Orders", "state = ?", "paid").Find(&users)   // with conditions
db.Joins("Company").Find(&users)                          // JOIN (single association)
```

Full details: `references/associations.md`.

### Transactions

```go
db.Transaction(func(tx *gorm.DB) error {
    err := tx.Create(&user).Error
    if err != nil {
        return err // return error → auto rollback
    }
    return nil // return nil → auto commit
})

// Manual control
tx := db.Begin()
// ...
tx.Commit() / tx.Rollback()
```

Full details: `references/transactions-migration.md`.

### Migrations

```go
db.AutoMigrate(&User{}, &Order{})           // auto create tables/add columns
db.Migrator().CreateTable(&Product{})       // manual create table
db.Migrator().AddColumn(&User{}, "Phone")   // add column
db.Migrator().CreateIndex(&User{}, "Name")  // create index
```

Full details: `references/transactions-migration.md`.

### Hooks

Models implement specific interface methods that are called before/after operations:

```go
func (u *User) BeforeCreate(tx *gorm.DB) error {
    u.UUID = uuid.New().String()
    return nil
}
```

Full hook list: `references/hooks-plugins-logger.md`.

### Generic Interface (v1.26+)

GORM 1.26+ provides type-safe generic API with compile-time field and type checking:

```go
// Generic queries
users, _ := gorm.G[User](db).Where("id > ?", 10).Find(ctx)
user, _ := gorm.G[User](db).First(ctx)

// Generic updates
rows, _ := gorm.G[User](db).Where("id = ?", 1).Update(ctx, "Name", "John")
```

Full details: `references/generics.md`.

## Code Generation Tools

### gen (`gorm.io/gen`)

Generates type-safe DAO code from database tables (code-generation based, not generics):

```go
// Configure generator
g := gen.NewGenerator(gen.Config{
    OutPath: "./query",
    Mode:    gen.WithDefaultQuery | gen.WithQueryInterface,
})

// Connect and generate
g.UseDB(db)
g.ApplyBasic(g.GenerateModel("users"), g.GenerateModel("orders"))
g.Execute()
```

Generated code uses the `field` package for type-safe field references and query building.

### cli (`gorm.io/cli`)

Next-gen CLI tool leveraging Go generics:

```bash
gorm gen                        # generate code from database
```

Full usage guides: `references/code-generation.md`.

## Reference Index

| Reference | Contents |
| :--- | :--- |
| `references/model-crud.md` | Model definitions, field tags, CRUD operations, complete chainable & finisher API reference |
| `references/associations.md` | HasOne, HasMany, BelongsTo, Many2Many, Preload, Association Mode |
| `references/advanced-queries.md` | Joins subqueries, Scopes, Clauses, Group/Having, Locking, DryRun, ToSQL |
| `references/transactions-migration.md` | Transactions, SavePoint, AutoMigrate, Migrator interface |
| `references/hooks-plugins-logger.md` | Lifecycle hooks, plugin system, Logger configuration, error handling |
| `references/generics.md` | `gorm.G[T]` generic API complete usage guide |
| `references/code-generation.md` | gen (code generation) and cli (CLI tool) usage guide |
| `references/troubleshooting.md` | Common issues and troubleshooting guide |

## Quick Troubleshooting

| Symptom | Common Cause |
| :--- | :--- |
| `ErrRecordNotFound` | `First`/`Last`/`Take` found no matching record (use `Find` instead to avoid this error) |
| `WHERE conditions required` | Performing unconditional `Update`/`Delete` without `AllowGlobalUpdate: true` |
| Zero-value fields not updated | `Updates` skips zero values by default; use `Select` explicitly or pointer field types |
| Soft-deleted records not found | Use `db.Unscoped()` to bypass soft-delete filter |
| Associations not loaded | Must explicitly call `Preload` or `Joins` — GORM does not auto-load associations |
| Nested transaction not rolling back | Check `DisableNestedTransaction` config; nested transactions use SavePoints |
| `AutoMigrate` no foreign keys | FK constraint creation is disabled by default; disable `DisableForeignKeyConstraintWhenMigrating` |
| Connection pool exhaustion | Check `SetMaxOpenConns`/`SetMaxIdleConns` on `sql.DB` |

Full troubleshooting guide: `references/troubleshooting.md`.

## Fallback

When this skill and reference materials cannot resolve the issue, **report the specifics to the user and ask for help** — do not silently guess. Include in your report:
- GORM version and database driver version
- Model definitions and the failing code snippet
- Complete error message or SQL log
- Expected SQL vs. actual SQL (use `Debug()` to obtain)
