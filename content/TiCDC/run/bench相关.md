# bench

官网首页[使用 TiUP bench 组件压测 TiDB](https://docs.pingcap.com/zh/tidb/stable/tiup-bench#%E4%BD%BF%E7%94%A8-tiup-bench-%E7%BB%84%E4%BB%B6%E5%8E%8B%E6%B5%8B-tidb)

## TPC-C

可以使用bench进行TPC-C测试，遵循以下步骤

- 通过 HASH 使用 4 个分区创建 4 个仓库：

  ```bash
  tiup bench tpcc --warehouses 4 --parts 4 prepare
  # 向指定集群写数据
  tiup bench tpcc -H xxxx -P xxxx -D tpcc10 --warehouses 10  prepare
  ```

- 运行 TPC-C 测试：

  ```bash
  tiup bench tpcc --warehouses 4 run
  ```

- 清理数据：

  ```bash
  tiup bench tpcc --warehouses 4 cleanup
  ```

- 检查一致性：

  ```bash
  tiup bench tpcc --warehouses 4 check
  ```

- 生成 CSV 文件：

  ```bash
  tiup bench tpcc --warehouses 4 prepare --output data
  ```

- 为指定的表生成 CSV 文件：

  ```bash
  tiup bench tpcc --warehouses 4 prepare --output data --tables history,orders
  ```

- 开启 pprof：

  ```bash
  tiup bench tpcc --warehouses 4 prepare --output data --pprof :10111
  ```

其中warehouse相当于标准数据集，每个大概有77mb的数据，由9张表组成。

## 多表测试

创建多表可以使用sysbench工具，参考链接：[如何用 Sysbench 测试 TiDB](https://docs.pingcap.com/zh/tidb/stable/benchmark-tidb-using-sysbench#%E5%A6%82%E4%BD%95%E7%94%A8-sysbench-%E6%B5%8B%E8%AF%95-tidb)。