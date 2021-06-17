新建的capture尝试选举成为owner[代码](https://github.com/pingcap/ticdc/blob/43c40f9d368a5f85beaf05080c3f11135648faa5/cdc/capture/capture.go#L200)：

```go
func (c *Capture) campaignOwner(ctx cdcContext.Context) error {
	// In most failure cases, we don't return error directly, just run another
	// campaign loop. We treat campaign loop as a special background routine.
	conf := config.GetGlobalServerConfig()
	ownerFlushInterval := time.Duration(conf.OwnerFlushInterval)
	failpoint.Inject("ownerFlushIntervalInject", func(val failpoint.Value) {
		ownerFlushInterval = time.Millisecond * time.Duration(val.(int))
	})
	// Limit the frequency of elections to avoid putting too much pressure on the etcd server
	rl := rate.NewLimiter(0.05, 2)
	for {
		select {
		case <-ctx.Done():
			return nil
		default:
		}
		err := rl.Wait(ctx)
		if err != nil {
			if errors.Cause(err) == context.Canceled {
				return nil
			}
			return errors.Trace(err)
		}
		// Campaign to be an owner, it blocks until it becomes the owner
		if err := c.campaign(ctx); err != nil {
			switch errors.Cause(err) {
			case context.Canceled:
				return nil
			case mvcc.ErrCompacted:
				// the revision we requested is compacted, just retry
				continue
			}
			log.Warn("campaign owner failed", zap.Error(err))
			// if campaign owner failed, restart capture
			return cerror.ErrCaptureSuicide.GenWithStackByArgs()
		}

		log.Info("campaign owner successfully", zap.String("capture-id", c.info.ID))
		owner := c.newOwner()
		c.setOwner(owner)
		err = c.runEtcdWorker(ctx, owner, model.NewGlobalState(), ownerFlushInterval)
		c.setOwner(nil)
		log.Info("run owner exited", zap.Error(err))
		// if owner exits, resign the owner key
		if resignErr := c.resign(ctx); resignErr != nil {
			// if resigning owner failed, return error to let capture exits
			return errors.Annotatef(resignErr, "resign owner failed, capture: %s", c.info.ID)
		}
		if err != nil {
			// for errors, return error and let capture exits or restart
			return errors.Trace(err)
		}
		// if owner exits normally, continue the campaign loop and try to election owner again
	}
}
```

