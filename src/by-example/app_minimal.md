# 最小应用

这是最小可能的 RTIC 应用:

```rust,noplayground
{{#include ../../examples/lm3s6965/examples/smallest.rs}}
```

RTIC 在设计时考虑了资源效率. RTIC 本身不依赖于任何动态内存分配, 因此 RAM 需求仅取决于应用. Flash 内存占用低于 1kB (包括中断向量表).

对于最小示例, 你可以预期类似如下:

```console
$ cargo xtask size --example smallest --backend thumbv7
```

```console
{{#include ../../ci/expected/lm3s6965/smallest.size}}
```

<!-- ---

从技术上讲, RTIC 会为每个 *software* 任务生成一个静态分配的 future (持有执行上下文, 包括 `Context` 结构体和栈上分配的变量). 同一静态优先级关联的 future 在执行期间共享一个异步栈.  -->