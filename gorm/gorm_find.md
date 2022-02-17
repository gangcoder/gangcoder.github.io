<!-- ---
title: Gorm find
date: 2020-11-08 08:48:38
category: showcode, gorm
--- -->

# Gorm 查找实现

查找记录SQL 实现。

![](images/gorm_find.svg)

调用逻辑：

```go
db.Find(&product, 2)
```

主要数据结构：

```go
type Where struct {
    Exprs []Expression
}

type IN struct {
    Column interface{}
    Values []interface{}
}

type Eq struct {
    Column interface{}
    Value  interface{}
}

type Select struct {
    Distinct   bool
    Columns    []Column
    Expression Expression
}

type Expression interface {
	Build(builder Builder)
}
```

## 1. 查找调用

```go
func (db *DB) Find(dest interface{}, conds ...interface{}) (tx *DB) {
    tx = db.getInstance()
    // 条件拼接
    if len(conds) > 0 {
        tx.Statement.AddClause(clause.Where{Exprs: tx.Statement.BuildCondition(conds[0], conds[1:]...)})
    }

    // 结果值
    tx.Statement.Dest = dest
    tx.callbacks.Query().Execute(tx)
    return
}
```

### 1.1 查询条件语句

```go
// BuildCondition 拼接查询条件
func (stmt *Statement) BuildCondition(query interface{}, args ...interface{}) (conds []clause.Expression) {
    // ...
    args = append([]interface{}{query}, args...)
    for _, arg := range args {
        switch v := arg.(type) {
        default:
            conds = append(conds, clause.IN{Column: clause.PrimaryColumn, Values: args})
        }
    }

    return
}
```

## 2. Query 查询实现

```go
func Query(db *gorm.DB) {
    // 拼接语句
    BuildQuerySQL(db)

    // ...
    rows, err := db.Statement.ConnPool.QueryContext(db.Statement.Context, db.Statement.SQL.String(), db.Statement.Vars...)
    
    // 处理返回结果
    gorm.Scan(rows, db, false)
}
```

### 2.1 拼接SQL

拼接语句。

```go
func BuildQuerySQL(db *gorm.DB) {
    // 处理软删除查询器
    if db.Statement.Schema != nil && !db.Statement.Unscoped {
        for _, c := range db.Statement.Schema.QueryClauses {
            db.Statement.AddClause(c)
        }
    }

    // 拼接查询语句
    clauseSelect := clause.Select{Distinct: db.Statement.Distinct}
    db.Statement.AddClauseIfNotExists(clause.From{})
    db.Statement.AddClauseIfNotExists(clauseSelect)

    db.Statement.Build("SELECT", "FROM", "WHERE", "GROUP BY", "ORDER BY", "LIMIT", "FOR")
}
```


### 2.2 SELECT 语句

```go
func (s Select) Build(builder Builder) {
    if len(s.Columns) > 0 {
        for idx, column := range s.Columns {
            if idx > 0 {
                builder.WriteByte(',')
            }
            builder.WriteQuoted(column)
        }
    }
}
```

### 2.3 FROM 语句

```go
func (from From) Build(builder Builder) {
    builder.WriteQuoted(currentTable)
    // ...
}
```

### 2.4 WHERE 语句

```go
// Build build where clause
func (where Where) Build(builder Builder) {
    // ...
    buildExprs(where.Exprs, builder, " AND ")
}
```

拼接Where 语句。

```go
func buildExprs(exprs []Expression, builder Builder, joinCond string) {
    // 片接where 语句
    for idx, expr := range exprs {
        // ...
        expr.Build(builder)
    }
}
```

IN 查询Where 条件实现：

```go
func (in IN) Build(builder Builder) {
    builder.WriteQuoted(in.Column)

    switch len(in.Values) {
    case 1:
        builder.WriteString(" = ")
        builder.AddVar(builder, in.Values...)
    }
}

IS 查询Where 条件实现：

func (eq Eq) Build(builder Builder) {
    builder.WriteQuoted(eq.Column)

    if eq.Value == nil {
        builder.WriteString(" IS NULL")
    }
}
```

## 3. 取出数据结果

```go
func Scan(rows *sql.Rows, db *DB, initialized bool) {
    switch dest := db.Statement.Dest.(type) {
    default:
        Schema := db.Statement.Schema
        switch db.Statement.ReflectValue.Kind() {
        case reflect.Struct:
            // ...
            if initialized || rows.Next() {
                for idx, column := range columns {
                    // 处理每列数据格式
                    if field := Schema.LookUpField(column); field != nil && field.Readable {
                        values[idx] = reflect.New(reflect.PtrTo(field.IndirectFieldType)).Interface()
                    }
                }

                db.RowsAffected++

                // 调用原生SQL 获取数据
                db.AddError(rows.Scan(values...))

                // 补充结果数据结构中的数值
                for idx, column := range columns {
                    if field := Schema.LookUpField(column); field != nil && field.Readable {
                        field.Set(db.Statement.ReflectValue, values[idx])
                    }
                    // ...
                }
            }
        }
    }
}
```

## 参考资料

- gorm.io/gorm/finisher_api.go