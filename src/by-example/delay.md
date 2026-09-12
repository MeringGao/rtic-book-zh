# 使用 Monotonics 实现延时与超时

表达最小时间需求的一种便捷方式就是延迟推进.

这可以通过实例化一个单调定时器来实现 (实现见 [`rtic-monotonics`]).
Monotonics 可以看作时间记录者, 它们测量从单调时钟启动以来所经过的时间 (通常是应用启动的时刻). 在 RTIC 中, monotonics 可用于延迟代码执行以及时间测量.

RTIC 中所有单调时钟的实现都保证在硬件寿命之内保持稳定, 也就是说理论上你可以让一个任务延迟多年也能成功执行 (假设没有硬件故障). 单调时钟的分辨率 (即你可以睡眠的最小可能延时) 取决于具体实现. 许多单调时钟的实现还允许你通过为硬件定时器设置预分频器来控制分辨率. 这在 [各个 `rtic-monotonics` 的 crate 文档](https://docs.rs/rtic-monotonics/latest/rtic_monotonics/#modules) 中有记载.

[`rtic-monotonics`]: https://github.com/rtic-rs/rtic/tree/master/rtic-monotonics
[`rtic-time`]: https://github.com/rtic-rs/rtic/tree/master/rtic-time
[`Monotonic`]: https://docs.rs/rtic-time/latest/rtic_time/trait.Monotonic.html
[实现一个 `Monotonic`]: ../monotonic_impl.md

## Delay

```rust,noplayground
...
{{#include ../../examples/lm3s6965/examples/async-timeout.rs:init}}
        ...
```

_software_ 任务可以 `await` 延时到期:

```rust,noplayground
#[task]
async fn foo(_cx: foo::Context) {
    ...
    Mono::delay(100.millis()).await;
    ...
}

```

<details>
<summary>完整示例</summary>

```rust,noplayground
{{#include ../../examples/lm3s6965/examples/async-delay.rs}}
```

```console
$ cargo xtask qemu --verbose --example async-delay
```

```console
{{#include ../../ci/expected/lm3s6965/async-delay.run}}
```

</details>

> 有兴趣为 [`Monotonic`] 贡献新的实现, 或者想了解更多关于 monotonics 内部工作原理的信息?
> 请查阅 [实现一个 `Monotonic`] 这一章!

## Timeout

Rust 的 [`Future`] (底层是 Rust 的 `async`/`await`) 是可组合的. 这使得在已完成的 `Future` 之间进行 `select` 成为可能.

[`Future`]: https://doc.rust-lang.org/std/future/trait.Future.html

一个常见的用例是带有关联超时的事务. 在下面的示例中, 我们引入一个伪 HAL 设备, 当你调用 `hal_get(n).await` 时, 它会执行某种想象中的事务. 我们根据输入参数 (`n`) 将其耗时建模为 `350ms + n * 100ms`.

使用 `futures` crate 中的 `select_biased` 宏, 它可能看起来像这样:

```rust,noplayground,noplayground
{{#include ../../examples/lm3s6965/examples/async-timeout.rs:select_biased}}
```

假设 `hal_get` 将耗时 450ms 完成, 那么 200ms 的短超时将在 `hal_get` 完成之前到期.

将超时延长到 1000ms 则会让 `hal_get` 先完成.

`select_biased` 可以组合任意数量的 future, 因此非常强大. 然而, 由于超时模式经常被使用, RTIC 内置了更符合人体工学的支持, 由 [`rtic-monotonics`] 和 [`rtic-time`] crate 提供. 下面是另一个示例, 使用 `Mono::delay_until` 和 `Mono::timeout_after`:

```rust,noplayground
{{#include ../../examples/lm3s6965/examples/async-timeout.rs:timeout_at_basic}}
```

在需要对时间进行精确控制 (无漂移) 的情况下, 我们可以使用 `Instant` 来表示精确的时间点, 使用 `Duration` 来表示时间跨度. 对 `Instant` 和 `Duration` 类型的操作来自 [`fugit`] crate.

[`fugit`]: https://crates.io/crates/fugit

`let mut instant = Mono::now()` 设置执行的起始时间.

我们希望相对于这个起始时间每 1000ms 调用一次 `hal_get`. 我们通过将 `instant` 增加 1000ms, 然后调用 `Mono::delay_until(instant).await` 来实现. 当我们在这个循环周围迭代时, 任何额外的延时都会被补偿, 因为我们是延时到 'previous + 1000' 而不是 'now + 1000' (后者会导致循环计时漂移).

为了展示上面 `select!` 异步超时示例的另一种替代方案, 我们将一个未来的时间点定义为 `timeout`, 然后调用 `Mono::timeout_at(timeout, hal_get(n)).await`.

对于循环的第一次迭代, 当 `n == 0` 时, `hal_get` 将耗时 350ms (如上所述), 并在超时之前完成. 对于第二次迭代, 延时为 450ms, 仍然在超时之前完成. 对于第三次迭代, 当 `n == 2` 时, `hal_get` 将耗时 550ms 完成, 此时我们将遇到超时.

<details>
<summary>完整示例</summary>

```rust,noplayground
{{#include ../../examples/lm3s6965/examples/async-timeout.rs}}
```

```console
$ cargo xtask qemu --verbose --example async-timeout
```

```console
{{#include ../../ci/expected/lm3s6965/async-timeout.run}}
```

</details>