# Transactions & Migrations — Complete Reference

## Table of Contents

- [Transactions](#transactions)
  - [Closure Transaction](#closure-transaction)
  - [Manual Transaction](#manual-transaction)
  - [Nested Transactions & SavePoint](#nested-transactions--savepoint)
  - [Transaction Configuration](#transaction-configuration)
- [Migrations](#migrations)
  - [AutoMigrate](#automigrate)
  - [Migrator Interface](#migrator-interface)
  - [Table Operations](#table-operations)
  - [Column Operations](#column-operations)
  - [Index Operations](#index-operations)
  - [Constraint Operations](#constraint-operations)
  - [View Operations](#view-operations)

## Transactions

### Closure Transaction

Recommended approach — auto handles Commit/Rollback:

```go
err := db.Transaction(func(tx *gorm.DB) error {
    if err := tx.Create(&user).Error; err != nil {
        return err // any returned error triggers Rollback
    }
    if err := tx.Create(&order).Error; err != nil {
        return err
    }
    // return nil → auto Commit
    return nil
})
```

**Panic-safe**: panics inside the closure automatically trigger Rollback.

### Manual Transaction

```go
tx := db.Begin()
if tx.Error != nil {
    return tx.Error
}

if err := tx.Create(&user).Error; err != nil {
    tx.Rollback()
    return err
}

tx.Commit()
```

### Transaction Options

```go
// Set isolation level
tx := db.Begin(&sql.TxOptions{
    Isolation: sql.LevelSerializable,
    ReadOnly:  true,
})

// Closure + options
err := db.Transaction(func(tx *gorm.DB) error {
    // ...
}, &sql.TxOptions{Isolation: sql.LevelReadCommitted})
```

### Nested Transactions & SavePoint

```go
// Inner transactions use SavePoints, not true nested transactions
db.Transaction(func(tx *gorm.DB) error {
    tx.Create(&user)

    // Nested transaction
    tx.Transaction(func(tx2 *gorm.DB) error {
        tx2.Create(&order)
        return nil // SavePoint released (committed)
    })

    // Inner rollback doesn't affect outer
    tx.Transaction(func(tx2 *gorm.DB) error {
        tx2.Create(&badOrder)
        return errors.New("rollback this savepoint only")
    })

    return nil // outer commits (user is persisted)
})
```

**Disable nested transactions**:

```go
db, _ := gorm.Open(mysql.Open(dsn), &gorm.Config{
    DisableNestedTransaction: true,
})
// Or session level
db.Session(&gorm.Session{DisableNestedTransaction: true})
```

When disabled, nested `Transaction` calls reuse the same transaction without SavePoints.

### SavePoint Manual Operations

```go
tx := db.Begin()
tx.Create(&user)
tx.SavePoint("sp1")
tx.Create(&order)
tx.RollbackTo("sp1")   // order is rolled back
tx.Commit()            // user is committed
```

### Transaction Configuration

```go
db, _ := gorm.Open(mysql.Open(dsn), &gorm.Config{
    SkipDefaultTransaction:    false,
    DefaultTransactionTimeout: 10 * time.Second,
})
```

## Migrations

### AutoMigrate

`AutoMigrate` automatically migrates schema: creates missing tables, adds missing columns and indexes. It does **NOT** drop existing columns or change column types.

```go
db.AutoMigrate(&User{}, &Order{}, &Product{})
db.Migrator().AutoMigrate(&User{})
```

**AutoMigrate limitations**:
- Won't drop unused columns
- Won't alter column types
- Won't create FK constraints by default

### Migrator Interface

Full interface available via `db.Migrator()`.

### Table Operations

```go
db.Migrator().CreateTable(&User{})
db.Migrator().DropTable(&User{})
db.Migrator().DropTable("users")
db.Migrator().HasTable(&User{})
db.Migrator().HasTable("users")
db.Migrator().RenameTable("users", "t_users")
tableList, _ := db.Migrator().GetTables()
```

### Column Operations

```go
db.Migrator().AddColumn(&User{}, "Phone")
db.Migrator().DropColumn(&User{}, "Phone")
db.Migrator().AlterColumn(&User{}, "Phone")
db.Migrator().RenameColumn(&User{}, "Phone", "Tel")
db.Migrator().HasColumn(&User{}, "Phone")

columns, _ := db.Migrator().ColumnTypes(&User{})
for _, col := range columns {
    name := col.Name()
    dbType := col.DatabaseTypeName()
    nullable, _ := col.Nullable()
}
```

### Index Operations

```go
db.Migrator().CreateIndex(&User{}, "Name")
db.Migrator().CreateIndex(&User{}, "idx_name_age")
db.Migrator().DropIndex(&User{}, "Name")
db.Migrator().HasIndex(&User{}, "Name")
db.Migrator().RenameIndex(&User{}, "old_name", "new_name")
indexes, _ := db.Migrator().GetIndexes(&User{})
```

### Constraint Operations

```go
db.Migrator().CreateConstraint(&User{}, "chk_user_age")
db.Migrator().DropConstraint(&User{}, "chk_user_age")
db.Migrator().HasConstraint(&User{}, "chk_user_age")
```

### View Operations

```go
db.Migrator().CreateView("active_users", gorm.ViewOption{
    Query: db.Model(&User{}).Where("active = ?", true),
})

db.Migrator().CreateView("active_users", gorm.ViewOption{
    Replace: true,
    Query:   db.Model(&User{}).Where("active = ? AND age > ?", true, 18),
})

db.Migrator().DropView("active_users")
```

### Custom Migration Logic

For changes AutoMigrate can't handle, write custom migration logic:

```go
m := db.Migrator()

if m.HasTable(&User{}) {
    if !m.HasColumn(&User{}, "Phone") {
        m.AddColumn(&User{}, "Phone")
    }
} else {
    m.CreateTable(&User{})
}

// Data migration
if m.HasColumn(&User{}, "OldField") {
    db.Exec("UPDATE users SET new_field = old_field")
    m.DropColumn(&User{}, "OldField")
}
```

### FK Constraint Configuration

```go
db, _ := gorm.Open(mysql.Open(dsn), &gorm.Config{
    DisableForeignKeyConstraintWhenMigrating: false,
})

// Ignore relationships during migration
db, _ := gorm.Open(mysql.Open(dsn), &gorm.Config{
    IgnoreRelationshipsWhenMigrating: true,
})
```
