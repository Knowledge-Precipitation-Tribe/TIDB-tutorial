# TiDB-tutorial
本项目是在学习 TiDB 及其周边生态工具 (CDC、DM) 的时候总结出的一些教程。

> TIDB Github：[pingcap/tidb](https://github.com/pingcap/tidb)
>
> TIDB官方文档：[TIDB](https://docs.pingcap.com/zh/tidb/stable)

## 目录
### TiDB
- [什么是TIDB](https://github.com/Knowledge-Precipitation-Tribe/TIDB-tutorial/blob/main/content/TiDB/1.%20%E4%BB%80%E4%B9%88%E6%98%AFTIDB.md)
- [安装TIDB](https://docs.pingcap.com/zh/tidb/stable/quick-start-with-tidb)
  - 启动：`tiup playground`
  - 连接TiDB：`tiup client`
- [TiDB原理](https://github.com/Knowledge-Precipitation-Tribe/TIDB-tutorial/blob/main/content/TiDB/2.%20TiDB%E5%8E%9F%E7%90%86.md)
  - TiDB架构
  - 存储
  - 计算
  - 调度
- [源码结构](https://github.com/Knowledge-Precipitation-Tribe/TIDB-tutorial/blob/main/content/TiDB/3.%20%E6%BA%90%E7%A0%81%E7%BB%93%E6%9E%84.md)
- [数据库操作](https://github.com/Knowledge-Precipitation-Tribe/TIDB-tutorial/tree/main/content/TiDB/%E6%95%B0%E6%8D%AE%E5%BA%93%E6%93%8D%E4%BD%9C)
  - [TiDB基本操作](https://github.com/Knowledge-Precipitation-Tribe/TIDB-tutorial/blob/main/content/TiDB/%E6%95%B0%E6%8D%AE%E5%BA%93%E6%93%8D%E4%BD%9C/TiDB%E5%9F%BA%E6%9C%AC%E6%93%8D%E4%BD%9C.md)
    - 数据库操作
    - 数据表操作
    - 索引操作
    - 记录的增删改查
    - 用户操作
  - [TiDB进阶操作](https://github.com/Knowledge-Precipitation-Tribe/TIDB-tutorial/blob/main/content/TiDB/%E6%95%B0%E6%8D%AE%E5%BA%93%E6%93%8D%E4%BD%9C/TiDB%E8%BF%9B%E9%98%B6%E6%93%8D%E4%BD%9C.md)
    - 读取历史数据
- [SQL流程](https://github.com/Knowledge-Precipitation-Tribe/TIDB-tutorial/blob/main/content/TiDB/4.%20SQL%E6%B5%81%E7%A8%8B.md)

### TiCDC
- [TiCDC术语表](https://github.com/Knowledge-Precipitation-Tribe/TIDB-tutorial/blob/main/content/TiCDC/0.%20TiCDC%E6%9C%AF%E8%AF%AD%E8%A1%A8.md)
- [什么是CDC](https://github.com/Knowledge-Precipitation-Tribe/TIDB-tutorial/blob/main/content/TiCDC/1.%20%E4%BB%80%E4%B9%88%E6%98%AFTiCDC.md)
- [CDC与TiKV连接](https://github.com/Knowledge-Precipitation-Tribe/TIDB-tutorial/blob/main/content/TiCDC/2.%20CDC%E4%B8%8ETiKV%E8%BF%9E%E6%8E%A5.md)
- [高可用设计](https://github.com/Knowledge-Precipitation-Tribe/TIDB-tutorial/blob/main/content/TiCDC/3.%20%E9%AB%98%E5%8F%AF%E7%94%A8%E8%AE%BE%E8%AE%A1.md)
### Chaos Mesh

- [什么是Chaos Mesh]()
- [Chaos Mesh基本功能]()
- 安装

## 推荐阅读

- [TiDB源码阅读](https://pingcap.com/blog-cn/#TiDB-%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB)
- [TiDB从入门到实践](https://www.bilibili.com/video/BV1Xz4y1Q79U?p=3&spm_id_from=pageDriver)
- [TiDB 精英进阶计划](https://www.bilibili.com/video/BV1pp4y1X7UQ?from=search&seid=14358703445510926872)
