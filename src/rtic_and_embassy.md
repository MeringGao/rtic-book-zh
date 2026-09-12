# RTIC vs. Embassy

## 差异

Embassy 同时提供硬件抽象层 (Hardware Abstraction Layer) 和 executor/runtime, 而 RTIC 旨在仅提供一个执行框架. 例如, embassy 提供 `embassy-stm32` (一个 HAL) 和 `embassy-executor` (一个 executor). 另一方面, RTIC 以 [`rtic`] 的形式提供框架, 用户负责提供 PAC 和 HAL 实现 (通常来自 [`stm32-rs`] 项目).

此外, RTIC 旨在尽可能在低层级提供对资源的独占访问, 理想情况下由某种形式的硬件保护来守护. 这允许访问硬件而不一定需要在软件层面使用锁定机制.

## 混合使用 Embassy 和 RTIC

由于大多数 Embassy 和 RTIC 库都是与运行时无关的, 一个项目中的许多细节可以在另一个项目中使用. 例如, 在由 `embassy-executor` 驱动的项目中使用 [`rtic-monotonics`] 是可行的, 而在 RTIC 项目中使用 [`embassy-sync`] (尽管推荐使用 [`rtic-sync`]) 也是可行的.

[`stm32-rs`]: https://github.com/stm32-rs
[`rtic`]: https://docs.rs/rtic/latest/rtic/
[`rtic-monotonics`]: https://docs.rs/rtic-monotonics/latest/rtic_monotonics/
[`embassy-sync`]: https://docs.rs/embassy-sync/latest/embassy_sync/
[`rtic-sync`]: https://docs.rs/rtic-sync/latest/rtic_sync/