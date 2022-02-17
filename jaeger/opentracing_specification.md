<!-- ---
title: OpenTracing 语义标准
date: 2019-06-30 05:15:57
category: src, jaeger, docs
--- -->

# OpenTracing 语义与标准

OpenTracing 定义了链路追踪跨编程语言的标准。

## 1. OpenTracing 数据模型

OpenTracing标准中有三个重要的相互关联的类型，分别是Tracer, Span 和 SpanContext。

Trace（调用链）：通过归属于此调用链的Span来隐性的定义。 一条Trace（调用链）可以被认为是一个由多个Span组成的有向无环图（DAG图）， Span 与Span 的关系被命名为References。

Span：可以被翻译为跨度，可以被理解为一次方法调用, 一个程序块的调用, 或者一次RPC/数据库访问。

Trace 示例，这里展示了在单个Trace中，span间的时间关系：

```
––|–––––––|–––––––|–––––––|–––––––|–––––––|–––––––|–––––––|–> time

 [Span A···················································]
   [Span B··············································]
      [Span D··········································]
    [Span C········································]
         [Span E·······]        [Span F··] [Span G··] [Span H··]
```

Tracer 接口用来创建Span，以及处理如何处理Inject(serialize) 和 Extract (deserialize)，用于跨进程边界传递。Tracer 需要具有以下行为：

- 创建一个新Span，返回一个已经启动Span实例
- 将SpanContext 上下文Inject（注入）到carrier
- 将SpanContext上下文从carrier中Extract（提取）


## 2. Span

每个Span 包含以下的状态:

- operation name，操作名称
- start timestamp，起始时间
- finish timestamp，结束时间
- Span Tag，一组键值对构成的Span标签集合
- Span Log，一组span的日志集合。 每次log操作包含一个键值对，以及一个时间戳。
- SpanContext，Span上下文对象
- References(Span间关系)，Span 间通过SpanContext 建立这种关系。

在代码实现上 span 需要具备如下功能：

- 获取Span 的SpanContext，返回Span 构建时传入的SpanContext。
- 复写操作名（operation name）
- 结束Span
- 为Span设置tag
- Log 结构化数据
- 设置一个baggage（随行数据）元素。
- 获取一个baggage 元素

### SpanContext

任何一个OpenTracing的实现，都需要将当前调用链的状态（例如：trace和span的id），依赖一个独特的SpanContext 去跨进程边界传输

相对于OpenTracing中其他的功能，SpanContext 更多的是一个“概念”。OpenTracing的使用者仅仅需要，在创建span、向传输协议Inject（注入）和从传输协议中Extract（提取）时，使用SpanContext 和references。

### Baggage

Baggage Items，Trace 的随行数据，是一个键值对集合，它存在于trace中。这些值将设置给Span，Span的SpanContext，以及所有和此Span有直接或者间接关系的本地Span。baggage 元素会随trace 一起保持在带内传递。

Baggage元素具有强大的功能，使得OpenTracing 能够实现全栈集成，例如：任意的应用程序数据，可以在移动端创建它，显然的，它会一直传递了系统最底层的存储系统，同时他也会产生巨大的开销，请小心使用此特性。

## 3. Span间关系

一个Span 可以与一个或者多个SpanContexts 存在因果关系。OpenTracing目前定义了两种关系：ChildOf（父子） 和 FollowsFrom（跟随）。这两种关系明确的给出了两个父子关系的Span的因果模型。 

### ChildOf

ChildOf（父子）: 一个span 可能是一个父级span 的孩子，即”ChildOf”关系。在”ChildOf”引用关系下，父级span某种程度上取决于子span。

表述一个”ChildOf”关系的父子节点关系的时序图：

```
[-Parent Span--------------]
        [-Child Span A----]
        [-Child Span B----]
    [-Child Span C----]
        [-Child Span D---------------]
        [-Child Span E----]
```

### FollowsFrom

FollowsFrom 引用: 一些父级节点不以任何方式依赖他们子节点的执行结果，这种情况下，我们说这些子span和父span之间是”FollowsFrom”的因果关系。

表述一个”FollowFrom”关系的父子节点关系的时序图：

```
[-Parent Span-]  [-Child Span-]
```

##  数据模型使用规范

数据模型Span Tag 和Log 的key 使用规范。

### 标准的Span tag

Span 的tag 作用于整个Span，也就是说，它会覆盖Span 的整个事件周期，所以无需指定特别的时间戳。

| Span tag 名称 | 类型 | 描述与实例 |
|-|-|-|
| component |string| 生成此Span所相关的软件包，框架，类库或模块。如 "grpc", "django", "JDBI"|
| db.instance | string | 数据库实例名称|
| http.method | string |	Span相关的HTTP请求方法 |
| peer.address | string | 远程地址 |
| peer.service | string | 远程服务名 |
| span.kind | string | 基于RPC的调用角色，"client" 或 "server"|


### Log field 清单

每个Span 的log 操作，都具有一个特定的时间戳，并包含一个或多个 field。

| Span log field 名称 | 类型 | 描述和实例 |
| error.kind | string | 错误类型（仅在event="error"时使用）|
| message | string | 简洁的，具有高可读性的一行事件描述 |
| stack | string | 针对特定平台的栈信息描述，不强制要求与错误相关|


### 典型场景建模

rpc 调用时常用的字段：

- span.kind: "client" 或 "server"。在Span开始时，设置此tag是十分重要的，它可能影响内部ID的生成。
- error: RPC调用是否发生错误
- peer.address, peer.hostname, peer.ipv4, peer.ipv6, peer.port
- peer.service: 描述RPC的对端信息

### 关于捕获错误

OpenTracing中，根据语言的不同，错误可以通过不同的方式来进行描述。

如果存在错误对象，它其中包含栈信息和错误信息，log时使用如下的field：

- event="error"
- error.object=<error object instance>
- message="..."
- stack="..." (可选)


## 参考资料

- []()