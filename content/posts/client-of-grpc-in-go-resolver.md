+++
title = "client of gRPC in go - resolver"
date = "2024-10-05T14:08:52+08:00"
draft = false
categories = ["go"]
tags = ["gRPC", "code"]
author = ["wiser"]
description = "Source code of grpc-go"
ShowWordCount = true
+++

## 前言

最近阅读了 gRPC 在 rust 中的实现，也想了解 gRPC 在 go 中的实现；在以前的项目中使用微服务架构，大量使用 gRPC 进行服务间的 RPC 调用，但是当时并没有深入研究，现在回想起来大概有以下几点原因：

1. 项目是 TOB 项目，并没有集群的概念，每一个服务就一个实例
2. 项目需求多，以实现功能为主要目标
3. 个人问题，没有养成追根溯源习惯

由于以上问题，没有很了解 gRPC client 在 go 是如何实现，并如何去合理使用的，现在把这些补一补。

## grpc-go

由于 go 的文档相比于 rust 而言，实在是混乱不堪；没有办法从文档入手，了解项目的整体。因此；直接从项目的 example 中入手，去了解是一个比较好的方法。

PS：文章中所贴代码均为 `grpc-go v1.70.0` 版本的代码。

![gRPC Go](/img/client-of-grpc-in-go/grpc-go.png)

上图是我整理的 grpc-go 的上层结构图，没有包括底层 HTTP2 的实现，这篇文章主要了解整体流程以及 `Resolver`。

以 `example/helloword/greeter_client/main.go` 中的代码为例：

```go
const (
	defaultName = "world"
)

var (
	addr = flag.String("addr", "localhost:50051", "the address to connect to")
	name = flag.String("name", defaultName, "Name to greet")
)

func main() {
	flag.Parse()
	// Set up a connection to the server.
	conn, err := grpc.NewClient(*addr, grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()
	c := pb.NewGreeterClient(conn)

	// Contact the server and print out its response.
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()
	r, err := c.SayHello(ctx, &pb.HelloRequest{Name: *name})
	if err != nil {
		log.Fatalf("could not greet: %v", err)
	}
	log.Printf("Greeting: %s", r.GetMessage())
}
```

可以看到，主要要关注两个函数调用以及一个方法调用，下面分别去了解它们。

### grpc.NewClient

```go
func NewClient(target string, opts ...DialOption) (conn *ClientConn, err error) {
	cc := &ClientConn{
		target: target,
		conns:  make(map[*addrConn]struct{}),
		dopts:  defaultDialOptions(),
	}

    ...

	// Determine the resolver to use.
	if err := cc.initParsedTargetAndResolverBuilder(); err != nil {
		return nil, err
	}

    ...

    // 通过解析 json，获取 ServiceConfig 配置
	if cc.dopts.defaultServiceConfigRawJSON != nil {
		scpr := parseServiceConfig(*cc.dopts.defaultServiceConfigRawJSON, cc.dopts.maxCallAttempts)
		if scpr.Err != nil {
			return nil, fmt.Errorf("%s: %v", invalidDefaultServiceConfigErrPrefix, scpr.Err)
		}
		cc.dopts.defaultServiceConfig, _ = scpr.Config.(*ServiceConfig)
	}
	cc.mkp = cc.dopts.copts.KeepaliveParams

    ...

	cc.csMgr = newConnectivityStateManager(cc.ctx, cc.channelz)
	cc.pickerWrapper = newPickerWrapper(cc.dopts.copts.StatsHandlers)

	cc.metricsRecorderList = stats.NewMetricsRecorderList(cc.dopts.copts.StatsHandlers)

	cc.initIdleStateLocked() // Safe to call without the lock, since nothing else has a reference to cc.
	cc.idlenessMgr = idle.NewManager((*idler)(cc), cc.dopts.idleTimeout)

	return cc, nil
}
```

上述代码省略了，配置的初始化，interceptor 的初始化以及证书，权限等，保留了我认为比较重要的部分。

从函数的参数，就可以明显感觉到 go 与 rust 的不同，rust 中如果有很多的配置项，较多会采用 [`builder pattern`](https://en.wikipedia.org/wiki/Builder_pattern) 的方式，go 初始化 gRPC client 就稍微随意一些了。注意 `DailOption` 是一个 `interface`。

```go
type ClientConn struct {
    ...

	target              string            // User's dial target.
	parsedTarget        resolver.Target   // 通过 target 构造的 URL
	authority           string            // See initAuthority().
	dopts               dialOptions       // Default and user specified dial options.
	channelz            *channelz.Channel // Channelz object.
	resolverBuilder     resolver.Builder  // See initParsedTargetAndResolverBuilder().
	idlenessMgr         *idle.Manager
	metricsRecorderList *stats.MetricsRecorderList

	csMgr              *connectivityStateManager
	pickerWrapper      *pickerWrapper

	// mu protects the following fields.
	// TODO: split mu so the same mutex isn't used for everything.
	mu              sync.RWMutex
	resolverWrapper *ccResolverWrapper         // Always recreated whenever entering idle to simplify Close.
	balancerWrapper *ccBalancerWrapper         // Always recreated whenever entering idle to simplify Close.
	sc              *ServiceConfig             // Latest service config received from the resolver.
	conns           map[*addrConn]struct{}     // Set to nil on close.

    ...
}
```

上面列举 `ClientConn` 结构包含的一些字段，其中：

- resolverBuilder：是用来构造 resolver 对象
- pickerWrapper：是 load balance 的 handle，通过这个去获取一个底层连接
- resolverWrapper：服务发现的接口包装，grpc-go 内置实现了 dns
- balancerWrapper：load balance 的接口包装，`grpc-go` 内置实现了很多的 load balance 策略，可以通过配置选择具体那种策略，这里就要比 rust 的 tonic 灵活多了
- conns：连接地址的集合

```go
func (cc *ClientConn) initParsedTargetAndResolverBuilder() error {
	logger.Infof("original dial target is: %q", cc.target)

	var rb resolver.Builder
	parsedTarget, err := parseTarget(cc.target)
	if err == nil {
		rb = cc.getResolver(parsedTarget.URL.Scheme)
		if rb != nil {
			cc.parsedTarget = parsedTarget
			cc.resolverBuilder = rb
			return nil
		}
	}

	// We are here because the user's dial target did not contain a scheme or
	// specified an unregistered scheme. We should fallback to the default
	// scheme, except when a custom dialer is specified in which case, we should
	// always use passthrough scheme. For either case, we need to respect any overridden
	// global defaults set by the user.
	defScheme := cc.dopts.defaultScheme
	if internal.UserSetDefaultScheme {
		defScheme = resolver.GetDefaultScheme()
	}

	canonicalTarget := defScheme + ":///" + cc.target

	parsedTarget, err = parseTarget(canonicalTarget)
	if err != nil {
		return err
	}
	rb = cc.getResolver(parsedTarget.URL.Scheme)
	if rb == nil {
		return fmt.Errorf("could not get resolver for default scheme: %q", parsedTarget.URL.Scheme)
	}
	cc.parsedTarget = parsedTarget
	cc.resolverBuilder = rb
	return nil
}
```

`ClientConn.initParsedTargetAndResolverBuilder` 方法解析传入的 tagert 地址，并根据地址的 schema 构建 `ResolverBuilder`，并存入 `ClientConn.resolverBuilder` 字段；`parseTarget` 函数会把一个合法的 target 解析成 `URL` 结构体并构造 `resolver.Target` 结构体，`ClientConn.getResolver` 方法通过 target 的 schema 先从传入的配置中查询是否有调用者实现的 `resolver.Builder`，否则就从默认的 `resolver.Builder` 中查询并返回，`grpc-go` 默认实现的 builder 有以下：

- `dns`: internal/resolver/dns/dns_resolver.go
- `passthrough`: internal/resolver/passthrough/passthrough.go
- `unix/unix-abstract`: internal/resolver/unix/unix.go

`ClientConn.initIdleStateLocked` 方法会初始化 `resolverWrapper` 以及 `balancerWrapper` 字段分别用于包装 `resolver` 以及 `balancer`；`NewClient` 函数的主要作用就是初始化。

### pb.NewGreeterClient

这个函数返回了 `greeterClient` 对象，该对象实现了 `GreeterClient` interface，这里的逻辑都是 protobuf 自动生成的。

### c.SayHello

由于 `ClientConn` 分别在 `call.go` 以及 `stream.go` 文件中实现了 `Invoke` 和 `NewStream` 方法，因此 `ClientConn` 实现了 `ClientConnInterface` interface。因此有了以下调用：

```go
func (c *greeterClient) SayHello(ctx context.Context, in *HelloRequest, opts ...grpc.CallOption) (*HelloReply, error) {
	cOpts := append([]grpc.CallOption{grpc.StaticMethod()}, opts...)
	out := new(HelloReply)
	err := c.cc.Invoke(ctx, Greeter_SayHello_FullMethodName, in, out, cOpts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}

func (cc *ClientConn) Invoke(ctx context.Context, method string, args, reply any, opts ...CallOption) error {
	// allow interceptor to see all applicable call options, which means those
	// configured as defaults from dial option as well as per-call options
	opts = combine(cc.dopts.callOptions, opts)

	if cc.dopts.unaryInt != nil {
		return cc.dopts.unaryInt(ctx, method, args, reply, cc, invoke, opts...)
	}
	return invoke(ctx, method, args, reply, cc, opts...)
}

func invoke(ctx context.Context, method string, req, reply any, cc *ClientConn, opts ...CallOption) error {
	cs, err := newClientStream(ctx, unaryStreamDesc, cc, method, opts...)
	if err != nil {
		return err
	}
	if err := cs.SendMsg(req); err != nil {
		return err
	}
	return cs.RecvMsg(reply)
}
```

`SayHello` 方法调用到最后，可以清除的看到 `SendMsg` 以及 `RecvMsg` 的调用，分别是发送 request 以及接收 response。

## Resolver

`resolver` 是解析目标 server 地址的接口，通过 `resolver` 通知 `balancer` 底层 endpoint 的变化。

重要的结构体包括以下两个：

```go
// State contains the current Resolver state relevant to the ClientConn.
type State struct {
	Addresses []Address // 最新的 server 地址信息

	Endpoints []Endpoint // 替代 Addresses

	ServiceConfig *serviceconfig.ParseResult

	Attributes *attributes.Attributes
}

type Address struct {
	// Addr is the server address on which a connection will be established.
	Addr string

	ServerName string

	Attributes *attributes.Attributes

    ...
}
```

重要的接口有：

```go
type ClientConn interface {
	UpdateState(State) error

	ReportError(error)

	// Deprecated: Use UpdateState instead.
	NewAddress(addresses []Address)

	// ParseServiceConfig parses the provided service config and returns an
	// object that provides the parsed config.
	ParseServiceConfig(serviceConfigJSON string) *serviceconfig.ParseResult
}

// Resolver watches for the updates on the specified target.
// Updates include address updates and service config updates.
type Resolver interface {
	// ResolveNow will be called by gRPC to try to resolve the target name
	// again. It's just a hint, resolver can ignore this if it's not necessary.
	//
	// It could be called multiple times concurrently.
	ResolveNow(ResolveNowOptions)
	// Close closes the resolver.
	Close()
}

// Builder creates a resolver that will be used to watch name resolution updates.
type Builder interface {
	// Build creates a new resolver for the given target.
	//
	// gRPC dial calls Build synchronously, and fails if the returned error is
	// not nil.
	Build(target Target, cc ClientConn, opts BuildOptions) (Resolver, error)
	// Scheme returns the scheme supported by this resolver.  Scheme is defined
	// at https://github.com/grpc/grpc/blob/master/doc/naming.md.  The returned
	// string should not contain uppercase characters, as they will not match
	// the parsed target's scheme as defined in RFC 3986.
	Scheme() string
}
```

可以参考 `examples/features/name_resolving/client/main.go` 给的🌰，传入自定义的 `resolver`；每个 `resolver` 需要实现上述的 `Builder` 以及 `Resolver` 接口；在上述的 `newClientStream` 函数中会调用 `Builder.Build` 方法，构造 `resolve`；

```go
func newClientStream(ctx context.Context, desc *StreamDesc, cc *ClientConn, method string, opts ...CallOption) (_ ClientStream, err error) {
	// Start tracking the RPC for idleness purposes. This is where a stream is
	// created for both streaming and unary RPCs, and hence is a good place to
	// track active RPC count.
	if err := cc.idlenessMgr.OnCallBegin(); err != nil {
		return nil, err
	}

    ...

	return newStream(ctx, func() {})
}
```

`ClientConn.idlenessMgr` 是管理 `ClientConn` 的状态的，不管是 gRPC unary request 还是 gRPC streaming reqeust 都会调用 `cc.idlenessMgr.OnCallBegin` 方法；在该方法中，会调用 `Manager.ExitIdleMode` 方法；最后调用 `m.enforcer.ExitIdleMode` 方法；`ExitIdleMode` 是接口 `Enforcer` 的其中一个方法；

```go
// Enforcer is the functionality provided by grpc.ClientConn to enter
// and exit from idle mode.
type Enforcer interface {
	ExitIdleMode() error
	EnterIdleMode()
}
```

从注释中，可以得知 `Enforcer` 是管理 `ClientConn` 是否进入 idle mode 的接口；

- `ExitIdleMode`：当 `ClientConn` 发起请求后，就会尝试退出 idle mode
- `EnterIdleMode`：当 `ClientConn` 在一定时间内（默认 30 分钟），没有请求；就会尝试进入 idle mode

```go
// OnCallBegin is invoked at the start of every RPC.
func (m *Manager) OnCallBegin() error {
    ...
	// Channel is either in idle mode or is in the process of moving to idle
	// mode. Attempt to exit idle mode to allow this RPC.
	if err := m.ExitIdleMode(); err != nil {
		// Undo the increment to calls count, and return an error causing the
		// RPC to fail.
		atomic.AddInt32(&m.activeCallsCount, -1)
		return err
	}
    ...
}


func (m *Manager) ExitIdleMode() error {
    ...

    // 退出 idle mode
	if err := m.enforcer.ExitIdleMode(); err != nil {
		return fmt.Errorf("failed to exit idle mode: %w", err)
	}

    ...

    // 启动定时器，当定时器超时就会进入 idle mode，
    // 每次检查是否要进入 idle mode 时，就会判断是否有 RPC 调用，
    // 如果有就重置定时器
	m.resetIdleTimerLocked(m.timeout)
	return nil
}
```

而在 `NewClient` 函数的最后，在初始化 `ClientConn.idlenessMgr` 字段时，把 `ClientConn` 对象断言成为 `idler` 对象，因此 `ClientConn` 就是实现了 `Enforcer` 接口的对象。

```go
type idler ClientConn

func (i *idler) ExitIdleMode() error {
	return (*ClientConn)(i).exitIdleMode()
}

// exitIdleMode moves the channel out of idle mode by recreating the name
// resolver and load balancer.  This should never be called directly; use
// cc.idlenessMgr.ExitIdleMode instead.
func (cc *ClientConn) exitIdleMode() (err error) {
    ...

	// This needs to be called without cc.mu because this builds a new resolver
	// which might update state or report error inline, which would then need to
	// acquire cc.mu.
	if err := cc.resolverWrapper.start(); err != nil {
		return err
	}

	cc.addTraceEvent("exiting idle mode")
	return nil
}

func (ccr *ccResolverWrapper) start() error {
	errCh := make(chan error)
	ccr.serializer.TrySchedule(func(ctx context.Context) {
		if ctx.Err() != nil {
			return
		}
		opts := resolver.BuildOptions{
			DisableServiceConfig: ccr.cc.dopts.disableServiceConfig,
			DialCreds:            ccr.cc.dopts.copts.TransportCredentials,
			CredsBundle:          ccr.cc.dopts.copts.CredsBundle,
			Dialer:               ccr.cc.dopts.copts.Dialer,
			Authority:            ccr.cc.authority,
		}
		var err error
		ccr.resolver, err = ccr.cc.resolverBuilder.Build(ccr.cc.parsedTarget, ccr, opts)
		errCh <- err
	})
	return <-errCh
}
```

在 `cc.resolverWrapper.start` 方法中会调用 `resolver` 实现的 `Build` 方法并赋值给 `ccResolverWrapper.resolver` 字段。并且在 `Build` 方法的调用中，在解析完 target server 地址后，会调用 `resolver` package 中 `ClientConn` 接口中的 `UpdateState` 方法来更新底层 endpoint，而实现 `ClientConn` 接口的就是 `ccResolverWrapper` 对象。

```go
func (ccr *ccResolverWrapper) UpdateState(s resolver.State) error {
	ccr.cc.mu.Lock()
	ccr.mu.Lock()
	if ccr.closed {
		ccr.mu.Unlock()
		ccr.cc.mu.Unlock()
		return nil
	}
	if s.Endpoints == nil {
		s.Endpoints = make([]resolver.Endpoint, 0, len(s.Addresses))
		for _, a := range s.Addresses {
			ep := resolver.Endpoint{Addresses: []resolver.Address{a}, Attributes: a.BalancerAttributes}
			ep.Addresses[0].BalancerAttributes = nil
			s.Endpoints = append(s.Endpoints, ep)
		}
	}
	ccr.addChannelzTraceEvent(s)
	ccr.curState = s
	ccr.mu.Unlock()
	return ccr.cc.updateResolverStateAndUnlock(s, nil)
}
```

到这里，`resolver` 的任务就基本完成了，后面就是 `balancer` 以及 `picker` 的工作。
