# Associations — Complete Reference

## Table of Contents

- [Association Types](#association-types)
  - [Belongs To](#belongs-to)
  - [Has One](#has-one)
  - [Has Many](#has-many)
  - [Many To Many](#many-to-many)
- [Preload](#preload)
- [Joins](#joins)
- [Association Mode](#association-mode)
- [Association Operations](#association-operations)
- [Foreign Key & Reference Conventions](#foreign-key--reference-conventions)

## Association Types

### Belongs To

```go
type User struct {
    gorm.Model
    Name      string
    CompanyID int      // foreign key
    Company   Company  // Belongs To
}

type Company struct {
    ID   int
    Name string
}
```

**Convention**: FK name = `{field}ID` (e.g., `CompanyID`); referenced key = associated model's `ID`.

Custom foreign key:

```go
type User struct {
    gorm.Model
    Name       string
    CompanyRef string
    Company    Company `gorm:"foreignKey:CompanyRef;references:Code"`
}

type Company struct {
    ID   int
    Code string // referenced key
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
    UserID uint   // foreign key
}
```

**Convention**: FK name = `{owner}ID` (e.g., `UserID`).

### Has Many

```go
type User struct {
    gorm.Model
    Orders []Order // Has Many
}

type Order struct {
    gorm.Model
    UserID int    // foreign key
    Amount float64
}
```

### Many To Many

```go
type User struct {
    gorm.Model
    Languages []Language `gorm:"many2many:user_languages"` // join table
}

type Language struct {
    gorm.Model
    Name string
}
```

Custom FK:

```go
type User struct {
    gorm.Model
    Languages []Language `gorm:"many2many:user_languages;foreignKey:UserID;joinForeignKey:UserRefer;references:Code;joinReferences:LangCode"`
}
```

Where:
- `foreignKey`: current model's FK in join table
- `references`: associated model's referenced field
- `joinForeignKey`: join table column pointing to current model
- `joinReferences`: join table column pointing to associated model

## Preload

GORM does NOT auto-load associations. You must explicitly call `Preload` or `Joins`.

```go
// Single layer
db.Preload("Orders").Find(&users)

// Nested
db.Preload("Orders.Products").Find(&users)

// Multiple associations
db.Preload("Orders").Preload("CreditCard").Find(&users)
```

### Conditional Preload

```go
// String condition
db.Preload("Orders", "state = ?", "paid").Find(&users)

// Custom query
db.Preload("Orders", func(db *gorm.DB) *gorm.DB {
    return db.Order("created_at DESC").Limit(3)
}).Find(&users)

// Nested + condition
db.Preload("Orders.Products", "active = ?", true).Find(&users)
```

### Preload All

```go
db.Preload(clause.Associations).Find(&users)
```

## Joins

`Joins` uses SQL JOIN to filter by association conditions. Association data is loaded into nested fields (only for Has One and Belongs To).

```go
// Join by association name
db.Joins("Company").Find(&users)

// Join with filter + load association data
db.Joins("Company", db.Where(&Company{Name: "ACME"})).Find(&users)

// INNER JOIN
db.InnerJoins("Company").Find(&users)

// Raw SQL JOIN
db.Joins("LEFT JOIN companies ON companies.id = users.company_id").Find(&users)
```

### Preload vs Joins

| | Preload | Joins |
| :--- | :--- | :--- |
| Loads association data | Yes | Only Has One / Belongs To |
| Filter by association condition | No (only limits preloaded scope) | Yes |
| SQL execution | Multiple queries | Single JOIN query |
| Association types | All | Has One, Belongs To |
| N+1 problem | Avoided (batch queries) | N/A |

## Association Mode

Get `*gorm.Association` for association operations:

```go
var user User
db.First(&user)
db.Model(&user).Association("Orders")
```

### Query Associations

```go
var orders []Order
db.Model(&user).Association("Orders").Find(&orders)
```

### Append

```go
db.Model(&user).Association("Orders").Append(&Order{Amount: 100})
db.Model(&user).Association("Orders").Append(&orders) // batch
```

### Replace

```go
db.Model(&user).Association("Orders").Replace([]Order{newOrder1, newOrder2})
```

### Delete

```go
// Has Many / Has One: sets FK to NULL
// Many2Many: deletes join table records (not the associated model records)
db.Model(&user).Association("Orders").Delete(&orders)
```

### Clear

```go
// Clears all associations
db.Model(&user).Association("Orders").Clear()
// Has Many / Has One: sets FK to NULL
// Many2Many: deletes join table records
```

### Count

```go
count := db.Model(&user).Association("Orders").Count()
```

## Association Operations

### Create with Associations

```go
db.Create(&User{
    Name: "John",
    Orders: []Order{{Amount: 100}, {Amount: 200}},
})

// Skip association creation
db.Session(&gorm.Session{FullSaveAssociations: false}).Create(&user)
// or set db.Config.FullSaveAssociations = false (default)
```

### Select / Omit Control

```go
// Skip Orders association on create
db.Omit("Orders").Create(&user)

// Select specific associations
db.Select("Orders").Create(&user)
```

## Foreign Key & Reference Conventions

GORM default FK conventions:

| Association | Default FK Name | FK Table |
| :--- | :--- | :--- |
| Belongs To | `{field}ID` | Current table |
| Has One | `{owner}ID` | Associated table |
| Has Many | `{owner}ID` | Associated table |
| Many2Many | Join table `{table1}_{table2}` | Join table |

Referenced key defaults to the first primary key field of the associated model.

### FK Constraints

```go
type User struct {
    gorm.Model
    Orders []Order `gorm:"constraint:OnUpdate:CASCADE,OnDelete:SET NULL"`
}
```

AutoMigrate does NOT create FK constraints by default. Set `DisableForeignKeyConstraintWhenMigrating: false` to enable.

### Polymorphism

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
    OwnerType string // "cats" or "dogs"
}
```
