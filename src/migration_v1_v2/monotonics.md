# 迁移到 `rtic-monotonics`

在旧版本的 `rtic` 中, 单调时钟是 `#[rtic::app]` 紧密耦合的一部分. 在新版本中, [`rtic-monotonics`] 以更解耦的方式提供它们.

`#[monotonic]` 属性已不再使用. 改用来自 [`rtic-monotonics`] 的一个 `create_X_token`. 该宏的调用会返回一个中断注册 token, 可用于构造所需单调时钟的实例.

`spawn_after` 与 `spawn_at` 也已移除. 改为使用由 `rtic_time::Monotonic` trait 实现提供的异步函数 `delay` 与 `delay_until`, 这些可通过 [`rtic-monotonics`] 获得.

请查看 [代码示例](./complete_example.md) 了解所需的整体改动概览.

有关当前单调时钟实现的更多信息, 请参阅 [`rtic-monotonics` 文档](https://docs.rs/rtic-monotonics), 以及 [示例](https://github.com/rtic-rs/rtic/tree/master/examples).

[`rtic-monotonics`]: https://github.com/rtic-rs/rtic
