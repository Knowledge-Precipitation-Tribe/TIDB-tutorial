代码位置[client_changefeed.go](https://github.com/pingcap/ticdc/blob/633f93591e6a25503f2e25d65c26dad54aa36fb7/cmd/client_changefeed.go#L630)

```go
Use:   "statistics",
		Short: "Periodically check and output the status of a replicaiton task (changefeed)",
		RunE: func(cmd *cobra.Command, args []string) error {
			ctx := defaultContext
      // 根据传递的参数指定每interval间隔进行查看
			tick := time.NewTicker(time.Duration(interval) * time.Second)
			lastTime := time.Now()
			var lastCount uint64
			for {
				select {
				case <-ctx.Done():
					if err := ctx.Err(); err != nil {
						return err
					}
        // 定时任务
				case <-tick.C:
					now := time.Now()
					status, _, err := cdcEtcdCli.GetChangeFeedStatus(ctx, changefeedID)
					if err != nil {
						return err
					}
					taskPositions, err := cdcEtcdCli.GetAllTaskPositions(ctx, changefeedID)
					if err != nil {
						return err
					}
					var count uint64
					for _, pinfo := range taskPositions {
						count += pinfo.Count
					}
          // 获取当前TS
					ts, _, err := pdCli.GetTS(ctx)
					if err != nil {
						return err
					}
          // 计算当前已经被cdc获取的数据和cdc已经同步到下哟数据的gap值，相当于啊lag
					sinkGap := oracle.ExtractPhysical(status.ResolvedTs) - oracle.ExtractPhysical(status.CheckpointTs)
					// 计算当前ts与已经同步到下游的间隔
          replicationGap := ts - oracle.ExtractPhysical(status.CheckpointTs)
					statistics := profileStatus{
						OPS:            (count - lastCount) / uint64(now.Unix()-lastTime.Unix()),
						SinkGap:        fmt.Sprintf("%dms", sinkGap),
						ReplicationGap: fmt.Sprintf("%dms", replicationGap),
						Count:          count,
					}
					jsonPrint(cmd, &statistics)
					lastCount = count
					lastTime = now
				}
			}
		},
```

该函数会定期检查并输出复制任务状态，其中的GetChangeFeedStatus和GetAllTaskPositions在其他的章节中进行过分析。

