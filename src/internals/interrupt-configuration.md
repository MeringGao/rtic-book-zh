# 中断配置

中断是 RTIC 应用运行的核心. 正确设置中断优先级并保证它们在运行时保持不变, 是应用内存安全的前提条件.

RTIC 框架将中断优先级作为编译期声明的内容暴露出来. 然而, 这种静态配置必须在应用初始化阶段被写入相应的寄存器. 中断的配置在 `init` 函数运行之前就已经完成.

下面的示例能够让你大致了解 RTIC 框架所运行的代码:

``` rust,noplayground
#[rtic::app(device = lm3s6965)]
mod app {
    #[init]
    fn init(c: init::Context) {
        // .. user code ..
    }

    #[idle]
    fn idle(c: idle::Context) -> ! {
        // .. user code ..
    }

    #[interrupt(binds = UART0, priority = 2)]
    fn foo(c: foo::Context) {
        // .. user code ..
    }
}
```

框架会生成一个形如下面的入口点:

``` rust,noplayground
// the real entry point of the program
#[no_mangle]
unsafe fn main() -> ! {
    // transforms a logical priority into a hardware / NVIC priority
    fn logical2hw(priority: u8) -> u8 {
        use lm3s6965::NVIC_PRIO_BITS;

        // the NVIC encodes priority in the higher bits of a bit
        // also a bigger numbers means lower priority
        ((1 << NVIC_PRIORITY_BITS) - priority) << (8 - NVIC_PRIO_BITS)
    }

    cortex_m::interrupt::disable();

    let mut core = cortex_m::Peripheral::steal();

    core.NVIC.enable(Interrupt::UART0);

    // value specified by the user
    let uart0_prio = 2;

    // check at compile time that the specified priority is within the supported range
    let _ = [(); (1 << NVIC_PRIORITY_BITS) - (uart0_prio as usize)];

    core.NVIC.set_priority(Interrupt::UART0, logical2hw(uart0_prio));

    // call into user code
    init(/* .. */);

    // ..

    cortex_m::interrupt::enable();

    // call into user code
    idle(/* .. */)
}
```
