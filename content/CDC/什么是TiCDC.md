# 简介

[TiCDC](https://github.com/pingcap/ticdc) 是一款通过拉取 TiKV 变更日志实现的 TiDB 增量数据同步工具，具有将数据还原到与上游任意 TSO 一致状态的能力，同时提供[开放数据协议](https://docs.pingcap.com/zh/tidb/stable/ticdc-open-protocol) (TiCDC Open Protocol)，支持其他系统订阅数据变更。（简单来说就是把TiKV中数据的变化实时同步到下游）

## 架构

TiCDC运行时是一种无状态节点，通过PD内部的etcd来实现高可用。TiCDC集群支持创建多个同步任务，向多个不同的下游进行数据同步。

TiCDC的整体架构如下图所示：

![](https://tva1.sinaimg.cn/large/008i3skNly1gqy53x420lj31xd0u07ek.jpg)

## 数据流转图

![](https://tva1.sinaimg.cn/large/008i3skNly1gre4mob9u9j31160is0wd.jpg)

### 系统角色

- capture：就是一个cdc的实例
- owner：owner 会维护全局的同步状态，会对集群的同步进行监控和适当的调度，owner 运行有以下逻辑：
  - table 同步调度调度 table 的同步任务，分发到某个节点或从某个节点删除
  - 维护运行的 processor 状态信息，对于异常节点进行清理
  - 执行处理 DDL，向下游同步 DDL
  - 更新每一个 changefeed 的全局的 CheckpointTS 和 ResolvedTS
    - 首先这个TS是事务上的TS
    - Checkpoint ts：
      - 代表cdc写到下游的事务的 ts，当前ts代表数据已经写入成功写入下游
      - 这个Ts是对于Changefeed 级别来说的
    - Resolved ts：
      - 代表TiKV 写到 ticdc 上的数据的ts，所有小于 resolved ts 的 commit 的事务已经完整发送到 ticdc 上
      - 这个TS是Region 级别的
    - Txn: Start ts=10, current TSO = 14, resolved ts <10
      - Commit ts = 15
      - Resolved ts = 16
      - Prewrite 10 + commit 15 ⇒ 完整的已经提交的事务
- processor：实际负责将TiKV中数据同步到下游的工作节点，它会负责几个table，并将这些表的 table id 按照 TiDB 内部的 key 编码逻辑进行编码得到一些需要获取 kv change log 的 key range，processor 会综合 key range 和 region 分布信息创建多个 EventFeed gRPC stream。具体流程如下
  - 首先每个table中的数据是存在一个区间的也就是key range
  - 然后每个range中的数据会拆分为多个region存在多个tikv中
  - cdc根据要同步的table定位到指定的region后会去pd中拿元信息，也就是对应的region存在那个TiKV中，并创建eventfeed连接到TiKV。
  - eventfeed会根据regionid定位到TiKV中对应的raft group leader节点上
  - 接下来就是leader会将数据变化推送到TiKV cdc component，eventfeed同步数据到cdc中

## 同步功能介绍

### sink支持

目前TiCDC sink模块支持同步数据到以下下游：

- MySQL协议兼容的数据库，提供最终一致性支持
- 以 TiCDC Open Protocol 输出到 Kafka，可实现行级别有序、最终一致性或严格事务一致性三种一致性保证。

### 同步顺序保证和一致性保证

#### 数据同步顺序

- TiCDC 对于所有的 DDL/DML 都能对外输出**至少一次**。
- TiCDC 在 TiKV/TiCDC 集群故障期间可能会重复发相同的 DDL/DML。对于重复的 DDL/DML：
  - MySQL sink 可以重复执行 DDL，对于在下游可重入的 DDL （譬如 truncate table）直接执行成功；对于在下游不可重入的 DDL（譬如 create table），执行失败，TiCDC 会忽略错误继续同步。
  - Kafka sink 会发送重复的消息，但重复消息不会破坏 Resolved Ts 的约束，用户可以在 Kafka 消费端进行过滤。

#### 数据同步一致性

- MySQL sink
  - TiCDC 不拆分单表事务，**保证**单表事务的原子性。
  - TiCDC **不保证**下游事务的执行顺序和上游完全一致。
  - TiCDC 以表为单位拆分跨表事务，**不保证**跨表事务的原子性。
  - TiCDC **保证**单行的更新与上游更新顺序一致。
- Kafka sink
  - TiCDC 提供不同的数据分发策略，可以按照表、主键或 ts 等策略分发数据到不同 Kafka partition。
  - 不同分发策略下 consumer 的不同实现方式，可以实现不同级别的一致性，包括行级别有序、最终一致性或跨表事务一致性。
  - TiCDC 没有提供 Kafka 消费端实现，只提供了 [TiCDC 开放数据协议](https://docs.pingcap.com/zh/tidb/stable/ticdc-open-protocol)，用户可以依据该协议实现 Kafka 数据的消费端。

## 同步限制

TiCDC 只能同步至少存在一个**有效索引**的表，**有效索引**的定义如下：

- 主键 (`PRIMARY KEY`) 为有效索引。
- 同时满足下列条件的唯一索引  (`UNIQUE INDEX`) 为有效索引：
  - 索引中每一列在表结构中明确定义非空 (`NOT NULL`)。
  - 索引中不存在虚拟生成列 (`VIRTUAL GENERATED COLUMNS`)。

TiCDC 从 4.0.8 版本开始，可通过修改任务配置来同步**没有有效索引**的表，但在数据一致性的保证上有所减弱。具体使用方法和注意事项参考[同步没有有效索引的表](https://docs.pingcap.com/zh/tidb/stable/manage-ticdc#同步没有有效索引的表)。

