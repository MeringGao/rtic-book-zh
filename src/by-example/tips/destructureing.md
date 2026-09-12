# 资源的解构

如果一个任务需要使用多个资源, 解构任务资源可能有助于提高可读性. 下面是两种拆分资源结构体的示例:

```rust,noplayground
{{#include ../../../../../examples/lm3s6965/examples/destructure.rs}}
```

```console
$ cargo xtask qemu --verbose --example destructure
```

```console
{{#include ../../../../../ci/expected/lm3s6965/destructure.run}}
```