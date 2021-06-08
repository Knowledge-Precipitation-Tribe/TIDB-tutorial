## 命令`cli`实现

项目整体使用Cobra实现，代码在[client.go](https://github.com/pingcap/ticdc/blob/master/cmd/client.go)中，其中会先连接pd集群，如果是https还会验证对应的TLS，否则使用http进行连接。接下来会初始化cdcEtcdCli和pdCli两个客户端，方便之后的其他包进行调用。主要代码如下：

```go
etcdCli, err := clientv3.New(clientv3.Config{
				Context:     defaultContext,
				Endpoints:   pdEndpoints,
				TLS:         tlsConfig,
				LogConfig:   &logConfig,
				DialTimeout: 30 * time.Second,
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
	// PD embeds an etcd server.
  return errors.Annotatef(err, "fail to open PD etcd client, pd-addr=\"%s\"", cliPdAddr)
}
cdcEtcdCli = kv.NewCDCEtcdClient(defaultContext, etcdCli)
pdCli, err = pd.NewClientWithContext(
				defaultContext, pdEndpoints, credential.PDSecurityOption(),
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
```

