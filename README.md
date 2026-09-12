# RTIC 中文文档

RTIC ( Real-Time Interrupt-driven Concurrency ) 是一个用于嵌入式系统的 Rust 实时并发框架. 本仓库是 RTIC 官方文档的完整中文翻译.

## 在线阅读

📖 **[https://meringgao.github.io/rtic-book-zh/](https://meringgao.github.io/rtic-book-zh/)**

## 项目结构

```
.
├── src/                # mdbook 源文件 ( 中文翻译的 markdown )
│   ├── preface.md
│   ├── starting_a_project.md
│   ├── by-example.md
│   ├── by-example/      # 示例入门
│   ├── monotonic_impl.md
│   ├── rtic_vs.md
│   ├── rtic_and_embassy.md
│   ├── awesome_rtic.md
│   ├── migration_v1_v2.md
│   ├── migration_v1_v2/
│   ├── internals.md
│   └── internals/       # 内部实现细节
├── theme/               # mdbook 主题
├── book.toml           # mdbook 配置
├── mermaid.min.js      # Mermaid 渲染支持
├── mermaid-init.js
└── .github/workflows/  # GitHub Actions 自动部署
```

## 本地构建

需要安装:
- [mdbook](https://rust-lang.github.io/mdBook/) v0.4+
- [mdbook-mermaid](https://github.com/badboy/mdbook-mermaid)

```bash
# 安装
cargo install mdbook mdbook-mermaid

# 构建
mdbook build

# 本地预览 ( http://localhost:3000 )
mdbook serve
```

## 翻译来源

- 官方仓库: <https://github.com/rtic-rs/rtic>
- 官方文档: <https://rtic.rs>
- 翻译日期: 2026 年 9 月

## 翻译约定

- 代码块保留英文 ( 包括 Rust 代码和 Mermaid 图表 )
- URL 保持原样
- 专业术语统一 ( 硬件任务 / 软件任务 / 资源 / 派生 / 单调时钟 / 临界区 等 )
- 中英文之间有空格
- 标点使用英文 + 空格 (`. `, `, `, `: `, `? `)

## 贡献

欢迎提交 Issue 和 PR 来改进翻译质量.

## 许可

本文档翻译基于原仓库的许可协议发布.
