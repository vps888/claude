# Claude Code 怎么用才顺手：别只把它当聊天工具

[![Status](https://img.shields.io/badge/Status-Active-brightgreen)]()
[![Updated](https://img.shields.io/badge/Updated-Jun_2026-blue)]()

> 这篇文章的完整版本已经整理到折腾侠主站。GitHub 这里保留摘要和原文入口，避免多处全文重复导致搜索引擎分散权重。

## 文章摘要

Claude Code 不能只当成一个更会写代码的聊天工具。它真正好用的地方，是能读仓库、改文件、跑命令、调用工具、分派子任务，并且在同一个项目里持续遵守你的工程约束。

这篇文章主要总结我高频使用 Claude Code 之后的一套工程化用法：怎么管理上下文，`CLAUDE.md` 应该写什么，Skills 和 Hooks 什么时候用，Subagents 怎么隔离复杂任务，以及最后如何用测试、构建、截图、日志来证明它真的做对了。

## 主要内容

- 为什么 Claude Code 更适合当成一个小型工程系统，而不是普通聊天框。
- 怎么把 Claude Code 拆成上下文、工具、工作流、硬约束、隔离执行、验证闭环六层。
- `CLAUDE.md` 怎么写才有用，哪些内容不应该塞进去。
- Skills、Hooks、Subagents 分别适合解决什么问题。
- 为什么复杂任务要先规划，再动手改代码。
- 怎么避免 Claude Code “看起来完成了，其实没有验证”的问题。

## 适合谁看

- 已经在用 Claude Code，但经常感觉上下文混乱的人。
- 想把 AI Coding 从临时问答升级成稳定工程流程的人。
- 需要让 Claude Code 长期参与真实项目开发、测试、提交和发布的人。
- 想设计 `CLAUDE.md`、Skills、Hooks、Subagents 使用规范的人。

## 原文链接

完整文章看这里：

https://zhetengxia.com/blog/claude-code-best-practices

如果你还在准备 Claude 账号、订阅和网络环境，也可以先看这篇：

https://zhetengxia.com/blog/claude-subscription-anti-ban-guide
