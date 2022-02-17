<!-- ---
title: Gorm Open
date: 2020-11-08 08:45:33
category: showcode, gorm
--- -->

# Gorm 创建连接

1. 开启mysql 连接
2. gorm 使用mysql 驱动

![](images/gorm_open.svg)

主要调用：

```go
db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
```

主要数据结构：

```go
type Dialector struct {
    *Config
}

// mysql 驱动配置
type Config struct {
    DriverName                string
    DSN                       string
    Conn                      gorm.ConnPool
}

// Dialector 接口
type Dialector interface {
    Name() string
    Initialize(*DB) error
    Explain(sql string, vars ...interface{}) string
}

// gorm 配置
type Config struct {
    // 是否不执行SQL
    DryRun bool
    // 语句构建器
    ClauseBuilders map[string]clause.ClauseBuilder
    ConnPool ConnPool
    // 数据库驱动
    Dialector
    callbacks  *callbacks
    cacheStore *sync.Map
}

// callbacks
type callbacks struct {
    processors map[string]*processor
}

type processor struct {
    db        *DB
    fns       []func(*DB)
    callbacks []*callback
}

type callback struct {
    name      string
    handler   func(*DB)
    processor *processor
}

// gorm 数据库实例
type DB struct {
    *Config
    RowsAffected int64
    Statement    *Statement
}

// Statement
type Statement struct {
    *DB
    Table                string
    Model                interface{}
    Dest                 interface{}
    Clauses              map[string]clause.Clause
    ConnPool             ConnPool
    Schema               *schema.Schema
    SQL                  strings.Builder
    Vars                 []interface{}
}
```

## 1. 开启mysql 连接

```go
mysql.Open(dsn)
```

```go
func Open(dsn string) gorm.Dialector {
    return &Dialector{Config: &Config{DSN: dsn}}
}
```

### 1.1 Mysql 连接初始化

```go
func (dialector Dialector) Initialize(db *gorm.DB) (err error) {
    // 注册默认回调
    callbacks.RegisterDefaultCallbacks(db, &callbacks.Config{})

    // 标准库sql 连接
    db.ConnPool, err = sql.Open(dialector.DriverName, dialector.DSN)

    // 注册mysql 默认语句构建器
    for k, v := range dialector.ClauseBuilders() {
        db.ClauseBuilders[k] = v
    }
    return
}
```

### 1.2 注册回调处理

```go
func RegisterDefaultCallbacks(db *gorm.DB, config *Config) {
    createCallback := db.Callback().Create()
    createCallback.Register("gorm:create", Create(config))

    queryCallback := db.Callback().Query()
    queryCallback.Register("gorm:query", Query)

    deleteCallback := db.Callback().Delete()
    deleteCallback.Register("gorm:delete", Delete)

    updateCallback := db.Callback().Update()
    updateCallback.Register("gorm:update", Update)

    db.Callback().Row().Register("gorm:row", RowQuery)
    db.Callback().Raw().Register("gorm:raw", RawExec)
}
```

注册逻辑：先取出对应的 processor 处理器，再将回调函数，注册上来。

```go
func (cs *callbacks) Create() *processor {
    return cs.processors["create"]
}
```

```go
createCallback.Register("gorm:create", Create(config))

func (p *processor) Register(name string, fn func(*DB)) error {
    return (&callback{processor: p}).Register(name, fn)
}

func (c *callback) Register(name string, fn func(*DB)) error {
    c.name = name
    c.handler = fn
    c.processor.callbacks = append(c.processor.callbacks, c)
    return c.processor.compile()
}
```

### 1.3 Mysql SQL 输出

```go
func (dialector Dialector) Explain(sql string, vars ...interface{}) string {
    return logger.ExplainSQL(sql, nil, `'`, vars...)
}
```

```go
func ExplainSQL(sql string, numericPlaceholder *regexp.Regexp, escaper string, avars ...interface{}) string {
    // 格式化数值
    var vars = make([]string, len(avars))
    for idx, v := range avars {
        convertParams(v, idx)
    }

    // 替换sql 中的预处理占位符
    var idx int
    var newSQL strings.Builder

    for _, v := range []byte(sql) {
        if v == '?' {
            if len(vars) > idx {
                newSQL.WriteString(vars[idx])
                idx++
                continue
            }
        }
        newSQL.WriteByte(v)
    }

    sql = newSQL.String()

    return sql
}
```

## 2. 使用mysql 驱动

```go
gorm.Open(mysql.Open(dsn), &gorm.Config{})
```

```go
// Open 基于数据库连接驱动，初始化数据库会话
func Open(dialector Dialector, config *Config) (db *DB, err error) {
    // ...
    config.Dialector = dialector

    db = &DB{Config: config, clone: 1}

    // 初始化语句回调处理
    db.callbacks = initializeCallbacks(db)

    // 初始化DB 连接
    err = config.Dialector.Initialize(db)

    // 预处理声明
    db.Statement = &Statement{
        DB:       db,
        ConnPool: db.ConnPool,
    }

    return
}
```

## 参考资料

- gorm.io/driver/mysql/mysql.go

