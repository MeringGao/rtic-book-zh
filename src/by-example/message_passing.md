# 消息传递与容量

软件任务支持消息传递, 这意味着软件任务可以带参数被派生:
`foo::spawn(1)` 将以参数 `1` 运行任务 `foo`.

容量设置任务的 spawn 队列大小. 如果未指定, 容量默认为 1.

在下面的示例中, 任务 `foo` 的容量为 `3`, 允许 `foo` 同时存在三个挂起的 spawn. 超出此容量将产生 `Error`.

任务的参数数量没有限制:

``` rust,noplayground
{{#include ../../examples/message_passing.rs}}
```

``` console
$ cargo xtask qemu --verbose --example message_passing
{{#include ../../ci/expected/message_passing.run}}
```