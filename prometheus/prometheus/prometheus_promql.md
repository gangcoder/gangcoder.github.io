<!-- ---
title: promql
date: 2019-04-18 20:12:42
category: src, prometheus, src
--- -->

# Promql 查询引擎实现

PromQL (Prometheus Query Language) Prometheus 查询语言实现。

1. 创建查询引擎实例
2. 创建查询
3. 求值程序执行

![](images/prometheus_promql.svg)

## 1. 创建查询引擎实例

数据模型

```go
// Engine 处理查询
type Engine struct {
	logger             log.Logger
	metrics            *engineMetrics //指标
	timeout            time.Duration
	gate               *gate.Gate
	maxSamplesPerQuery int
}

// Query 从查询条件中解析出的查询语句实例接口
type Query interface {
	// Exec 执行查询
	Exec(ctx context.Context) *Result
	// Statement 返回查询语句的 statement
	Statement() Statement
}

// query 查询语句实例的实现结构
type query struct {
	// 数据存储器，可供查询数据
	queryable storage.Queryable
	// 原始条件语句
	q string
	// 查询语句 Statement
	stmt Statement
	// 查询引擎
	ng *Engine
}

// EvalStmt 包含待执行语句的SQL 声明
type EvalStmt struct {
	Expr Expr // 执行表达式
	Start, End time.Time
	Interval time.Duration
}

//evaluator 基于数据引擎，在指定时间段内对表达式求职
type evaluator struct {
	startTimestamp int64
	endTimestamp   int64
	interval       int64

	maxSamples     int
	currentSamples int
	logger         log.Logger
}
```

创建实例

```go
// github.com/prometheus/prometheus/cmd/prometheus/main.go
queryEngine = promql.NewEngine(opts)

// github.com/prometheus/prometheus/promql/engine.go
// NewEngine 创建查询实例
func NewEngine(opts EngineOpts) *Engine {
	// ...
	return &Engine{
		timeout:            opts.Timeout,
		logger:             opts.Logger,
		metrics:            metrics,
		maxSamplesPerQuery: opts.MaxSamples,
		activeQueryTracker: opts.ActiveQueryTracker,
	}
}
```

## 2. 创建查询

```go
// github.com/prometheus/prometheus/cmd/prometheus/main.go
// 基于查询引擎和存储服务，创建用于业务的查询函数
QueryFunc: rules.EngineQueryFunc(queryEngine, fanoutStorage),

// github.com/prometheus/prometheus/promql/engine.go
// QueryFunc 查询函数，处理 PromQL 查询
type QueryFunc func(ctx context.Context, q string, t time.Time) (promql.Vector, error)

// EngineQueryFunc 
func EngineQueryFunc(engine *promql.Engine, q storage.Queryable) QueryFunc {
	return func(ctx context.Context, qs string, t time.Time) (promql.Vector, error) {
		// 创建查询语句实例 Query
		q, err := engine.NewInstantQuery(q, qs, t)
		
		// 执行查询
		res := q.Exec(ctx)
		
		// 处理结果数据
		switch v := res.Value.(type) {
		case promql.Vector:
			return v, nil
		case promql.Scalar:
			return promql.Vector{promql.Sample{
				Point:  promql.Point(v),
				Metric: labels.Labels{},
			}}, nil
		default:
			return nil, errors.New("rule result is not a vector or scalar")
		}
	}
}
```

### 2.1 创建查询语句实例 Query

```go
//NewInstantQuery 创建瞬时类查询语句的Query 实例
func (ng *Engine) NewInstantQuery(q storage.Queryable, qs string, ts time.Time) (Query, error) {
	// 从查询条件解析出查询语句
	expr, err := ParseExpr(qs)
	//...
    qry := ng.newQuery(q, expr, ts, ts, 0)
	qry.q = qs

	return qry, nil
}

// 创建查询语句实例
func (ng *Engine) newQuery(q storage.Queryable, expr Expr, start, end time.Time, interval time.Duration) *query {
	es := &EvalStmt{
		Expr:     expr,
		Start:    start,
		End:      end,
		Interval: interval,
	}
	qry := &query{
		stmt:      es,
		ng:        ng,
		stats:     stats.NewQueryTimers(),
		queryable: q,
	}
	return qry
}
```

### 2.2 查询语句执行

```go
// Exec 查询端口
func (q *query) Exec(ctx context.Context) *Result {
    //...
    res, warnings, err := q.ng.exec(ctx, q)
	return &Result{Err: err, Value: res, Warnings: warnings}
}

// exec 查询引擎执行查询
func (ng *Engine) exec(ctx context.Context, q *query) (Value, error) {
    //...
    switch s := q.Statement().(type) {
	case *EvalStmt:
		return ng.execEvalStmt(ctx, q, s)
	}
}

// execEvalStmt 计算评估声明的评估条件
func (ng *Engine) execEvalStmt(ctx context.Context, query *query, s *EvalStmt) (Value, error) {
	// 数据查询，查询结果会绑定到 s.Expr
	querier, warnings, err := ng.populateSeries(ctxPrepare, query.queryable, s)

	//...
    // Range evaluation.
	evaluator := &evaluator{
		startTimestamp:      start,
		endTimestamp:        start,
		interval:            1,
		ctx:                 ctxInnerEval,
		maxSamples:          ng.maxSamplesPerQuery,
		defaultEvalInterval: GetDefaultEvaluationInterval(),
		logger:              ng.logger,
	}

    val, err := evaluator.Eval(s.Expr)
    switch result := val.(type) {
	case Matrix:
		mat = result
	case String:
		return result, warnings, nil

    switch s.Expr.Type() {
	case ValueTypeVector:
		// 查询查询语句类型，返回不同格式结果
		vector := make(Vector, len(mat))
		for i, s := range mat {
			vector[i] = Sample{Metric: s.Metric, Point: Point{V: s.Points[0].V, T: start}}
		}
		return vector, warnings, nil
	case ValueTypeScalar:
		return Scalar{V: mat[0].Points[0].V, T: start}, warnings, nil
	case ValueTypeMatrix:
		return mat, warnings, nil
	}
}

func (ng *Engine) populateSeries(ctx context.Context, q storage.Queryable, s *EvalStmt) (storage.Querier, storage.Warnings, error) {
	querier, err := q.Querier(ctx, timestamp.FromTime(mint), timestamp.FromTime(s.End))

	// Inspect 会将s.Expr 当做node 参数传入匿名函数中
	Inspect(s.Expr, func(node Node, path []Node) error {
		switch n := node.(type) {
		case *VectorSelector:
			set, wrn, err = querier.Select(params, n.LabelMatchers...)
			// 查询出来的数据，会绑定到 s.Expr 的node 字段上
			n.unexpandedSeriesSet = set
			// ...
		}
		return nil
	})

	return querier, warnings, err
}
```

## 3. 求值程序执行

```go
// github.com/prometheus/prometheus/promql/engine.go
// 执行查询求值语句
func (ev *evaluator) Eval(expr Expr) (v Value, err error) {
	//...
	return ev.eval(expr), nil
}

func (ev *evaluator) eval(expr Expr) Value {
	switch e := expr.(type) {
	case *AggregateExpr:
	// ...
	case *Call:
	// ...
	case *ParenExpr:
	// ...
	case *UnaryExpr:
	// ...
	case *BinaryExpr:
	// ...
	case *NumberLiteral:
	// ...
	}
}
```

## 参考资料

- github.com/prometheus/prometheus/cmd/prometheus/main.go
- github.com/prometheus/prometheus/promql/engine.go

