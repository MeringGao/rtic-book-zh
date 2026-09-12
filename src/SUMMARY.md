# 目录

[前言](./preface.md)

---

- [开始新项目](./starting_a_project.md)
- [RTIC 示例入门](./by-example.md)
  - [`app`](./by-example/app.md)
  - [硬件任务](./by-example/hardware_tasks.md)
  - [软件任务与 `spawn`](./by-example/software_tasks.md)
  - [资源](./by-example/resources.md)
  - [init 任务](./by-example/app_init.md)
  - [idle 任务](./by-example/app_idle.md)
  - [基于 Channel 的通信](./by-example/channel.md)
  - [使用单调时钟实现延时与超时](./by-example/delay.md)
  - [最小 app](./by-example/app_minimal.md)
  - [技巧与窍门](./by-example/tips/index.md)
    - [解构资源](./by-example/tips/destructureing.md)
    - [消息传递时避免拷贝](./by-example/tips/indirection.md)
    - [`'static` 超能力](./by-example/tips/static_lifetimes.md)
    - [查看生成的代码](./by-example/tips/view_code.md)
- [单调时钟与 Timer Queue](./monotonic_impl.md)
- [RTIC vs 其他方案](./rtic_vs.md)
- [RTIC 与 Embassy](./rtic_and_embassy.md)
- [精选 RTIC 案例](./awesome_rtic.md)

---

- [从 v1.0.x 迁移到 v2.0.0](./migration_v1_v2.md)
  - [迁移到 `rtic-monotonics`](./migration_v1_v2/monotonics.md)
  - [软件任务现在必须使用 `async`](./migration_v1_v2/async_tasks.md)
  - [使用并理解 `rtic-sync`](./migration_v1_v2/rtic-sync.md)
  - [迁移代码示例](./migration_v1_v2/complete_example.md)

---

- [深入原理](./internals.md)
  - [目标架构](./internals/targets.md)
  <!--- [中断配置](./internals/interrupt-configuration.md)-->
  <!--- [非可重入性](./internals/non-reentrancy.md)-->
  <!--- [访问控制](./internals/access.md)-->
  <!--- [Late resources](./internals/late-resources.md)-->
  <!--- [临界区](./internals/critical-sections.md)-->
  <!--- [优先级天花板分析](./internals/ceilings.md)-->
  <!--- [软件任务](./internals/tasks.md)-->
  <!--- [Timer queue](./internals/timer-queue.md)-->

  <!-- - [定义任务](./by-example/app_task.md) -->
  <!-- - [软件任务与 `spawn`](./by-example/software_tasks.md)
    - [消息传递与 `capacity`](./by-example/message_passing.md)
    - [任务优先级](./by-example/app_priorities.md)
    - [单调时钟与 `spawn_{at/after}`](./by-example/monotonic.md)
  -->
