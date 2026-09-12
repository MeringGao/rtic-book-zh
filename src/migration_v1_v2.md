# 从 v1.0.x 迁移到 v2.0.0

将项目从 RTIC `v1.0.x` 迁移到 `v2.0.0` 需要完成以下步骤:

1. `v2.1.0` 可在 Rust 1.75 稳定版上运行 ( **推荐** ), 而更早的版本需要通过 [`#![type_alias_impl_trait]`](https://github.com/rust-lang/rust/issues/63063) 使用 `nightly` 编译器.
2. 从 `v1.0.x` 内置的单调时钟迁移到 `rtic-time` 与 `rtic-monotonics`, 并替换 `spawn_after`、`spawn_at`.
3. 软件任务现在必须是 `async` 的, 并需要正确使用它们.
4. 理解并使用 `rtic-sync` 提供的数据类型.

有关改动的详细说明, 请参考各子章节.

如果你想查看所需改动的代码示例, 可以查阅 [完整的迁移示例页面](./migration_v1_v2/complete_example.md).

#### TL;DR (Too Long; Didn't Read)

1. 用由 `rtic-monotonics` 提供的 `async` 函数 `delay`、`delay_until` ( 以及相关函数 ) 取代 `spawn_after` 与 `spawn_at`.
2. 软件任务现在_必须_是 `async fn`. 只要任务中存在 `await`, 就允许永不返回. 你仍然可以对共享资源使用 `lock`.
3. 使用 `rtic_sync::arbiter::Arbiter` 来 `await` 对共享资源的访问, 使用 `rtic_sync::channel::Channel` 在任务之间通信, 取代 `spawn` 新任务.
