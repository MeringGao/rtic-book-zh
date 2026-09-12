# `'static` 超能力

在 `#[init]`, `#[idle]` 以及永不返回的软件任务中, `local` 资源具有 `'static` 生命周期.

在需要预分配以及/或在任务, 驱动或某些其他对象之间拆分资源时非常有用. 当驱动 (例如 USB 驱动) 需要分配内存, 以及使用可拆分的数据结构 (例如 [`heapless::spsc::Queue`]) 时, 这非常方便.

在下面的示例中, 两个不同的任务共享一个 [`heapless::spsc::Queue`], 以无锁方式访问这个共享队列.

[`heapless::spsc::Queue`]: https://docs.rs/heapless/0.7.5/heapless/spsc/struct.Queue.html

```rust,noplayground
{{#include ../../../../../examples/lm3s6965/examples/static-resources-in-init.rs}}
```

运行这个程序会产生预期的输出.

```console
$ cargo xtask qemu --verbose --example static-resources-in-init
```

```console
{{#include ../../../../../ci/expected/lm3s6965/static-resources-in-init.run}}
```