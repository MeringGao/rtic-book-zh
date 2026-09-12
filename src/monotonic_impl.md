# 单调时钟背后的原理

在内部, 所有的单调时钟 (Monotonic) 都使用一个 [Timer Queue](#timer-queue), 它是一个优先级队列, 其条目描述了各个 `Future` 应当完成的时刻.

## 为调度实现 `Monotonic` 计时器

[`rtic-time`] 框架非常灵活, 因为它可以使用任何支持比较匹配 (compare-match) 并可选地支持溢出中断的计时器来进行调度. 将计时器用于 RTIC 的唯一要求是实现 [`rtic-time::Monotonic`] trait.

对于 RTIC 2.0, 我们假设用户在实现 [`Monotonic`] 时使用一个时间库, 例如 [`fugit`], 作为所有基于时间的操作的基础. 这些库使得正确地实现 [`Monotonic`] trait 变得容易得多, 允许在系统中使用几乎任何计时器进行调度.

trait 文档化了每个方法的要求. 在 [`rtic-monotonics`] 中有参考实现可供借鉴.

- [`Systick based`], 以固定的 interrupt (core tick) 速率运行 —— 简单, 但有一些开销, 并且只能在 core 运行期间使用
- [`RP2040 Timer`], 一个 "规范" 的实现, 支持长时间等待而无需 interrupt. 清晰地演示了如何使用 [`TimerQueue`] 来处理调度.
- [`nRF52 timers`] 为 nRF52 的 RTC 和普通计时器实现了 monotonic 与 Timer Queue

通常, 硬件计数器只有 16 或 32 位的宽度, 这意味着它们会在几分钟到几年后溢出, 具体取决于计数器外设的频率. 为了解决这个问题, 这类计数器的单调时钟实现必须使用原子溢出计数器 (例如 [`portable_atomic::AtomicU32`]) 来正确计算当前时间瞬间. 这会导致溢出计数器和时间寄存器之间存在竞态条件, 这个问题并不容易解决. 因此, 必须使用半周期计数器 (half period counter). 请阅读并参考 [`rtic_time::half_period_counter`], 它提供了实现此逻辑的辅助方法和示例.

## 贡献

贡献新的 `Monotonic` 实现可以通过多种方式完成:
* 在 [`rtic-monotonics`] 中的一个 feature flag 后面实现该 trait, 然后创建一个 PR 将其纳入主 RTIC 仓库. 这样, 实现就在主仓库中, RTIC 可以保证它们的正确性, 并在新版本发布时更新它们.
* 在外部仓库中实现更改. 这样做不会将它们纳入 [`rtic-monotonics`], 但将来可能会更容易纳入.

[`rtic-monotonics`]: https://github.com/rtic-rs/rtic/tree/master/rtic-monotonics/
[`fugit`]: https://docs.rs/fugit/
[`Systick based`]: https://github.com/rtic-rs/rtic/blob/master/rtic-monotonics/src/systick.rs
[`rtic-monotonics`]:  https://github.com/rtic-rs/rtic/blob/master/rtic-monotonics
[`RP2040 Timer`]: https://github.com/rtic-rs/rtic/blob/master/rtic-monotonics/src/rp2040.rs
[`nRF52 timers`]: https://github.com/rtic-rs/rtic/blob/master/rtic-monotonics/src/nrf.rs
[`rtic-time`]: https://docs.rs/rtic-time/latest/rtic_time
[`rtic-time::Monotonic`]: https://docs.rs/rtic-time/latest/rtic_time/trait.Monotonic.html
[`Monotonic`]: https://docs.rs/rtic-time/latest/rtic_time/trait.Monotonic.html
[`TimerQueue`]: https://docs.rs/rtic-time/latest/rtic_time/timer_queue/struct.TimerQueue.html
[`rtic_time::half_period_counter`]: https://docs.rs/rtic-time/latest/rtic_time/half_period_counter/
[`portable_atomic::AtomicU32`]: https://docs.rs/portable-atomic/latest/portable_atomic/struct.AtomicU32.html

## Timer Queue

Timer Queue 实现为基于列表的优先级队列, 其中列表节点是在等待 monotonic 时 `await` 一个 Future 所创建的 `Future` 的静态分配部分. 因此, Timer Queue 在运行时是不可失败的 (其大小和分配在编译时确定).

与通道实现类似, Timer Queue 的实现依赖于一个全局的 *Critical Section* (CS) 来防止竞态. 在示例中, CS 实现由平台 crate 提供, 例如 `cortex-m/critical-section-single-core` 或 `esp32c3/critical-section`.