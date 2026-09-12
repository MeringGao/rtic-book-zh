# 后台任务 `#[idle]`

用 `idle` 属性标记的函数可以可选地出现在模块中. 它将成为特殊的 _idle task_, 并且必须具有签名 `fn(idle::Context) -> !`.

当存在时, 运行时将在 `init` 之后执行 `idle` 任务. 与 `init` 不同, `idle` 会在 _中断使能_ 的状态下运行, 并且永远不能返回, 正如 `-> !` 函数签名所表明的那样.
[Rust 类型 `!` 意味着 "永不返回"][nevertype].

[nevertype]: https://doc.rust-lang.org/core/primitive.never.html

与 `init` 中一样, 本地声明的资源具有 `'static` 生命周期, 可以安全地访问.

下面的示例显示了 `idle` 在 `init` 之后运行.

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/idle.rs}}
```

```console
$ cargo xtask qemu --verbose --example idle
```

```console
{{#include ../../../../ci/expected/lm3s6965/idle.run}}
```

默认情况下, RTIC 的 `idle` 任务不会尝试针对任何特定目标进行优化.

一个常见且有用的优化是启用 [SLEEPONEXIT], 允许 MCU 在到达 `idle` 时进入睡眠状态.

> **注意**: 除非另行配置, 否则某些硬件在睡眠模式下会禁用调试单元.
>
> 请查阅你的硬件相关文档, 因为这超出了 RTIC 的范畴.

下面的示例展示了如何通过设置
[`SLEEPONEXIT`][SLEEPONEXIT] 并提供一个自定义的 `idle` 任务 (将默认的 [`nop()`][NOP] 替换为 [`wfi()`][WFI]) 来启用睡眠.

[SLEEPONEXIT]: https://developer.arm.com/documentation/100737/0100/Power-management/Sleep-mode/Sleep-on-exit-bit
[WFI]: https://developer.arm.com/documentation/dui0662/b/The-Cortex-M0--Instruction-Set/Miscellaneous-instructions/WFI
[NOP]: https://developer.arm.com/documentation/dui0662/b/The-Cortex-M0--Instruction-Set/Miscellaneous-instructions/NOP

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/idle-wfi.rs}}
```

```console
$ cargo xtask qemu --verbose --example idle-wfi
```

```console
{{#include ../../../../ci/expected/lm3s6965/idle-wfi.run}}
```

> **注意**: `idle` 任务不能与运行在优先级零的 _software_ 任务一起使用. 原因是 `idle` 以永不返回的 Rust 函数运行在优先级零, 那么优先级零的执行器就无法将控制权交给同优先级的 _software_ 任务.