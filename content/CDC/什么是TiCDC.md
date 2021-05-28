# 简介

[TiCDC](https://github.com/pingcap/ticdc) 是一款通过拉取 TiKV 变更日志实现的 TiDB 增量数据同步工具，具有将数据还原到与上游任意 TSO 一致状态的能力，同时提供[开放数据协议](https://docs.pingcap.com/zh/tidb/stable/ticdc-open-protocol) (TiCDC Open Protocol)，支持其他系统订阅数据变更。

## 架构

TiCDC运行时是一种无状态节点，通过PD内部的etcd来实现高可用。TiCDC集群支持创建多个同步任务，向多个不同的下游进行数据同步。

TiCDC的整体架构如下图所示：

![](https://tva1.sinaimg.cn/large/008i3skNly1gqy53x420lj31xd0u07ek.jpg)

### 系统角色

（这里描述的不清楚）

![](https://tva1.sinaimg.cn/large/008i3skNly1gqy54rr2h4j30k307jtaa.jpg)

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

