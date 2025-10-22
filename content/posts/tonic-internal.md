+++
title = "tonic internal"
date = "2024-09-26"
author = ["wiser"]
description = "Source code for tonic."

[taxonomies]
tags=["gRPC", "code"]
categories=["rust"]
+++

## 前言

公司线上 etcd 集群由于运维同学的不当操作；导致 etcd 集群异常，经过修复后；etcd 集群恢复正常，但是服务没有自动恢复，进而需要分析代码为何服务无法自动恢复。

代码中使用了 [`etcd-client`](https://docs.rs/etcd-client/latest/etcd_client/) 库作为 client 去连接 etcd，该库的底层使用的是 `tonic` 发起 gRPC 请求 etcd。

因此；这篇文章的重点放在 `tonic` 库的实现，顺便解决上层业务代码的问题。

PS: 阅读的 `tonic` 原码版本为 `v0.12.3`

## 文档

首先；阅读 rust 代码，第一步也是笔者认为的重要的一步是 [`tonic`](https://docs.rs/tonic/latest/tonic) 文档的阅读，快速了解库中有哪些 `modules`，`macros`，`struct`，`enum` 以及最重要的 `trait`。

从文档中，可以知道有以下 `modules`

| module    | description                                                               |
| --------- | ------------------------------------------------------------------------- |
| body      | HTTP specific body utilities.                                             |
| client    | Generic client implementation.                                            |
| codec     | Generic encoding and decoding.                                            |
| metadata  | Contains data structures and utilities for handling gRPC custom metadata. |
| server    | Generic server implementation.                                            |
| service   | Utilities for using Tower services with Tonic.                            |
| transport | Batteries included server and client.                                     |

笔者认为重要的有以下：

- `client`：高层抽象，gRPC client 的实现
- `server`：高层抽象，gRPC server 的实现
- `service`：利用 [tower service trait] 实现的工具
- `transport`：底层连接抽象，在 server 端以及 client 端都有使用

`trait` 实现了以下两个：

| trait                | description                                       |
| -------------------- | ------------------------------------------------- |
| IntoRequest          | Trait implemented by RPC request types.           |
| IntoStreamingRequest | Trait implemented by RPC streaming request types. |

从名称中，就可以知道实现 `IntoRequest` trait 的对象可以发起 gRPC unary call，而实现 `IntoStreamingRequest` trait 的对象可以发起 gRPC streaming call。

~~其余的 `struct` 以及 `enum` ，这里暂时按下不表。~~

### client

通过 tonic 给出的 [helloword example](https://github.com/hyperium/tonic/blob/4b8d2c46aa57e40b1e80077f4f7b7d4679027bb5/examples/src/helloworld/client.rs#L8C1-L21C2) 分析 client 端是如何实现。

`GreeterClient` 是 tonic 利用 [prost](https://docs.rs/prost/latest/prost/) 库通过 protobuf 生成的 gRPC client 对象。

```rs
/// The greeting service definition.
#[derive(Debug, Clone)]
pub struct GreeterClient<T> {
    inner: tonic::client::Grpc<T>,
}
impl GreeterClient<tonic::transport::Channel> {
    /// Attempt to create a new client by connecting to a given endpoint.
    pub async fn connect<D>(dst: D) -> Result<Self, tonic::transport::Error>
    where
        D: TryInto<tonic::transport::Endpoint>,
        D::Error: Into<StdError>,
    {
        let conn = tonic::transport::Endpoint::new(dst)?.connect().await?;
        Ok(Self::new(conn))
    }
}
```

`GreeterClient` 就是 `tonic::client::Grpc<T>` 的封装，通过 `connect` 方法创建，`connect` 方法接收一个参数 `D: TryInto<tonic::transport::Endpoint>`，传入 `Endpoint::new` 方法，返回一个 `Endpoint` 对象，这个对象就是 gRPC server 端地址的封装；然后通过 `Endpoint::connect` 方法创建一个 `tonic::transport::Channel` 对象。这个方法设置了 HTTP 连接的参数，然后传入 `Channel::connect` 方法创建 `tonic::transport::Channel` 对象。

```rs
pub(crate) async fn connect<C>(connector: C, endpoint: Endpoint) -> Result<Self, super::Error>
where
    C: Service<Uri> + Send + 'static,
    C::Error: Into<crate::Error> + Send,
    C::Future: Unpin + Send,
    C::Response: rt::Read + rt::Write + HyperConnection + Unpin + Send + 'static,
{
    // buffer_size 是可以异步发送的请求数，默认 1024 个
    let buffer_size = endpoint.buffer_size.unwrap_or(DEFAULT_BUFFER_SIZE);
    // executor 默认是 tokio，在 endpoint 初始化中设置
    let executor = endpoint.executor.clone();

    // 创建 HTTP 连接
    let svc = Connection::connect(connector, endpoint)
        .await
        .map_err(super::Error::from_source)?;
    let (svc, worker) = Buffer::pair(Either::A(svc), buffer_size);
    executor.execute(worker);

    Ok(Channel { svc })
}
```

从这里开始就进入了 tonic 库的底层设计，`tower::buffer::Buffer::pair` 其实是 `tokio::mpsc::unbounded_channel` 方法的封装，最终的效果就是发送端被 `tonic::transport::Channel` 所拥有，而接收端被上述的 `worker(tower::buffer::worker::Worker)` 变量拥有；**利用 `tower::buffer::Buffer` 把底层连接和上层 client 做区分，这样对于使用者来说，就可以在多个 task 中使用 client 发送 gRPC 请求了。**

```rs
impl<T, Request> Future for Worker<T, Request>
where
    T: Service<Request>,
    T::Error: Into<crate::BoxError>,
{
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        if self.finish {
            return Poll::Ready(());
        }

        loop {
            // 从 rx 中获取请求
            match ready!(self.poll_next_msg(cx)) {
                Some((msg, first)) => {
                    let _guard = msg.span.enter();
                    if let Some(ref failed) = self.failed {
                        let _ = msg.tx.send(Err(failed.clone()));
                        continue;
                    }

                    // ...

                    // 利用 service trait 中 poll_ready 判断底层连接是否正常，是否可以发送数据
                    match self.service.poll_ready(cx) {
                        Poll::Ready(Ok(())) => {
                            // 利用 service trait 中 call 发送请求
                            let response = self.service.call(msg.request);

                            // 返回响应给请求方
                            let _ = msg.tx.send(Ok(response));
                        }
                        // 如果底层连接返回 pending，则返回 pending
                        Poll::Pending => {
                            // Put out current message back in its slot.
                            drop(_guard);
                            self.current_message = Some(msg);
                            return Poll::Pending;
                        }
                        // 如果底层连接异常，则返回异常，并关闭 channel
                        Poll::Ready(Err(e)) => {
                            let error = e.into();
                            drop(_guard);
                            self.failed(error);
                            let _ = msg.tx.send(Err(self
                                .failed
                                .as_ref()
                                .expect("Worker::failed did not set self.failed?")
                                .clone()));
                            // Wake any tasks waiting on channel capacity.
                            self.close_semaphore();
                        }
                    }
                }
                None => {
                    // No more more requests _ever_.
                    self.finish = true;
                    return Poll::Ready(());
                }
            }
        }
    }
}
```

获取到 `tonic::transport::Channel` 后就被封装到 `tonic::client::Grpc` 中，最后生成 `GreeterClient` 返回给调用者。

### send gRPC unary request

调用 `client.say_hello(request)` 方法，就发送了一个 gRPC unary request，会被上述的 `worker(tower::buffer::worker::Worker)` 对象接收，该对象实现 `Future` trait，通过 `Connection` 对象实现的 `tower::Service` trait 发送请求给 gRPC server。

到这里，利用 gRPC client 发送请求的流程就结束了，但是在 `etcd-client` 库中使用的并不是简单的 unary call API，而是 streaming call，因此需要看 `etcd-client` 库是如何使用 `tonic` 的。

### balance channel

从上述分析 client 发送 gRPC unary request 过程中，可以发现 `tonic::transport::Channel` 是向底层连接发送请求的唯一通道，那么如果有多个 endpoint，又该如何处理？balance 的策略又是如何？

如果需要我们自己实现，大概逻辑如下：

1. 通过多个 endpoint 创建多个 client，并封装一个对象进行管理
2. 实现 load balance 策略，发送请求时，pick 一个 client 发送请求
3. 实现动态管理 endpoint 即 client 的逻辑

但是，`tonic` 其实通过 `tonic::transport::Channel::balance_channel` 方法优雅地实现了上述逻辑。

```rs
/// Balance a list of [`Endpoint`]'s.
///
/// This creates a [`Channel`] that will listen to a stream of change events and will add or remove provided endpoints.
pub fn balance_channel<K>(capacity: usize) -> (Self, Sender<Change<K, Endpoint>>)
where
    K: Hash + Eq + Send + Clone + 'static,
{
    Self::balance_channel_with_executor(capacity, SharedExec::tokio())
}

/// Balance a list of [`Endpoint`]'s.
///
/// This creates a [`Channel`] that will listen to a stream of change events and will add or remove provided endpoints.
///
/// The [`Channel`] will use the given executor to spawn async tasks.
pub fn balance_channel_with_executor<K, E>(
    capacity: usize,
    executor: E,
) -> (Self, Sender<Change<K, Endpoint>>)
where
    K: Hash + Eq + Send + Clone + 'static,
    E: Executor<Pin<Box<dyn Future<Output = ()> + Send>>> + Send + Sync + 'static,
{
    let (tx, rx) = channel(capacity);
    // 创建 discover 对象，用于管理底层 endpoint
    let list = DynamicServiceStream::new(rx);
    (Self::balance(list, DEFAULT_BUFFER_SIZE, executor), tx)
}

pub(crate) fn balance<D, E>(discover: D, buffer_size: usize, executor: E) -> Self
where
    D: Discover<Service = Connection> + Unpin + Send + 'static,
    D::Error: Into<crate::Error>,
    D::Key: Hash + Send + Clone,
    E: Executor<BoxFuture<'static, ()>> + Send + Sync + 'static,
{
    let svc = Balance::new(discover);

    let svc = BoxService::new(svc);
    let (svc, worker) = Buffer::pair(Either::B(svc), buffer_size);
    executor.execute(Box::pin(worker));

    Channel { svc }
}
```

从上面的代码中，可以清晰的看到 `tonic::transport::Channel::balance_channel` 除了返回 `tonic::transport::Channel` 还多返回一个 `tokio::mpsc::Sender<Change<K, Endpoint>>`，并且从 `Sender` 的发送数据中，就可以得出这个 `Sender` 是动态管理 endpoint 的 handle。

![Balance Channel](/img/client-of-grpc-in-rust/balance_channel.png)

上图是通过 `balance_channel` 方法返回的 `tonic::transport::Channel` 封装，可以看到有很多层，重要的有以下：

- `tonic::transport::channel::service::discover::DynamicServiceStream`：动态更新 endpoint
- `tower::balance::p2c::service::Balance`：[p2c](https://docs.rs/tower/latest/tower/balance/p2c/index.html) balance 策略
- `tower::reconnect::Reconnect`：重连策略

**`DynamicServiceStream`**

```rs
pub(crate) struct DynamicServiceStream<K: Hash + Eq + Clone> {
    changes: Receiver<Change<K, Endpoint>>,
}

impl<K: Hash + Eq + Clone> Stream for DynamicServiceStream<K> {
    type Item = DiscoverResult<K, Connection, crate::Error>;

    fn poll_next(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>> {
        let c = &mut self.changes;
        match Pin::new(&mut *c).poll_recv(cx) {
            Poll::Pending | Poll::Ready(None) => Poll::Pending,
            Poll::Ready(Some(change)) => match change {
                Change::Insert(k, endpoint) => {
                    let mut http = HttpConnector::new();
                    http.set_nodelay(endpoint.tcp_nodelay);
                    http.set_keepalive(endpoint.tcp_keepalive);
                    http.set_connect_timeout(endpoint.connect_timeout);
                    http.enforce_http(false);

                    let connection = Connection::lazy(endpoint.connector(http), endpoint);
                    let change = Ok(Change::Insert(k, connection));
                    Poll::Ready(Some(change))
                }
                Change::Remove(k) => Poll::Ready(Some(Ok(Change::Remove(k)))),
            },
        }
    }
}
```
`DynamicServiceStream` 很简单，除了一个 `DynamicServiceStream::new` 方法外就没有其他方法了，但是它实现了 `Stream` trait 并且 `Item` 是 `Result`。而在 [`tower::discover`](https://github.com/tower-rs/tower/blob/tower-0.4.13/tower/src/discover/mod.rs) module 中有如下实现：

```rs
impl<K, S, E, D: ?Sized> Discover for D
where
    D: TryStream<Ok = Change<K, S>, Error = E>,
    K: Eq,
{
    type Key = K;
    type Service = S;
    type Error = E;

    fn poll_discover(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>,
    ) -> Poll<Option<Result<D::Ok, D::Error>>> {
        TryStream::try_poll_next(self, cx)
    }
}
```

即只要实现 `TryStream` trait 的对象就自动实现 `tower::discover::Discover` trait，而在 `TryStream` trait 定义的文件中又有如下实现：

```rs
impl<S, T, E> TryStream for S
where
    S: ?Sized + Stream<Item = Result<T, E>>,
{
    type Ok = T;
    type Error = E;

    fn try_poll_next(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>,
    ) -> Poll<Option<Result<Self::Ok, Self::Error>>> {
        self.poll_next(cx)
    }
}
```

这样，**`DynamicServiceStream` 就实现了 `Discover` trait。**

**`Balance`**

`Balance` 对象的初始化如下，`discover` 参数就是上面的 `DynamicServiceStream`

```rs
impl<D, Req> Balance<D, Req>
where
    D: Discover,
    D::Key: Hash,
    D::Service: Service<Req>,
    <D::Service as Service<Req>>::Error: Into<crate::BoxError>,
{
    /// Constructs a load balancer that uses operating system entropy.
    pub fn new(discover: D) -> Self {
        Self::from_rng(discover, &mut rand::thread_rng()).expect("ThreadRNG must be valid")
    }
}
```

在这个 `Balance` 对象中，可以根据 `p2c` 策略 pick 一个连接发送请求，也可以动态的增删连接；这里的逻辑是在 `tower` 库中实现，可以另开一篇文章解析，暂时就不细说了。

**`Reconnect`**

`Reconnect` layer 也是 `tower` 库中的实现逻辑，这里要着重说明的是，**当使用 `tower::Service` trait 的 `poll_ready` 方法时，如果连接建立失败，也是会返回 `Poll::Ready(Ok(()))` 的。**

#### 总结

通过 `balance_channel` 方法建立 client 后，可以得到 `tonic::transport::Channel` 和 `tokio::mpsc::Sender`，通过 `tower::transport::Channel` 封装的 client 可以发送 gRPC 请求，而 `tokio::mpsc::Sender` 可以通过发送 `Change` 实现动态管理底层 endpoint。
```rs
/// A change in the service set.
#[derive(Debug)]
pub enum Change<K, V> {
    /// A new service identified by key `K` was identified.
    Insert(K, V),
    /// The service identified by key `K` disappeared.
    Remove(K),
}
```
