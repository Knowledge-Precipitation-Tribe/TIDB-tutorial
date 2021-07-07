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

在[TableInfo](https://github.com/pingcap/ticdc/blob/284dd55766f4f57b4c3550057c219bd1c92e5331/cdc/model/schema_storage.go#L38)结构体中存储了下面这些信息：

```go
type TableInfo struct {
	*model.TableInfo
	SchemaID         int64
	TableName        TableName
	TableInfoVersion uint64
	columnsOffset    map[int64]int
	indicesOffset    map[int64]int
	uniqueColumns    map[int64]struct{}

	// It's a mapping from ColumnID to the offset of the columns in row changed events.
	RowColumnsOffset map[int64]int

	ColumnsFlag map[int64]ColumnFlagType

	// only for new row format decoder
	handleColID []int64

	// the mounter will choose this index to output delete events
	// special value:
	// HandleIndexPKIsHandle(-1) : pk is handle
	// HandleIndexTableIneligible(-2) : the table is not eligible
	HandleIndexID int64

	IndexColumnsOffset [][]int
	rowColInfos        []rowcodec.ColInfo
	rowColFieldTps     map[int64]*types.FieldType
}
```

其中TableInfo信息如下，根据其Collate字段就可以判断该表的排序规则。

```go
type TableInfo struct {
	ID      int64  `json:"id"`
	Name    CIStr  `json:"name"`
	Charset string `json:"charset"`
	Collate string `json:"collate"`
	// Columns are listed in the order in which they appear in the schema.
	Columns     []*ColumnInfo     `json:"cols"`
	Indices     []*IndexInfo      `json:"index_info"`
	Constraints []*ConstraintInfo `json:"constraint_info"`
	ForeignKeys []*FKInfo         `json:"fk_info"`
	State       SchemaState       `json:"state"`
	// PKIsHandle is true when primary key is a single integer column.
	PKIsHandle bool `json:"pk_is_handle"`
	// IsCommonHandle is true when clustered index feature is
	// enabled and the primary key is not a single integer column.
	IsCommonHandle bool `json:"is_common_handle"`
	// CommonHandleVersion is the version of the clustered index.
	// 0 for the clustered index created == 5.0.0 RC.
	// 1 for the clustered index created > 5.0.0 RC.
	CommonHandleVersion uint16 `json:"common_handle_version"`

	Comment         string `json:"comment"`
	AutoIncID       int64  `json:"auto_inc_id"`
	AutoIdCache     int64  `json:"auto_id_cache"`
	AutoRandID      int64  `json:"auto_rand_id"`
	MaxColumnID     int64  `json:"max_col_id"`
	MaxIndexID      int64  `json:"max_idx_id"`
	MaxConstraintID int64  `json:"max_cst_id"`
	// UpdateTS is used to record the timestamp of updating the table's schema information.
	// These changing schema operations don't include 'truncate table' and 'rename table'.
	UpdateTS uint64 `json:"update_timestamp"`
	// OldSchemaID :
	// Because auto increment ID has schemaID as prefix,
	// We need to save original schemaID to keep autoID unchanged
	// while renaming a table from one database to another.
	// TODO: Remove it.
	// Now it only uses for compatibility with the old version that already uses this field.
	OldSchemaID int64 `json:"old_schema_id,omitempty"`

	// ShardRowIDBits specify if the implicit row ID is sharded.
	ShardRowIDBits uint64
	// MaxShardRowIDBits uses to record the max ShardRowIDBits be used so far.
	MaxShardRowIDBits uint64 `json:"max_shard_row_id_bits"`
	// AutoRandomBits is used to set the bit number to shard automatically when PKIsHandle.
	AutoRandomBits uint64 `json:"auto_random_bits"`
	// PreSplitRegions specify the pre-split region when create table.
	// The pre-split region num is 2^(PreSplitRegions-1).
	// And the PreSplitRegions should less than or equal to ShardRowIDBits.
	PreSplitRegions uint64 `json:"pre_split_regions"`

	Partition *PartitionInfo `json:"partition"`

	Compression string `json:"compression"`

	View *ViewInfo `json:"view"`

	Sequence *SequenceInfo `json:"sequence"`

	// Lock represent the table lock info.
	Lock *TableLockInfo `json:"Lock"`

	// Version means the version of the table info.
	Version uint16 `json:"version"`

	// TiFlashReplica means the TiFlash replica info.
	TiFlashReplica *TiFlashReplicaInfo `json:"tiflash_replica"`

	// IsColumnar means the table is column-oriented.
	// It's true when the engine of the table is TiFlash only.
	IsColumnar bool `json:"is_columnar"`

	TempTableType `json:"temp_table_type"`
}
```

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
CREATE DATABASE tidb_test;
USE tidb_test;
CREATE TABLE IF NOT EXISTS `user`(
   `id` INT UNSIGNED,
   `name` VARCHAR(100) NOT NULL,
   PRIMARY KEY ( `id` )
);

INSERT INTO user VALUES(1, "hello");
INSERT INTO user VALUES(2, "world");
DELETE FROM user WHERE id=2;
INSERT INTO user VALUES(3, "go");
UPDATE user SET name="java" WHERE id=3;
```

开启Old Value

```log
# insert操作
[2021/06/30 17:23:02.242 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":425995791581577217,\"commit-ts\":425995791581577218,\"row-id\":1,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":425995789930070019,\"replica-id\":0,\"columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":1},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"aGVsbG8=\"}],\"pre-columns\":null}"]

# delete操作
[2021/06/30 17:23:14.499 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":425995794308136962,\"commit-ts\":425995794308136963,\"row-id\":2,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":425995789930070019,\"replica-id\":0,\"columns\":null,\"pre-columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":2},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"d29ybGQ=\"}]}"]

# update操作
[2021/06/30 17:23:24.301 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":425995797204828161,\"commit-ts\":425995797204828162,\"row-id\":3,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":425995789930070019,\"replica-id\":0,\"columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":3},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"amF2YQ==\"}],\"pre-columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":3},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"Z28=\"}]}"]
```
不开启Old Value

```log
# insert操作
[2021/06/30 16:51:32.942 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":425995296216449025,\"commit-ts\":425995296216449026,\"row-id\":1,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":425995291864072195,\"replica-id\":0,\"columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":1},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"aGVsbG8=\"}],\"pre-columns\":null}"]

# delete操作
[2021/06/30 16:52:25.846 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":425995310083604481,\"commit-ts\":425995310083604482,\"row-id\":2,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":425995291864072195,\"replica-id\":0,\"columns\":null,\"pre-columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":2},null]}"]

# update操作
[2021/06/30 17:16:43.675 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":425995691829755906,\"commit-ts\":425995691829755907,\"row-id\":3,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":425995291864072195,\"replica-id\":0,\"columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":3},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"amF2YQ==\"}],\"pre-columns\":null}"]
```

new collation ,puller true, old value false

```
[2021/07/06 19:16:32.949 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426133472354500609,\"commit-ts\":426133472354500610,\"row-id\":1,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":426133470833541131,\"replica-id\":0,\"columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":1},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"aGVsbG8=\"}],\"pre-columns\":null}"]
[2021/07/06 19:16:38.672 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426133473875197953,\"commit-ts\":426133473875197954,\"row-id\":2,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":426133470833541131,\"replica-id\":0,\"columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":2},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"d29ybGQ=\"}],\"pre-columns\":null}"]
[2021/07/06 19:16:43.675 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426133475250929665,\"commit-ts\":426133475250929666,\"row-id\":2,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":426133470833541131,\"replica-id\":0,\"columns\":null,\"pre-columns\":null}"]
[2021/07/06 19:16:48.516 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426133476496113665,\"commit-ts\":426133476496113666,\"row-id\":3,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":426133470833541131,\"replica-id\":0,\"columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":3},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"Z28=\"}],\"pre-columns\":null}"]
[2021/07/06 19:16:53.673 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426133477845893121,\"commit-ts\":426133477845893122,\"row-id\":3,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":426133470833541131,\"replica-id\":0,\"columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":3},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"amF2YQ==\"}],\"pre-columns\":null}"]
```

new collation ,puller false, old value true

```
[2021/07/06 19:25:17.840 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426133610113007617,\"commit-ts\":426133610113007618,\"row-id\":1,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":426133608802025484,\"replica-id\":0,\"columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":1},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"aGVsbG8=\"}],\"pre-columns\":null}"]
[2021/07/06 19:25:24.240 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426133611371560961,\"commit-ts\":426133611371560962,\"row-id\":2,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":426133608802025484,\"replica-id\":0,\"columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":2},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"d29ybGQ=\"}],\"pre-columns\":null}"]
[2021/07/06 19:25:27.841 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426133612629590017,\"commit-ts\":426133612629590018,\"row-id\":2,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":426133608802025484,\"replica-id\":0,\"columns\":null,\"pre-columns\":null}"]
[2021/07/06 19:25:32.015 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426133613809238017,\"commit-ts\":426133613809238018,\"row-id\":3,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":426133608802025484,\"replica-id\":0,\"columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":3},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"Z28=\"}],\"pre-columns\":null}"]
[2021/07/06 19:25:36.941 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426133615015100417,\"commit-ts\":426133615015100418,\"row-id\":3,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":426133608802025484,\"replica-id\":0,\"columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":3},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"amF2YQ==\"}],\"pre-columns\":null}"]

```

new collation ,puller true, old value true

```
[2021/07/06 19:31:41.689 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426133710552432641,\"commit-ts\":426133710552432642,\"row-id\":1,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":426133709529808907,\"replica-id\":0,\"columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":1},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"aGVsbG8=\"}],\"pre-columns\":null}"]
[2021/07/06 19:31:47.916 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426133711784771585,\"commit-ts\":426133711784771586,\"row-id\":2,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":426133709529808907,\"replica-id\":0,\"columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":2},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"d29ybGQ=\"}],\"pre-columns\":null}"]
[2021/07/06 19:31:52.487 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426133713357897729,\"commit-ts\":426133713357897730,\"row-id\":2,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":426133709529808907,\"replica-id\":0,\"columns\":null,\"pre-columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":2},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"d29ybGQ=\"}]}"]
[2021/07/06 19:31:56.488 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426133714576080897,\"commit-ts\":426133714576080898,\"row-id\":3,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":426133709529808907,\"replica-id\":0,\"columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":3},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"Z28=\"}],\"pre-columns\":null}"]
[2021/07/06 19:32:02.887 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426133715860324353,\"commit-ts\":426133715860324354,\"row-id\":3,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":426133709529808907,\"replica-id\":0,\"columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":3},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"amF2YQ==\"}],\"pre-columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":3},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"Z28=\"}]}"]
```

开启oldvalue, new collation关闭

```
[2021/07/07 10:48:43.585 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426148134405537793,\"commit-ts\":426148134405537794,\"row-id\":1,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":426148133146722315,\"replica-id\":0,\"columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":1},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"aGVsbG8=\"}],\"pre-columns\":null}"]
[2021/07/07 10:48:49.586 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426148135794900993,\"commit-ts\":426148135794900994,\"row-id\":2,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":426148133146722315,\"replica-id\":0,\"columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":2},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"d29ybGQ=\"}],\"pre-columns\":null}"]
[2021/07/07 10:48:54.588 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426148137091989505,\"commit-ts\":426148137091989506,\"row-id\":2,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":426148133146722315,\"replica-id\":0,\"columns\":null,\"pre-columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":2},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"d29ybGQ=\"}]}"]
[2021/07/07 10:49:01.884 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426148138796187649,\"commit-ts\":426148138796187650,\"row-id\":3,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":426148133146722315,\"replica-id\":0,\"columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":3},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"Z28=\"}],\"pre-columns\":null}"]
[2021/07/07 10:49:06.620 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426148140264194049,\"commit-ts\":426148140264194050,\"row-id\":3,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":426148133146722315,\"replica-id\":0,\"columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":3},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"amF2YQ==\"}],\"pre-columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":3},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"Z28=\"}]}"]
```

关闭old value，new collation关闭

```
# insert操作
[2021/07/07 10:53:40.457 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426148211942752257,\"commit-ts\":426148211942752258,\"row-id\":1,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":426148210553126915,\"replica-id\":0,\"columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":1},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"aGVsbG8=\"}],\"pre-columns\":null}"]

# insert操作
[2021/07/07 10:53:46.315 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426148213541568513,\"commit-ts\":426148213541568515,\"row-id\":2,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":426148210553126915,\"replica-id\":0,\"columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":2},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"d29ybGQ=\"}],\"pre-columns\":null}"]

# delete操作
[2021/07/07 10:53:51.385 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426148214826074115,\"commit-ts\":426148214826074116,\"row-id\":2,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":426148210553126915,\"replica-id\":0,\"columns\":null,\"pre-columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":2},null]}"]

# insert操作
[2021/07/07 10:53:55.386 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426148216045043713,\"commit-ts\":426148216045043714,\"row-id\":3,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":426148210553126915,\"replica-id\":0,\"columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":3},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"Z28=\"}],\"pre-columns\":null}"]

# update操作
[2021/07/07 10:54:00.388 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426148217329811457,\"commit-ts\":426148217329811458,\"row-id\":3,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":426148210553126915,\"replica-id\":0,\"columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":3},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"amF2YQ==\"}],\"pre-columns\":null}"]
```

关闭old value，new collation开启

```
# insert操作
[2021/07/07 10:58:34.436 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426148289032486915,\"commit-ts\":426148289032486916,\"row-id\":1,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":426148287564480523,\"replica-id\":0,\"columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":1},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"aGVsbG8=\"}],\"pre-columns\":null}"]

# insert操作
[2021/07/07 10:58:39.637 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426148290383314945,\"commit-ts\":426148290383314946,\"row-id\":2,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":426148287564480523,\"replica-id\":0,\"columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":2},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"d29ybGQ=\"}],\"pre-columns\":null}"]

# delete操作
[2021/07/07 10:58:44.488 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426148291680403457,\"commit-ts\":426148291680403458,\"row-id\":2,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":426148287564480523,\"replica-id\":0,\"columns\":null,\"pre-columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":2},null]}"]

# insert操作
[2021/07/07 10:58:48.418 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426148292964646913,\"commit-ts\":426148292964646914,\"row-id\":3,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":426148287564480523,\"replica-id\":0,\"columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":3},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"Z28=\"}],\"pre-columns\":null}"]

# update操作
[2021/07/07 10:58:54.636 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426148294315212801,\"commit-ts\":426148294315212802,\"row-id\":3,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":426148287564480523,\"replica-id\":0,\"columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":3},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"amF2YQ==\"}],\"pre-columns\":null}"]
```

开启old value，new collation开启

```
[2021/07/07 11:01:56.480 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426148341813346305,\"commit-ts\":426148341813346306,\"row-id\":1,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":426148340450197514,\"replica-id\":0,\"columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":1},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"aGVsbG8=\"}],\"pre-columns\":null}"]
[2021/07/07 11:02:00.708 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426148343255138305,\"commit-ts\":426148343255138306,\"row-id\":2,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":426148340450197514,\"replica-id\":0,\"columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":2},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"d29ybGQ=\"}],\"pre-columns\":null}"]
[2021/07/07 11:02:05.965 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426148344580276225,\"commit-ts\":426148344580276226,\"row-id\":2,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":426148340450197514,\"replica-id\":0,\"columns\":null,\"pre-columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":2},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"d29ybGQ=\"}]}"]
[2021/07/07 11:02:14.459 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426148346453557249,\"commit-ts\":426148346453557250,\"row-id\":3,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":426148340450197514,\"replica-id\":0,\"columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":3},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"Z28=\"}],\"pre-columns\":null}"]
[2021/07/07 11:02:18.059 +08:00] [DEBUG] [black_hole.go:41] ["BlockHoleSink: EmitRowChangedEvents"] [row="{\"start-ts\":426148347804123138,\"commit-ts\":426148347804123139,\"row-id\":3,\"table\":{\"db-name\":\"tidb_test\",\"tbl-name\":\"user\",\"tbl-id\":55,\"is-partition\":false},\"table-info-version\":426148340450197514,\"replica-id\":0,\"columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":3},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"amF2YQ==\"}],\"pre-columns\":[{\"name\":\"id\",\"type\":3,\"flag\":139,\"value\":3},{\"name\":\"name\",\"type\":15,\"flag\":0,\"value\":\"Z28=\"}]}"]
```

