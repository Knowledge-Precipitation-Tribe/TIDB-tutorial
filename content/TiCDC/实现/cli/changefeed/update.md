代码位置[client_changefeed.go](https://github.com/pingcap/ticdc/blob/1c3653e292835b674fa47f0be7ac463ef64593fe/cmd/client_changefeed.go#L497)

```go
// 首先获取changfeed的信息
old, err := cdcEtcdCli.GetChangeFeedInfo(ctx, changefeedID)
if err != nil {
   return err
}
// 将新的配置与老的进行对比，判断是否有变化
changelog, err := diff.Diff(old, info)
			if err != nil {
				return err
			}
			if len(changelog) == 0 {
				cmd.Printf("changefeed config is the same with the old one, do nothing\n")
				return nil
			}
// 确认修改后会保存changfeed的信息
err = cdcEtcdCli.SaveChangeFeedInfo(ctx, info, changefeedID)
			if err != nil {
				return err
			}
```

该方法会更新一个changefeed的信息，其中的[cdcEtcdCli.SaveChangeFeedInfo](https://github.com/pingcap/ticdc/blob/633f93591e6a25503f2e25d65c26dad54aa36fb7/cdc/kv/etcd.go#L326)代码如下

```go
// SaveChangeFeedInfo stores change feed info into etcd
// TODO: this should be called from outer system, such as from a TiDB client
func (c CDCEtcdClient) SaveChangeFeedInfo(ctx context.Context, info *model.ChangeFeedInfo, changeFeedID string) error {
	// /tidb/cdc/changefeed/info/changfeedID
  key := GetEtcdKeyChangeFeedInfo(changeFeedID)
	value, err := info.Marshal()
	if err != nil {
		return errors.Trace(err)
	}
	_, err = c.Client.Put(ctx, key, value)
	return cerror.WrapError(cerror.ErrPDEtcdAPIError, err)
}
```

