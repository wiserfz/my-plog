+++
title = "etcd auth"
date = "2025-02-06"
author = ["wiser"]
description = "Enable etcd authentication."

[taxonomies]
tags = ["etcd", "code"]
categories = ["go"]
+++

# Overview

由于安全要求，etcd 需要开启 authentication，因此；记录以下如何在不停服务的情况下开启 etcd authentication。

由于 etcd 3.5 更改了 auth 相关的 WAL entries，因此；如果在 etcd 3.4 下开启 authentication 则后续无法升级到 etcd 3.5。见 [etcd upgrade 3.5](https://etcd.io/docs/v3.5/upgrades/upgrade_3_5/#upgrade-checklists)，因此；需要先把 etcd 升级到 3.5 之后在开启 authentication。

## Operations

### Step 1

创建 root 用户以及 root role

```bash
# there is one special user, root, and one special role, root
etcdctl user add root:password
etcdctl role add root

etcdctl user grant-role root root
```

### Step 2

根据目前所有的 key 的使用情况，创建 roles。这些 roles 用来给不同的 user 授权。有的应用需要  readwrite，有的是 read，有的是 write。

```bash
# add other roles by root user if authentication is enabled need pass <--user root:password> flag
etcdctl role add role1
# and grant permission for other roles
etcdctl role grant-permission --prefix=true role1 read /xxx/
```

### Step 3

给每个应用创建并分发 user:password，并更新到应用进程中（动态更新或重启）。

```bash
# create other user if authentication is enabled need pass --user flag
etcdctl user add user1:password

# grant role for user
etcdctl user grant-role user1 role1
```

### Step 4

所有应用都已经使用 user:password 并准备好后，etcd 集群启用 authentication。

```bash
etcdctl auth enable
```

## Analyzation

通常 etcd client 实现会实现配置传递 endpoints，username，password 等配置。在启用 Authenticatioin 的情况下，会成功返回 AuthenticateResponse 并带上可用的 token。在 client 中会使用这个 token 另外建立一个 TCP 连接，来请求 etcd server，见[etcd v3 authentication design](https://etcd.io/docs/v3.5/learning/design-auth-v3/#authentication)。

### Go etcd client

在 [etcd](https://github.com/etcd-io/etcd) 仓库中，有 go 的 etcd client 实现，那么 etcd go 是如何实现 credentials 的。

```go
func newClient(cfg *Config) (*Client, error) {
	// ...

	if cfg.Username != "" && cfg.Password != "" {
		client.Username = cfg.Username
		client.Password = cfg.Password
		client.authTokenBundle = credentials.NewPerRPCCredentialBundle()
	}

	// ...

	// Use a provided endpoint target so that for https:// without any tls config given, then
	// grpc will assume the certificate server name is the endpoint host.
	conn, err := client.dialWithBalancer()
	if err != nil {
		client.cancel()
		client.resolver.Close()
		// TODO: Error like `fmt.Errorf(dialing [%s] failed: %v, strings.Join(cfg.Endpoints, ";"), err)` would help with debugging a lot.
		return nil, err
	}
	client.conn = conn

	client.Cluster = NewCluster(client)
	client.KV = NewKV(client)
	client.Lease = NewLease(client)
	client.Watcher = NewWatcher(client)
	client.Auth = NewAuth(client)
	client.Maintenance = NewMaintenance(client)

	// get token with established connection
	ctx, cancel = client.ctx, func() {}
	if client.cfg.DialTimeout > 0 {
		ctx, cancel = context.WithTimeout(ctx, client.cfg.DialTimeout)
	}
	err = client.getToken(ctx)
	if err != nil {
		client.Close()
		cancel()
		// TODO: Consider fmt.Errorf("communicating with [%s] failed: %v", strings.Join(cfg.Endpoints, ";"), err)
		return nil, err
	}
	cancel()

	if cfg.RejectOldCluster {
		if err := client.checkVersion(); err != nil {
			client.Close()
			return nil, err
		}
	}

	go client.autoSync()
	return client, nil

}
```

设置 `client.authTokenBundle` 为 `client/v3/credentials.perRPCCredentialBundle`，其中 `perRPCCredential` 实现 `grpc-go` 库中的 `grpc/credentials.PerRPCCredentials` 接口；然后，通过 `client.getToken` 方法通过 `Authenticate` 方法请求 etcd 获取 token 并设置到 `client.authTokenBundle` 中，这样请求就可以通过 `grpc/credentials.PerRPCCredentials` 接口获取到 Token。

go etcd client 利用 `grpc-go` 中的 Interceptor 机制设置了 unary interceptor 以及 stream interceptor。通过 `client.dialWithBalancer` 方法调用 `client.dial` 方法以及  `client.dialSetupOpts` 方法。

```go
func (c *Client) dialSetupOpts(creds grpccredentials.TransportCredentials, dopts ...grpc.DialOption) []grpc.DialOption {
	var opts []grpc.DialOption

	// ...

	// Interceptor retry and backoff.
	// TODO: Replace all of clientv3/retry.go with RetryPolicy:
	// https://github.com/grpc/grpc-proto/blob/cdd9ed5c3d3f87aef62f373b93361cf7bddc620d/grpc/service_config/service_config.proto#L130
	rrBackoff := withBackoff(c.roundRobinQuorumBackoff(backoffWaitBetween, backoffJitterFraction))
	opts = append(opts,
		// Disable stream retry by default since go-grpc-middleware/retry does not support client streams.
		// Streams that are safe to retry are enabled individually.
		grpc.WithStreamInterceptor(c.streamClientInterceptor(withMax(0), rrBackoff)),
		grpc.WithUnaryInterceptor(c.unaryClientInterceptor(withMax(unaryMaxRetries), rrBackoff)),
	)

	return opts
}
```

**gRPC stream request 是没有 retry 策略的，但是 gRPC unary request 是有 backoff retry 策略。**

```go
func (c *Client) streamClientInterceptor(optFuncs ...retryOption) grpc.StreamClientInterceptor {
	intOpts := reuseOrNewWithCallOptions(defaultOptions, optFuncs)
	return func(ctx context.Context, desc *grpc.StreamDesc, cc *grpc.ClientConn, method string, streamer grpc.Streamer, opts ...grpc.CallOption) (grpc.ClientStream, error) {
		ctx = withVersion(ctx)
		// getToken automatically. Otherwise, auth token may be invalid after watch reconnection because the token has expired
		// (see https://github.com/etcd-io/etcd/issues/11954 for more).
		err := c.getToken(ctx)
		if err != nil {
			c.GetLogger().Error("clientv3/retry_interceptor: getToken failed", zap.Error(err))
			return nil, err
		}
		grpcOpts, retryOpts := filterCallOptions(opts)
		callOpts := reuseOrNewWithCallOptions(intOpts, retryOpts)
		// short circuit for simplicity, and avoiding allocations.
		if callOpts.max == 0 {
			return streamer(ctx, desc, cc, method, grpcOpts...)
		}
		if desc.ClientStreams {
			return nil, status.Errorf(codes.Unimplemented, "clientv3/retry_interceptor: cannot retry on ClientStreams, set Disable()")
		}
		newStreamer, err := streamer(ctx, desc, cc, method, grpcOpts...)
		if err != nil {
			c.GetLogger().Error("streamer failed to create ClientStream", zap.Error(err))
			return nil, err // TODO(mwitkow): Maybe dial and transport errors should be retriable?
		}
		retryingStreamer := &serverStreamingRetryingStream{
			client:       c,
			ClientStream: newStreamer,
			callOpts:     callOpts,
			ctx:          ctx,
			streamerCall: func(ctx context.Context) (grpc.ClientStream, error) {
				return streamer(ctx, desc, cc, method, grpcOpts...)
			},
		}
		return retryingStreamer, nil
	}
}
```

设置了 `streamClientInterceptor` 后，每一次 stream request 都会重新获取一次 auth token，防止 token 过期。

```go
func (c *Client) unaryClientInterceptor(optFuncs ...retryOption) grpc.UnaryClientInterceptor {
	intOpts := reuseOrNewWithCallOptions(defaultOptions, optFuncs)
	return func(ctx context.Context, method string, req, reply any, cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
		ctx = withVersion(ctx)
		grpcOpts, retryOpts := filterCallOptions(opts)
		callOpts := reuseOrNewWithCallOptions(intOpts, retryOpts)
		// short circuit for simplicity, and avoiding allocations.
		if callOpts.max == 0 {
			return invoker(ctx, method, req, reply, cc, grpcOpts...)
		}
		var lastErr error
		for attempt := uint(0); attempt < callOpts.max; attempt++ {
			if err := waitRetryBackoff(ctx, attempt, callOpts); err != nil {
				return err
			}

			// ...

			lastErr = invoker(ctx, method, req, reply, cc, grpcOpts...)
			if lastErr == nil {
				return nil
			}

			// ...

			if isContextError(lastErr) {
				if ctx.Err() != nil {
					// its the context deadline or cancellation.
					return lastErr
				}
				// its the callCtx deadline or cancellation, in which case try again.
				continue
			}
			if c.shouldRefreshToken(lastErr, callOpts) {
				gtErr := c.refreshToken(ctx)
				if gtErr != nil {
					c.GetLogger().Warn(
						"retrying of unary invoker failed to fetch new auth token",
						zap.String("target", cc.Target()),
						zap.Error(gtErr),
					)
					return gtErr // lastErr must be invalid auth token
				}
				continue
			}
			if !isSafeRetry(c, lastErr, callOpts) {
				return lastErr
			}
		}
		return lastErr
	}
}
```

而 `unaryClientInterceptor` 则是通过 retry 策略判断上一次请求的错误判断是否更新 token。

- 如果错误是 `ErrUserEmpty`，则当 `client.authTokenBundle` 不为 nil 时即设置了 username 以及 password，则重新获取 token
- 如果错误是 `ErrInvalidAuthToken` 或者 `ErrAuthOldRevision` 时，且设置了 `retryAuth` flag，则重新获取 token

## Auth token ttl

etcd 默认 auth token 是 simple 并且有过期时间，通过 `--auth-token-ttl` 设置，默认为 300s；它的机制是在 token 没有过期的时间内，建立的 stream 都可以正常的读写数据，但是；当 token 过期，在使用过期的 token 建立的 stream 是无法正常使用的。由于 go 的 etcd client 利用 gRPC interceptor 机制解决了在建立新的 stream 时刷新 token 的问题，因此这里使用 rust 的 [etcd-client](https://docs.rs/etcd-client/latest/etcd_client/index.html) 来复现这个问题，设置 etcd `auth-token-ttl=5`；

```rs
#[tokio::main]
async fn main() {
    let shutdown = handle_signals();
    tokio::pin!(shutdown);

    let endpoints = "http://127.0.0.1:12379,http://127.0.0.1:22379,http://127.0.0.1:32379";
    let endpoints: Vec<_> = endpoints.split(',').map(|addr| addr.trim()).collect();
    let opts = etcd_client::ConnectOptions::new().with_user("user1", "test");

    let mut client = Client::connect(endpoints, Some(opts)).await.unwrap();
    let opt = WatchOptions::new().with_prefix();

    let sleep = tokio::time::sleep(Duration::from_secs(30));
    tokio::pin!(sleep);

    'a: loop {
        // client
        //     .auth_client()
        //     .set_client_auth("user1".to_string(), "test".to_string())
        //     .await
        //     .unwrap();
        // println!("set client auth success");

        let (mut watcher, mut stream) = client
            .watch("/registry/foo", Some(opt.clone()))
            .await
            .unwrap();

        loop {
            println!("start watch loop =======");
            tokio::select! {
                _ = &mut shutdown => {
                    println!("shutdown signal received");
                    break 'a;
                }

                _ = &mut sleep => {
                    println!("sleep timeout");
                    watcher.cancel().await.unwrap();
                    sleep.as_mut().reset(Instant::now() + Duration::from_secs(300));
                    continue 'a;
                }

                res = stream.message() => {
                    let resp = res.inspect_err(|err| println!("watch response error: {}", err)).unwrap()
                        .ok_or_else(|| println!("Channel of etcd watch response closed")).unwrap();

                    println!("watch resp: {:?}", resp);
                }
            }
        }
    }
}
```

在刚开始的 30s 内是可以正常接收到 watch response 的，因为 watch stream 是在 token 没有过期的时间内设置的，但是；当 30s 过后，由于 token 过期，在建立的 stream 是非法的，无法接收 watch response。

### Fix

每次建立新的 watch stream 时，需要重新设置 auth token，即把注释的代码打开即可。
