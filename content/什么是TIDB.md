# 什么是TIDB

[TiDB](https://github.com/pingcap/tidb) 是 [PingCAP](https://pingcap.com/about-cn/) 公司自主设计、研发的开源分布式关系型数据库，是一款同时支持在线事务处理与在线分析处理 (Hybrid Transactional and Analytical Processing, HTAP）的融合型分布式数据库产品，具备水平扩容或者缩容、金融级高可用、实时 HTAP、云原生的分布式数据库、兼容 MySQL 5.7 协议和 MySQL 生态等重要特性。目标是为用户提供一站式 OLTP (Online Transactional Processing)、OLAP (Online Analytical Processing)、HTAP 解决方案。TiDB 适合高可用、强一致要求较高、数据规模较大等各种应用场景。

> HTAP：在互联网浪潮出现之前，企业的数据量普遍不大，特别是核心的业务数据，通常一个单机的数据库就可以保存。那时候的存储并不需要复杂的架构，所有的线上请求 (OLTP, Online Transactional Processing) 和后台分析 (OLAP, Online Analytical Processing) 都跑在同一个数据库实例上。
>
> 随着互联网的发展，企业的业务数据量不断增多，单机数据库的容量限制制约了其在海量数据场景下的使用。因此在实际应用中，为了面对各种需求，OLTP、OLAP 在技术上分道扬镳，在很多企业架构中，这两类任务处理由不同团队完成。当物联网大数据应用不断深入，具有海量的传感器数据要求实时更新和查询，对数据库的性能要求也越来越高，此时，新的问题随之出现：
>
> 1. **OLAP 和 OLTP 系统间通常会有几分钟甚至几小时的时延，OLAP 数据库和 OLTP 数据库之间的一致性无法保证，难以满足对分析的实时性要求很高的业务场景。**
> 2. **企业需要维护不同的数据库以便支持两类不同的任务，管理和维护成本高。**
>
> 因此，能够统一支持事务处理和工作负载分析的数据库成为众多企业的需求。在此背景下，由 Gartner 提出的 HTAP（混合事务 / 分析处理，Hybrid Transactional/Analytical Processing）成为希望。基于创新的计算存储框架，HTAP数据库能够在一份数据上同时支撑业务系统运行和 OLAP 场景，避免在传统架构中，在线与离线数据库之间大量的数据交互。此外，HTAP 基于分布式架构，支持弹性扩容，可按需扩展吞吐或存储，轻松应对高并发、海量数据场景。
>
> 转载自：https://zhuanlan.zhihu.com/p/118592173 by TiDB Robot
>
> OLTP：强调支持短时间内大量并发的事务操作（增删改查）能力，每个操作涉及的数据量都很小，强调事务的强一致性。
>
> OLAP：偏向于更复杂的只读查询，读取海量数据进行分析计算，查询时间往往比较长，例如双十一结束之后对所有交易数据进行分析。

# TIDB的五大核心特点

- 一键水平扩容或者缩容
- 金融级高可用
- 实时HTAP
- 云原生的分布式数据库
- 兼容 MySQL 5.7 协议和 MySQL 生态

