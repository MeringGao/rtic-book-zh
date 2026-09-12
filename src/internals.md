# 底层原理

**本章目前仍在完善中, 完整后会重新发布**

本节从*较高层面*介绍 RTIC 框架的内部实现. 过程宏 (`#[app]`) 进行的解析与代码生成等底层细节不在此处展开. 重点放在对用户规约的分析, 以及运行时所使用的数据结构上.

强烈建议你在深入这些内容之前, 先阅读 embedonomicon 中关于 [并发](https://github.com/rust-embedded/embedonomicon/pull/48) 的章节.

[concurrency]: https://github.com/rust-embedded/embedonomicon/pull/48
