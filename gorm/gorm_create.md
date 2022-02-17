<!-- ---
title: Gorm Statement
date: 2020-11-08 08:47:19
category: showcode, gorm
--- -->

# Gorm 增加实现

创建记录SQL 实现。

![](images/gorm_create.svg)

调用逻辑：

```go
db.Create(&Product{Code: "D42", Price: 100})
```

主要数据结构：

```go
type Schema struct {
    Table                     string
    Fields                    []*Field
    FieldsByName              map[string]*Field
    FieldsByDBName            map[string]*Field
}

type Field struct {
    Name                  string
    BindNames             []string
    DataType              DataType
    Comment               string
    Tag                   reflect.StructTag
}

// insert 语句
type Insert struct {
    Table    Table
    Modifier string
}

// values 语句
type Values struct {
    Columns []Column
    Values  [][]interface{}
}
```

## 1. Create 调用

```go
func (db *DB) Create(value interface{}) (tx *DB) {
    // 获取数据库实例
    tx = db.getInstance()

    // 数据值
    tx.Statement.Dest = value

    // 增加语句处理
    tx.callbacks.Create().Execute(tx)
    return
}
```

### 1.1 获取数据库实例

```go
func (db *DB) getInstance() *DB {
    // 复制一份DB 配置，用于新的SQL 处理
    tx := &DB{Config: db.Config}

    if db.clone == 1 {
        // 复制新的预处理声明实例
        tx.Statement = &Statement{
            DB:       tx,
            ConnPool: db.Statement.ConnPool,
        }
    }
    
    return tx
}
```

### 1.2 语句执行

```go
func (p *processor) Execute(db *DB) {
    stmt := db.Statement
    stmt.Model = stmt.Dest
    
    // 解析表名
    stmt.Parse(stmt.Model)
    
    // 结果值
    stmt.ReflectValue = reflect.ValueOf(stmt.Dest)

    // 执行SQL 语句回调
    for _, f := range p.fns {
        f(db)
    }

    // 打印SQL 语句
    db.Dialector.Explain(stmt.SQL.String(), stmt.Vars...)
}
```

## 2. 数据库信息解析

解析数据库表名与结构信息。

### 2.1 解析表信息

```go
func (stmt *Statement) Parse(value interface{}) (err error) {
    // 解析数据库表名与结构信息
    stmt.Schema, err = schema.Parse(value, stmt.DB.cacheStore, stmt.DB.NamingStrategy)
    
    stmt.Table = stmt.Schema.Table
    return err
}


// get data type from dialector
func Parse(dest interface{}, cacheStore *sync.Map, namer Namer) (*Schema, error) {
    // 表名
    modelType := reflect.ValueOf(dest).Type()
    tableName := namer.TableName(modelType.Name())
    
    // 表信息
    schema := &Schema{
        Table:          tableName,
        FieldsByName:   map[string]*Field{},
    }

    // 表结构解析
    for i := 0; i < modelType.NumField(); i++ {
        if fieldStruct := modelType.Field(i); ast.IsExported(fieldStruct.Name) {
            // ...
            field := schema.ParseField(fieldStruct)
            schema.Fields = append(schema.Fields, field)
        }
    }

    return schema, schema.err
}
```

### 2.2 解析字段信息

```go
func (schema *Schema) ParseField(fieldStruct reflect.StructField) *Field {
    field := &Field{
        Name:              fieldStruct.Name,
        FieldType:         fieldStruct.Type,
        StructField:       fieldStruct,
    }

    // 字段类型
    switch reflect.Indirect(fieldValue).Kind() {
    case reflect.Bool:
        field.DataType = Bool
    case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
        field.DataType = Int
    }

    return field
}
```

## 3. Create 语句处理

```go
func Create(config *Config) func(db *gorm.DB) {
    return func(db *gorm.DB) {
        // 拼接SQL 语句        
        db.Statement.AddClauseIfNotExists(clause.Insert{})
        db.Statement.AddClause(ConvertToCreateValues(db.Statement))
        db.Statement.Build("INSERT", "VALUES", "ON CONFLICT")
        
        // 执行 SQL
        result, err := db.Statement.ConnPool.ExecContext(db.Statement.Context, db.Statement.SQL.String(), db.Statement.Vars...)
        
        // 获取返回的记录ID
        insertID, err := result.LastInsertId()
        db.Statement.Schema.PrioritizedPrimaryField.Set(db.Statement.ReflectValue, insertID)
    }
}
```

### 3.1 添加 Insert 语句

```go
func (stmt *Statement) AddClauseIfNotExists(v clause.Interface) {
    if c, ok := stmt.Clauses[v.Name()]; !ok || c.Expression == nil {
        stmt.AddClause(v)
    }
}

func (stmt *Statement) AddClause(v clause.Interface) {
    name := v.Name()
    c := stmt.Clauses[name]
    c.Name = name
    v.MergeClause(&c)
    stmt.Clauses[name] = c
}
```

```go
// MergeClause 合并insert 语句
func (insert Insert) MergeClause(clause *Clause) {
    // ...
    // 添加insert 语句
    clause.Expression = insert
}
```


Insert Build：

```go
func (insert Insert) Build(builder Builder) {
    builder.WriteString("INTO ")
    if insert.Table.Name == "" {
        builder.WriteQuoted(currentTable)
    } else {
        builder.WriteQuoted(insert.Table)
    }
}
```

### 3.2 添加 Values 语句

```go
db.Statement.AddClause(ConvertToCreateValues(db.Statement))
```

```go
func ConvertToCreateValues(stmt *gorm.Statement) (values clause.Values) {
    switch value := stmt.Dest.(type) {
    default:
        values = clause.Values{Columns: make([]clause.Column, 0, len(stmt.Schema.DBNames))}

        // 数据库表结构
        for _, db := range stmt.Schema.DBNames {
            // 数据库表字段
            values.Columns = append(values.Columns, clause.Column{Name: db})
        }

        switch stmt.ReflectValue.Kind() {
        case reflect.Struct:
            for idx, column := range values.Columns {
                // 获取数据值
                field := stmt.Schema.FieldsByDBName[column.Name]
                values.Values[0][idx], isZero = field.ValueOf(stmt.ReflectValue)
                // ...
            }
    }

    return values
}
```

Values 语句部分生成逻辑处理。

```go
// Build build from clause
func (values Values) Build(builder Builder) {
    builder.WriteByte('(')
    for idx, column := range values.Columns {
        if idx > 0 {
            builder.WriteByte(',')
        }
        builder.WriteQuoted(column)
    }
    builder.WriteByte(')')

    builder.WriteString(" VALUES ")

    for idx, value := range values.Values {
        if idx > 0 {
            builder.WriteByte(',')
        }

        builder.WriteByte('(')
        builder.AddVar(builder, value...)
        builder.WriteByte(')')
    }
}
```

```go
func (stmt *Statement) AddVar(writer clause.Writer, vars ...interface{}) {
    for idx, v := range vars {
        if idx > 0 {
            writer.WriteByte(',')
        }

        switch v := v.(type) {
        default:
            switch rv := reflect.ValueOf(v); rv.Kind() {
            default:
                stmt.Vars = append(stmt.Vars, v)
                stmt.DB.Dialector.BindVarTo(writer, stmt, v)
            }
        }
    }
}
```

### 3.3 SQL 生成

```go
func (stmt *Statement) Build(clauses ...string) {
    for _, name := range clauses {
        if c, ok := stmt.Clauses[name]; ok {
            c.Build(stmt)
        }
    }
}
```

```go
func (c Clause) Build(builder Builder) {
    // ...
    if c.Name != "" {
        builder.WriteString(c.Name)
        builder.WriteByte(' ')
    }

    c.Expression.Build(builder)
}
```

### 3.4 SQL 执行

```go
result, err := db.Statement.ConnPool.ExecContext(db.Statement.Context, db.Statement.SQL.String(), db.Statement.Vars...)
```

```go
// database/sql/sql.go
func (tx *Tx) ExecContext(ctx context.Context, query string, args ...interface{}) (Result, error) {
    dc, release, err := tx.grabConn(ctx)
    // ...
    return tx.db.execDC(ctx, dc, release, query, args)
}
```

## 参考资料

- gorm.io/gorm/finisher_api.go
- gorm.io/gorm/callbacks/create.go
