# 任务优先级

## 优先级

`priority` 参数声明每个 `task` 的静态优先级.

对于 Cortex-M, 任务的优先级范围可以是 `0..=(1 << NVIC_PRIO_BITS)`, 其中 `NVIC_PRIO_BITS` 是 `device` crate 中定义的常量.

省略 `priority` 参数时, 任务优先级默认为 `0`. `idle` 任务具有不可配置的静态优先级 `0`, 即最低优先级.

> 在 RTIC 中, 数字越大表示优先级越高, 这与 Cortex-M 在 NVIC 外设中的表示相反.
> 也就是说, 数字 `10` 的优先级比数字 `9` **更高**.

当多个任务都准备好执行时, 具有最高静态优先级的任务优先获得执行权.

下面的场景演示了任务的优先级:
在执行低优先级任务 B 期间派生一个高优先级任务 A, 会挂起任务 B. 由于任务 A 具有更高的优先级, 它会抢占任务 B, 任务 B 被挂起直到任务 A 完成执行. 因此, 当任务 A 完成时, 任务 B 会恢复执行.

```text
Task Priority
  ┌────────────────────────────────────────────────────────┐
  │                                                        │
  │                                                        │
 3 │                      Preempts                          │
 2 │                    A─────────►                         │
 1 │          B─────────► - - - - B────────►                │
 0 │Idle┌─────►                   Resumes  ┌──────────►     │
  ├────┴──────────────────────────────────┴────────────────┤
  │                                                        │
  └────────────────────────────────────────────────────────┘Time
```

下面的示例展示了基于优先级的任务调度:

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/preempt.rs}}
```

```console
$ cargo xtask qemu --verbose --example preempt
{{#include ../../../../ci/expected/lm3s6965/preempt.run}}
```

注意, 任务 `bar` 并 _不会_ 抢占任务 `baz`, 因为它的优先级与 `baz` _相同_. `bar` 在 `baz` 返回后先于 `foo` 运行. 当 `bar` 返回时, `foo` 可以恢复执行.

再补充一点关于优先级的说明: 选择一个超过设备所支持的优先级会导致编译错误. 由于 Rust 语言的限制, 这个错误信息是晦涩难懂的, 例如如果 `example/common.rs` 中任务 `uart0_interrupt` 的 `priority = 9`, 错误如下:

由于 Rust 语言的限制, 这个错误信息是晦涩难懂的, 例如如果 `example/common.rs` 中任务 `uart0_interrupt` 的 `priority = 9`, 错误如下:

```text
   error[E0080]: evaluation of constant value failed
  --> examples/common.rs:10:1
   |
10 | #[rtic::app(device = lm3s6965, dispatchers = [SSI0, QEI0])]
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ attempt to compute `8_usize - 9_usize`, which would overflow
   |
   = note: this error originates in the attribute macro `rtic::app` (in Nightly builds, run with -Z macro-backtrace for more info)

```

错误消息错误地指向宏的起始位置, 但至少被减去的值 (这里是 9) 会提示是哪个任务导致了错误.