# 使用 `async` 软件任务

软件任务有一些变化, 概述如下.

### 软件任务现在必须是 `async` 的

所有软件任务现在都必须是 `async` 的.

#### 所需的改动

项目中所有未绑定到中断的任务现在都必须是 `async fn`. 例如:

``` rust,noplayground
#[task(
    local = [ some_resource ],
    shared = [ my_shared_resource ],
    priority = 2
)]
fn my_task(cx: my_task::Context) {
    cx.local.some_resource.do_trick();
    cx.shared.my_shared_resource.lock(|s| s.do_shared_thing());
}
```

变为

``` rust,noplayground
#[task(
    local = [ some_resource ],
    shared = [ my_shared_resource ],
    priority = 2
)]
async fn my_task(cx: my_task::Context) {
    cx.local.some_resource.do_trick();
    cx.shared.my_shared_resource.lock(|s| s.do_shared_thing());
}
```

## 软件任务现在可以永久运行

新的 `async` 软件任务允许永久运行, 但有一个前提条件: **任务的无限循环中必须存在一个 `await`**. 这种任务的示例如下:

``` rust,noplayground
#[task(local = [ my_channel ] )]
async fn my_task_that_runs_forever(cx: my_task_that_runs_forever::Context) {
    loop {
        let value = cx.local.my_channel.recv().await;
        do_something_with_value(value);
    }
}
```

## `spawn_after` 与 `spawn_at` 已被移除

正如 [迁移到 `rtic-monotonics`](./monotonics.md) 一节所述, `spawn_after` 与 `spawn_at` 已不再可用.
