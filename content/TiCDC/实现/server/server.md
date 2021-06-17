代码位置[server.go](https://github.com/pingcap/ticdc/blob/master/cmd/server.go)

```go
func runEServer(cmd *cobra.Command, args []string) error {
  // 验证参数
	conf, err := loadAndVerifyServerConfig(cmd)
	if err != nil {
		return errors.Trace(err)
	}
  // initCmd initializes the logger, the default context and returns its cancel function.
	cancel := initCmd(cmd, &logutil.Config{
		File:  conf.LogFile,
		Level: conf.LogLevel,
	})
	defer cancel()
  // GetTimezone returns the timezone specified by the name
  // 默认的name是system，获取本地时间
	tz, err := util.GetTimezone(conf.TZ)
	if err != nil {
		return errors.Annotate(err, "can not load timezone, Please specify the time zone through environment variable `TZ` or command line parameters `--tz`")
	}
  // / StoreGlobalServerConfig stores a new config to the globalServerConfig. It mostly uses in the test to avoid some data races.
	config.StoreGlobalServerConfig(conf)
	// 将时间和capture地址写入context
  ctx := util.PutTimezoneInCtx(defaultContext, tz)
  // 默认地址：127.0.0.1:8300
	ctx = util.PutCaptureAddrInCtx(ctx, conf.AdvertiseAddr)

	version.LogVersionInfo()
	if util.FailpointBuild {
		for _, path := range failpoint.List() {
			status, err := failpoint.Status(path)
			if err != nil {
				log.Error("fail to get failpoint status", zap.Error(err))
			}
			log.Info("failpoint enabled", zap.String("path", path), zap.String("status", status))
		}
	}

	logHTTPProxies()
  // 根据config创建cdc的server
	server, err := cdc.NewServer(strings.Split(serverPdAddr, ","))
	if err != nil {
		return errors.Annotate(err, "new server")
	}
  // 运行cdc server
	err = server.Run(ctx)
	if err != nil && errors.Cause(err) != context.Canceled {
		log.Error("run server", zap.String("error", errors.ErrorStack(err)))
		return errors.Annotate(err, "run server")
	}
	server.Close()
	sorter.UnifiedSorterCleanUp()
	log.Info("cdc server exits successfully")

	return nil
}
```

其中server的结构题对应如下

```go
// Server is the capture server
type Server struct {
	captureV2 *capture.Capture

	capture      *Capture
	owner        *Owner
	ownerLock    sync.RWMutex
	statusServer *http.Server
	pdClient     pd.Client
	etcdClient   *kv.CDCEtcdClient
	kvStorage    tidbkv.Storage
	pdEndpoints  []string
}
```

其中的[server.Run](https://github.com/pingcap/ticdc/blob/43c40f9d368a5f85beaf05080c3f11135648faa5/cdc/server.go#L86)代码如下：

```go
func (s *Server) Run(ctx context.Context) error {
	conf := config.GetGlobalServerConfig()

	grpcTLSOption, err := conf.Security.ToGRPCDialOption()
	if err != nil {
		return errors.Trace(err)
	}
  // 与pd创建连接
	pdClient, err := pd.NewClientWithContext(
		ctx, s.pdEndpoints, conf.Security.PDSecurityOption(),
		pd.WithGRPCDialOptions(
			grpcTLSOption,
			grpc.WithBlock(),
			grpc.WithConnectParams(grpc.ConnectParams{
				Backoff: backoff.Config{
					BaseDelay:  time.Second,
					Multiplier: 1.1,
					Jitter:     0.1,
					MaxDelay:   3 * time.Second,
				},
				MinConnectTimeout: 3 * time.Second,
			}),
		))
	if err != nil {
		return cerror.WrapError(cerror.ErrServerNewPDClient, err)
	}
	s.pdClient = pdClient
  // 新版本的owner和processor
	if config.NewReplicaImpl {
		tlsConfig, err := conf.Security.ToTLSConfig()
		if err != nil {
			return errors.Trace(err)
		}
		logConfig := logutil.DefaultZapLoggerConfig
		logConfig.Level = zap.NewAtomicLevelAt(zapcore.ErrorLevel)
		etcdCli, err := clientv3.New(clientv3.Config{
      // 与pd的etcd创建连接
			Endpoints:   s.pdEndpoints,
			TLS:         tlsConfig,
			Context:     ctx,
			LogConfig:   &logConfig,
			DialTimeout: 5 * time.Second,
			DialOptions: []grpc.DialOption{
				grpcTLSOption,
				grpc.WithBlock(),
				grpc.WithConnectParams(grpc.ConnectParams{
					Backoff: backoff.Config{
						BaseDelay:  time.Second,
						Multiplier: 1.1,
						Jitter:     0.1,
						MaxDelay:   3 * time.Second,
					},
					MinConnectTimeout: 3 * time.Second,
				}),
			},
		})
		if err != nil {
			return errors.Annotate(cerror.WrapError(cerror.ErrNewCaptureFailed, err), "new etcd client")
		}
		etcdClient := kv.NewCDCEtcdClient(ctx, etcdCli)
		s.etcdClient = &etcdClient
	}
	// To not block CDC server startup, we need to warn instead of error
	// when TiKV is incompatible.
	errorTiKVIncompatible := false
	err = version.CheckClusterVersion(ctx, s.pdClient, s.pdEndpoints[0], conf.Security, errorTiKVIncompatible)
	if err != nil {
		return err
	}
	err = s.startStatusHTTP()
	if err != nil {
		return err
	}

	kv.InitWorkerPool()
	kvStore, err := kv.CreateTiStore(strings.Join(s.pdEndpoints, ","), conf.Security)
	if err != nil {
		return errors.Trace(err)
	}
	defer func() {
		err := kvStore.Close()
		if err != nil {
			log.Warn("kv store close failed", zap.Error(err))
		}
	}()
	s.kvStorage = kvStore
	ctx = util.PutKVStorageInCtx(ctx, kvStore)
	if config.NewReplicaImpl {
		s.captureV2 = capture.NewCapture(s.pdClient, s.kvStorage, s.etcdClient)
		return s.run(ctx)
	}
	// When a capture suicided, restart it
	for {
		if err := s.run(ctx); cerror.ErrCaptureSuicide.NotEqual(err) {
			return err
		}
		log.Info("server recovered", zap.String("capture-id", s.capture.info.ID))
	}
}
```

