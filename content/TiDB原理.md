# TiDB原理

## TiDB架构

TiDB 分布式数据库将整体架构拆分成了多个模块，各模块之间互相通信，组成完整的 TiDB 系统。对应的架构图如下：

![TiDB架构](https://download.pingcap.com/images/docs-cn/tidb-architecture-v3.1.png)

- TiDB Server：SQL 层，对外暴露 MySQL 协议的连接 endpoint，负责接受客户端的连接，执行 SQL 解析和优化，最终生成分布式执行计划。TiDB 层本身是无状态的，实践中可以启动多个 TiDB 实例，通过负载均衡组件（如 LVS、HAProxy 或 F5）对外提供统一的接入地址，客户端的连接可以均匀地分摊在多个 TiDB 实例上以达到负载均衡的效果。TiDB Server 本身并不存储数据，只是解析 SQL，将实际的数据读取请求转发给底层的存储节点 TiKV（或 TiFlash）。
- PD (Placement Driver) Server：整个 TiDB 集群的元信息管理模块，负责存储每个 TiKV 节点实时的数据分布情况和集群的整体拓扑结构，提供 TiDB Dashboard 管控界面，并为分布式事务分配事务 ID。PD 不仅存储元信息，同时还会根据 TiKV 节点实时上报的数据分布状态，下发数据调度命令给具体的 TiKV 节点，可以说是整个集群的“大脑”。此外，PD 本身也是由至少 3 个节点构成，拥有高可用的能力。建议部署奇数个 PD 节点。
- 存储节点
  - TiKV Server：负责存储数据，从外部看 TiKV 是一个分布式的提供事务的 Key-Value 存储引擎。存储数据的基本单位是 Region，每个 Region 负责存储一个 Key Range（从 StartKey 到 EndKey 的左闭右开区间）的数据，每个 TiKV 节点会负责多个 Region。TiKV 的 API 在 KV 键值对层面提供对分布式事务的原生支持，默认提供了 SI (Snapshot Isolation) 的隔离级别，这也是 TiDB 在 SQL 层面支持分布式事务的核心。TiDB 的 SQL 层做完 SQL 解析后，会将 SQL 的执行计划转换为对 TiKV API 的实际调用。所以，数据都存储在 TiKV 中。另外，TiKV 中的数据都会自动维护多副本（默认为三副本），天然支持高可用和自动故障转移。
  - TiFlash：TiFlash 是一类特殊的存储节点。和普通 TiKV 节点不一样的是，在 TiFlash 内部，数据是以列式的形式进行存储，主要的功能是为分析型的场景加速。

![](https://tva1.sinaimg.cn/large/008i3skNly1gqfmiyv8w2j30px0f5tcw.jpg)

![](https://tva1.sinaimg.cn/large/008i3skNly1gqfmowq57wj30px0bmn10.jpg)

## SQL层架构

这幅图大体描述了 SQL 核心模块，大家可以从左边开始，顺着箭头的方向看。

![](https://tva1.sinaimg.cn/large/008eGmZEly1gpnuiojrl4j30p20a2mz8.jpg)

### Protocol Layer

最左边是 TiDB 的 Protocol Layer，这里是与 Client 交互的接口，目前 TiDB 只支持 MySQL 协议，相关的代码都在 `server` 包中。

这一层的主要功能是管理客户端 connection，解析 MySQL 命令并返回执行结果。具体的实现是按照 MySQL 协议实现，具体的协议可以参考 [MySQL 协议文档](https://dev.mysql.com/doc/internals/en/client-server-protocol.html)。这个模块我们认为是当前实现最好的一个 MySQL 协议组件，如果大家的项目中需要用到 MySQL 协议解析、处理的功能，可以参考或引用这个模块。

连接建立的逻辑在 server.go 的 [Run()](https://github.com/pingcap/tidb/blob/source-code/server/server.go#L236) 方法中，主要是下面两行：

```go
conn, err := s.listener.Accept()
go s.onConn(conn)
```

单个 Session 处理命令的入口方法是调用 clientConn 类的 [dispatch 方法](https://github.com/pingcap/tidb/blob/source-code/server/conn.go#L465)，这里会解析协议并转给不同的处理函数。

### SQL Layer

大体上讲，一条 SQL 语句需要经过，语法解析–>合法性验证–>制定查询计划–>优化查询计划–>根据计划生成查询器–>执行并返回结果 等一系列流程。这个主干对应于 TiDB 的下列包：

| Package    | 作用                                          |
| :--------- | :-------------------------------------------- |
| tidb       | Protocol 层和 SQL 层之间的接口                |
| parser     | 语法解析                                      |
| plan       | 合法性验证 + 制定查询计划 + 优化查询计划      |
| executor   | 执行器生成以及执行                            |
| distsql    | 通过 TiKV Client 向 TiKV 发送以及汇总返回结果 |
| store/tikv | TiKV Client                                   |

### KV API Layer

TiDB 依赖于底层的存储引擎提供数据的存取功能，但是并不是依赖于特定的存储引擎（比如 TiKV），而是对存储引擎提出一些要求，满足这些要求的引擎都能使用（其中 TiKV 是最合适的一款）。

最基本的要求是『带事务的 Key-Value 引擎，且提供 Go 语言的 Driver』，再高级一点的要求是『支持分布式计算接口』，这样 TiDB 可以把一些计算请求下推到 存储引擎上进行。

这些要求都可以在 `kv` 这个包的[接口](https://github.com/pingcap/tidb/blob/source-code/kv/kv.go)中找到，存储引擎需要提供实现了这些接口的 Go 语言 Driver，然后 TiDB 利用这些接口操作底层数据。

对于最基本的要求，可以重点看这几个接口：

- [Transaction](https://github.com/pingcap/tidb/blob/source-code/kv/kv.go#L121)：事务基本操作
- [Retriever ](https://github.com/pingcap/tidb/blob/source-code/kv/kv.go#L75)：读取数据的接口
- [Mutator](https://github.com/pingcap/tidb/blob/source-code/kv/kv.go#L91)：修改数据的接口
- [Storage](https://github.com/pingcap/tidb/blob/source-code/kv/kv.go#L229)：Driver 提供的基本功能
- [Snapshot](https://github.com/pingcap/tidb/blob/source-code/kv/kv.go#L214)：在数据 Snapshot 上面的操作
- [Iterator](https://github.com/pingcap/tidb/blob/source-code/kv/kv.go#L255)：`Seek` 返回的结果，可以用于遍历数据

有了上面这些接口，可以对数据做各种所需要的操作，完成全部 SQL 功能，但是为了更高效的进行运算，我们还定义了一个高级计算接口，可以关注这三个 Interface/struct :

- [Client](https://github.com/pingcap/tidb/blob/source-code/kv/kv.go#L150)：向下层发送请求以及获取下层存储引擎的计算能力
- [Request](https://github.com/pingcap/tidb/blob/source-code/kv/kv.go#L176): 请求的内容
- [Response](https://github.com/pingcap/tidb/blob/source-code/kv/kv.go#L204): 返回结果的抽象

## 存储

> 参考来源：[三篇文章了解 TiDB 技术内幕 - 说存储](https://pingcap.com/blog-cn/tidb-internal-1/)

数据库最为核心的作用就是把数据保存下来，对于数据存储一般有两种方式

- 存储在内容中，访问速度快，但是一旦停机或服务重启数据就会丢失
- 存储在硬盘中，可以持久化存储，但是如果磁盘出现问题也会导致数据丢失，还可以在这个基础上做冗余备份，但是也存在下面的一些问题：
  - 如何做容灾处理
  - 如何保证数据的一致性？
  - 数据保存下来后，是否方便读取？
  - 保存的数据如何修改？如何支持并发的修改？
  - 如何原子地修改多条记录？

抛开数据库，在TiDB中是如何做数据存储的呢？TiDB中使用的是TiKV项目。TiKV使用的Key-Value模型，并且提供了有序的遍历方法。对于TiKV有以下两点很重要：

- TiKV可以看作一个巨大的Map
- Map中的键值对是按照key的二进制顺序有序存储的，可以通过Seek定位某一个Key的位置，然后不断调用Next方法获取比这个Key更大的键值对。

对于任何持久化存储，终究还是要讲数据存储到磁盘上，但是TiKV并不是直接将数据写入磁盘，而是将数据保存在[RocksDB](https://github.com/facebook/rocksdb)中，而具体的数据落地由RocksDB负责。

到现在为止解决了数据的本地存储问题，但是如何保证数据不丢失呢？最简单的办法就是做数据冗余，这样一台机器挂了也没问题，但是如何保证数据的一致性也是一个问题，TiKV中使用的是优化的[Raft协议](https://zhuanlan.zhihu.com/p/25735592)，TiKV用Raft协议来做数据复制，每个数据的变更都会落地为一条Raft日志，通过 Raft 的日志复制功能，将数据安全可靠地同步到 Group 的多数节点中。

![](https://download.pingcap.com/images/blog-cn/tidb-internal-1/2.png)

总结下来，通过单机的 RocksDB，我们可以将数据快速地存储在磁盘上；通过 Raft，我们可以将数据复制到多台机器上，以防单机失效。数据的写入是通过 Raft 这一层的接口写入，而不是直接写 RocksDB。通过实现 Raft，我们拥有了一个分布式的 KV，现在再也不用担心某台机器挂掉了。

但是还存在一些问题，对于海量的数据存储，肯定不能存储在单机上，必须要将数据分散在多台机器上，而对于键值对来说一般有以下两种方案：

- 按Key做Hash，然后根据Hash结果将数据存储在不同节点上
- 分Range，某个连续的Key段都保存在一个存储节点上，TiKV使用的就是这种方式。

TiKV会将整个键值对分为很多段，每一段都是一系列连续的Key，每一段叫做Region，而且尽量保持每个 Region 中保存的数据不超过一定的大小(这个大小可以配置，目前默认是 96mb)。每一个 Region 都可以用 StartKey 到 EndKey 这样一个左闭右开区间来描述。

将数据分Region后就可以做以下两件事

- 以Region为单位将数据分散在集群的节点上，并且保证每个节点上服务的Region数量差不多。系统会有组件来负责将Region尽可能均匀的散布在集群中所有节点上，这样一方面实现了存储容量的水平拓展，另一方面也实现了负载均衡。而且也会有单独的组件记录当前Key是属于哪个Region中。
- 以Region为单位做Raft的复制和成员管理，一个Region数据回保存多个副本，每个副本叫做一个Replica，每个Replica通过Raft来保持数据的一致。一个 Region 的多个 Replica 会保存在不同的节点上，构成一个 Raft Group。其中一个 Replica 会作为这个 Group 的 Leader，其他的 Replica 作为 Follower。所有的读和写都是通过 Leader 进行，再由 Leader 复制给 Follower。如下图所示![KeyValue](https://download.pingcap.com/images/blog-cn/tidb-internal-1/4.png)

通过上面的方式就有了一个分布式的具备一定容灾能力的 KeyValue 系统，不用再担心数据存不下，或者是磁盘故障丢失数据的问题。但是还有问题需要解决，那就是如何保证并发行。

TiKV为了实现更好的并发行能使用的是MVCC，也就是在每个Key后面添加Version来实现，对于同一个 Key 的多个版本，我们把版本号较大的放在前面，版本号小的放在后面，这样当用户通过一个 Key + Version 来获取 Value 的时候，可以将 Key 和 Version 构造出 MVCC 的 Key，也就是 Key-Version。然后可以直接 Seek(Key-Version)，定位到第一个大于等于这个 Key-Version 的位置。

## 计算

前面的内容中介绍了如何存储数据，那么现在的主要问题变为，如何在KV接收上保存Table以及如何在KV结构上运行SQL语句。SQL和KV结构之间存在巨大的差别，如何能够方便高效的进行映射是一个很重要的问题。一个好的映射方案必须有利于对数据操作的需求。那么我们先看一下对数据的操作有哪些需求，分别有哪些特点。对于一个 Table 来说，**需要存储的数据包括三部分：**

1. 表的元信息
2. Table 中的 Row
3. 索引数据

对于 Row，可以选择行存或者列存，这两种各有优缺点。TiDB 面向的首要目标是 OLTP 业务，这类业务需要支持快速地读取、保存、修改、删除一行数据，所以采用**行存**是比较合适的。

对于 Index，TiDB 不止需要支持 Primary Index，还需要支持 Secondary Index。Index 的作用的辅助查询，提升查询性能，以及保证某些 Constraint。查询的时候有两种模式，一种是点查，比如通过 Primary Key 或者 Unique Key 的等值条件进行查询，如 `select name from user where id=1;` ，这种需要通过索引快速定位到某一行数据；另一种是 Range 查询，如 `select name from user where age > 30 and age < 35;`，这个时候需要通过`idxAge`索引查询 age 在 30 和 35 之间的那些数据。Index 还分为 Unique Index 和 非 Unique Index，这两种都需要支持。

以上就是数据存储的特点，还需要看看对于**数据操作的需求**，主要考虑insert/update/delete/select这四种语句。

- 对于 Insert 语句，需要将 Row 写入 KV，并且建立好索引数据。

- 对于 Update 语句，需要将 Row 更新的同时，更新索引数据（如果有必要）。

- 对于 Delete 语句，需要在删除 Row 的同时，将索引也删除。

- 对于 Select 语句，情况会复杂一些。
  - 首先我们需要能够简单快速地读取一行数据，所以每个 Row 需要有一个 ID （显示或隐式的 ID）。
  - 其次可能会读取连续多行数据，比如 `Select * from user;`。
  - 最后还有通过索引读取数据的需求，对索引的使用可能是点查或者是范围查询。

针对于上面的需求，我们手中有的是一个**全局有序的分布式Key-Value引擎**。全局有序这一点重要，可以帮助我们解决不少问题。比如对于快速获取一行数据，假设我们能够构造出某一个或者某几个 Key，定位到这一行，我们就能利用 TiKV 提供的 Seek 方法快速定位到这一行数据所在位置。再比如对于扫描全表的需求，如果能够映射为一个 Key 的 Range，从 StartKey 扫描到 EndKey，那么就可以简单的通过这种方式获得全表数据。操作 Index 数据也是类似的思路。接下来让我们看看 TiDB 是如何做的。

首先来看一下数据是如何存储的，首先TiDB会为每个表分配一个TableID，每一个索引都会分配一个 IndexID，每一行分配一个 RowID（如果表有整数型的 Primary Key，那么会用 Primary Key 的值当做 RowID），其中 TableID 在整个集群内唯一，IndexID/RowID 在表内唯一，这些 ID 都是 int64 类型。

每行数据按照如下规则进行编码成为键值对：

```
Key: tablePrefix{tableID}_recordPrefixSep{rowID}
Value: [col1, col2, col3, col4]
```

其中 Key 的 `tablePrefix`/`recordPrefixSep` 都是特定的字符串常量，用于在 KV 空间内区分其他数据。

对于 Index 数据，会按照如下规则编码成键值对：

```fallback
Key: tablePrefix{tableID}_indexPrefixSep{indexID}_indexedColumnsValue
Value: rowID
```

Index 数据还需要考虑 Unique Index 和非 Unique Index 两种情况，对于 Unique Index，可以按照上述编码规则。但是对于非 Unique Index，通过这种编码并不能构造出唯一的 Key，因为同一个 Index 的 `tablePrefix{tableID}_indexPrefixSep{indexID}` 都一样，可能有多行数据的 `ColumnsValue` 是一样的，所以对于**非 Unique Index 的编码做了一点调整**：

```fallback
Key: tablePrefix{tableID}_indexPrefixSep{indexID}_indexedColumnsValue_rowID
Value: null
```

这样能够对索引中的每行数据构造出唯一的 Key。 注意上述编码规则中的 Key 里面的各种 xxPrefix 都是字符串常量，作用都是区分命名空间，以免不同类型的数据之间相互冲突，定义如下：

```fallback
var(
	tablePrefix     = []byte{'t'}
	recordPrefixSep = []byte("_r")
	indexPrefixSep  = []byte("_i")
)
```

另外请大家注意，上述方案中，无论是 Row 还是 Index 的 Key 编码方案，一个 Table 内部所有的 Row 都有相同的前缀，一个 Index 的数据也都有相同的前缀。这样具体相同的前缀的数据，在 TiKV 的 Key 空间内，是排列在一起。同时只要我们小心地设计后缀部分的编码方案，保证编码前和编码后的比较关系不变，那么就可以将 Row 或者 Index 数据有序地保存在 TiKV 中。这种**`保证编码前和编码后的比较关系不变` **的方案我们称为 Memcomparable，对于任何类型的值，两个对象编码前的原始类型比较结果，和编码成 byte 数组后（注意，TiKV 中的 Key 和 Value 都是原始的 byte 数组）的比较结果保持一致。具体的编码方案参见 TiDB 的 [codec 包](https://github.com/pingcap/tidb/tree/master/util/codec)。采用这种编码后，一个表的所有 Row 数据就会按照 RowID 的顺序排列在 TiKV 的 Key 空间中，某一个 Index 的数据也会按照 Index 的 ColumnValue 顺序排列在 Key 空间内。

有了这样的存储方式后，我们再看一下之前提到的需求是否可以被满足。

- 首先我们通过这个映射方案，将 Row 和 Index 数据都转换为 Key-Value 数据，且每一行、每一条索引数据都是有唯一的 Key。
- 其次，这种映射方案对于点查、范围查询都很友好，我们可以很容易地构造出某行、某条索引所对应的 Key，或者是某一块相邻的行、相邻的索引值所对应的 Key 范围。
- 最后，在保证表中的一些 Constraint 的时候，可以通过构造并检查某个 Key 是否存在来判断是否能够满足相应的 Constraint。

下面通过几个例子来理解一下，假设表中有三条数据：

```
1, "TiDB", "SQL Layer", 10
2, "TiKV", "KV Engine", 20
3, "PD", "Manager", 30
```

那么首先每行数据都会映射为一个 Key-Value pair，注意这个表有一个 Int 类型的 Primary Key，所以 RowID 的值即为这个 Primary Key 的值。假设这个表的 Table ID 为 10，其 Row 的数据为：

```
t10_r1 --> ["TiDB", "SQL Layer", 10]
t10_r2 --> ["TiKV", "KV Engine", 20]
t10_r3 --> ["PD", "Manager", 30]
```

除了 Primary Key 之外，这个表还有一个 Index，假设这个 Index 的 ID 为 1，则其数据为：

```fallback
t10_i1_10_1 --> null
t10_i1_20_2 --> null
t10_i1_30_3 --> null
```

大家可以结合上面的编码规则来理解这个例子，希望大家能理解我们为什么选择了这个映射方案，这样做的目的是什么。

上面介绍了如何存储数据，那么对于Database/Table的元信息如何存储呢？在TiDB中每个 Database/Table 都被分配了一个唯一的 ID，这个 ID 作为唯一标识，并且在编码为 Key-Value 时，这个 ID 都会编码到 Key 中，再加上 `m_` 前缀。这样可以构造出一个Key，Value中存储的是序列化后的元信息。 除此之外，还有一个专门的 Key-Value 存储当前 Schema 信息的版本。TiDB 使用 Google F1 的 Online Schema 变更算法，有一个后台线程在不断的检查 TiKV 上面存储的 Schema 版本是否发生变化，并且保证在一定时间内一定能够获取版本的变化（如果确实发生了变化）。

## 调度

TiKV集群是TiDB数据库的分布式KV存储引擎，数据以 Region 为单位进行复制和管理，每个 Region 会有多个 Replica（副本），这些 Replica 会分布在不同的 TiKV 节点上，其中 Leader 负责读/写，Follower 负责同步 Leader 发来的 raft log。了解了这些信息后，请思考下面这些问题：

- 如何保证同一个 Region 的多个 Replica 分布在不同的节点上？更进一步，如果在一台机器上启动多个 TiKV 实例，会有什么问题？
- TiKV 集群进行跨机房部署用于容灾的时候，如何保证一个机房掉线，不会丢失 Raft Group 的多个 Replica？
- 添加一个节点进入 TiKV 集群之后，如何将集群中其他节点上的数据搬过来?
- 当一个节点掉线时，会出现什么问题？整个集群需要做什么事情？如果节点只是短暂掉线（重启服务），那么如何处理？如果节点是长时间掉线（磁盘故障，数据全部丢失），需要如何处理？
- 假设集群需要每个 Raft Group 有 N 个副本，那么对于单个 Raft Group 来说，Replica 数量可能会不够多（例如节点掉线，失去副本），也可能会过于多（例如掉线的节点又回复正常，自动加入集群）。那么如何调节 Replica 个数？
- 读/写都是通过 Leader 进行，如果 Leader 只集中在少量节点上，会对集群有什么影响？
- 并不是所有的 Region 都被频繁的访问，可能访问热点只在少数几个 Region，这个时候我们需要做什么？
- 集群在做负载均衡的时候，往往需要搬迁数据，这种数据的迁移会不会占用大量的网络带宽、磁盘 IO 以及 CPU？进而影响在线服务？

以上的问题如果没有一个全局的调度是很难同时满足以上的需求，因此我们需要一个中心节点来对系统的整体情况进行把控和调整，所以就有了PD模块。

对于上面的需求，可以对其进行分解为两大类：

- 分布式高可用存储系统，必须满足的需求
  - 副本数量不能多也不能少
  - 副本需要分布在不同的机器上
  - 新加节点后，可以将其他节点上的副本迁移过来
  - 节点下线后，需要将该节点的数据迁移走
- 良好的分布式系统需要优化的地方
  - 维持整个集群的 Leader 分布均匀
  - 维持每个节点的储存容量均匀
  - 维持访问热点分布均匀
  - 控制 Balance 的速度，避免影响在线服务
  - 管理节点状态，包括手动上线/下线节点，以及自动下线失效节点

满足第一类需求后，整个系统将具备多副本容错、动态扩容/缩容、容忍节点掉线以及自动错误恢复的功能。满足第二类需求后，可以使得整体系统的负载更加均匀、且可以方便的管理。

为了满足这些需求，首先我们需要收集足够的信息，比如每个节点的状态、每个 Raft Group 的信息、业务访问操作的统计等；其次需要设置一些策略，PD 根据这些信息以及调度的策略，制定出尽量满足前面所述需求的调度计划；最后需要一些基本的操作，来完成调度计划。

对于调度来说，最基本的操作可以分为下面三件事：

- 增加一个 Replica
- 删除一个 Replica
- 将 Leader 角色在一个 Raft Group 的不同 Replica 之间 transfer

刚好 Raft 协议能够满足这三种需求，通过 AddReplica、RemoveReplica、TransferLeader 这三个命令，可以支撑上述三种基本操作。

到现在为止我们知道可以通过Raft来做基本的调度了，但是对于调度来说我们得需要集群的信息才能做进一步调度，简单来说我们需要知道每个TiKV节点的状态以及每个Region的状态。TiKV集群会向PD汇报两类消息：

- **每个 TiKV 节点会定期向 PD 汇报节点的整体信息**：TiKB节点(Store)与PD之间存在心跳包，一方面PD 通过心跳包检测每个 Store 是否存活，以及是否有新加入的 Store；另一方面，心跳包中也会携带这个 [Store 的状态信息](https://github.com/pingcap/kvproto/blob/master/proto/pdpb.proto#L294)，主要包括：
  - 总磁盘容量
  - 可用磁盘容量
  - 承载的 Region 数量
  - 数据写入速度
  - 发送/接受的 Snapshot 数量（Replica 之间可能会通过 Snapshot(快照复制) 同步数据）
  - 是否过载
  - 标签信息（标签是具备层级关系的一系列 Tag）
- **每个 Raft Group 的 Leader 会定期向 PD 汇报信息**：每个 Raft Group 的 Leader 和 PD 之间存在心跳包，用于汇报这个 [Region 的状态](https://github.com/pingcap/kvproto/blob/master/proto/pdpb.proto#L207)，主要包括下面几点信息：
  - Leader 的位置
  - Followers 的位置
  - 掉线 Replica 的个数
  - 数据写入/读取的速度

PD 不断的通过这两类心跳消息收集整个集群的信息，再以这些信息作为决策的依据。除此之外，PD还可以通过管理接口接受额外的信息，用来做更准确的决策。**比如当某个 Store 的心跳包中断的时候，PD 并不能判断这个节点是临时失效还是永久失效，只能经过一段时间的等待（默认是 30 分钟）**，如果一直没有心跳包，就认为是 Store 已经下线，再决定需要将这个 Store 上面的 Region 都调度走。但是有的时候，是运维人员主动将某台机器下线，这个时候，可以通过 PD 的管理接口通知 PD 该 Store 不可用，PD 就可以马上判断需要将这个 Store 上面的 Region 都调度走。

> TODO：探索一下为什么有心跳包的情况下还需要等待30分钟才能判断节点下线。每次心跳的时间间隔是多久

在收集到了以上的信息以后，就需要执行具体的调度策略了。

- **一个 Region 的 Replica 数量正确**：当 PD 通过某个 Region Leader 的心跳包发现这个 Region 的 Replica 数量不满足要求时，需要通过 Add/Remove Replica 操作调整 Replica 数量。出现这种情况的可能原因是：

  - 某个节点掉线，上面的数据全部丢失，导致一些 Region 的 Replica 数量不足
  - 某个掉线节点又恢复服务，自动接入集群，这样之前已经补足了 Replica 的 Region 的 Replica 数量多过，需要删除某个 Replica
  - 管理员调整了副本策略，修改了 [max-replicas](https://github.com/pingcap/pd/blob/master/conf/config.toml#L54) 的配置

- **一个 Raft Group 中的多个 Replica 不在同一个位置**：『一个 Raft Group 中的多个 Replica 不在同一个位置』，这里用的是『同一个位置』而不是『同一个节点』。在一般情况下，PD 只会保证多个 Replica 不落在一个节点上，以避免单个节点失效导致多个 Replica 丢失。在实际部署中，还可能出现下面这些需求：

  - 多个节点部署在同一台物理机器上
  - TiKV 节点分布在多个机架上，希望单个机架掉电时，也能保证系统可用性
  - TiKV 节点分布在多个互联网数据中心(IDC)中，希望单个机房掉电时，也能保证系统可用

  这些需求本质上都是某一个节点具备共同的位置属性，构成一个最小的容错单元，我们希望这个单元内部不会存在一个 Region 的多个 Replica。这个时候，可以给节点配置 [lables](https://github.com/pingcap/tikv/blob/master/etc/config-template.toml#L16) 并且通过在 PD 上配置 [location-labels](https://github.com/pingcap/pd/blob/master/conf/config.toml#L59) 来指明哪些 lable 是位置标识，需要在 Replica 分配的时候尽量保证不会有一个 Region 的多个 Replica 所在结点有相同的位置标识。

- **副本在 Store 之间的分布均匀分配**：每个副本中存储的数据容量上限是固定的，所以我们维持每个节点上面，副本数量的均衡，会使得总体的负载更均衡。

- **Leader 数量在 Store 之间均匀分配**：Raft 协议要读取和写入都通过 Leader 进行，所以计算的负载主要在 Leader 上面，PD 会尽可能将 Leader 在节点间分散开。

- **访问热点数量在 Store 之间均匀分配**：每个 Store 以及 Region Leader 在上报信息时携带了当前访问负载的信息，比如 Key 的读取/写入速度。PD 会检测出访问热点，且将其在节点之间分散开。

- **各个 Store 的存储空间占用大致相等**：每个 Store 启动的时候都会指定一个 Capacity 参数，表明这个 Store 的存储空间上限，PD 在做调度的时候，会考虑节点的存储空间剩余量。

- **控制调度速度，避免影响在线服务**：调度操作需要耗费 CPU、内存、磁盘 IO 以及网络带宽，我们需要避免对线上服务造成太大影响。PD 会对当前正在进行的操作数量进行控制，默认的速度控制是比较保守的，如果希望加快调度(比如已经停服务升级，增加新节点，希望尽快调度)，那么可以通过 pd-ctl 手动加快调度速度。

- **支持手动下线节点**：当通过 pd-ctl 手动下线节点后，PD 会在一定的速率控制下，将节点上的数据调度走。当调度完成后，就会将这个节点置为下线状态。

