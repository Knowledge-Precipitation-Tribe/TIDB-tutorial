代码位置[client_changefeed.go](https://github.com/pingcap/ticdc/blob/1c3653e292835b674fa47f0be7ac463ef64593fe/cmd/client_changefeed.go#L90)

```go
Use:   "pause",
			Short: "Pause a replicaiton task (changefeed)",
			RunE: func(cmd *cobra.Command, args []string) error {
				ctx := defaultContext
				job := model.AdminJob{
					CfID: changefeedID,
					Type: model.AdminStop,
				}
				return applyAdminChangefeed(ctx, job, getCredential())
			},
```

该方法会暂停一个同步的任务也就是一个changefeed，可以看到这里创建了一个job，其类型为AdminStop，[具体逻辑](https://github.com/pingcap/ticdc/blob/1c3653e292835b674fa47f0be7ac463ef64593fe/cmd/util.go#L143)如下

```go
func applyAdminChangefeed(ctx context.Context, job model.AdminJob, credential *security.Credential) error {
  // 获取cdc capture的Owner，因为Owner会负责整个任务的调度
	owner, err := getOwnerCapture(ctx)
	if err != nil {
		return err
	}
	scheme := "http"
	if credential.IsTLSEnabled() {
		scheme = "https"
	}
	addr := fmt.Sprintf("%s://%s/capture/owner/admin", scheme, owner.AdvertiseAddr)
	cli, err := httputil.NewClient(credential)
	if err != nil {
		return err
	}
	forceRemoveOpt := "false"
	if job.Opts != nil && job.Opts.ForceRemove {
		forceRemoveOpt = "true"
	}
  // 发送任务下线请求
	resp, err := cli.PostForm(addr, url.Values(map[string][]string{
		cdc.APIOpVarAdminJob:           {fmt.Sprint(int(job.Type))},
		cdc.APIOpVarChangefeedID:       {job.CfID},
		cdc.APIOpForceRemoveChangefeed: {forceRemoveOpt},
	}))
	if err != nil {
		return err
	}
	if resp.StatusCode < 200 || resp.StatusCode >= 300 {
		body, err := ioutil.ReadAll(resp.Body)
		if err != nil {
			return errors.BadRequestf("admin changefeed failed")
		}
		return errors.BadRequestf("%s", string(body))
	}
	return nil
}
```

