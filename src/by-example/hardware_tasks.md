# 硬件任务

RTIC 的核心是使用硬件中断控制器 (cortex-m 上的 [ARM NVIC][NVIC]) 来调度和启动任务. 除 `pre-init` (一个隐藏的 "任务"), `#[init]` 和 `#[idle]` 之外, 所有任务都作为中断处理函数运行.

要将任务绑定到中断, 请使用 `#[task]` 属性的参数 `binds = InterruptName`. 该任务随即成为该硬件中断向量的中断处理函数.

所有绑定到显式中断的任务称为 _硬件任务_, 因为它们是对某个硬件事件做出反应而开始执行的.

指定一个不存在的中断名将导致编译错误. 中断名称通常由 [PAC 或 HAL][pacorhal] crate 定义.

任何可用的中断向量都应该可用. 特定设备可能会将特定的中断优先级绑定到用户代码无法控制的中断向量. 参见例如 [nRF "softdevice"](https://github.com/rtic-rs/rtic/issues/434).

请注意不要使用被硬件特性内部使用的中断向量, RTIC 不知道这些硬件相关的细节.

[pacorhal]: https://docs.rust-embedded.org/book/start/registers.html
[NVIC]: https://developer.arm.com/documentation/100166/0001/Nested-Vectored-Interrupt-Controller/NVIC-functional-description/NVIC-interrupts

## 示例

下面的示例演示了使用 `#[task(binds = InterruptName)]` 属性来声明一个绑定到中断处理函数的硬件任务.

```rust,noplayground
{{#include ../../examples/lm3s6965/examples/hardware.rs}}
```

```console
$ cargo xtask qemu --verbose --example hardware
```

```console
{{#include ../../ci/expected/lm3s6965/hardware.run}}
```