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

疑问：

- TiCDC中如何区分一张表是否开启了new collation
- 为什么开了new collation 就必须要开 old value 才能正确同步
- 希望的情况：
  - 无论new collation是否开启都以开启old value形式拉取TiKV变更数据
  - 用户配置开启old value：
    - 向下游输出的是带有old value的数据
  - 用户配置不开启old value：
    - 向下游输出的是不带有old value的数据
- 如何测试当前向下游输出的数据是否带有old value
  - sink-uri:TiDB,然后检查是否带有old vaue(如何操作)