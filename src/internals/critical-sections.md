# 临界区

当一个资源 ( 静态变量 ) 被两个或更多运行在不同优先级的任务共享时, 需要某种形式的互斥来保证无数据竞争地修改该内存. 在 RTIC 中, 我们使用基于优先级的临界区来保证互斥 ( 参见 [优先级上限协议][icpp] ).

[icpp]: https://en.wikipedia.org/wiki/Priority_ceiling_protocol

临界区的工作原理是临时提升任务的*动态*优先级. 当一个任务处于该临界区中时, 所有其他可能请求该资源的任务都*不允许启动*.

对于某个特定资源, 动态优先级需要提高到多少才能保证互斥? [上限分析](ceilings.md) 负责回答这个问题, 将在下一节讨论. 本节将聚焦于临界区的实现.

## 资源代理

为简单起见, 我们看一个由两个运行在不同优先级的任务共享的资源. 显然其中一个任务可以抢占另一个; 为了避免数据竞争, *较低优先级*的任务在需要修改该共享内存时必须使用临界区. 另一方面, 较高优先级的任务可以直接修改该共享内存, 因为它不会被较低优先级的任务抢占. 为了强制让较低优先级任务使用临界区, 我们向它下发一个*资源代理*, 而向较高优先级任务下发一个唯一引用 (`&mut-`).

下面的示例展示了下发给每个任务的不同类型:

``` rust,noplayground
#[rtic::app(device = ..)]
mut app {
    struct Resources {
        #[init(0)]
        x: u64,
    }

    #[interrupt(binds = UART0, priority = 1, resources = [x])]
    fn foo(c: foo::Context) {
        // resource proxy
        let mut x: resources::x = c.resources.x;

        x.lock(|x: &mut u64| {
            // critical section
            *x += 1
        });
    }

    #[interrupt(binds = UART1, priority = 2, resources = [x])]
    fn bar(c: bar::Context) {
        let mut x: &mut u64 = c.resources.x;

        *x += 1;
    }

    // ..
}
```

接下来我们看看框架是如何创建这些类型的.

``` rust,noplayground
fn foo(c: foo::Context) {
    // .. user code ..
}

fn bar(c: bar::Context) {
    // .. user code ..
}

pub mod resources {
    pub struct x {
        // ..
    }
}

pub mod foo {
    pub struct Resources {
        pub x: resources::x,
    }

    pub struct Context {
        pub resources: Resources,
        // ..
    }
}

pub mod bar {
    pub struct Resources<'a> {
        pub x: &'a mut u64,
    }

    pub struct Context {
        pub resources: Resources,
        // ..
    }
}

mod app {
    static mut x: u64 = 0;

    impl rtic::Mutex for resources::x {
        type T = u64;

        fn lock<R>(&mut self, f: impl FnOnce(&mut u64) -> R) -> R {
            // we'll check this in detail later
        }
    }

    #[no_mangle]
    unsafe fn UART0() {
        foo(foo::Context {
            resources: foo::Resources {
                x: resources::x::new(/* .. */),
            },
            // ..
        })
    }

    #[no_mangle]
    unsafe fn UART1() {
        bar(bar::Context {
            resources: bar::Resources {
                x: &mut x,
            },
            // ..
        })
    }
}
```

## `lock`

现在我们聚焦于临界区本身. 在这个示例中, 我们需要将动态优先级至少提升到 `2` 才能避免数据竞争. 在 Cortex-M 架构上, 动态优先级可以通过写入 `BASEPRI` 寄存器来修改.

`BASEPRI` 寄存器的语义如下:

- 向 `BASEPRI` 写入 `0` 会禁用它的功能.
- 向 `BASEPRI` 写入非零值会改变中断抢占所需的优先级. 但只有当写入的值*低于*当前执行上下文的优先级时才会生效, 需要注意的是, 较低的硬件优先级数值表示较高的逻辑优先级.

因此任意时刻的动态优先级可以按如下公式计算

``` rust,noplayground
dynamic_priority = max(hw2logical(BASEPRI), hw2logical(static_priority))
```

其中 `static_priority` 是 NVIC 中为当前中断配置的优先级, 或者当当前上下文为 `idle` 时的逻辑 `0`.

在这个具体示例中, 我们可以这样实现临界区:

> **注意**: 这是一个简化版的实现

``` rust,noplayground
impl rtic::Mutex for resources::x {
    type T = u64;

    fn lock<R, F>(&mut self, f: F) -> R
    where
        F: FnOnce(&mut u64) -> R,
    {
        unsafe {
            // start of critical section: raise dynamic priority to `2`
            asm!("msr BASEPRI, 192" : : : "memory" : "volatile");

            // run user code within the critical section
            let r = f(&mut x);

            // end of critical section: restore dynamic priority to its static value (`1`)
            asm!("msr BASEPRI, 0" : : : "memory" : "volatile");

            r
        }
    }
}
```

这里需要注意的是在 `asm!` 块中使用 `"memory"` clobber. 它可以防止编译器将跨过它的内存操作进行重排序. 这一点很重要, 因为在临界区外访问变量 `x` 将导致数据竞争.

需要特别注意的是, `lock` 方法的签名本身就能阻止对它的嵌套调用. 这是内存安全所必需的, 因为嵌套调用会产生对 `x` 的多个唯一引用 (`&mut-`), 违反 Rust 的别名规则. 如下所示:

``` rust,noplayground
#[interrupt(binds = UART0, priority = 1, resources = [x])]
fn foo(c: foo::Context) {
    // resource proxy
    let mut res: resources::x = c.resources.x;

    res.lock(|x: &mut u64| {
        res.lock(|alias: &mut u64| {
            //~^ error: `res` has already been uniquely borrowed (`&mut-`)
            // ..
        });
    });
}
```

## 嵌套

对*同一*资源嵌套调用 `lock` 必须被编译器拒绝, 以保证内存安全; 但对*不同*资源嵌套调用 `lock` 是合法操作. 在这种情况下, 我们要确保嵌套的临界区绝不会降低动态优先级 ( 那样是不健全的 ), 同时也要尽量减少对 `BASEPRI` 寄存器的写入次数以及编译器屏障的次数. 为此, 我们使用一个栈变量来跟踪任务的动态优先级, 并据此决定是否真正写入 `BASEPRI`. 在实际中, 该栈变量会被编译器优化掉, 但它仍然为编译器提供了额外的信息.

考虑下面这段程序:

``` rust,noplayground
#[rtic::app(device = ..)]
mod app {
    struct Resources {
        #[init(0)]
        x: u64,
        #[init(0)]
        y: u64,
    }

    #[init]
    fn init() {
        rtic::pend(Interrupt::UART0);
    }

    #[interrupt(binds = UART0, priority = 1, resources = [x, y])]
    fn foo(c: foo::Context) {
        let mut x = c.resources.x;
        let mut y = c.resources.y;

        y.lock(|y| {
            *y += 1;

            *x.lock(|x| {
                x += 1;
            });

            *y += 1;
        });

        // mid-point

        x.lock(|x| {
            *x += 1;

            y.lock(|y| {
                *y += 1;
            });

            *x += 1;
        })
    }

    #[interrupt(binds = UART1, priority = 2, resources = [x])]
    fn bar(c: foo::Context) {
        // ..
    }

    #[interrupt(binds = UART2, priority = 3, resources = [y])]
    fn baz(c: foo::Context) {
        // ..
    }

    // ..
}
```

框架生成的代码形如:

``` rust,noplayground
// omitted: user code

pub mod resources {
    pub struct x<'a> {
        priority: &'a Cell<u8>,
    }

    impl<'a> x<'a> {
        pub unsafe fn new(priority: &'a Cell<u8>) -> Self {
            x { priority }
        }

        pub unsafe fn priority(&self) -> &Cell<u8> {
            self.priority
        }
    }

    // repeat for `y`
}

pub mod foo {
    pub struct Context {
        pub resources: Resources,
        // ..
    }

    pub struct Resources<'a> {
        pub x: resources::x<'a>,
        pub y: resources::y<'a>,
    }
}

mod app {
    use cortex_m::register::basepri;

    #[no_mangle]
    unsafe fn UART1() {
        // the static priority of this interrupt (as specified by the user)
        const PRIORITY: u8 = 2;

        // take a snashot of the BASEPRI
        let initial = basepri::read();

        let priority = Cell::new(PRIORITY);
        bar(bar::Context {
            resources: bar::Resources::new(&priority),
            // ..
        });

        // roll back the BASEPRI to the snapshot value we took before
        basepri::write(initial); // same as the `asm!` block we saw before
    }

    // similarly for `UART0` / `foo` and `UART2` / `baz`

    impl<'a> rtic::Mutex for resources::x<'a> {
        type T = u64;

        fn lock<R>(&mut self, f: impl FnOnce(&mut u64) -> R) -> R {
            unsafe {
                // the priority ceiling of this resource
                const CEILING: u8 = 2;

                let current = self.priority().get();
                if current < CEILING {
                    // raise dynamic priority
                    self.priority().set(CEILING);
                    basepri::write(logical2hw(CEILING));

                    let r = f(&mut y);

                    // restore dynamic priority
                    basepri::write(logical2hw(current));
                    self.priority().set(current);

                    r
                } else {
                    // dynamic priority is high enough
                    f(&mut y)
                }
            }
        }
    }

    // repeat for resource `y`
}
```

最终, 编译器会把函数 `foo` 优化成类似下面的样子:

``` rust,noplayground
fn foo(c: foo::Context) {
    // NOTE: BASEPRI contains the value `0` (its reset value) at this point

    // raise dynamic priority to `3`
    unsafe { basepri::write(160) }

    // the two operations on `y` are merged into one
    y += 2;

    // BASEPRI is not modified to access `x` because the dynamic priority is high enough
    x += 1;

    // lower (restore) the dynamic priority to `1`
    unsafe { basepri::write(224) }

    // mid-point

    // raise dynamic priority to `2`
    unsafe { basepri::write(192) }

    x += 1;

    // raise dynamic priority to `3`
    unsafe { basepri::write(160) }

    y += 1;

    // lower (restore) the dynamic priority to `2`
    unsafe { basepri::write(192) }

    // NOTE: it would be sound to merge this operation on `x` with the previous one but
    // compiler fences are coarse grained and prevent such optimization
    x += 1;

    // lower (restore) the dynamic priority to `1`
    unsafe { basepri::write(224) }

    // NOTE: BASEPRI contains the value `224` at this point
    // the UART0 handler will restore the value to `0` before returning
}
```

## BASEPRI 不变量

RTIC 框架需要维持的一个不变量是: 在一个*中断*处理函数开始时的 BASEPRI 值, 必须与该中断处理函数返回时的 BASEPRI 值相同. BASEPRI 可以在中断处理函数执行过程中发生变化, 但一个中断处理函数从开始到结束的整个执行过程, 不应造成 BASEPRI 的可观察变化.

必须维持该不变量, 以避免通过抢占抬升某个处理函数的动态优先级. 在下面的例子中可以最直观地观察到这一点:

``` rust,noplayground
#[rtic::app(device = ..)]
mod app {
    struct Resources {
        #[init(0)]
        x: u64,
    }

    #[init]
    fn init() {
        // `foo` will run right after `init` returns
        rtic::pend(Interrupt::UART0);
    }

    #[task(binds = UART0, priority = 1)]
    fn foo() {
        // BASEPRI is `0` at this point; the dynamic priority is currently `1`

        // `bar` will preempt `foo` at this point
        rtic::pend(Interrupt::UART1);

        // BASEPRI is `192` at this point (due to a bug); the dynamic priority is now `2`
        // this function returns to `idle`
    }

    #[task(binds = UART1, priority = 2, resources = [x])]
    fn bar() {
        // BASEPRI is `0` (dynamic priority = 2)

        x.lock(|x| {
            // BASEPRI is raised to `160` (dynamic priority = 3)

            // ..
        });

        // BASEPRI is restored to `192` (dynamic priority = 2)
    }

    #[idle]
    fn idle() -> ! {
        // BASEPRI is `192` (due to a bug); dynamic priority = 2

        // this has no effect due to the BASEPRI value
        // the task `foo` will never be executed again
        rtic::pend(Interrupt::UART0);

        loop {
            // ..
        }
    }

    #[task(binds = UART2, priority = 3, resources = [x])]
    fn baz() {
        // ..
    }

}
```

重要: 假设我们*忘记*在 `UART1` 中回滚 `BASEPRI` —— 这将是 RTIC 代码生成器的一个 bug.

``` rust,noplayground
// code generated by RTIC

mod app {
    // ..

    #[no_mangle]
    unsafe fn UART1() {
        // the static priority of this interrupt (as specified by the user)
        const PRIORITY: u8 = 2;

        // take a snashot of the BASEPRI
        let initial = basepri::read();

        let priority = Cell::new(PRIORITY);
        bar(bar::Context {
            resources: bar::Resources::new(&priority),
            // ..
        });

        // BUG: FORGOT to roll back the BASEPRI to the snapshot value we took before
        basepri::write(initial);
    }
}
```

其后果是 `idle` 将以 `2` 的动态优先级运行, 而事实上系统将再也不会以低于 `2` 的动态优先级运行. 这虽然不会损害程序的内存安全, 但会影响任务调度: 在这个特定情况下, 优先级为 `1` 的任务将永远得不到运行机会.
