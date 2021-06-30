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
将数据写到下游，然后打印输出。（具体操作形式需要探索一下）