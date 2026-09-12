# 资源使用

RTIC 框架管理着共享资源和任务本地资源, 允许持久化数据存储并在不使用 `unsafe` 代码的情况下进行安全访问.

RTIC 资源仅对 `#[app]` 模块内声明的函数可见, 框架让用户可以完全控制 (按任务粒度) 资源的可访问性.

系统级资源的声明是通过在 `#[app]` 模块内给 **两个** `struct` 添加 `#[local]` 和 `#[shared]` 属性完成. 这些结构体中的每个字段对应一个不同的资源 (通过字段名标识). 这两组资源之间的区别将在下面介绍.

每个任务必须在其元数据属性中使用 `local` 和 `shared` 参数声明它打算访问的资源. 每个参数接受一个资源标识符列表. 列出的资源会通过 `Context` 结构体的 `local` 和 `shared` 字段提供给上下文.

`init` 任务返回系统级 (`#[shared]` 和 `#[local]`) 资源的初始值.

<!-- 以及应用使用的一组初始化定时器. 单调定时器将在 [Monotonic & `spawn_{at/after}`](./monotonic.md) 中进一步讨论. -->

## `#[local]` 资源

`#[local]` 资源对特定任务是本地可访问的, 意味着只有该任务能够访问该资源, 并且不需要锁或临界区. 这允许在 `#[init]` 中初始化的资源 (通常是驱动或大型对象) 之后被传递给某个特定任务.

因此, 任务的 `#[local]` 资源只能被唯一的一个任务访问. 尝试将同一个 `#[local]` 资源分配给多个任务会导致编译错误.

`#[local]` 资源的类型必须实现 [`Send`] trait, 因为它们要从 `init` 发送到目标任务, 跨越线程边界.

[`Send`]: https://doc.rust-lang.org/stable/core/marker/trait.Send.html

下方示例应用包含三个任务 `foo`, `bar` 和 `idle`, 每个任务都可以访问它自己的 `#[local]` 资源.

```rust,noplayground
{{#include ../../examples/lm3s6965/examples/locals.rs}}
```

运行该示例:

```console
$ cargo xtask qemu --verbose --example locals
```

```console
{{#include ../../ci/expected/lm3s6965/locals.run}}
```

`#[init]` 和 `#[idle]` 中的本地资源具有 `'static` 生命周期. 这一点是安全的, 因为这两个任务都是不可重入的.

### 任务本地初始化资源

本地资源也可以直接在资源声明中指定, 就像这样: `#[task(local = [my_var: TYPE = INITIAL_VALUE, ...])]`; 这允许创建不需要在 `#[init]` 中初始化的本地资源.

`#[task(local = [..])]` 资源的类型既不需要是 [`Send`], 也不需要是 [`Sync`], 因为它们不会跨越任何线程边界.

[`Sync`]: https://doc.rust-lang.org/stable/core/marker/trait.Sync.html

下面的示例展示了不同的用法和生命周期:

```rust,noplayground
{{#include ../../examples/lm3s6965/examples/declared_locals.rs}}
```

你可以运行该应用, 但由于该示例只是为了演示生命周期特性, 不会有任何输出 (编译过即可).

```console
$ cargo build --target thumbv7m-none-eabi --example declared_locals
```

<!-- {{#include ../../ci/expected/lm3s6965/declared_locals.run}} -->

## `#[shared]` 资源与 `lock`

访问 `#[shared]` 资源需要临界区, 以避免数据竞争. 为此, 所传入 `Context` 的 `shared` 字段为每个任务可访问的共享资源实现了 [`Mutex`] trait. 该 trait 只有一个方法 [`lock`], 它会在临界区中运行其闭包参数.

[`Mutex`]: ../../api/rtic/trait.Mutex.html
[`lock`]: ../../api/rtic/trait.Mutex.html#method.lock

`lock` API 所创建的临界区基于动态优先级: 它会将上下文的动态优先级临时提升到一个 _上限_ 优先级, 防止其他任务抢占该临界区. 这种同步协议被称为 [Immediate Ceiling Priority Protocol (ICPP)][icpp], 符合 RTIC 基于 [Stack Resource Policy (SRP)][srp] 的调度.

[icpp]: https://en.wikipedia.org/wiki/Priority_ceiling_protocol
[srp]: https://en.wikipedia.org/wiki/Stack_Resource_Policy

在下面的示例中, 我们有三个优先级从一到三的中断处理函数. 两个较低优先级的中断处理函数会争抢一个 `shared` 资源, 它们必须成功锁定资源才能访问其数据. 最高优先级的中断处理函数不访问 `shared` 资源, 因此它可以自由地抢占由最低优先级处理函数创建的临界区.

```rust,noplayground
{{#include ../../examples/lm3s6965/examples/lock.rs}}
```

```console
$ cargo xtask qemu --verbose --example lock
```

```console
{{#include ../../ci/expected/lm3s6965/lock.run}}
```

`#[shared]` 资源的类型必须是 [`Send`].

## 多重 lock

作为 `lock` 的扩展, 为了减少向右缩进, 锁可以以元组形式获取. 下面的示例展示了这种用法:

```rust,noplayground
{{#include ../../examples/lm3s6965/examples/multilock.rs}}
```

```console
$ cargo xtask qemu --verbose --example multilock
```

```console
{{#include ../../ci/expected/lm3s6965/multilock.run}}
```

## 仅共享 (`&-`) 访问

默认情况下, 框架假设所有任务都需要对资源进行独占可变访问 (`&mut-`), 但可以通过在 `shared` 列表中使用 `&resource_name` 语法来指定某个任务仅需要对资源进行共享访问 (`&-`).

指定对资源的共享访问 (`&-`) 的优点是, 即便资源被多个在不同优先级上运行的任务争用, 也不需要锁来访问资源. 缺点是任务只能获得对资源的共享引用 (`&-`), 从而限制了其能执行的操作, 但在共享引用足够用的情况下, 这种方式能减少所需的锁的数量. 除了简单的不可变数据之外, 当资源类型安全地实现了内部可变性 (使用合适的锁或原子操作) 时, 这种共享访问也很有用.

请注意, 在此版本的 RTIC 中, 不可能从不同任务同时请求对同一资源的独占访问 (`&mut-`) 和共享访问 (`&-`). 尝试这样做会导致编译错误.

在下面的示例中, 一个 key (例如加密密钥) 在运行时被加载 (或创建) (由 `init` 返回), 然后被两个在不同优先级上运行的任务使用, 不需要任何锁.

```rust,noplayground
{{#include ../../examples/lm3s6965/examples/only-shared-access.rs}}
```

```console
$ cargo xtask qemu --verbose --example only-shared-access
```

```console
{{#include ../../ci/expected/lm3s6965/only-shared-access.run}}
```

## 共享资源的无锁访问

对于只被 _同一_ 优先级上运行的任务访问的 `#[shared]` 资源, _不需要_ 临界区. 在这种情况下, 你可以通过在资源声明上添加 `#[lock_free]` 字段级属性来选择不使用 `lock` API (见下方示例).

<!-- 注意, 这只是为了减少不必要的资源锁定代码的便利手段, 因为即便使用了
`lock` API, 在运行时由于底层资源上限抢占的工作机制, 框架也**不会**产生临界区. -->

为了遵守 Rust 的 [别名][aliasing] 规则, 资源既可以通过多个不可变引用访问, 也可以通过单个可变引用访问 (但不能同时两者并存).

[aliasing]: https://doc.rust-lang.org/nomicon/aliasing.html

在由不同优先级运行的任务共享的资源上使用 `#[lock_free]` 将导致 _编译时_ 错误, 因为不使用 `lock` API 会违反上述别名规则. 同样, 对于每个优先级, 只能有单个 _software_ 任务访问某个共享资源 (因为 `async` 任务可能会让出执行权给同优先级的其他 _software_ 或 _hardware_ 任务). 然而, 在这个单任务限制下, 我们观察到该资源实际上不再是 `shared`, 而是 `local`. 因此, 对一个 `#[lock_free]` 共享资源使用 `#[lock_free]` 将导致 _编译时_ 错误, 在适用的情况下请改用 `#[local]` 资源.

```rust,noplayground
{{#include ../../examples/lm3s6965/examples/lock-free.rs}}
```

```console
$ cargo xtask qemu --verbose --example lock-free
```

```console
{{#include ../../ci/expected/lm3s6965/lock-free.run}}
```