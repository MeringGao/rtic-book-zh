# 检查生成的代码

`#[rtic::app]` 是一个过程宏, 用于生成支撑代码.
如果出于某种原因你需要检查该宏生成的代码, 你有两种选择:

* 你可以检查 `target` 目录下的 `rtic-expansion.rs` 文件.
* 使用 [`cargo-expand`] 子命令

## 使用生成的 `rtic-expansion.rs`

该文件的位置取决于构建方式.

在主 RTIC 仓库中使用 `cargo xtask build-example` 会根据使用的 "platform" 将文件放在不同位置:
```
$ cargo xtask example-build --example smallest
$ cargo xtask example-build --example monotonic --platform esp32-c3

$ fd -u rtic-expansion.rs
examples/esp32c3/target/rtic-expansion.rs
examples/lm3s6965/target/rtic-expansion.rs
```

在普通的 cargo 项目中, 它直接放在 `target` 文件夹下.

该文件包含 `#[rtic::app]` 项的展开结果 (不是你的整个程序!), 来自 _最后构建的_ (通过 `cargo build` 或 `cargo check`) RTIC 应用.
展开代码默认不格式化, 因此你需要在阅读前对它运行 `rustfmt`.

``` console
$ cargo build --example smallest --target thumbv7m-none-eabi
```

``` console
$ rustfmt target/rtic-expansion.rs
```

``` console
$ tail target/rtic-expansion.rs
```

``` rust,noplayground
#[doc = r" Implementation details"]
mod app {
    #[doc = r" Always include the device crate which contains the vector table"]
    use lm3s6965 as _;
    #[no_mangle]
    unsafe extern "C" fn main() -> ! {
        rtic::export::interrupt::disable();
        let mut core: rtic::export::Peripherals = core::mem::transmute(());
        core.SCB.scr.modify(|r| r | 1 << 1);
        rtic::export::interrupt::enable();
        loop {
            rtic::export::wfi()
        }
    }
}
```

## 使用 `cargo-expand` 工具

如果不可用, 请安装:

```
$ cargo install cargo-expand
```

这个子命令会展开 _所有_ 宏, 包括 `#[rtic::app]` 属性以及 crate 中的模块, 并将输出打印到控制台.


[`cargo-expand`]: https://crates.io/crates/cargo-expand

``` console
# produces the same output as before
```

``` console
cargo expand --example smallest | tail
```