# 使用 `rtic-sync`

`rtic-sync` 提供了一组原语, 可用于在异步上下文里进行消息传递和资源共享.

其中重要的结构体有:

* `Arbiter`, 允许你在异步上下文中等待对共享资源的访问, 而无需使用 `lock`.
* `Channel`, 允许你在任务之间 ( 包括 `async` 任务和非 `async` 任务 ) 通信.

有关这些结构体的更多信息, 请参阅 [`rtic-sync` 文档](https://docs.rs/rtic-sync).
