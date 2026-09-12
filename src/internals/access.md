# 访问控制

RTIC 的核心基础之一是访问控制. 控制程序的哪些部分能访问哪些静态变量, 对于强制保证内存安全至关重要.

静态变量用于在中断处理函数之间, 或者中断处理函数与最底层的执行上下文 `main` 之间共享状态. 在普通的 Rust 代码中, 很难对哪些函数可以访问静态变量进行细粒度的控制, 因为处于同一作用域内的任意函数都可以访问静态变量. 模块提供了一定程度的访问控制能力, 但灵活性仍不够.

为了实现这种细粒度的访问控制, 使得任务只能访问它们在 RTIC 属性中所指定的静态变量 ( 资源 ), RTIC 框架执行了一次源码层面的转换. 该转换将用户指定的资源 ( 静态变量 ) 放置在一个模块*内部*, 而将用户代码放在该模块*外部*. 这使得用户代码无法直接引用这些静态变量.

随后, 通过一个 `Resources` 结构体把对资源的访问交给每个任务. 该结构体的字段对应该任务有权访问的资源. 每个任务都有一个这样的 `Resources` 结构体, 并以对静态变量的唯一引用 (`&mut-`) 或资源代理 ( 参见 [临界区](critical-sections.md) 一节 ) 进行初始化.

下面的代码展示了这种源码级别转换的一个示例:

``` rust,noplayground
#[rtic::app(device = ..)]
mod app {
    static mut X: u64: 0;
    static mut Y: bool: 0;

    #[init(resources = [Y])]
    fn init(c: init::Context) {
        // .. user code ..
    }

    #[interrupt(binds = UART0, resources = [X])]
    fn foo(c: foo::Context) {
        // .. user code ..
    }

    #[interrupt(binds = UART1, resources = [X, Y])]
    fn bar(c: bar::Context) {
        // .. user code ..
    }

    // ..
}
```

框架生成的代码形如:

``` rust,noplayground
fn init(c: init::Context) {
    // .. user code ..
}

fn foo(c: foo::Context) {
    // .. user code ..
}

fn bar(c: bar::Context) {
    // .. user code ..
}

// Public API
pub mod init {
    pub struct Context<'a> {
        pub resources: Resources<'a>,
        // ..
    }

    pub struct Resources<'a> {
        pub Y: &'a mut bool,
    }
}

pub mod foo {
    pub struct Context<'a> {
        pub resources: Resources<'a>,
        // ..
    }

    pub struct Resources<'a> {
        pub X: &'a mut u64,
    }
}

pub mod bar {
    pub struct Context<'a> {
        pub resources: Resources<'a>,
        // ..
    }

    pub struct Resources<'a> {
        pub X: &'a mut u64,
        pub Y: &'a mut bool,
    }
}

/// Implementation details
mod app {
    // everything inside this module is hidden from user code

    static mut X: u64 = 0;
    static mut Y: bool = 0;

    // the real entry point of the program
    unsafe fn main() -> ! {
        interrupt::disable();

        // ..

        // call into user code; pass references to the static variables
        init(init::Context {
            resources: init::Resources {
                X: &mut X,
            },
            // ..
        });

        // ..

        interrupt::enable();

        // ..
    }

    // interrupt handler that `foo` binds to
    #[no_mangle]
    unsafe fn UART0() {
        // call into user code; pass references to the static variables
        foo(foo::Context {
            resources: foo::Resources {
                X: &mut X,
            },
            // ..
        });
    }

    // interrupt handler that `bar` binds to
    #[no_mangle]
    unsafe fn UART1() {
        // call into user code; pass references to the static variables
        bar(bar::Context {
            resources: bar::Resources {
                X: &mut X,
                Y: &mut Y,
            },
            // ..
        });
    }
}
```
