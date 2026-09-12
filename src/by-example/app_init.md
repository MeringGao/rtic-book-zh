# 应用初始化与 `#[init]` 任务

RTIC 应用需要一个 `init` 任务来设置系统. 对应的 `init` 函数必须具有签名 `fn(init::Context) -> (Shared, Local)`, 其中 `Shared` 和 `Local` 是用户定义的结构体.

`init` 任务在系统复位后执行, 在 [可选的 `pre-init` 代码段][^pre-init] 之后以及 RTIC 内部始终发生的初始化之后执行.

`init` 任务和可选的 `pre-init` 任务在 _中断禁用_ 的状态下运行, 并对 Cortex-M 拥有独占访问 (可以通过 `cs` 获得 `critical_section::CriticalSection` token).

设备特定的外设通过 `init::Context` 的 `core` 和 `device` 字段获得.

[^pre-init]: [https://docs.rs/cortex-m-rt/latest/cortex_m_rt/attr.pre_init.html](https://docs.rs/cortex-m-rt/latest/cortex_m_rt/attr.pre_init.html)

## 示例

下面的示例展示了 `core`, `device` 和 `cs` 字段的类型, 并演示了具有 `'static` 生命周期的 `local` 变量的使用. 这样的变量可以从 `init` 任务委托给 RTIC 应用的其他任务.

`device` 字段仅在 `peripherals` 参数设置为默认值 `true` 时可用.
在极少数情况下, 如果你想要实现一个超精简的应用, 你可以显式将 `peripherals` 设为 `false`.

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/init.rs}}
```

运行该示例将向控制台打印 `init`, 然后退出 QEMU 进程.

```console
$ cargo xtask qemu --verbose --example init
```

```console
{{#include ../../../../ci/expected/lm3s6965/init.run}}
```