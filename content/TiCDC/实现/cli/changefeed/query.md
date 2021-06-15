该方法用于查询指定changfeed的状态，代码位于[client_changefeed.go](https://github.com/pingcap/ticdc/blob/1c3653e292835b674fa47f0be7ac463ef64593fe/cmd/client_changefeed.go#L195)。

```go
    Use:   "query",
		Short: "Query information and status of a replicaiton task (changefeed)",
		RunE: func(cmd *cobra.Command, args []string) error {
			ctx := defaultContext

      // 如果该参数为true，则输出简化的任务状态
      // applyOwnerChangefeedQuery在list中有进行分析
			if simplified {
				resp, err := applyOwnerChangefeedQuery(ctx, changefeedID, getCredential())
				if err != nil {
					return err
				}
				cmd.Println(resp)
				return nil
			}
      // 获取changfeed的信息
			info, err := cdcEtcdCli.GetChangeFeedInfo(ctx, changefeedID)
			if err != nil && cerror.ErrChangeFeedNotExists.NotEqual(err) {
				return err
			}
      // 获取changefeed状态
			status, _, err := cdcEtcdCli.GetChangeFeedStatus(ctx, changefeedID)
			if err != nil && cerror.ErrChangeFeedNotExists.NotEqual(err) {
				return err
			}
			if err != nil && cerror.ErrChangeFeedNotExists.Equal(err) {
				log.Error("This changefeed does not exist", zap.String("changefeed", changefeedID))
				return err
			}
      // 因为每个changefeed会被分为多个task，所以需要获取当前hcangefeed所有task的位置
			taskPositions, err := cdcEtcdCli.GetAllTaskPositions(ctx, changefeedID)
			if err != nil && cerror.ErrChangeFeedNotExists.NotEqual(err) {
				return err
			}
			var count uint64
			for _, pinfo := range taskPositions {
				count += pinfo.Count
			}
      // 获取当前changefeed下所有task的状态
			processorInfos, err := cdcEtcdCli.GetAllTaskStatus(ctx, changefeedID)
			if err != nil {
				return err
			}
      // 将获取到的信息整合，最后封装为meta进行返回
			taskStatus := make([]captureTaskStatus, 0, len(processorInfos))
			for captureID, status := range processorInfos {
				taskStatus = append(taskStatus, captureTaskStatus{CaptureID: captureID, TaskStatus: status})
			}
			meta := &cfMeta{Info: info, Status: status, Count: count, TaskStatus: taskStatus}
			if info == nil {
				log.Warn("This changefeed has been deleted, the residual meta data will be completely deleted within 24 hours.", zap.String("changgefeed", changefeedID))
			}
			return jsonPrint(cmd, meta)
```

其中的[cdcEtcdCli.GetChangeFeedInfo](https://github.com/pingcap/ticdc/blob/1c3653e292835b674fa47f0be7ac463ef64593fe/cdc/kv/etcd.go#L200)代码如下：

```go
func (c CDCEtcdClient) GetChangeFeedInfo(ctx context.Context, id string) (*model.ChangeFeedInfo, error) {
  // EtcdKeyBase/changefeed/info/key
	key := GetEtcdKeyChangeFeedInfo(id)
  // 根据指定key获取信息
	resp, err := c.Client.Get(ctx, key)
	if err != nil {
		return nil, cerror.WrapError(cerror.ErrPDEtcdAPIError, err)
	}
	if resp.Count == 0 {
		return nil, cerror.ErrChangeFeedNotExists.GenWithStackByArgs(key)
	}
	detail := &model.ChangeFeedInfo{}
	err = detail.Unmarshal(resp.Kvs[0].Value)
	return detail, errors.Trace(err)
}
```

其中的[cdcEtcdCli.GetChangeFeedStatus](https://github.com/pingcap/ticdc/blob/1c3653e292835b674fa47f0be7ac463ef64593fe/cdc/kv/etcd.go#L245)代码如下：

```go
func (c CDCEtcdClient) GetChangeFeedStatus(ctx context.Context, id string) (*model.ChangeFeedStatus, int64, error) {
  // JobKeyPrefix + "/" + changeFeedID
	key := GetEtcdKeyJob(id)
  // 根据指定key获取信息
	resp, err := c.Client.Get(ctx, key)
	if err != nil {
		return nil, 0, cerror.WrapError(cerror.ErrPDEtcdAPIError, err)
	}
	if resp.Count == 0 {
		return nil, 0, cerror.ErrChangeFeedNotExists.GenWithStackByArgs(key)
	}
	info := &model.ChangeFeedStatus{}
	err = info.Unmarshal(resp.Kvs[0].Value)
	return info, resp.Kvs[0].ModRevision, errors.Trace(err)
}
```

其中的[cdcEtcdCli.GetAllTaskPositions](https://github.com/pingcap/ticdc/blob/1c3653e292835b674fa47f0be7ac463ef64593fe/cdc/kv/etcd.go#L341)代码如下：

```go
func (c CDCEtcdClient) GetAllTaskPositions(ctx context.Context, changefeedID string) (map[string]*model.TaskPosition, error) {
  // TaskPositionKeyPrefix = /tidb/cdc/task/position
  // 当前方法会返回所有task的position
  // TODO：需要看一下create中的逻辑是什么，为什么将changefeed作为前缀，然后返回所有position的做法
	resp, err := c.Client.Get(ctx, TaskPositionKeyPrefix, clientv3.WithPrefix())
	if err != nil {
		return nil, cerror.WrapError(cerror.ErrPDEtcdAPIError, err)
	}
	positions := make(map[string]*model.TaskPosition, resp.Count)
	for _, rawKv := range resp.Kvs {
		changeFeed, err := model.ExtractKeySuffix(string(rawKv.Key))
		if err != nil {
			return nil, err
		}
    // 获取changefeed前缀，如果当前ID与要查询的ID不同则continue
		endIndex := len(rawKv.Key) - len(changeFeed) - 1
		captureID, err := model.ExtractKeySuffix(string(rawKv.Key[0:endIndex]))
		if err != nil {
			return nil, err
		}
		if changeFeed != changefeedID {
			continue
		}
    // 将task信息添加到positions中
		info := &model.TaskPosition{}
		err = info.Unmarshal(rawKv.Value)
		if err != nil {
			return nil, cerror.ErrDecodeFailed.GenWithStackByArgs("failed to unmarshal task position: %s", err)
		}
		positions[captureID] = info
	}
	return positions, nil
}
```

其中的[cdcEtcdCli.GetAllTaskStatus](https://github.com/pingcap/ticdc/blob/1c3653e292835b674fa47f0be7ac463ef64593fe/cdc/kv/etcd.go#L427)代码如下(具体逻辑与上一个函数差不多)：

```go
func (c CDCEtcdClient) GetAllTaskStatus(ctx context.Context, changefeedID string) (model.ProcessorsInfos, error) {
  // TaskStatusKeyPrefix = /tidb/cdc/task/status
	resp, err := c.Client.Get(ctx, TaskStatusKeyPrefix, clientv3.WithPrefix())
	if err != nil {
		return nil, cerror.WrapError(cerror.ErrPDEtcdAPIError, err)
	}
	pinfo := make(map[string]*model.TaskStatus, resp.Count)
	for _, rawKv := range resp.Kvs {
		changeFeed, err := model.ExtractKeySuffix(string(rawKv.Key))
		if err != nil {
			return nil, err
		}
		endIndex := len(rawKv.Key) - len(changeFeed) - 1
		captureID, err := model.ExtractKeySuffix(string(rawKv.Key[0:endIndex]))
		if err != nil {
			return nil, err
		}
		if changeFeed != changefeedID {
			continue
		}
		info := &model.TaskStatus{}
		err = info.Unmarshal(rawKv.Value)
		if err != nil {
			return nil, cerror.ErrDecodeFailed.GenWithStackByArgs("failed to unmarshal task status: %s", err)
		}
		info.ModRevision = rawKv.ModRevision
		pinfo[captureID] = info
	}
	return pinfo, nil
}

```



