# Troubleshooting Guide

## Table of Contents

- [Query Issues](#query-issues)
- [Write Issues](#write-issues)
- [Association Issues](#association-issues)
- [Migration Issues](#migration-issues)
- [Transaction Issues](#transaction-issues)
- [Performance Issues](#performance-issues)
- [Connection Issues](#connection-issues)
- [Error Quick Reference](#error-quick-reference)

## Query Issues

### `First`/`Last`/`Take` returns `ErrRecordNotFound`

This is expected behavior — these methods return this error when no record is found. To avoid:

```go
// Use Find instead (doesn't return ErrRecordNotFound)
var users []User
db.Where("id = ?", 999).Find(&users)

// Or check explicitly
if errors.Is(err, gorm.ErrRecordNotFound) {
    // handle not found
}
```

### Soft-deleted records not appearing

```go
// Use Unscoped to bypass soft delete filter
db.Unscoped().Find(&users)

// Only find deleted records
db.Unscoped().Where("deleted_at IS NOT NULL").Find(&users)
```

### Preload not working

GORM does **NOT** auto-load associations. Always call `Preload` explicitly:

```go
// ❌ Associations won't load
db.Find(&users)

// ✅ Explicit preload
db.Preload("Orders").Find(&users)
```

### Unexpected query results

Enable Debug to see the generated SQL:

```go
db.Debug().Where("name = ?", "John").Find(&users)
// Output: [2024-01-01 12:00:00] [2.00ms] SELECT * FROM `users` WHERE name = 'John'
```

## Write Issues

### Zero-value fields not being updated

`Updates(struct)` skips zero values by default. Solutions:

```go
// Option 1: use map
db.Model(&user).Updates(map[string]any{"age": 0, "active": false})

// Option 2: use Select
db.Model(&user).Select("Age", "Active").Updates(User{Age: 0, Active: false})
db.Model(&user).Select("*").Updates(User{Age: 0})

// Option 3: pointer fields in model
type User struct {
    Age    *int
    Active *bool
}
```

### Batch update/delete error: `WHERE conditions required`

GORM's safety guard prevents accidental full-table updates:

```go
// ❌ No WHERE condition
db.Model(&User{}).Update("name", "John") // ErrMissingWhereClause

// ✅ Add condition
db.Model(&User{}).Where("id = ?", 1).Update("name", "John")

// ✅ Allow global update
db.Session(&gorm.Session{AllowGlobalUpdate: true}).Model(&User{}).Update("name", "John")
```

### `Save` inserts instead of updating

`Save` decides based on whether the PK is a zero value:

```go
user := User{Name: "John"} // ID is zero
db.Save(&user)             // INSERT, not UPDATE

// ✅ Correct
user := User{ID: 1, Name: "Jane"}
db.Save(&user)             // UPDATE
```

### Primary key not backfilled after create

Ensure you pass a pointer to `Create`:

```go
user := User{Name: "John"}
db.Create(&user) // ✅ user.ID is backfilled
// ❌ db.Create(user) — won't backfill
```

## Association Issues

### Many2Many association creation fails

Ensure join table name follows convention (alphabetically sorted model names):

```go
// Default join table: user_languages
db.AutoMigrate(&User{}, &Language{})

// Custom join table
type User struct {
    Languages []Language `gorm:"many2many:custom_table"`
}
```

### Foreign key constraints not created

AutoMigrate does **NOT** create FK constraints by default:

```go
db, _ := gorm.Open(mysql.Open(dsn), &gorm.Config{
    DisableForeignKeyConstraintWhenMigrating: false,
})
```

### `FullSaveAssociations` causes unexpected updates

`Create`/`Updates` do not handle associations by default. With `FullSaveAssociations` enabled, associated objects are also created/updated:

```go
db.Session(&gorm.Session{FullSaveAssociations: true}).Create(&user)
// May unexpectedly create/update associated records
```

## Migration Issues

### AutoMigrate doesn't change column types

AutoMigrate only adds new columns, never modifies existing column types or drops columns:

```go
// Use Migrator for manual migrations
if !db.Migrator().HasColumn(&User{}, "Phone") {
    db.Migrator().AddColumn(&User{}, "Phone")
} else {
    db.Migrator().AlterColumn(&User{}, "Phone")
}
```

### AutoMigrate missing indexes

Ensure proper index tags on your model:

```go
type User struct {
    ID   uint   `gorm:"primaryKey"`
    Name string `gorm:"index"`
    Code string `gorm:"uniqueIndex"`
    Age  int    `gorm:"index:idx_name_age,priority:2"` // composite index
}
```

## Transaction Issues

### Nested transaction rollback behavior unexpected

By default, nested transactions use SavePoints. Inner rollback only rolls back to the SavePoint:

```go
db.Transaction(func(tx *gorm.DB) error {
    tx.Create(&user) // will be committed
    tx.Transaction(func(tx2 *gorm.DB) error {
        tx2.Create(&order)
        return errors.New("rollback") // only order rolled back
    })
    return nil // user committed
})
```

To disable nested SavePoints:

```go
db, _ := gorm.Open(mysql.Open(dsn), &gorm.Config{
    DisableNestedTransaction: true,
})
```

### Panic in manual transaction not rolled back

Closure transactions auto-handle panics. Manual transactions do not:

```go
// ✅ Closure handles panic
db.Transaction(func(tx *gorm.DB) error {
    panic("oops") // auto rollback
})

// ❌ Manual — use defer + recover
tx := db.Begin()
defer func() {
    if r := recover(); r != nil {
        tx.Rollback()
    }
}()
```

## Performance Issues

### N+1 queries

Avoid per-record queries in loops:

```go
// ❌ N+1 problem
var users []User
db.Find(&users)
for _, u := range users {
    db.Where("user_id = ?", u.ID).Find(&u.Orders)
}

// ✅ Use Preload
db.Preload("Orders").Find(&users)
```

### OOM with large datasets

Use `FindInBatches`:

```go
db.FindInBatches(&users, 100, func(tx *gorm.DB, batch int) error {
    return nil
})
```

### Slow query debugging

```go
newLogger := logger.New(
    log.New(os.Stdout, "\r\n", log.LstdFlags),
    logger.Config{
        SlowThreshold: 200 * time.Millisecond,
        LogLevel:      logger.Warn,
    },
)
```

## Connection Issues

### Connection pool exhaustion

```go
sqlDB, err := db.DB()
sqlDB.SetMaxOpenConns(100)
sqlDB.SetMaxIdleConns(10)
sqlDB.SetConnMaxLifetime(time.Hour)
```

### Context timeout not working

Ensure `WithContext` is called:

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

db.WithContext(ctx).Find(&users)
```

> GORM's `Session.Context` does not automatically propagate. Each new `Session` or chain may lose the Context.

## Error Quick Reference

| Error | Meaning | Common Cause |
| :--- | :--- | :--- |
| `ErrRecordNotFound` | Record not found | `First`/`Last`/`Take` matched no records |
| `ErrMissingWhereClause` | WHERE required | `Update`/`Delete` without conditions, `AllowGlobalUpdate` not set |
| `ErrInvalidTransaction` | Invalid transaction | `Commit`/`Rollback` on non-transaction connection |
| `ErrPrimaryKeyRequired` | PK required | `FindInBatches` needs a primary key field |
| `ErrDuplicatedKey` | Duplicate key | Unique constraint violation (needs `TranslateError: true`) |
| `ErrForeignKeyViolated` | FK violation | Foreign key constraint violation (needs `TranslateError: true`) |
| `ErrInvalidValue` | Invalid value | `dest` is not a pointer or not struct/slice |
| `ErrEmptySlice` | Empty slice | `Create` with empty slice |
| `ErrDryRunModeUnsupported` | DryRun unsupported | `Row`/`Rows` called in DryRun mode |
| `ErrUnsupportedDriver` | Driver unsupported | `SavePoint`/`RollbackTo` not implemented by driver |
| `ErrUnsupportedRelation` | Unsupported relation | Association name doesn't exist |
| `ErrPreloadNotAllowed` | Preload not allowed | `Count` used with `Preload` |
| `ErrInvalidDB` | Invalid DB connection | `db.DB()` can't get `*sql.DB` |
