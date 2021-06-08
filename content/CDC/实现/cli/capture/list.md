## 命令`cdc cli capture list`实现

项目整体使用Cobra实现，通过子命令的方式实现链式调用，命令创建的代码位置在[client_capture.go](https://github.com/pingcap/ticdc/blob/master/cmd/client_capture.go)中，如下所示：

```go
func newListCaptureCommand() *cobra.Command {
	command := &cobra.Command{
		Use:   "list",
		Short: "List all captures in TiCDC cluster",
		RunE: func(cmd *cobra.Command, args []string) error {
			ctx := defaultContext
			captures, err := getAllCaptures(ctx)
			if err != nil {
				return err
			}
			return jsonPrint(cmd, captures)
		},
	}
	return command
}
```

其中list的主要逻辑是getAllCaptures函数，它位于[util.go](https://github.com/pingcap/ticdc/blob/master/cmd/util.go)中，函数实现如下：

```go
func getAllCaptures(ctx context.Context) ([]*capture, error) {
	_, raw, err := cdcEtcdCli.GetCaptures(ctx)
	if err != nil {
		return nil, err
	}
	ownerID, err := cdcEtcdCli.GetOwnerID(ctx, kv.CaptureOwnerKey)
	if err != nil && errors.Cause(err) != concurrency.ErrElectionNoLeader {
		return nil, err
	}
	captures := make([]*capture, 0, len(raw))
	for _, c := range raw {
		isOwner := c.ID == ownerID
		captures = append(captures,
			&capture{ID: c.ID, IsOwner: isOwner, AdvertiseAddr: c.AdvertiseAddr})
	}
	return captures, nil
}
```

可以看到这里面用到了cli中的cdcEtcdCli来获取一些数据信息，主要的函数有GetCaptures和GetOwnerID这两个。这两个函数都位于[cdc/kv/etcd.go](https://github.com/pingcap/ticdc/blob/master/cdc/kv/etcd.go)中，其中GetCaptures的实现如下：

```go
func (c CDCEtcdClient) GetCaptures(ctx context.Context)(int64[]*model.CaptureInfo, error) {
  // 预先定义的常量 CaptureInfoKeyPrefix is the capture info path that is saved to etcd
	// CaptureInfoKeyPrefix = EtcdKeyBase + "/capture"
	key := CaptureInfoKeyPrefix

	resp, err := c.Client.Get(ctx, key, clientv3.WithPrefix())
	if err != nil {
		return 0, nil, cerror.WrapError(cerror.ErrPDEtcdAPIError, err)
	}
	revision := resp.Header.Revision
	infos := make([]*model.CaptureInfo, 0, resp.Count)
	for _, kv := range resp.Kvs {
		info := &model.CaptureInfo{}
		err := info.Unmarshal(kv.Value)
		if err != nil {
			return 0, nil, errors.Trace(err)
		}
		infos = append(infos, info)
	}
	return revision, infos, nil
}
```

主要就是获取ETCD指定目录下的数据列表。GetOwnerID实现如下：

```go
func (c CDCEtcdClient) GetOwnerID(ctx context.Context, key string) (string, error) {
  // 传入的key为预先定的 CaptureOwnerKey is the capture owner path that is saved to etcd
	// CaptureOwnerKey = EtcdKeyBase + "/owner"
	resp, err := c.Client.Get(ctx, key, clientv3.WithFirstCreate()...)
	if err != nil {
		return "", cerror.WrapError(cerror.ErrPDEtcdAPIError, err)
	}
	if len(resp.Kvs) == 0 {
		return "", concurrency.ErrElectionNoLeader
	}
	return string(resp.Kvs[0].Value), nil
}
```

通过上面这两个函数就可以实现在ETCD中查找对应的capture list。

