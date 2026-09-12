# RTIC 示例入门

本书的这一部分通过由浅入深的示例向新用户介绍 RTIC 框架.

本书这一部分的所有示例都属于 [RTIC 仓库][repoexamples], 位于 `examples` 目录中. 这些示例可以在 QEMU 上运行 (模拟 cortex M3 目标), 因此无需特殊硬件即可跟随学习.

[repoexamples]: https://github.com/rtic-rs/rtic/tree/master/rtic/examples

## 运行示例

要使用 QEMU 运行这些示例, 你需要 `qemu-system-arm` 程序. 查看 [嵌入式 Rust 书籍][the embedded Rust book] 获取关于如何搭建包含 QEMU 的嵌入式开发环境的说明.

[the embedded Rust book]: https://rust-embedded.github.io/book/intro/install.html

要在本地使用 QEMU 运行 `examples/` 中的示例:

```
cargo xtask qemu
```

这会针对默认的 `thumbv7m-none-eabi` 设备 `lm3s6965` 运行所有示例.

要限制运行的示例, 可以使用 `--example <example name>` 标志, 名称是示例的文件名.

假设依赖已就绪, 运行:

```console
$ cargo xtask qemu --example locals
```

会产生如下输出:

```console
   Finished dev [unoptimized + debuginfo] target(s) in 0.07s
    Running `target/debug/xtask qemu --example locals`
INFO  xtask::run > QEMU run for platform: Lm3s6965, backend: Thumbv7
INFO  xtask::run > 👟 Build example locals (thumbv7m-none-eabi, release, "thumbv7-backend", in examples/lm3s6965)
INFO  xtask::run > ✅ Success.
INFO  xtask::run > 👟 Run example locals in QEMU (thumbv7m-none-eabi, release, "thumbv7-backend", in examples/lm3s6965)
INFO  xtask::run > ✅ Success.
INFO  xtask::results > ✅ Success: Build example locals (thumbv7m-none-eabi, release, "thumbv7-backend", in examples/lm3s6965)
INFO  xtask::results > ✅ Success: Run example locals in QEMU (thumbv7m-none-eabi, release, "thumbv7-backend", in examples/lm3s6965)
INFO  xtask::results > 🚀🚀🚀 All tasks succeeded 🚀🚀🚀
```

示例通过固然很好, 这也是 RTIC CI 配置的一部分, 但出于本书的目的, 我们必须添加 `--verbose` 标志 (简写 `-v`) 才能看到实际的程序输出:

```console
❯ cargo xtask qemu --verbose --example locals
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.04s
     Running `target/debug/xtask qemu --verbose --example locals`
 DEBUG xtask > Stderr of child processes is inherited: false
 DEBUG xtask > Partial features: false
 INFO  xtask::run > QEMU run for platform: Lm3s6965, backend: Thumbv7
 INFO  xtask::run > 👟 Build example locals (thumbv7m-none-eabi, release, "thumbv7-backend", in examples/lm3s6965)
 INFO  xtask::run > ✅ Success.
 INFO  xtask::run > 👟 Run example locals in QEMU (thumbv7m-none-eabi, release, "thumbv7-backend", in examples/lm3s6965)
 INFO  xtask::run > ✅ Success.
 INFO  xtask::results > ✅ Success: Build example locals (thumbv7m-none-eabi, release, "thumbv7-backend", in examples/lm3s6965)
    cd examples/lm3s6965 && cargo build --target thumbv7m-none-eabi --features thumbv7-backend --release --example locals
 DEBUG xtask::results >
cd examples/lm3s6965 && cargo build --target thumbv7m-none-eabi --features thumbv7-backend --release --example locals
Stderr:
    Finished `release` profile [optimized] target(s) in 0.03s
 INFO  xtask::results > ✅ Success: Run example locals in QEMU (thumbv7m-none-eabi, release, "thumbv7-backend", in examples/lm3s6965)
    cd examples/lm3s6965 && cargo run --target thumbv7m-none-eabi --features thumbv7-backend --release --example locals
 DEBUG xtask::results >
cd examples/lm3s6965 && cargo run --target thumbv7m-none-eabi --features thumbv7-backend --release --example locals
Stdout:
bar: local_to_bar = 1
foo: local_to_foo = 1
idle: local_to_idle = 1

Stderr:
    Finished `release` profile [optimized] target(s) in 0.03s
     Running `qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb -nographic -semihosting-config enable=on,target=native -kernel target/thumbv7m-none-eabi/release/examples/locals`
Timer with period zero, disable

 INFO  xtask::results > 🚀🚀🚀 All tasks succeeded 🚀🚀🚀
```

注意输出末尾 `Stdout:` 后面的内容, 程序输出应该包含这些行:

```console
{{#include ../../../ci/expected/lm3s6965/locals.run}}
```

> **注意**:
> 关于 `cargo xtask` 的其他有用选项, 请参阅:
> ```
> cargo xtask qemu --help
> ```
>
> `--platform` 标志允许更改运行示例的设备, 目前 `lm3s6965` 是支持最好的设备, 正在持续增加对其他设备的支持, 包括 ARM 和 RISC-V
