代码位置[client_changefeed.go](https://github.com/pingcap/ticdc/blob/1c3653e292835b674fa47f0be7ac463ef64593fe/cmd/client_changefeed.go#L90)

```go
Use:   "resume",
			Short: "Resume a paused replicaiton task (changefeed)",
			RunE: func(cmd *cobra.Command, args []string) error {
				ctx := defaultContext
				job := model.AdminJob{
					CfID: changefeedID,
					Type: model.AdminResume,
				}
				if err := resumeChangefeedCheck(ctx, cmd); err != nil {
					return err
				}
				return applyAdminChangefeed(ctx, job, getCredential())
			},
```

该方法会恢复一个同步的任务也就是一个changefeed，可以看到这里创建了一个job，其类型为AdminResume，然后调用的方法仍为[applyAdminChangefeed](https://github.com/pingcap/ticdc/blob/1c3653e292835b674fa47f0be7ac463ef64593fe/cmd/util.go#L143)。

