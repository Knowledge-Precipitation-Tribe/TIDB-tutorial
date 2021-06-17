新建的capture变成processor开始运行，[代码](https://github.com/pingcap/ticdc/blob/43c40f9d368a5f85beaf05080c3f11135648faa5/cdc/capture/capture.go#L256)如下：

```go
func (c *Capture) runEtcdWorker(ctx cdcContext.Context, reactor orchestrator.Reactor, reactorState orchestrator.ReactorState, timerInterval time.Duration) error {
	etcdWorker, err := orchestrator.NewEtcdWorker(ctx.GlobalVars().EtcdClient.Client, kv.EtcdKeyBase, reactor, reactorState)
	if err != nil {
		return errors.Trace(err)
	}
	if err := etcdWorker.Run(ctx, c.session, timerInterval); err != nil {
		// We check ttl of lease instead of check `session.Done`, because
		// `session.Done` is only notified when etcd client establish a
		// new keepalive request, there could be a time window as long as
		// 1/3 of session ttl that `session.Done` can't be triggered even
		// the lease is already revoked.
		switch {
		case cerror.ErrEtcdSessionDone.Equal(err),
			cerror.ErrLeaseExpired.Equal(err):
			return cerror.ErrCaptureSuicide.GenWithStackByArgs()
		}
		lease, inErr := ctx.GlobalVars().EtcdClient.Client.TimeToLive(ctx, c.session.Lease())
		if inErr != nil {
			return cerror.WrapError(cerror.ErrPDEtcdAPIError, inErr)
		}
		if lease.TTL == int64(-1) {
			log.Warn("session is disconnected", zap.Error(err))
			return cerror.ErrCaptureSuicide.GenWithStackByArgs()
		}
		return errors.Trace(err)
	}
	return nil
}
```

