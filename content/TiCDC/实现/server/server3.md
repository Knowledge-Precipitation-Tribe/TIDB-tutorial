```go
TODO LIST:
- 在竞选成功为owner后，开始runETCDworker，之后为什么要将capture的owner置nil
```

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
    // 需要注意这里会阻塞住
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
    // 竞选成功的话会将capture的owner重新设置
		log.Info("campaign owner successfully", zap.String("capture-id", c.info.ID))
		owner := c.newOwner()
		c.setOwner(owner)
    // 开始对ETCD中的任务进行监控
		err = c.runEtcdWorker(ctx, owner, model.NewGlobalState(), ownerFlushInterval)
    // TODO：这里为什么要置空
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

其中的[c.campaign](https://github.com/pingcap/ticdc/blob/43c40f9d368a5f85beaf05080c3f11135648faa5/cdc/capture/capture.go#L302)代码如下：

```go
func (c *Capture) campaign(ctx cdcContext.Context) error {
	failpoint.Inject("capture-campaign-compacted-error", func() {
		failpoint.Return(errors.Trace(mvcc.ErrCompacted))
	})
  // 通过ETCD中的选举策略进行选举
	return cerror.WrapError(cerror.ErrCaptureCampaignOwner, c.election.Campaign(ctx, c.info.ID))
}
```

其中的[c.runEtcdWorker](https://github.com/pingcap/ticdc/blob/43c40f9d368a5f85beaf05080c3f11135648faa5/cdc/capture/capture.go#L256)代码如下：

```go
func (c *Capture) runEtcdWorker(ctx cdcContext.Context, reactor orchestrator.Reactor, reactorState orchestrator.ReactorState, timerInterval time.Duration) error {
	// 创建worker，用于监控之后目录变化并做任务的安排
  // reactor为owner
  etcdWorker, err := orchestrator.NewEtcdWorker(ctx.GlobalVars().EtcdClient.Client, kv.EtcdKeyBase, reactor, reactorState)
	if err != nil {
		return errors.Trace(err)
	}
  // 开始执行具体的监控的逻辑
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

其中的[etcdWorker.Run](https://github.com/pingcap/ticdc/blob/43c40f9d368a5f85beaf05080c3f11135648faa5/pkg/orchestrator/etcd_worker.go#L74)代码如下

```go
func (worker *EtcdWorker) Run(ctx context.Context, session *concurrency.Session, timerInterval time.Duration) error {
	defer worker.cleanUp()
  // 同步raw状态
	err := worker.syncRawState(ctx)
	if err != nil {
		return errors.Trace(err)
	}

	ctx1, cancel := context.WithCancel(ctx)
	defer cancel()

	ticker := time.NewTicker(timerInterval)
	defer ticker.Stop()

  // 调用ETCD的watch功能
	watchCh := worker.client.Watch(ctx1, worker.prefix.String(), clientv3.WithPrefix(), clientv3.WithRev(worker.revision+1))
	var (
		pendingPatches []DataPatch
		exiting        bool
		sessionDone    <-chan struct{}
	)
	if session != nil {
		sessionDone = session.Done()
	} else {
		// should never be closed
		sessionDone = make(chan struct{})
	}
	lastReceivedEventTime := time.Now()

	for {
		var response clientv3.WatchResponse
		select {
		case <-ctx.Done():
			return ctx.Err()
		case <-sessionDone:
			return cerrors.ErrEtcdSessionDone.GenWithStackByArgs()
		case <-ticker.C:
			// There is no new event to handle on timer ticks, so we have nothing here.
			if time.Since(lastReceivedEventTime) > etcdRequestProgressDuration {
				if err := worker.client.RequestProgress(ctx); err != nil {
					log.Warn("failed to request progress for etcd watcher", zap.Error(err))
				}
			}
		case response = <-watchCh:
			// In this select case, we receive new events from Etcd, and call handleEvent if appropriate.
      // 收到事件变化

			if err := response.Err(); err != nil {
				return errors.Trace(err)
			}
			lastReceivedEventTime = time.Now()

			// Check whether the response is stale.
			if worker.revision >= response.Header.GetRevision() {
				continue
			}
			worker.revision = response.Header.GetRevision()

			// ProgressNotify implies no new events.
			if response.IsProgressNotify() {
				continue
			}

			for _, event := range response.Events {
				// handleEvent will apply the event to our internal `rawState`.
        // event分为两种：1.PUT 2.DELETE，将对应rawstate进行对应的逻辑
        // PUT就将对应的key放入或更新到rawstate中 worker.rawState[util.NewEtcdKeyFromBytes(event.Kv.Key)] = value
        // DELETE就将对应的key删除 delete(worker.rawState, util.NewEtcdKeyFromBytes(event.Kv.Key))
				err := worker.handleEvent(ctx, event)
				if err != nil {
					return errors.Trace(err)
				}
			}
		}

		if len(pendingPatches) > 0 {
			// Here we have some patches yet to be uploaded to Etcd.
			err := worker.applyPatches(ctx, pendingPatches, session)
			if err != nil {
				if cerrors.ErrEtcdTryAgain.Equal(errors.Cause(err)) {
					continue
				}
				return errors.Trace(err)
			}
			// If we are here, all patches have been successfully applied to Etcd.
			// `applyPatches` is all-or-none, so in case of success, we should clear all the pendingPatches.
			pendingPatches = pendingPatches[:0]
		} else {
			if exiting {
				// If exiting is true here, it means that the reactor returned `ErrReactorFinished` last tick, and all pending patches is applied.
				return nil
			}
			if worker.revision < worker.barrierRev {
				// We hold off notifying the Reactor because barrierRev has not been reached.
				// This usually happens when a committed write Txn has not been received by Watch.
				continue
			}

			// We are safe to update the ReactorState only if there is no pending patch.
			if err := worker.applyUpdates(); err != nil {
				return errors.Trace(err)
			}
			nextState, err := worker.reactor.Tick(ctx, worker.state)
			if err != nil {
				if !cerrors.ErrReactorFinished.Equal(errors.Cause(err)) {
					return errors.Trace(err)
				}
				// normal exit
				exiting = true
			}
			worker.state = nextState
			pendingPatches = append(pendingPatches, nextState.GetPatches()...)
		}
	}
}
```

其中的[worker.syncRawState]()代码如下：

```go
func (worker *EtcdWorker) syncRawState(ctx context.Context) error {
  // worker.prefix.String() = /tidb/cdc
  // 监控cdc下面的任务变化
	resp, err := worker.client.Get(ctx, worker.prefix.String(), clientv3.WithPrefix())
	if err != nil {
		return errors.Trace(err)
	}

	worker.rawState = make(map[util.EtcdKey][]byte)
	for _, kv := range resp.Kvs {
		key := util.NewEtcdKeyFromBytes(kv.Key)
		worker.rawState[key] = kv.Value
		err := worker.state.Update(key, kv.Value, true)
		if err != nil {
			return errors.Trace(err)
		}
	}

	worker.revision = resp.Header.Revision
	return nil
}
```

