# 通过通道通信.

通道可用于在运行的任务之间传递数据. 通道本质上是一个等待队列, 允许多生产者单接收者的任务. 通道在 `init` 任务中构造, 由静态分配的内存支持. 发送和接收端点被分发给 _software_ 任务:

```rust,noplayground
...
const CAPACITY: usize = 5;
#[init]
    fn init(_: init::Context) -> (Shared, Local) {
        let (s, r) = make_channel!(u32, CAPACITY);
        receiver::spawn(r).unwrap();
        sender1::spawn(s.clone()).unwrap();
        sender2::spawn(s.clone()).unwrap();
        ...
```

在这种情况下, 通道保存 `u32` 类型的数据, 容量为 5 个元素.

通道也可以在 _hardware_ 任务中使用, 但只能以非 `async` 方式, 通过 [Try API](#try-api).

## 发送数据

`send` 方法将消息发送到通道, 如下所示:

```rust,noplayground
#[task]
async fn sender1(_c: sender1::Context, mut sender: Sender<'static, u32, CAPACITY>) {
    hprintln!("Sender 1 sending: 1");
    sender.send(1).await.unwrap();
}
```

## 接收数据

接收者可以 `await` 传入的消息:

```rust,noplayground
#[task]
async fn receiver(_c: receiver::Context, mut receiver: Receiver<'static, u32, CAPACITY>) {
    while let Ok(val) = receiver.recv().await {
        hprintln!("Receiver got: {}", val);
        ...
    }
}
```

通道是使用一个小的 (全局) _Critical Section_ (CS) 实现的, 用于防止竞争条件. 用户必须提供一个 CS 实现. 示例中, CS 实现由平台 crate 提供, 例如 `cortex-m/critical-section-single-core` 或 `esp32c3/critical-section`.

完整的示例:

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/async-channel.rs}}
```

```console
$ cargo xtask qemu --verbose --example async-channel
```

```console
{{#include ../../../../ci/expected/lm3s6965/async-channel.run}}
```

发送端点也可以被 `await`. 如果通道容量尚未达到上限, `await` 发送端可以立即推进, 而在容量已满的情况下, 发送端会一直阻塞直到队列中有空闲位置. 这样数据就永远不会丢失.

在下面的示例中, `CAPACITY` 被减少到 1, 这强制发送任务等待直到通道中的数据被接收.

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/async-channel-done.rs}}
```

查看输出, 我们会发现 `Sender 2` 将一直等待, 直到 `Sender 1` 发送的数据被接收.

> **注意** 同一优先级的 _software_ 任务彼此异步执行, 因此 **不能** 假定严格的顺序. (这里呈现的顺序仅适用于当前的实现, 在 RTIC 框架的不同发行版之间可能发生变化.)

```console
$ cargo xtask qemu --verbose --example async-channel-done
{{#include ../../../../ci/expected/lm3s6965/async-channel-done.run}}
```

## 错误处理

如果所有发送端都已 drop, 则 `await` 一个空的接收通道会导致错误. 这允许优雅地实现不同类型的关闭操作.

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/async-channel-no-sender.rs}}
```

```console
$ cargo xtask qemu --verbose --example async-channel-no-sender
```

```console
{{#include ../../../../ci/expected/lm3s6965/async-channel-no-sender.run}}
```

类似地, 如果接收端已被 drop, `await` 一个发送通道会导致错误. 这允许优雅地实现应用级错误处理.

产生的错误会将数据返回给发送端, 允许发送端采取适当的操作 (例如将数据存储起来以便稍后重发).

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/async-channel-no-receiver.rs}}
```

```console
$ cargo xtask qemu --verbose --example async-channel-no-receiver
```

```console
{{#include ../../../../ci/expected/lm3s6965/async-channel-no-receiver.run}}
```

## Try API

使用 Try API, 你可以在不要求操作必须成功的非 `async` 上下文中, 从通道发送或向通道接收数据.

该 API 通过 `Receiver::try_recv` 和 `Sender::try_send` 暴露.

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/async-channel-try.rs}}
```

```console
$ cargo xtask qemu --verbose --example async-channel-try
```

```console
{{#include ../../../../ci/expected/lm3s6965/async-channel-try.run}}
```