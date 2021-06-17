用于分析[server run](https://github.com/pingcap/ticdc/blob/43c40f9d368a5f85beaf05080c3f11135648faa5/cdc/server.go#L294)函数

```go
func (s *Server) run(ctx context.Context) (err error) {
	// 老版本的owner和processor进行下面的操作
  if !config.NewReplicaImpl {
    // 将之前创建TiStore取出
		kvStorage, err := util.KVStorageFromCtx(ctx)
		if err != nil {
			return errors.Trace(err)
		}
    // 创建capture
		capture, err := NewCapture(ctx, s.pdEndpoints, s.pdClient, kvStorage)
		if err != nil {
			return err
		}
		s.capture = capture
		s.etcdClient = &capture.etcdClient
	}
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()

	wg, cctx := errgroup.WithContext(ctx)
  // 新版本执行以下操作
	if config.NewReplicaImpl {
		wg.Go(func() error {
			return s.captureV2.Run(cctx)
		})
	} else {
		wg.Go(func() error {
			return s.campaignOwnerLoop(cctx)
		})

		wg.Go(func() error {
			return s.capture.Run(cctx)
		})
	}
  // 检查etcd健康
	wg.Go(func() error {
		return s.etcdHealthChecker(cctx)
	})

  // RunWorkerPool runs the worker pool used by the heapSorters
  // It **must** be running for Unified Sorter to work.
	wg.Go(func() error {
		return sorter.RunWorkerPool(cctx)
	})

  // RunWorkerPool runs the worker pool used by the region worker in kv client v2
  // It must be running before region worker starts to work
	wg.Go(func() error {
		return kv.RunWorkerPool(cctx)
	})

	return wg.Wait()
}
```

其中的[captureV2.Run](https://github.com/pingcap/ticdc/blob/43c40f9d368a5f85beaf05080c3f11135648faa5/cdc/capture/capture.go#L105)代码如下：

```go
func (c *Capture) Run(ctx context.Context) error {
	defer log.Info("the capture routine has exited")
	// Limit the frequency of reset capture to avoid frequent recreating of resources
  // 添加限流器
	rl := rate.NewLimiter(0.05, 2)
	for {
		select {
		case <-ctx.Done():
			return nil
		default:
		}
		ctx, cancel := context.WithCancel(ctx)
		c.cancel = cancel
		err := rl.Wait(ctx)
		if err != nil {
			if errors.Cause(err) == context.Canceled {
				return nil
			}
			return errors.Trace(err)
		}
		err = c.reset()
		if err != nil {
			return errors.Trace(err)
		}
		err = c.run(ctx)
		// if capture suicided, reset the capture and run again.
		// if the canceled error throw, there are two possible scenarios:
		//   1. the internal context canceled, it means some error happened in the internal, and the routine is exited, we should restart the capture
		//   2. the parent context canceled, it means that the caller of the capture hope the capture to exit, and this loop will return in the above `select` block
		// TODO: make sure the internal cancel should return the real error instead of context.Canceled
		if cerror.ErrCaptureSuicide.Equal(err) || context.Canceled == errors.Cause(err) {
			log.Info("capture recovered", zap.String("capture-id", c.info.ID))
			continue
		}
		return errors.Trace(err)
	}
}
```

其中的[capture run](https://github.com/pingcap/ticdc/blob/43c40f9d368a5f85beaf05080c3f11135648faa5/cdc/capture/capture.go#L142)代码如下：

```go
func (c *Capture) run(stdCtx context.Context) error {
   ctx := cdcContext.NewContext(stdCtx, &cdcContext.GlobalVars{
      PDClient:    c.pdClient,
      KVStorage:   c.kvStorage,
      CaptureInfo: c.info,
      EtcdClient:  c.etcdClient,
   })
   // 在etcd中注册自己的信息
   err := c.register(ctx)
   if err != nil {
      return errors.Trace(err)
   }
   defer func() {
      timeoutCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
      if err := ctx.GlobalVars().EtcdClient.DeleteCaptureInfo(timeoutCtx, c.info.ID); err != nil {
         log.Warn("failed to delete capture info when capture exited", zap.Error(err))
      }
      cancel()
   }()
   wg := new(sync.WaitGroup)
   wg.Add(2)
   var ownerErr, processorErr error
   go func() {
      defer wg.Done()
      defer c.AsyncClose()
      // when the campaignOwner returns an error, it means that the the owner throws an unrecoverable serious errors
      // (recoverable errors are intercepted in the owner tick)
      // so we should also stop the processor and let capture restart or exit
      // 尝试选举成为owner
      ownerErr = c.campaignOwner(ctx)
      log.Info("the owner routine has exited", zap.Error(ownerErr))
   }()
   go func() {
      defer wg.Done()
      defer c.AsyncClose()
      conf := config.GetGlobalServerConfig()
      processorFlushInterval := time.Duration(conf.ProcessorFlushInterval)
      // when the etcd worker of processor returns an error, it means that the the processor throws an unrecoverable serious errors
      // (recoverable errors are intercepted in the processor tick)
      // so we should also stop the owner and let capture restart or exit
      // 开始执行processor的工作
      processorErr = c.runEtcdWorker(ctx, c.processorManager, model.NewGlobalState(), processorFlushInterval)
      log.Info("the processor routine has exited", zap.Error(processorErr))
   }()
   wg.Wait()
   if ownerErr != nil {
      return errors.Annotate(ownerErr, "owner exited with error")
   }
   if processorErr != nil {
      return errors.Annotate(processorErr, "processor exited with error")
   }
   return nil
}
```