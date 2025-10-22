+++
title = "memory profiling for rust"
date = "2024-05-03"
author = ["wiser"]
description = "Rust program memory profiling with heaptrack."

[taxonomies]
tags=["profiling", "heap"]
categories=["rust"]
+++

## Overview

由于线上某个服务经常出现崩溃并且还会造成雪崩效应，每次出现故障后，不得不立即采取扩容2倍机器的方式来解决故障；并且 review 该服务的代码后，发现逻辑混乱，错误处理缺失，没有相应的日志信息等等问题；因此决定对该服务进行重构，利用 rust 重写，并解决程序容易崩溃，问题不好排查等等线上问题。

重构后，在 2C4G 上的机器进行基准测试后发现，重构后的服务最多只能承载 3000 左右的设备连接；但是，旧的服务可以轻松承载 4w 以上的设备连接，因此需要对代码进行 memory profiling，对代码进行优化解决内存不足的问题。

## Heaptrack

[Heaptrack](https://github.com/KDE/heaptrack) 是一个强大的 Linux 的堆内存分析工具，它可以帮助我们：

- 找到需要优化的热点，并减少应用程序的内存占用
- 找出内存泄漏
- 找出内存分配的热点
- 找出临时分配的内存

具体可以参考这篇博客[^1]，这里详细介绍了 heaptrack。

## Profiling

对一个 release build 的 rust 程序进行 profiling，首先需要开启 `debug info`，在 `Cargo.toml` 中添加以下配置：

```toml
[profile.release]
debug = 1
```

可以在 Cargo book 的 Profiles[^2] 参考更多配置。

如果，想要对 rust 标准库的代码进行 profiling，那么以上的配置还是不够的。因为 rust 的标准库不是通过 debug info 进行构建的。因此，如果想要对标准库中的代码进行 profiling，通常我们需要通过 rust 的原码构建自己的编译器，并且在 `config.toml` 中添加以下配置：

```toml
[rust]
debuginfo-level = 1
```
不过，rust 的原码的性能都是极好的可以参考标准库的引导[^3] 自己进行测试也可以参考这个结果[^4]。

构建好程序后，需要在 linux 平台上运行，无论是通过 `heaptrack <your application and its parameters>` 还是通过 attach 的方式 `heaptrack --pid $(pidof <your application>)`；最终都会得到一个 `gz` 后缀文件的结果，可以通过 `heaptrack_gui` 程序解析这个文件并得到火焰图。

## heaptrack_gui

通过原码构建的方式[^5]编译得到 heaptrack_gui，推荐通过 homebrew 的方式进行安装。

![Flame Graph](/img/heaptrack-rust/heaptrack-flame-graph.png)

通过上述火焰图，得出 [rumqttc::MqttState](https://docs.rs/rumqttc/latest/rumqttc/struct.MqttState.html) 在初始化时占用大量内存。

```rs
/// Creates new mqtt state. Same state should be used during a
/// connection for persistent sessions while new state should
/// instantiated for clean sessions
pub fn new(max_inflight: u16, manual_acks: bool) -> Self {
    MqttState {
        await_pingresp: false,
        collision_ping_count: 0,
        last_incoming: Instant::now(),
        last_outgoing: Instant::now(),
        last_pkid: 0,
        last_puback: 0,
        inflight: 0,
        max_inflight,
        // index 0 is wasted as 0 is not a valid packet id
        outgoing_pub: vec![None; max_inflight as usize + 1], // max_inflight default 100
        outgoing_rel: vec![None; max_inflight as usize + 1],
        incoming_pub: vec![None; std::u16::MAX as usize + 1],
        collision: None,
        // TODO: Optimize these sizes later
        events: VecDeque::with_capacity(100),
        write: BytesMut::with_capacity(10 * 1024),
        manual_acks,
    }
}
```

在初始化 `MqttState` 时；大概需要分配 300 Kb 内存，以下为字段所占内存计算：

```bash
outgoing_pub: 144 * 101 = 14544 bytes
outgoing_rel: 4 * 101 = 404 bytes
incomeing_pub: 4 * 65536 = 262144 bytes
events: 144 * 100 = 14400 bytes
write: 10240 bytes

total: 301732 bytes
```

与基准测试的数据基本符合，因此；定位到问题是由于构建 MQTT client 对象时，分配的内存过多导致的内存使用过高。

经过上述分析，最终通过复用 `rumqttc` 库的 codec 实现自己的 MQTT client 解决内存占用过大的问题。

## Reference

[^1]: https://milianw.de/blog/heaptrack-a-heap-memory-profiler-for-linux.html
[^2]: https://doc.rust-lang.org/cargo/reference/profiles.html
[^3]: https://std-dev-guide.rust-lang.org/development/perf-benchmarking.html
[^4]: https://benchmarksgame-team.pages.debian.net/benchmarksgame/fastest/rust-go.html
[^5]: https://github.com/KDE/heaptrack#heaptrack_gui-dependencies
