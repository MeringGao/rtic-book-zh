# 创建一个新项目

从零开始一个 RTIC 项目时, 推荐遵循 RTIC 的 [`defmt-app-template`].

如果你面向 ARMv6-M 或 ARMv8-M-base 架构, 请查阅 [目标架构](./internals/targets.md) 一节, 了解需要注意的硬件限制.

[`defmt-app-template`]: https://github.com/rtic-rs/defmt-app-template

它会给你一个 RTIC 应用, 支持通过 [`defmt`] 进行 RTT 日志, 以及通过 [`flip-link`] 实现栈溢出保护. 社区也提供了大量示例供参考:

如果需要灵感, 你可以查看 [RTIC 示例].

## RISC-V 设备上的 RTIC

尽管 RTIC 最初是为 ARM Cortex-M 开发的, 但也可以在 RISC-V 设备上使用 RTIC. 不过, RISC-V 的生态更加异构. 为解决这一问题, RTIC 目前实现了三种不同的后端:

- **`riscv-esp32c3-backend`**: 该后端为 ESP32-C3 SoC 提供支持. 在这些设备上, RTIC 与其 Cortex-M 版本非常相似.

- **`riscv-esp32c6-backend`**: 该后端为 ESP32-C6 SoC 提供支持. 在这些设备上, RTIC 与其 Cortex-M 版本非常相似.

- **`riscv-mecall-backend`**: 该后端支持**任何** RISC-V 设备. 在该后端中, 挂起的任务会触发 Machine Environment Call 异常. 该异常源的处理程序根据优先级派发挂起的任务. 该后端的行为与 `riscv-clint-backend` 等价. 该后端的主要区别在于所有任务**必须**是 [软件任务](./by-example/software_tasks.md). 此外, 无需在 `#[app]` 属性中提供调度器列表, RTIC 会在编译时生成它们.

- **`riscv-clint-backend`**: 该后端支持带有 CLINT 外设的设备. 它与 `riscv-mecall-backend` 等价, 但不是通过触发异常, 而是通过 CLINT 的 `MSIP` 寄存器触发软件中断.

[`defmt`]: https://github.com/knurling-rs/defmt/
[`flip-link`]: https://github.com/knurling-rs/flip-link/
[RTIC examples]: https://github.com/rtic-rs/rtic/tree/master/examples
