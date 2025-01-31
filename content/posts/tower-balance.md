+++
title = "tower balance"
date = "2024-12-14T15:48:01+08:00"
draft = false
categories = ["rust"]
tags = ["code", "tower"]
author = ["wiser"]
description = "Balance policy in tower crate"
ShowWordCount = true
+++

## 前言

在之前分析 tonic 的[文章](/posts/client-of-grpc-in-rust/)中，由于 `tonic` 中大量使用的 `tower` 库中的组件，并且没有分析 `tonic` 中的 balance 策略，相当于还有一部分的东西是缺失的，因此；在这篇文章中补全。

## 概述 {#Overview}

从 [`tower`](https://docs.rs/tower/latest/tower/) 文档的概述中，就可以得知 `tower` 为网络请求的客户端和服务端提供了模块化并且可重用的可靠组件；而这些组件的核心就是 `Service` trait。

`tower` 库包含四个 crates

- tower：负责组件的实现逻辑
- tower-service：`Service` trait 的定义
- tower-layer：`Layer` trait 的定义
- tower-test：集成测试模块

从文档，可以知道要想了解 tower 中组件的实现，必须先了解 `Service` trait 的定义以及如何使用，因此；这篇文章着重于 `Service` trait 以及 `balance` module。

## Service

从 `tower` 的文档，最醒目的一行就是对 `Service` trait 的描述：

**`async fn(request) -> Result<Response, Error>`**

这即是描述 `Service` trait 也描述一个网络请求的模型，`Service` trait 为实现这个请求模型做了以下定义：

```rs
pub trait Service<Request> {
    type Response;
    type Error;
    type Future: Future<Output = Result<Self::Response, Self::Error>>;

    // Required methods
    fn poll_ready(
        &mut self,
        cx: &mut Context<'_>,
    ) -> Poll<Result<(), Self::Error>>;
    fn call(&mut self, req: Request) -> Self::Future;
}
```

`Service` trait 定义了三个关联类型（associated type）分别是 `Response`，`Error` 以及 `Future`：

- Response: 请求返回的响应
- Error：请求发生错误返回的 Error
- Future：发起请求的 Future trait

以及定义了两个 required methods。

### poll_ready

一个请求是否能否发送，需要有很多条件，比如操作系统的 buffer 是否有空闲，在比如底层连接状态是否正常等，这些都是需要在执行发送请求前提前进行判断的，`poll_ready` 就是进行判断请求能否发送，它会返回以下三种情况：

- `Poll::Ready(Ok(()))`：表示请求可以被发送
- `Poll::Pending`：表示请求暂时不能被处理，因为底层的 Service 还因为各种各样的原因还没有准备好，这个请求需要被阻塞以等待 service 准备好
- `Poll::Ready(Err(_))`：表示底层 service 错误，并且不可以在被使用

**⚠️  注意：**

1. 在发送请求之前，`poll_ready` 方法可以被重复调用，但是必须返回 `Poll::Ready(Ok(()))` 或者 `Poll::Ready(Err(_))`
2. `poll_ready` 方法调用必须在 `call` 方法调用之前；因为底层的 Service 对象可能会被多个资源使用者共享，因此不能假设，底层服务一直都是 Ready 状态，如果底层服务被释放掉，则调用的对象也要及时释放这个 `Service` 对象，举个🌰：如果底层连接的 socket 因为某种错误被操作系统回收，但是因为没有调用 `poll_ready` 方法，而直接调用 `call` 方法这种情况。

### call

异步的处理请求，如果没有调用 `poll_ready` 就调用 `call` 则有可能造成程序 panic

## Balance

目前，tower 中实现的 load balance 只有 `p2c(power of two random choices)`，在该 module 下面有四个 struct 的实现：

- Balance：p2c load balance
- MakeBalance：构建 p2c balance 的对象
- MakeBalanceLayer：通过 Layer trait 构建 p2c balance 对象
- MakeFuture：构建 p2c balance 的 Future

因此，着重了解 `Balance` 对象中有哪些字段声明。

```rs
pub struct Balance<D, Req>
where
    D: Discover,
    D::Key: Hash,
{
    discover: D,

    services: ReadyCache<D::Key, D::Service, Req>,
    ready_index: Option<usize>,

    rng: Box<dyn Rng + Send + Sync>,

    _req: PhantomData<Req>,
}
```

- discover：要求是要有实现 `tower::discover::Discover` trait 的对象，动态管理底层 Service 对象
- services：底层 Services 集合
- ready_index：p2c 策略 pick 好的 Service index
- rng：随机因子
- _req：关联的泛型 Req 的声明字段

在[概述](#Overview)中有说明，tower 中实现的组件都以 `Service` trait 为核心，因此；`Balance` 对象实现的 `Service` trait 就是实际 p2c 策略实现的逻辑。

```rs
impl<D, Req> Service<Req> for Balance<D, Req>
where
    D: Discover + Unpin,
    D::Key: Hash + Clone,
    D::Error: Into<crate::BoxError>,
    D::Service: Service<Req> + Load,
    <D::Service as Load>::Metric: std::fmt::Debug,
    <D::Service as Service<Req>>::Error: Into<crate::BoxError>,
{
    type Response = <D::Service as Service<Req>>::Response;
    type Error = crate::BoxError;
    type Future = future::MapErr<
        <D::Service as Service<Req>>::Future,
        fn(<D::Service as Service<Req>>::Error) -> crate::BoxError,
    >;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        // 接收从 Discover 中收到的 Change 对象，动态管理底层 Service
        let _ = self.update_pending_from_discover(cx)?;
        // 不断的 poll 底层 Service，并把它们变为 ready 状态
        self.promote_pending_to_ready(cx);

        loop {
            // p2c load balance 实现逻辑
            if let Some(index) = self.ready_index.take() {
                match self.services.check_ready_index(cx, index) {
                    Ok(true) => {
                        // The service remains ready.
                        self.ready_index = Some(index);
                        return Poll::Ready(Ok(()));
                    }
                    Ok(false) => {
                        // The service is no longer ready. Try to find a new one.
                        trace!("ready service became unavailable");
                    }
                    Err(Failed(_, error)) => {
                        // The ready endpoint failed, so log the error and try
                        // to find a new one.
                        debug!(%error, "endpoint failed");
                    }
                }
            }

            // Select a new service by comparing two at random and using the
            // lesser-loaded service.
            self.ready_index = self.p2c_ready_index();
            if self.ready_index.is_none() {
                debug_assert_eq!(self.services.ready_len(), 0);
                // We have previously registered interest in updates from
                // discover and pending services.
                return Poll::Pending;
            }
        }
    }

    fn call(&mut self, request: Req) -> Self::Future {
        let index = self.ready_index.take().expect("called before ready");
        self.services
            .call_ready_index(index, request)
            .map_err(Into::into)
    }
}
```

上述代码就是，整个 `Balance` 的核心逻辑了，可以看到，对比 `grpc-go` 中 balancer 的实现要简洁很多，而且从我个人角度来看也实现的更加优雅，`grpc-go` 中的实现调用，在我阅读期间简直让我头昏脑胀，不得不用时序图来表明调用逻辑，具体可以看这篇[文章](/posts/client-of-grpc-in-go-balancer/)。

### poll_ready

```rs
/// Polls `discover` for updates, adding new items to `not_ready`.
///
/// Removals may alter the order of either `ready` or `not_ready`.
fn update_pending_from_discover(
    &mut self,
    cx: &mut Context<'_>,
) -> Poll<Option<Result<(), error::Discover>>> {
    debug!("updating from discover");
    loop {
        match ready!(Pin::new(&mut self.discover).poll_discover(cx))
            .transpose()
            .map_err(|e| error::Discover(e.into()))?
        {
            None => return Poll::Ready(None),
            Some(Change::Remove(key)) => {
                trace!("remove");
                self.services.evict(&key);
            }
            Some(Change::Insert(key, svc)) => {
                trace!("insert");
                // If this service already existed in the set, it will be
                // replaced as the new one becomes ready.
                self.services.push(key, svc);
            }
        }
    }
}
```

不断的 poll `Discover` 对象并从中接收 `tower::discover::Change` 对象，如果是 `Change::Remove` 则从 services 集合中删除底层 service，如果是 `Change::Insert` 则把 service 加入 services 集合中。

```rs
fn promote_pending_to_ready(&mut self, cx: &mut Context<'_>) {
    loop {
        match self.services.poll_pending(cx) {
            Poll::Ready(Ok(())) => {
                // There are no remaining pending services.
                debug_assert_eq!(self.services.pending_len(), 0);
                break;
            }
            Poll::Pending => {
                // None of the pending services are ready.
                debug_assert!(self.services.pending_len() > 0);
                break;
            }
            Poll::Ready(Err(error)) => {
                // An individual service was lost; continue processing
                // pending services.
                debug!(%error, "dropping failed endpoint");
            }
        }
    }
    trace!(
        ready = %self.services.ready_len(),
        pending = %self.services.pending_len(),
        "poll_unready"
    );
}
```

`promote_pending_to_ready` 方法不断的 poll services 集合；

- 返回 `Poll::Ready(Ok(()))`：表示底层有可用的 service 并返回
- 返回 `Poll::Pending`：表示底层的所有 service 都不可用，阻塞调用，等待 Event 事件唤醒，并继续 poll services 集合，知道有可用 service 返回
- 返回 `Poll::Ready(Err(_))`：表示该 service 内部出错，不可用；继续 poll 下一个 service

然后，通过 p2c 策略从 services 集合中的 ready 对象集合中获取一个 service 并把这个 service 的 index 存入 `ready_index` 字段中，为 `call` 调用做准备

### call

通过 `poll_ready` 方法调用，可用的 service 的 index 已经存在 `ready_index` 字段中，从中取出 index 并通过 `self.services.call_ready_index` 发送请求，等待响应。

## ReadyCache

上述中 services 集合是利用的 `tower::ready_cache::ReadyCache` 对象。

```rs
pub struct ReadyCache<K, S, Req>
where
    K: Eq + Hash,
{
    pending: FuturesUnordered<Pending<K, S, Req>>,

    pending_cancel_txs: IndexMap<K, CancelTx>,

    ready: IndexMap<K, (S, CancelPair)>,
}
```

- `pending`：`FuturesUnordered` 是一个实现 `Stream` trait 的对象，暂时不可用的 service 集合
- `pending_cancel_txs`：当 service 不可用或者被删除时，通过 `CancelTx` 删除 service
- `ready`：可用的 service 集合

当一个新的 service 通过 `update_pending_from_discover` 方法中的 `self.services.push(key, svc)` 方法调用，会被统一加入到 `pending` 对象中，因此；调用 `self.services.poll_pending` 方法，会 poll `pending` 中的每一个对象。

```rs
pub fn poll_pending(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), error::Failed<K>>> {
    loop {
        match Pin::new(&mut self.pending).poll_next(cx) {
            Poll::Pending => return Poll::Pending,
            Poll::Ready(None) => return Poll::Ready(Ok(())),
            Poll::Ready(Some(Ok((key, svc, cancel_rx)))) => {
                trace!("endpoint ready");
                let cancel_tx = self.pending_cancel_txs.swap_remove(&key);
                if let Some(cancel_tx) = cancel_tx {
                    // Keep track of the cancelation so that it need not be
                    // recreated after the service is used.
                    self.ready.insert(key, (svc, (cancel_tx, cancel_rx)));
                } else {
                    assert!(
                        cancel_tx.is_some(),
                        "services that become ready must have a pending cancelation"
                    );
                }
            }
            Poll::Ready(Some(Err(PendingError::Canceled(_)))) => {
                debug!("endpoint canceled");
                // The cancellation for this service was removed in order to
                // cause this cancellation.
            }
            Poll::Ready(Some(Err(PendingError::Inner(key, e)))) => {
                let cancel_tx = self.pending_cancel_txs.swap_remove(&key);
                assert!(
                    cancel_tx.is_some(),
                    "services that return an error must have a pending cancelation"
                );
                return Err(error::Failed(key, e.into())).into();
            }
        }
    }
}
```

poll 每个其中的 servicer 对象只会返回以下结果：

- `Poll::Pending`：该服务暂时不可用，因此继续留在 `pending` 中
- `Poll::Ready(None)`：表示 `pending` 集合中没有 service 对象了
- `Poll::Ready(Ok(_))`：表示该 service 已经处于 ready 状态，可以发送请求了，并把它加入到 `ready` 集合中
- `Poll::Ready(Err(_))`: 这里把 Error 分成两种类型，一种是 `PendingError::Canceled` 表示已经被 cancel 掉的 service，service 已经被删除掉了，因此这里忽略；另一种错误是 `PendingError::Inner`，这里表示 service 内部错误，不能被忽略处理，需要调用 CancelTx，并把 service cancel 掉

这样，就把所有处于 `pending` 集合中的 service 都处理了一边，并把可用的 service 加入到 `ready` 集合中，暂时不可用的 service 继续留在 `pending` 中，发生错误的 service 进行 cancel 处理。

```rs
/// Calls a ready service by index.
///
/// # Panics
///
/// If the specified index is out of range.
pub fn call_ready_index(&mut self, index: usize, req: Req) -> S::Future {
    let (key, (mut svc, cancel)) = self
        .ready
        .swap_remove_index(index)
        .expect("check_ready_index was not called");

    let fut = svc.call(req);

    // If a new version of this service has been added to the
    // unready set, don't overwrite it.
    if !self.pending_contains(&key) {
        self.push_pending(key, svc, cancel);
    }

    fut
}
```

`call_ready_index` 方法会通过 index 从 `ready` 集合中获取 service 并调用该 service 的 `Service.Call` 方法，并返回 `Service` trait 中定义的关联类型 Future，让调用者通过 `await` 获取 Response。

同时，会把从 ready 集合中取出的 service 对象，加入到 `pending` 集合中，这样等待，下一次的调用；如果该 service 继续可以使用，那么就在把它从 `pending` 集合在加入到 `ready` 集合中，通过从 `pending` -> `ready`，以及 `ready` -> `pending` 这样的循环调用，实现服务请求的可靠使用。
