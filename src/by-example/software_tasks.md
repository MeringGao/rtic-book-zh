# 软件任务与 spawn

RTIC 中的软件任务概念与 [硬件任务](./hardware_tasks.md) 有很多共同之处. 核心区别在于, 软件任务并不显式绑定到特定的中断向量, 而是绑定到一个以软件任务预期优先级运行的 "dispatcher" 中断向量 (见下文).

与 _hardware_ 任务类似, 函数上使用的 `#[task]` 属性将其声明为一个任务. 属性中缺少 `binds = InterruptName` 参数则将该函数声明为 _software task_.

静态方法 `task_name::spawn()` 派生 (启动) 一个软件任务, 假设没有更高优先级的任务正在运行, 该任务将立即开始执行.

_software_ 任务本身以 `async` Rust 函数的形式给出, 这允许用户可选地 `await` 未来的事件. 这使得反应式编程 (通过 _hardware_ 任务) 与顺序式编程 (通过 _software_ 任务) 能够混合在一起.

_software_ 任务假定运行到完成 (并返回), 而 _software_ 任务可以被启动一次 (`spawned`) 并永远运行, 条件是任何循环 (执行路径) 至少被一个 `await` (让出操作) 所打断.

## Dispatchers

同一优先级的所有 _software_ 任务共享一个中断处理函数, 该中断处理函数充当调度软件任务的异步执行器. 这个 dispatcher 列表 `dispatchers = [FreeInterrupt1, FreeInterrupt2, ...]` 是 `#[app]` 属性的一个参数, 你可以在其中定义一组空闲且可用的中断.

每个充当 dispatcher 的中断向量被分配一个优先级, 这意味着 dispatcher 列表需要覆盖软件任务使用的所有优先级.

示例: 对于一个使用三个不同优先级的软件任务的应用, `dispatchers =` 参数至少需要 3 项.

如果没有提供足够的 dispatcher, 或者 dispatcher 列表与 _hardware_ 任务绑定的中断发生冲突, 框架会报编译错误.

请看下面的示例:

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/spawn.rs}}
```

```console
$ cargo xtask qemu --verbose --example spawn
```

```console
{{#include ../../../../ci/expected/lm3s6965/spawn.run}}
```

只要 _software_ 任务已经运行到完成 (返回), 你就可以再次 `spawn` 它.

在下面的示例中, 我们从 `idle` 任务中 `spawn` _software_ 任务 `foo`. 由于 _software_ 任务的优先级是 1 (高于 `idle`), dispatcher 将执行 `foo` (抢占 `idle`). 因为 `foo` 会运行到完成, 所以再次 `spawn` `foo` 任务是允许的.

从技术上讲, 异步执行器会对 `foo` _future_ 进行 `poll`, 在这种情况下, _future_ 会停留在 _completed_ 状态.

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/spawn_loop.rs}}
```

```console
$ cargo xtask qemu --verbose --example spawn_loop
```

```console
{{#include ../../../../ci/expected/lm3s6965/spawn_loop.run}}
```

尝试对一个已经派生 (正在运行) 的任务再次 `spawn` 将导致错误. 注意, 错误是在 `foo` 任务实际运行之前报告的. 这是因为 _software_ 任务的实际执行由 dispatcher 中断 (`SSIO`) 处理, 而该中断直到我们退出 `init` 任务才被启用. (记住, `init` 在临界区运行, 即所有中断都被禁用.)

从技术上讲, 对一个不处于 _completed_ 状态的 _future_ 进行 `spawn` 被视为错误.

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/spawn_err.rs}}
```

```console
$ cargo xtask qemu --verbose --example spawn_err
```

```console
{{#include ../../../../ci/expected/lm3s6965/spawn_err.run}}
```

## 传递参数

你也可以在 spawn 时传递参数, 如下所示.

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/spawn_arguments.rs}}
```

```console
$ cargo xtask qemu --verbose --example spawn_arguments
```

```console
{{#include ../../../../ci/expected/lm3s6965/spawn_arguments.run}}
```

## 永不返回的任务

任务有两种签名之一: `async fn({name}::Context, ..)` 或 `async fn({name}::Context, ..) -> !`. 后者定义了一个 *永不返回 (divergent)* 的任务, 即永不返回的任务. 永不返回任务的关键优势是它们接收一个 `'static` 上下文, `local` 资源具有 `'static` 生命周期. 此外, 使用这种签名使得任务的意图变得明确, 清楚地区分了短期任务和无限期运行的任务. 注意不要让同一优先级的其他任务挨饿, 请确保通过 `.await` 让出控制权.

## 优先级为零的任务

在 RTIC 中, 任务彼此抢占式运行, 优先级零 (0) 是最低优先级. 你可以使用优先级为零的任务来处理后台工作, 而没有任何严格的实时性要求.

从概念上讲, 可以认为这样的任务运行在应用的 `main` 线程中, 因此关联的资源不需要 [Send] 约束.

[Send]: https://doc.rust-lang.org/nomicon/send-and-sync.html

```rust,noplayground
{{#include ../../../../examples/lm3s6965/examples/zero-prio-task.rs}}
```

```console
$ cargo xtask qemu --verbose --example zero-prio-task
```

```console
{{#include ../../../../ci/expected/lm3s6965/zero-prio-task.run}}
```

> **注意**: 优先级为零的 _software_ 任务不能与 [idle] 任务共存. 原因是 `idle` 以永不返回的 Rust 函数运行在优先级零, 那么优先级零的执行器就无法将控制权交给同优先级的 _software_ 任务.

---

应用侧安全性: 从技术上讲, RTIC 框架确保 `poll` 永远不会在任何 _software_ 任务的 _completed_ future 上执行, 从而遵循 Rust async 的 soundness 规则.