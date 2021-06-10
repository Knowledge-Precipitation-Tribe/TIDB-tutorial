p参考：[如何用-sysbench-测试-tidb](https://docs.pingcap.com/zh/tidb/stable/benchmark-tidb-using-sysbench#%E5%A6%82%E4%BD%95%E7%94%A8-sysbench-%E6%B5%8B%E8%AF%95-tidb)

## 修改集群yml文件

首先修改集群yml文件，添加config如下

```bash
server_configs:
  tidb:
    log.level: "error"
    prepared-plan-cache.enabled: true
  tikv:
    log-level: "error"
    rocksdb.defaultcf.block-cache-size: "8GB"
    rocksdb.writecf.block-cache-size: "2GB"
```

## 重启集群

```bash
tiup cluster restart 集群名
```

看到如下`Restarted cluster 集群名 successfully`则表示启动成功。

## 下载Sysbench

Sysbench可在 [Sysbench Release 1.0.14 页面](https://github.com/akopytov/sysbench/releases/tag/1.0.14)下载。

## 书写配置文件

案例如下

```
mysql-host=172.16.30.33
mysql-port=4000
mysql-user=root
mysql-password=password
mysql-db=sbtest
time=600
threads=16
report-interval=10
db-driver=mysql
```

## 登陆集群mysql修改配置

```bash
mysql --host xxxx.xxxx.xxxx.xxxx --port 4000 -u root
```

在数据导入前，需要对 TiDB 进行简单设置。在 MySQL 客户端中执行如下命令：

```bash
set global tidb_disable_txn_auto_retry = off;
```

然后退出客户端。

重新启动 MySQL 客户端执行以下 SQL 语句，首先删除之前已经存在的数据。

```bash
drop database sbtest;
```

再创建数据库 `sbtest`：

```sql
create database sbtest;
```

## 调整sysbench数据导入顺序

此步骤不清楚原文件是否修改，故未进行操作。-

## 导入数据

```bash
sysbench --config-file=config oltp_point_select --tables=50 --table-size=10000000 prepare
```

