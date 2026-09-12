# 不可重入

在 RTIC 中, 任务处理函数是*不可重入*的. 重入任务处理函数会破坏 Rust 的别名规则, 从而导致*未定义行为*. 任务处理函数可能通过以下两种方式之一发生重入: 软件方式或硬件方式.

## 软件方式

要通过软件方式重入某个任务处理函数, 必须通过 FFI 调用其底层的中断处理函数 ( 见下例 ). FFI 需要 `unsafe` 代码, 因此不鼓励最终用户直接调用中断处理函数.

``` rust,noplayground
#[rtic::app(device = ..)]
mod app {
    #[init]
    fn init(c: init::Context) { .. }

    #[interrupt(binds = UART0)]
    fn foo(c: foo::Context) {
        static mut X: u64 = 0;

        let x: &mut u64 = X;

        // ..

        //~ `bar` can preempt `foo` at this point

        // ..
    }

    #[interrupt(binds = UART1, priority = 2)]
    fn bar(c: foo::Context) {
        extern "C" {
            fn UART0();
        }

        // this interrupt handler will invoke task handler `foo` resulting
        // in aliasing of the static variable `X`
        unsafe { UART0() }
    }
}
```

RTIC 框架必须生成用于调用用户定义任务处理函数的中断处理代码. 我们会特别小心地使这些处理函数无法被用户代码直接调用.

上面的示例会被展开为:

``` rust,noplayground
fn foo(c: foo::Context) {
    // .. user code ..
}

fn bar(c: bar::Context) {
    // .. user code ..
}

mod app {
    // everything in this block is not visible to user code

    #[no_mangle]
    unsafe fn USART0() {
        foo(..);
    }

    #[no_mangle]
    unsafe fn USART1() {
        bar(..);
    }
}
```

## 硬件方式

任务处理函数也可以在没有软件介入的情况下被重入. 如果同一个处理函数被分配给向量表中的两个或多个中断, 就可能发生这种情况, 但 RTIC 框架的语法并不允许这种配置.
