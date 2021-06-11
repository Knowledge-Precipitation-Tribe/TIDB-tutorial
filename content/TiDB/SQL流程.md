# SQL执行流程

对于一条SQL语句处理框架如下：

- 协议解析和转换，拿到语句内容：所有的逻辑都在server这个包中，主要分为两块：
  - 连接的建立和管理，每个连接对应一个session
  - 是否建立了连接，在这个连接的基础上进行操作。
- SQL核心层，生成查询计划：这一部分最为复杂，主要因为下面三个方面
  - SQL语言本身非常复杂，语句的种类，数据类型，操作符，语法组合多
  - SQL是一门表意的语言，只是说要什么数据，而不说如何拿数据，所以需要一些复杂的逻辑选择如何拿数据，也就是选择一个好的查询计划。
  - 底层是一个分布式存储引擎，所以就要考虑在查询时下层的数据是分片的，网络有可能无法通信等情况，所以需要一些复杂的逻辑处理这些情况，并且需要一种机制将这些处理逻辑封装起来
- 从存储引擎中获取数据，进行计算并返回结果：对于这一部分可以认为是两块：
  - 第一块是KV接口层，主要作用是将请求路由到正确的KV Server，接收返回消息传给SQL层，并在此过程中处理各种异常逻辑
  - 第二块是KV server的具体实现，由于TiKV比较复杂，可以从Mock-TiKV入手。

对于SQL核心层主要设计下面几个接口：

- [Session](https://github.com/pingcap/tidb/blob/source-code/session.go#L62)
- [RecordSet](https://github.com/pingcap/tidb/blob/source-code/ast/ast.go#L136)
- [Plan](https://github.com/pingcap/tidb/blob/source-code/plan/plan.go#L30)
- [LogicalPlan](https://github.com/pingcap/tidb/blob/source-code/plan/plan.go#L140)
- [PhysicalPlan](https://github.com/pingcap/tidb/blob/source-code/plan/plan.go#L190)
- [Executor](https://github.com/pingcap/tidb/blob/source-code/executor/executor.go#L190)

## 协议层入口

