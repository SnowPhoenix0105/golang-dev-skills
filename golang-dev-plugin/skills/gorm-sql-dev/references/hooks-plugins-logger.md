# Hooks, Plugins, Logger & Error Handling — Complete Reference

## Table of Contents

- [Hooks](#hooks)
  - [Lifecycle Hooks](#lifecycle-hooks)
  - [Model-Level Hooks](#model-level-hooks)
  - [Skipping Hooks](#skipping-hooks)
- [Plugin System](#plugin-system)
- [Logger](#logger)
- [Error Handling](#error-handling)
- [Error Translation](#error-translation)

## Hooks

### Lifecycle Hooks

GORM provides hook interfaces before/after each CRUD operation. Implement the corresponding interface method on your model.

| Hook | Trigger |
| :--- | :--- |
| `BeforeSave` | Before create/update |
| `AfterSave` | After create/update |
| `BeforeCreate` | Before create |
| `AfterCreate` | After create |
| `BeforeUpdate` | Before update |
| `AfterUpdate` | After update |
| `BeforeDelete` | Before delete |
| `AfterDelete` | After delete |
| `AfterFind` | After query |

```go
func (u *User) BeforeCreate(tx *gorm.DB) error {
    if u.UUID == "" {
        u.UUID = uuid.New().String()
    }
    return nil
}

func (u *User) BeforeUpdate(tx *gorm.DB) error {
    if u.Age < 0 {
        return errors.New("age cannot be negative")
    }
    return nil
}

func (u *User) AfterFind(tx *gorm.DB) error {
    u.Name = strings.ToUpper(u.Name)
    return nil
}
```

**Key rules**:
- Returning `error` aborts the operation and rolls back the transaction
- `BeforeSave` runs before `BeforeCreate`/`BeforeUpdate`
- `AfterSave` runs after `AfterCreate`/`AfterUpdate`
- Hooks execute within the transaction by default

### Model-Level Hooks

Register global hooks via callbacks instead of interface methods:

```go
db.Callback().Create().Before("gorm:before_create").Register("my_plugin:before_create", func(db *gorm.DB) {
    // runs before every Create
})

db.Callback().Query().After("gorm:after_query").Register("my_plugin:after_query", func(db *gorm.DB) {
    // runs after every query
})
```

### Skipping Hooks

```go
// Session level
db.Session(&gorm.Session{SkipHooks: true}).Create(&user)

// UpdateColumn / UpdateColumns skip hooks
db.Model(&user).UpdateColumn("name", "John")
db.Model(&user).UpdateColumns(User{Name: "John", Age: 0})

// Batch Delete skips hooks by default
// To trigger hooks, query then delete individually
db.Where("age > ?", 60).Find(&users)
for _, user := range users {
    db.Delete(&user)
}
```

## Plugin System

### Built-in Plugin — DBResolver

```go
import "gorm.io/plugin/dbresolver"

// Read/write splitting
db.Use(dbresolver.Register(dbresolver.Config{
    Sources:  []gorm.Dialector{mysql.Open("dsn-master")},
    Replicas: []gorm.Dialector{mysql.Open("dsn-slave1"), mysql.Open("dsn-slave2")},
    Policy:   dbresolver.RandomPolicy{},
}))

db.Clauses(dbresolver.Write).Create(&user)  // writes to master
db.Clauses(dbresolver.Read).Find(&users)     // reads from replica
```

### Custom Plugin

```go
type MyPlugin struct{}

func (p MyPlugin) Name() string {
    return "my_plugin"
}

func (p MyPlugin) Initialize(db *gorm.DB) error {
    db.Callback().Create().Before("gorm:create").Register("my_plugin:before_create", func(db *gorm.DB) {
        // custom logic
    })
    return nil
}

db.Use(&MyPlugin{})
```

### Callback Operations

```go
db.Callback().Create().Before("gorm:before_create").Register("my_callback", func(db *gorm.DB) {
    // ...
})
db.Callback().Create().Remove("gorm:before_create")
db.Callback().Create().Replace("gorm:before_create", func(db *gorm.DB) {
    // new logic
})
```

## Logger

### Default Logger

```go
db, _ := gorm.Open(mysql.Open(dsn), &gorm.Config{
    Logger: logger.Default.LogMode(logger.Info),
})

// Or create a new Logger
newLogger := logger.New(
    log.New(os.Stdout, "\r\n", log.LstdFlags),
    logger.Config{
        SlowThreshold:             200 * time.Millisecond,
        LogLevel:                  logger.Warn,
        IgnoreRecordNotFoundError: true,
        ParameterizedQueries:      false,
        Colorful:                  true,
    },
)
```

### Log Levels

| Level | Output |
| :--- | :--- |
| `logger.Silent` | No output |
| `logger.Error` | Errors only |
| `logger.Warn` | Errors + slow queries |
| `logger.Info` | All SQL |

### Debug Mode

```go
db.Debug().First(&user)
db.Session(&gorm.Session{Logger: db.Logger.LogMode(logger.Info)}).First(&user)
```

### Custom Logger

Implement `logger.Interface`:

```go
type CustomLogger struct{}

func (l CustomLogger) LogMode(level logger.LogLevel) logger.Interface { return l }
func (l CustomLogger) Info(ctx context.Context, msg string, data ...interface{}) {}
func (l CustomLogger) Warn(ctx context.Context, msg string, data ...interface{}) {}
func (l CustomLogger) Error(ctx context.Context, msg string, data ...interface{}) {}
func (l CustomLogger) Trace(ctx context.Context, begin time.Time, fc func() (string, int64), err error) {}
```

### slog Logger (Go 1.21+)

```go
import "gorm.io/gorm/logger/slog"

db, _ := gorm.Open(mysql.Open(dsn), &gorm.Config{
    Logger: slog.New(slog.Default().Handler()),
})
```

## Error Handling

### Basic Pattern

```go
result := db.First(&user)
if result.Error != nil {
    // handle error
}

// or
if err := db.First(&user).Error; err != nil {
    // handle error
}
```

### Check Rows Affected

```go
result := db.Create(&user)
if result.RowsAffected == 0 {
    // no records affected
}
```

### Common Error Types

```go
// Record not found
if errors.Is(result.Error, gorm.ErrRecordNotFound) { ... }

// Missing WHERE clause
if errors.Is(result.Error, gorm.ErrMissingWhereClause) { ... }

// Duplicate key
if errors.Is(result.Error, gorm.ErrDuplicatedKey) { ... }

// FK constraint violation
if errors.Is(result.Error, gorm.ErrForeignKeyViolated) { ... }
```

## Error Translation

```go
db, _ := gorm.Open(mysql.Open(dsn), &gorm.Config{
    TranslateError: true,
})

// With translation, DB-specific errors become gorm.ErrDuplicatedKey, etc.
result := db.Create(&user)
if errors.Is(result.Error, gorm.ErrDuplicatedKey) {
    fmt.Println("email already exists")
}
```

Error translation requires driver support for `ErrorTranslator` interface.
