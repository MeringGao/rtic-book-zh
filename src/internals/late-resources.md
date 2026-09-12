# 后初始化资源

有些资源是在 `init` 函数返回之后的运行时才被初始化的. 重要的是, 这些资源 ( 静态变量 ) 必须在任务被允许运行之前完成完整的初始化, 也就是说, 它们的初始化必须发生在中断被屏蔽的状态下.

下面的示例展示了框架为初始化后初始化资源所生成的那种代码.

``` rust,noplayground
#[rtic::app(device = ..)]
mod app {
    struct Resources {
        x: Thing,
    }

    #[init]
    fn init() -> init::LateResources {
        // ..

        init::LateResources {
            x: Thing::new(..),
        }
    }

    #[task(binds = UART0, resources = [x])]
    fn foo(c: foo::Context) {
        let x: &mut Thing = c.resources.x;

        x.frob();

        // ..
    }

    // ..
}
```

框架生成的代码形如:

``` rust,noplayground
fn init(c: init::Context) -> init::LateResources {
    // .. user code ..
}

fn foo(c: foo::Context) {
    // .. user code ..
}

// Public API
pub mod init {
    pub struct LateResources {
        pub x: Thing,
    }

    // ..
}

pub mod foo {
    pub struct Resources<'a> {
        pub x: &'a mut Thing,
    }

    pub struct Context<'a> {
        pub resources: Resources<'a>,
        // ..
    }
}

/// Implementation details
mod app {
    // uninitialized static
    static mut x: MaybeUninit<Thing> = MaybeUninit::uninit();

    #[no_mangle]
    unsafe fn main() -> ! {
        cortex_m::interrupt::disable();

        // ..

        let late = init(..);

        // initialization of late resources
        x.as_mut_ptr().write(late.x);

        cortex_m::interrupt::enable(); //~ compiler fence

        // exceptions, interrupts and tasks can preempt `main` at this point

        idle(..)
    }

    #[no_mangle]
    unsafe fn UART0() {
        foo(foo::Context {
            resources: foo::Resources {
                // `x` has been initialized at this point
                x: &mut *x.as_mut_ptr(),
            },
            // ..
        })
    }
}
```

这里有一个重要的细节: `interrupt::enable` 的行为就像一个*编译器屏障*, 它可以防止编译器将对 `X` 的写入重排到 `interrupt::enable` 之后. 如果编译器真的做了这样的重排, 那么这次写入与 `foo` 在 `X` 上的任何操作之间就会产生数据竞争.

流水线结构更复杂的架构可能需要内存屏障 (`atomic::fence`), 而不仅仅是编译器屏障, 才能在中断重新使能之前完整地将这次写入刷出. ARM Cortex-M 架构在单核场景下不需要内存屏障.
