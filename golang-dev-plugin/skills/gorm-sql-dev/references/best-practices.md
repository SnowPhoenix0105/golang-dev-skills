# GORM Best Practices

This document collects core patterns and recommended practices for GORM usage. For detailed API signatures, see `references/model-crud.md` and `references/advanced-queries.md`.

## Table of Contents

- [N+1 Query Prevention](#n1-query-prevention)
- [Zero-Value Update Strategies](#zero-value-update-strategies)
- [Connection Pool Configuration](#connection-pool-configuration)
- [Soft Delete Design](#soft-delete-design)
- [Transaction Patterns](#transaction-patterns)
- [Migration Strategy](#migration-strategy)
- [Error Handling](#error-handling)
- [Batch Operations & Performance](#batch-operations--performance)
- [Logger Configuration](#logger-configuration)

---

## N+1 Query Prevention

N+1 is the most common GORM performance pitfall: querying associated data one by one in a loop, generating an exponential number of SQL calls.

### Use Preload (Recommended)

```go
// ❌ N+1: one Orders query per user
var users []User
db.Find(&users)
for _, u := range users {
    db.Where("user_id = ?", u.ID).Find(&u.Orders) // N extra queries
}

// ✅ Preload: one query for all associations
db.Preload("Orders").Find(&users)

// Conditional Preload
db.Preload("Orders", "status = ?", "active").Find(&users)

// Nested Preload
db.Preload("Orders.Items").Find(&users)
```

### Preload vs Joins

| Scenario | Recommendation |
| :--- | :--- |
| Load complete associated objects | `Preload` |
| Filter by association field without loading data | `Joins` + `Select` |
| Large associated datasets (thousands of rows) | `Joins` + pagination |
| Multi-level nested associations | Chained `Preload` |

```go
// Preload: load association data into memory
db.Preload("Orders").Find(&users)

// Joins: filter by association field without loading Orders
db.Joins("JOIN orders ON orders.user_id = users.id").
    Where("orders.amount > ?", 100).Find(&users)
```

---

## Zero-Value Update Strategies

`Updates(struct)` skips zero-value fields (`0`, `""`, `false`, `nil`) by default. Three strategies for updating zero values:

### Option 1: Use map (simple cases)

```go
db.Model(&user).Updates(map[string]interface{}{"age": 0, "active": false})
```

### Option 2: Use Select to specify fields (recommended)

```go
// Specific fields
db.Model(&user).Select("Age", "Active").Updates(User{Age: 0, Active: false})

// All fields (including zero values)
db.Model(&user).Select("*").Updates(User{Age: 0})
```

### Option 3: Pointer types in models (large projects)

```go
type User struct {
    Age    *int   // distinguish "zero value" from "not set"
    Active *bool
}

// nil = don't update, non-nil = update to this value
db.Model(&user).Updates(User{Age: ptr(0), Active: ptr(false)})
```

### Strategy Selection

| Scenario | Recommended |
| :--- | :--- |
| Occasional zero-value update | map or Select |
| Business logic distinguishes "not set" vs "zero" | Pointer types |
| Batch update of zero-value fields | Select("*") |

---

## Connection Pool Configuration

Default pool settings may not suit production. Configure explicitly:

```go
sqlDB, err := db.DB()
sqlDB.SetMaxOpenConns(100)             // Max open connections (default: unlimited)
sqlDB.SetMaxIdleConns(10)              // Max idle connections (default: 2)
sqlDB.SetConnMaxLifetime(time.Hour)    // Max connection lifetime
sqlDB.SetConnMaxIdleTime(10 * time.Minute) // Max idle time
```

### Recommended Values

| Parameter | Development | Production | Notes |
| :--- | :--- | :--- | :--- |
| MaxOpenConns | 10-25 | 50-100 | Based on database max connections |
| MaxIdleConns | 5-10 | 10-25 | ~25% of MaxOpenConns |
| ConnMaxLifetime | 1h | 30min-1h | Shorter than database wait_timeout |

> Use `db.Session()` for per-query connection tuning rather than modifying the global DB instance.

---

## Soft Delete Design

GORM supports soft deletes via the `gorm.DeletedAt` field.

### Basic Model

```go
type User struct {
    ID        uint
    Name      string
    DeletedAt gorm.DeletedAt // automatically enables soft delete
}
```

### Key Behaviors

```go
// Delete: sets DeletedAt timestamp, doesn't actually delete
db.Delete(&user)

// Query: automatically filters out deleted records (WHERE deleted_at IS NULL)
db.Find(&users)

// Query including deleted
db.Unscoped().Find(&users)

// Only deleted records
db.Unscoped().Where("deleted_at IS NOT NULL").Find(&users)

// Hard delete
db.Unscoped().Delete(&user)
```

### Caveats

- **Unique indexes**: Soft-deleted records still occupy indexes. Use `gorm.DeletedAt` (not `time.Time`) — when not deleted, the field is NULL, and NULL doesn't participate in unique constraints in MySQL.
- **Association soft deletes**: Handle association deletion manually or via Hooks.
- **Restore**: `db.Unscoped().Model(&user).Update("deleted_at", nil)`.

---

## Transaction Patterns

### Closure Transaction (Recommended)

```go
// Auto commit/rollback, auto panics recovery
err := db.Transaction(func(tx *gorm.DB) error {
    if err := tx.Create(&user).Error; err != nil {
        return err // auto rollback
    }
    if err := tx.Create(&order).Error; err != nil {
        return err // auto rollback
    }
    return nil // auto commit
})
```

### Manual Transaction (fine-grained control)

```go
tx := db.Begin()
defer func() {
    if r := recover(); r != nil {
        tx.Rollback()
    }
}()

if err := tx.Create(&user).Error; err != nil {
    tx.Rollback()
    return err
}
if err := tx.Create(&order).Error; err != nil {
    tx.Rollback()
    return err
}
tx.Commit()
```

### Nested Transactions

Nested transactions use SavePoints by default:

```go
db.Transaction(func(tx *gorm.DB) error {
    tx.Create(&user)
    tx.Transaction(func(tx2 *gorm.DB) error {
        tx2.Create(&order) // inner
        return errors.New("rollback") // only rolls back to SavePoint
    })
    return nil // user is still committed!
})
```

To disable nested SavePoints (rollback entire transaction together):

```go
db, _ := gorm.Open(mysql.Open(dsn), &gorm.Config{
    DisableNestedTransaction: true,
})
```

---

## Migration Strategy

### AutoMigrate Limitations

`db.AutoMigrate(&User{})` **can only**: create missing tables, add new columns, create indexes.

**Cannot**: modify column types, drop columns, rename columns, create foreign key constraints (disabled by default), modify constraints.

### Incremental Migration

```go
// Development: AutoMigrate
db.AutoMigrate(&User{}, &Order{})

// Production changes: use Migrator for manual handling
m := db.Migrator()

// Add column
if !m.HasColumn(&User{}, "Phone") {
    m.AddColumn(&User{}, "Phone")
}

// Alter column type (database-dependent)
m.AlterColumn(&User{}, "Phone")

// Create index
m.CreateIndex(&User{}, "idx_name")

// Rename column
m.RenameColumn(&User{}, "OldName", "NewName")
```

### Foreign Key Constraints

AutoMigrate does **not** create foreign key constraints by default. To enable:

```go
db, _ := gorm.Open(mysql.Open(dsn), &gorm.Config{
    DisableForeignKeyConstraintWhenMigrating: false,
})
```

---

## Error Handling

### ErrRecordNotFound

`First`/`Last`/`Take` return `gorm.ErrRecordNotFound` when no record found. `Find` does not.

```go
// Distinguish "not found" from other errors
if errors.Is(err, gorm.ErrRecordNotFound) {
    // Record doesn't exist
}

// If separation isn't needed, just use Find
var users []User
db.Where("id = ?", 999).Find(&users)
if len(users) == 0 {
    // No records
}
```

### TranslateError

When enabled, GORM translates database errors to `ErrDuplicatedKey`, `ErrForeignKeyViolated`, etc.:

```go
db, _ := gorm.Open(mysql.Open(dsn), &gorm.Config{
    TranslateError: true,
})

// Check for unique constraint violation
if errors.Is(err, gorm.ErrDuplicatedKey) {
    // Handle duplicate key
}
```

---

## Batch Operations & Performance

### Batch Insert

```go
// ✅ Batch insert (one SQL)
db.Create(&users) // users is []User

// ✅ Insert in batches (large datasets)
db.CreateInBatches(users, 100) // 100 per batch

// ❌ Loop single inserts
for _, u := range users {
    db.Create(&u) // N SQL statements
}
```

### Large Dataset Queries

```go
// ✅ Process in batches to avoid OOM
db.FindInBatches(&results, 100, func(tx *gorm.DB, batch int) error {
    for _, r := range results {
        // up to 100 rows per batch
    }
    return nil
})
```

### PrepareStmt

PrepareStmt caches prepared statements, reducing SQL parsing overhead. But for workloads with many dynamic SQL conditions, the cache may be constantly invalidated, hurting performance:

```go
// Enable globally
db, _ := gorm.Open(mysql.Open(dsn), &gorm.Config{
    PrepareStmt: true,
})

// Disable for a single query
db.Session(&gorm.Session{PrepareStmt: false}).Find(&users)
```

### Skip Hooks

Skip unnecessary hooks for batch operations to significantly improve throughput:

```go
db.Session(&gorm.Session{SkipHooks: true}).CreateInBatches(users, 100)
```

---

## Logger Configuration

### Development

```go
newLogger := logger.New(
    log.New(os.Stdout, "\r\n", log.LstdFlags),
    logger.Config{
        SlowThreshold:             200 * time.Millisecond,
        LogLevel:                  logger.Info, // log all SQL
        IgnoreRecordNotFoundError: true,        // suppress ErrRecordNotFound
        ParameterizedQueries:      false,       // show parameter values
        Colorful:                  true,
    },
)
```

### Production

```go
newLogger := logger.New(
    log.New(os.Stdout, "\r\n", log.LstdFlags),
    logger.Config{
        SlowThreshold:             200 * time.Millisecond,
        LogLevel:                  logger.Warn, // only slow queries and errors
        IgnoreRecordNotFoundError: true,
        ParameterizedQueries:      true,        // hide parameter values
    },
)
```

### Debug Tips

```go
// Debug a single query
db.Debug().Where("name = ?", "John").Find(&users)
// Output: [2024-01-01 12:00:00] [row:2] SELECT * FROM `users` WHERE name = 'John'

// View SQL without executing
stmt := db.Session(&gorm.Session{DryRun: true}).Find(&users).Statement
fmt.Println(stmt.SQL.String())
```
