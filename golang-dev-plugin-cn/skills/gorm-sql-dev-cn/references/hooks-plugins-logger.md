# 钩子、插件、Logger 与错误处理完整参考

## 目录

- [钩子（Hooks）](#钩子hooks)
  - [生命周期钩子](#生命周期钩子)
  - [模型级钩子](#模型级钩子)
  - [跳过钩子](#跳过钩子)
- [插件系统](#插件系统)
- [Logger](#logger)
- [错误处理](#错误处理)
- [错误转换](#错误转换)

## 钩子（Hooks）

### 生命周期钩子

GORM 在每个 CRUD 操作前后提供钩子接口。模型实现对应的接口方法即可自动调用。

| 钩子 | 触发时机 |
| :--- | :--- |
| `BeforeSave` | 创建/更新前 |
| `AfterSave` | 创建/更新后 |
| `BeforeCreate` | 创建前 |
| `AfterCreate` | 创建后 |
| `BeforeUpdate` | 更新前 |
| `AfterUpdate` | 更新后 |
| `BeforeDelete` | 删除前 |
| `AfterDelete` | 删除后 |
| `AfterFind` | 查询后 |

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
    // 查询后处理：格式化、解密等
    u.Name = strings.ToUpper(u.Name)
    return nil
}
```

**重要规则**：
- 返回 `error` 会中止当前操作并回滚事务
- `BeforeSave` 在 `BeforeCreate` / `BeforeUpdate` 之前调用
- `AfterSave` 在 `AfterCreate` / `AfterUpdate` 之后调用
- 钩子默认在事务内执行

### 模型级钩子

除了实现接口方法，也可以通过回调注册：

```go
// 使用回调注册全局钩子
db.Callback().Create().Before("gorm:before_create").Register("my_plugin:before_create", func(db *gorm.DB) {
    // 所有 Create 操作前执行
})

db.Callback().Query().After("gorm:after_query").Register("my_plugin:after_query", func(db *gorm.DB) {
    // 所有查询后执行
})
```

### 跳过钩子

```go
// Session 级别跳过
db.Session(&gorm.Session{SkipHooks: true}).Create(&user)

// UpdateColumn / UpdateColumns 跳过钩子
db.Model(&user).UpdateColumn("name", "张三")
db.Model(&user).UpdateColumns(User{Name: "张三", Age: 0})

// Delete 批量删除默认跳过钩子（性能考虑）
// 如需触发钩子，逐条查询后删除
db.Where("age > ?", 60).Find(&users)
for _, user := range users {
    db.Delete(&user)
}
```

## 插件系统

GORM 支持通过插件扩展功能。

### 内置插件示例 — DBResolver

```go
import "gorm.io/plugin/dbresolver"

// 读写分离
db.Use(dbresolver.Register(dbresolver.Config{
    Sources:  []gorm.Dialector{mysql.Open("dsn-master")},
    Replicas: []gorm.Dialector{mysql.Open("dsn-slave1"), mysql.Open("dsn-slave2")},
    Policy:   dbresolver.RandomPolicy{},
}))

// 使用
db.Clauses(dbresolver.Write).Create(&user)  // 主库
db.Clauses(dbresolver.Read).Find(&users)     // 从库
```

### 自定义插件

```go
type MyPlugin struct{}

func (p MyPlugin) Name() string {
    return "my_plugin"
}

func (p MyPlugin) Initialize(db *gorm.DB) error {
    // 注册回调
    db.Callback().Create().Before("gorm:create").Register("my_plugin:before_create", func(db *gorm.DB) {
        // 自定义逻辑
    })
    return nil
}

// 注册插件
db.Use(&MyPlugin{})
```

### 回调操作

```go
// 注册回调
db.Callback().Create().Before("gorm:before_create").Register("my_callback", func(db *gorm.DB) {
    // ...
})

// 移除回调
db.Callback().Create().Remove("gorm:before_create")

// 替换回调
db.Callback().Create().Replace("gorm:before_create", func(db *gorm.DB) {
    // 新逻辑
})

// 获取已注册的回调名
names := db.Callback().Create().Get("gorm:create")
```

## Logger

### 默认 Logger

```go
// 配置 Logger
db, _ := gorm.Open(mysql.Open(dsn), &gorm.Config{
    Logger: logger.Default.LogMode(logger.Info),
})

// 或新建 Logger
newLogger := logger.New(
    log.New(os.Stdout, "\r\n", log.LstdFlags),
    logger.Config{
        SlowThreshold:             200 * time.Millisecond, // 慢查询阈值
        LogLevel:                  logger.Warn,            // 日志级别
        IgnoreRecordNotFoundError: true,                   // 忽略 ErrRecordNotFound
        ParameterizedQueries:      false,                  // 隐藏 SQL 参数值
        Colorful:                  true,                   // 彩色输出
    },
)

db, _ := gorm.Open(mysql.Open(dsn), &gorm.Config{Logger: newLogger})
```

### 日志级别

| 级别 | 输出内容 |
| :--- | :--- |
| `logger.Silent` | 无输出 |
| `logger.Error` | 仅错误 |
| `logger.Warn` | 错误 + 慢查询 |
| `logger.Info` | 所有 SQL |

### Debug 模式

```go
// 链式 API
db.Debug().First(&user)

// Session 级别
db.Session(&gorm.Session{Logger: db.Logger.LogMode(logger.Info)}).First(&user)
```

### 自定义 Logger

```go
// 实现 logger.Interface
type CustomLogger struct{}

func (l CustomLogger) LogMode(level logger.LogLevel) logger.Interface { return l }
func (l CustomLogger) Info(ctx context.Context, msg string, data ...interface{}) {}
func (l CustomLogger) Warn(ctx context.Context, msg string, data ...interface{}) {}
func (l CustomLogger) Error(ctx context.Context, msg string, data ...interface{}) {}
func (l CustomLogger) Trace(ctx context.Context, begin time.Time, fc func() (sql string, rowsAffected int64), err error) {}
```

### slog Logger（Go 1.21+）

```go
import "gorm.io/gorm/logger/slog"

db, _ := gorm.Open(mysql.Open(dsn), &gorm.Config{
    Logger: slog.New(slog.Default().Handler()),
})
```

## 错误处理

### 基础模式

GORM 操作返回 `*gorm.DB`，通过 `Error` 字段检查错误：

```go
result := db.First(&user)
if result.Error != nil {
    // 处理错误
}

// 或
err = db.First(&user).Error
if err != nil {
    // 处理错误
}
```

### 检查受影响行数

```go
result := db.Create(&user)
if result.RowsAffected == 0 {
    // 没有记录被影响
}
```

### 常见错误类型

```go
// 记录未找到
if errors.Is(result.Error, gorm.ErrRecordNotFound) {
    // First/Last/Take 未找到
}

// 缺少 WHERE 条件
if errors.Is(result.Error, gorm.ErrMissingWhereClause) {
    // Update/Delete 缺少条件
}

// 重复键
if errors.Is(result.Error, gorm.ErrDuplicatedKey) {
    // 违反唯一约束
}

// 外键约束
if errors.Is(result.Error, gorm.ErrForeignKeyViolated) {
    // 违反外键约束
}
```

### 错误链

GORM 用 `fmt.Errorf("%v; %w", ...)` 累积错误：

```go
// 多个错误会被串联
db.Where("name = ?", name).First(&user)
db.Where("age > ?", age).First(&user) // 这会清除前一个 Where 的效果，但错误会累积

// 检查是否包含某个错误
if errors.Is(db.Error, gorm.ErrRecordNotFound) {
    // ...
}
```

完整错误定义位于 `gorm.io/gorm` 的 `errors.go`。

## 错误转换

```go
db, _ := gorm.Open(mysql.Open(dsn), &gorm.Config{
    TranslateError: true, // 启用错误翻译
})

// 启用后，数据库驱动会将数据库特定错误转为 gorm.ErrDuplicatedKey 等
result := db.Create(&user)
if errors.Is(result.Error, gorm.ErrDuplicatedKey) {
    fmt.Println("邮箱已存在")
}
```

错误翻译需要驱动支持 `ErrorTranslator` 接口。
