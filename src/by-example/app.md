# `#[app]` 属性与 RTIC 应用

## `app` 属性的要求

所有 RTIC 应用都使用 [`app`] 属性 (`#[app(..)]`). 该属性仅作用于包含 RTIC 应用的 `mod`-item.

`app` 属性有一个必填的 `device` 参数, 其值为一个 _path_. 这个 path 必须是一个指向 _peripheral access crate_ (PAC) 的完整路径, 该 PAC 由 [`svd2rust`] **v0.14.x** 或更新版本生成.

`app` 属性会展开为合适的入口点, 从而取代 [`cortex_m_rt::entry`] 属性的使用.

[`app`]: ../../api/rtic_macros/attr.app.html
[`svd2rust`]: https://crates.io/crates/svd2rust
[`cortex_m_rt::entry`]: https://docs.rs/cortex-m-rt-macros/latest/cortex_m_rt_macros/attr.entry.html

## 结构与零成本并发

RTIC `app` 是单核应用的可执行系统模型, 它声明了一组 `local` 和 `shared` 资源, 并由 `init`, `idle`, 以及 _hardware_ 和 _software_ 任务操作.

- `init` 在任何其他任务之前运行, 并返回 `local` 和 `shared` 资源.
- 任务 (无论是硬件任务还是软件任务) 根据其关联的静态优先级抢占式运行.
- 硬件任务绑定到底层的硬件中断.
- 软件任务由一组异步执行器调度, 每个软件任务优先级对应一个执行器.
- `idle` 具有最低优先级, 可用于后台工作, 以及/或让系统进入休眠直到被某个事件唤醒.

在编译时, 任务/资源模型会按照 Stack Resource Policy (SRP) 进行分析, 生成的执行代码具有以下突出特性:

- 在单一共享栈上保证无竞争的资源访问和无死锁的执行.
- 硬件任务的调度由硬件直接完成.
- 软件任务的调度由为该应用量身定制的自动生成的异步执行器完成.

总体而言, 生成的代码相对于手写实现不引入任何额外开销, 因此在 Rust 术语中 RTIC 提供了零成本的并发抽象.

## 优先级

RTIC 中的优先级通过传给 `#[task]` 属性的 `priority = N` 参数指定 (其中 N 是正数). 所有 `#[task]` 都可以有优先级. 如果任务的优先级未指定, 则设置为默认值 0.

RTIC 中的优先级遵循 _值越大越重要_ 的方案. 例如, 优先级为 2 的任务将抢占优先级为 1 的任务.

## RTIC 应用示例

为了让你对 RTIC 有个直观感受, 下面的示例包含了常用的特性.
在接下来的章节中, 我们将逐一详细介绍每个特性.

```rust,noplayground
{{#include ../../examples/lm3s6965/examples/common.rs}}
```