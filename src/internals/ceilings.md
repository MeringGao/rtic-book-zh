# 上限分析

一个资源的*优先级上限*, 或简称*上限*, 是任何任务为安全访问该资源内存而必须具备的动态优先级. 上限分析相对简单, 但对 RTIC 应用的内存安全至关重要.

要计算一个资源的上限, 首先必须收集能够访问该资源的任务列表 —— 由于 RTIC 框架在编译期对资源实施访问控制, 它在编译期也拥有这些信息. 该资源的上限就是这些任务中最高的逻辑优先级.

`init` 和 `idle` 并不是严格意义上的任务, 但它们也可以访问资源, 因此需要纳入上限分析. `idle` 被视为一个逻辑优先级为 `0` 的任务, 而 `init` 则完全被排除在分析之外 —— 其原因在于 `init` 从不使用 ( 也不需要 ) 临界区来访问静态变量.

在上一节中我们展示了, 共享资源在任务面前可能呈现为唯一引用 (`&mut-`) 或一个代理, 具体取决于访问它的任务. 给任务下发哪种形式取决于任务优先级与该资源上限的关系: 如果任务优先级与资源上限相等, 则任务获得对资源内存的唯一引用 (`&mut-`); 否则任务获得一个代理 —— 这一规则同样适用于 `idle`. `init` 是特殊的: 它始终获得对资源的唯一引用 (`&mut-`).

下面用一个示例来说明上限分析:

``` rust,noplayground
#[rtic::app(device = ..)]
mod app {
    struct Resources {
        // accessed by `foo` (prio = 1) and `bar` (prio = 2)
        // -> CEILING = 2
        #[init(0)]
        x: u64,

        // accessed by `idle` (prio = 0)
        // -> CEILING = 0
        #[init(0)]
        y: u64,
    }

    #[init(resources = [x])]
    fn init(c: init::Context) {
        // unique reference because this is `init`
        let x: &mut u64 = c.resources.x;

        // unique reference because this is `init`
        let y: &mut u64 = c.resources.y;

        // ..
    }

    // PRIORITY = 0
    #[idle(resources = [y])]
    fn idle(c: idle::Context) -> ! {
        // unique reference because priority (0) == resource ceiling (0)
        let y: &'static mut u64 = c.resources.y;

        loop {
            // ..
        }
    }

    #[interrupt(binds = UART0, priority = 1, resources = [x])]
    fn foo(c: foo::Context) {
        // resource proxy because task priority (1) < resource ceiling (2)
        let x: resources::x = c.resources.x;

        // ..
    }

    #[interrupt(binds = UART1, priority = 2, resources = [x])]
    fn bar(c: foo::Context) {
        // unique reference because task priority (2) == resource ceiling (2)
        let x: &mut u64 = c.resources.x;

        // ..
    }

    // ..
}
```
