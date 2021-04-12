# TiDB原理

## TiDB架构

TiDB 分布式数据库将整体架构拆分成了多个模块，各模块之间互相通信，组成完整的 TiDB 系统。对应的架构图如下：

![TiDB架构](https://download.pingcap.com/images/docs-cn/tidb-architecture-v3.1.png)

- TiDB Server：SQL 层，对外暴露 MySQL 协议的连接 endpoint，负责接受客户端的连接，执行 SQL 解析和优化，最终生成分布式执行计划。TiDB 层本身是无状态的，实践中可以启动多个 TiDB 实例，通过负载均衡组件（如 LVS、HAProxy 或 F5）对外提供统一的接入地址，客户端的连接可以均匀地分摊在多个 TiDB 实例上以达到负载均衡的效果。TiDB Server 本身并不存储数据，只是解析 SQL，将实际的数据读取请求转发给底层的存储节点 TiKV（或 TiFlash）。
- PD (Placement Driver) Server：整个 TiDB 集群的元信息管理模块，负责存储每个 TiKV 节点实时的数据分布情况和集群的整体拓扑结构，提供 TiDB Dashboard 管控界面，并为分布式事务分配事务 ID。PD 不仅存储元信息，同时还会根据 TiKV 节点实时上报的数据分布状态，下发数据调度命令给具体的 TiKV 节点，可以说是整个集群的“大脑”。此外，PD 本身也是由至少 3 个节点构成，拥有高可用的能力。建议部署奇数个 PD 节点。
- 存储节点
  - TiKV Server：负责存储数据，从外部看 TiKV 是一个分布式的提供事务的 Key-Value 存储引擎。存储数据的基本单位是 Region，每个 Region 负责存储一个 Key Range（从 StartKey 到 EndKey 的左闭右开区间）的数据，每个 TiKV 节点会负责多个 Region。TiKV 的 API 在 KV 键值对层面提供对分布式事务的原生支持，默认提供了 SI (Snapshot Isolation) 的隔离级别，这也是 TiDB 在 SQL 层面支持分布式事务的核心。TiDB 的 SQL 层做完 SQL 解析后，会将 SQL 的执行计划转换为对 TiKV API 的实际调用。所以，数据都存储在 TiKV 中。另外，TiKV 中的数据都会自动维护多副本（默认为三副本），天然支持高可用和自动故障转移。
  - TiFlash：TiFlash 是一类特殊的存储节点。和普通 TiKV 节点不一样的是，在 TiFlash 内部，数据是以列式的形式进行存储，主要的功能是为分析型的场景加速。

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

TiKV的分布式事务使用的是 [Percolator](https://www.usenix.org/legacy/event/osdi10/tech/full_papers/Peng.pdf) 模型，采用乐观锁的形式，事务的执行过程中，不会检测写写冲突，只有在提交过程中，才会做冲突检测，冲突的双方中比较早完成提交的会写入成功，另一方会尝试重新执行整个事务。

