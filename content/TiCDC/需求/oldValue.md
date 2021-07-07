# TiDB中的new collation和old value

## new collation

TiDB内部支持的新的排序规则，对应一个表。使用如下配置开启new collation

```toml
new_collations_enabled_on_first_bootstrap = true
```

## old value

Old Value 是指被 update/delete 的某行数据在 update/delete 之前的值。在许多 CDC 协议输出给下游的数据中都包含 Old Value。

## 现在的实现方式

在TiKV中暴露了这样的一个rpc接口，支持TiCDC可以读取Old Value

```protobuf
enum ExtraOp {
    Noop = 0;
    ReadOldValue = 1;
}

message ChangeDataRequest {
    Header header = 1;
    uint64 region_id = 2;
    metapb.RegionEpoch region_epoch = 3;

    uint64 checkpoint_ts = 4;
    bytes start_key = 5;
    bytes end_key = 6;
    // Used for CDC to identify events corresponding to different requests.
    uint64 request_id = 7;
    kvrpcpb.ExtraOp extra_op = 8;
}
```

在TiCDC中通过[config中的enableOldValue](https://github.com/pingcap/ticdc/blob/3fd3637a106639ba17273ec841d713f32ac5c741/cdc/kv/client.go#L775)来判断是否需要读Old Value，并构造RPC[请求](https://github.com/pingcap/ticdc/blob/3fd3637a106639ba17273ec841d713f32ac5c741/cdc/kv/client.go#L781)。

```go
extraOp := kvrpcpb.ExtraOp_Noop
if s.enableOldValue {
	extraOp = kvrpcpb.ExtraOp_ReadOldValue
}

req := &cdcpb.ChangeDataRequest{
			Header: &cdcpb.Header{
				ClusterId:    s.client.clusterID,
				TicdcVersion: version.ReleaseSemver(),
			},
			RegionId:     regionID,
			RequestId:    requestID,
			RegionEpoch:  rpcCtx.Meta.RegionEpoch,
			CheckpointTs: sri.ts,
			StartKey:     sri.span.Start,
			EndKey:       sri.span.End,
			ExtraOp:      extraOp,
		}
```
### TiCDC中如何区分一张表是否开启了new collation

需要在TiCDC中获取上游集群的mysql.tidb表中的new_collation_enable字段，通过它来真实反应集群是否开启了new collation。

### 为什么开了new collation 就必须要开 old value 才能正确同步

关于TiDB中表，Row，索引与TiKV对应的关系可以查看：[三篇文章了解 TiDB 技术内幕 - 说计算](https://pingcap.com/blog-cn/tidb-internal-2/)。简单来说就是因为如果一个表使用了new collation，那么它对应的index编排方式会发生改变，导致无法通过变更后的new value所在key反向解析到原数据是怎么样的，导致TiCDC无法还原对应的SQL，最终导致同步失败，所以如果一个表开启了new collation那么就需要开启old value来拉去TiKV数据。


## 新的需求
- 用户配置开启old value：
  - 以开启old value形式拉取TiKV变更数据
  - 向下游输出的是带有old value的数据
- 用户配置不开启old value：
  - 如果表使用了new collation以开启old value形式拉取TiKV变更数据，同时向下游输出前需要将old value删除
  - 如果表没有使用new collation则以不开启old value的形式拉取TiKV变更数据(防止性能降低)

## 测试

开启debug模式并修改log位置，[pkg/config/config.go第165行](https://github.com/pingcap/ticdc/blob/284dd55766f4f57b4c3550057c219bd1c92e5331/pkg/config/config.go#L165)

```go
var defaultServerConfig = &ServerConfig{
	Addr:          "127.0.0.1:8300",
	AdvertiseAddr: "",
	LogFile:       "",
	LogLevel:      "info",
	Log: &LogConfig{
		File: &LogFileConfig{
			MaxSize:    300,
			MaxDays:    0,
			MaxBackups: 0,
		},
```

修改是否同步old value，[pkg/config/config.go第46行](https://github.com/pingcap/ticdc/blob/284dd55766f4f57b4c3550057c219bd1c92e5331/pkg/config/config.go#L46)。

```go
var defaultReplicaConfig = &ReplicaConfig{
	CaseSensitive:    true,
	EnableOldValue:   true,
	CheckGCSafePoint: true,
	Filter: &FilterConfig{
		Rules: []string{"*.*"},
	},
	Mounter: &MounterConfig{
		WorkerNum: 16,
	},
	Sink: &SinkConfig{
		Protocol: "default",
	},
	Cyclic: &CyclicConfig{
		Enable: false,
	},
	Scheduler: &SchedulerConfig{
		Tp:          "table-number",
		PollingTime: -1,
	},
}
```

将数据写到blackhole，然后在日志中查找对应的log

```bash
grep "BlockHoleSink: EmitRowChangedEvents" ticdc.log
```
测试脚本

```mysql
DROP DATABASE IF EXISTS `tidb_test`;
CREATE DATABASE `tidb_test`;
USE `tidb_test`;
DROP TABLE IF EXISTS `employee`;
CREATE TABLE `employee` (
  `user_id` varchar(11) NOT NULL,
  `user_name` varchar(50) NOT NULL,
  `authority` int(11) DEFAULT '1' COMMENT '用户的权限',
  PRIMARY KEY (`user_id`),
  KEY `FK_authority` (`user_id`,`user_name`),
  CONSTRAINT `FK_authority` FOREIGN KEY (`user_id`, `user_name`) REFERENCES `user` (`user_id`, `user_name`) ON DELETE CASCADE ON UPDATE CASCADE
);

INSERT INTO `employee` VALUES ('1', 'szh', '0');
INSERT INTO `employee` VALUES ('2', 'sss', '0');
INSERT INTO `employee` VALUES ('3', 'ssk', '1');
INSERT INTO `employee` VALUES ('4', '2', '1');
INSERT INTO `employee` VALUES ('5', '3', '1');
INSERT INTO `employee` VALUES ('6', 'sds', '1');
INSERT INTO `employee` VALUES ('7', 'sd', '1');

DROP TABLE IF EXISTS `user`;
CREATE TABLE `user` (
  `user_id` varchar(11) NOT NULL AUTO_INCREMENT,
  `user_name` varchar(50) NOT NULL,
  `user_pass` varchar(50) NOT NULL,
  PRIMARY KEY (`user_id`),
  UNIQUE KEY `user_name` (`user_name`),
  KEY `user_id` (`user_id`,`user_name`)
);

INSERT INTO `user` VALUES ('1', 'szh', '123');
INSERT INTO `user` VALUES ('2', 'sss', '123');
INSERT INTO `user` VALUES ('3', 'ssk', '132');
INSERT INTO `user` VALUES ('4', '2', '3');
INSERT INTO `user` VALUES ('5', '3', '123');
INSERT INTO `user` VALUES ('6', 'sds', '123');
INSERT INTO `user` VALUES ('7', 'sd', '123');

DELETE FROM `employee` WHERE `user_id`='2';
UPDATE `employee` SET `user_name`="java" WHERE `user_id`='3';
```

关闭old value，关闭new collation

```
# insert
[2021/07/07 12:28:31.771 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426149702189973517,\"commit-ts\":426149702189973518,\"row-id\":5,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":57,\"is-partition\":false},\"table-info-version\":426149702157729810,\"replica-id\":0,\"columns\":[{\"name\":\"user_id\",\"type\":15,\"flag\":58,\"value\":\"NQ==\"},{\"name\":\"user_name\",\"type\":15,\"flag\":48,\"value\":\"Mw==\"},{\"name\":\"user_pass\",\"type\":15,\"flag\":0,\"value\":\"MTIz\"}],\"pre-columns\":null}"]

# delete
[2021/07/07 12:28:31.771 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426149702189973518,\"commit-ts\":426149702189973519,\"row-id\":6,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":57,\"is-partition\":false},\"table-info-version\":426149702157729810,\"replica-id\":0,\"columns\":[{\"name\":\"user_id\",\"type\":15,\"flag\":58,\"value\":\"Ng==\"},{\"name\":\"user_name\",\"type\":15,\"flag\":48,\"value\":\"c2Rz\"},{\"name\":\"user_pass\",\"type\":15,\"flag\":0,\"value\":\"MTIz\"}],\"pre-columns\":null}"]

# update
[2021/07/07 12:28:31.771 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426149702189973519,\"commit-ts\":426149702189973520,\"row-id\":7,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":57,\"is-partition\":false},\"table-info-version\":426149702157729810,\"replica-id\":0,\"columns\":[{\"name\":\"user_id\",\"type\":15,\"flag\":58,\"value\":\"Nw==\"},{\"name\":\"user_name\",\"type\":15,\"flag\":48,\"value\":\"c2Q=\"},{\"name\":\"user_pass\",\"type\":15,\"flag\":0,\"value\":\"MTIz\"}],\"pre-columns\":null}"]
```

关闭old value，开启new collation

```
# insert
[2021/07/07 12:24:25.421 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426149636137811981,\"commit-ts\":426149636137811982,\"row-id\":5,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":57,\"is-partition\":false},\"table-info-version\":426149636098228251,\"replica-id\":0,\"columns\":[{\"name\":\"user_id\",\"type\":15,\"flag\":58,\"value\":\"NQ==\"},{\"name\":\"user_name\",\"type\":15,\"flag\":48,\"value\":\"Mw==\"},{\"name\":\"user_pass\",\"type\":15,\"flag\":0,\"value\":\"MTIz\"}],\"pre-columns\":null}"]

# delete
[2021/07/07 12:24:25.421 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426149636137811982,\"commit-ts\":426149636137811983,\"row-id\":6,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":57,\"is-partition\":false},\"table-info-version\":426149636098228251,\"replica-id\":0,\"columns\":[{\"name\":\"user_id\",\"type\":15,\"flag\":58,\"value\":\"Ng==\"},{\"name\":\"user_name\",\"type\":15,\"flag\":48,\"value\":\"c2Rz\"},{\"name\":\"user_pass\",\"type\":15,\"flag\":0,\"value\":\"MTIz\"}],\"pre-columns\":null}"]

# update
[2021/07/07 12:24:25.421 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426149636137811983,\"commit-ts\":426149636137811984,\"row-id\":7,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":57,\"is-partition\":false},\"table-info-version\":426149636098228251,\"replica-id\":0,\"columns\":[{\"name\":\"user_id\",\"type\":15,\"flag\":58,\"value\":\"Nw==\"},{\"name\":\"user_name\",\"type\":15,\"flag\":48,\"value\":\"c2Q=\"},{\"name\":\"user_pass\",\"type\":15,\"flag\":0,\"value\":\"MTIz\"}],\"pre-columns\":null}"]
```

