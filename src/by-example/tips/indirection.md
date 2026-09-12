# 使用间接层加速消息传递

消息传递总是涉及将 payload 从发送者拷贝到一个静态变量, 然后再从该静态变量拷贝到接收者. 因此, 发送像 `[u8; 128]` 这样的大缓冲区作为消息涉及两次开销很大的 `memcpy`.

间接层可以最小化消息传递的开销: 我们可以不按值发送缓冲区, 而是发送一个指向该缓冲区的拥有型指针.

可以使用全局内存分配器来实现间接层 (`alloc::Box`, `alloc::Rc` 等), 这需要使用 Rust v1.37.0 之后的 nightly 通道, 或者使用静态分配的内存池, 例如 [`heapless::Pool`].

[`heapless::Pool`]: https://docs.rs/heapless/latest/heapless/pool/index.html

由于这种方法的示例完全脱离了 RTIC 共享资源和本地资源的资源模型, 程序将依赖于内存分配器的正确性, 此处即 `heapless::pool`.

下面是一个使用 `heapless::Pool` 来 "装箱" 128 字节缓冲区的示例.

```rust,noplayground
{{#include ../../../examples/lm3s6965/examples/pool.rs}}
```

```console
$ cargo xtask qemu --verbose --example pool
```

```console
{{#include ../../../ci/expected/lm3s6965/pool.run}}
```