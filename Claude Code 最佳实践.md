# Claude Code 怎么用才顺手：别只把它当聊天工具

[![Status](https://img.shields.io/badge/Status-Active-brightgreen)]()
[![Updated](https://img.shields.io/badge/Updated-May_2026-blue)]()

> 可以参考我的 [2026 最新 Claude 订阅防封全攻略：小白也能搞定的低成本防封实操](https://zhetengxia.com/blog/claude-subscription-anti-ban-guide) 这篇文章稳定订阅Claude Code。

自去年开始接触 Claude Code 之后，基本上就离不开了，一致在高强度的使用。今天闲下来总结一下我的一些感受。然后让 Claude Code 再给我润色一下。
Claude Code 不能只当成一个更会写代码的工具，它真正好用的地方是能读仓库、改文件、跑命令、调用工具、分派子任务，并且在同一个项目里持续遵守你的工程约束。问题也在这里：能力一多，如果没有上下文治理、权限边界和验证闭环，很快就会变成“看起来很努力，实际上到处跑偏”。

我的理解很简单：Claude Code 想用稳，核心不是多写几句神奇 prompt，而是把它当成一个工程系统来管理。

## 1. 先把 Claude Code 拆成六层

日常使用中最容易混在一起的是：规则、工具、自动化、子代理和验证。拆开之后就清楚很多。

![Claude Code 工程化六层框架](./claude-code-best-practices-diagram.jpg)

| 层         | 主要载体                          | 解决什么问题                                            | 常见误用                                     |
| ---------- | --------------------------------- | ------------------------------------------------------- | -------------------------------------------- |
| 上下文层   | `CLAUDE.md` / rules / memory      | 告诉 Claude 每次进项目都必须知道什么                    | 写成团队 wiki，关键约束被背景噪声淹没        |
| 动作能力层 | Tools / MCP                       | 让 Claude 访问 GitHub、数据库、Sentry、浏览器等外部系统 | 工具接太多、命名太泛，模型选错工具           |
| 工作流层   | Skills                            | 按需加载一套明确方法：什么时候用、怎么做、怎么停        | 一个 Skill 覆盖 review、debug、deploy 多件事 |
| 硬约束层   | Hooks / permissions / sandbox     | 把不能靠 Claude 自觉执行的事自动化或阻断                | 用 Hook 做复杂业务推理，输出又长又吵         |
| 隔离执行层 | Subagents / worktrees             | 把搜索、审查、长日志分析放到独立上下文                  | 权限和主线程一样宽，隔离失去意义             |
| 验证闭环层 | tests / lint / screenshots / logs | 用证据证明 Claude 做对了                                | 只听 Claude 说“完成了”，没有验证命令         |

这六层里，任何一层过度膨胀都会出问题：CLAUDE.md 写成百科全书会污染上下文；MCP 接太多会吃掉大量固定 token；Subagent 没边界会让状态漂移；验证缺失会让你不知道结果到底对不对。

## 2. 上下文管理比 prompt 更重要

很多 Claude Code 的翻车，不是模型不聪明，而是上下文太杂乱。

上下文里常驻的东西越多，留给真正代码和对话历史的空间就越少。系统指令、工具定义、MCP schema、CLAUDE.md、记忆、历史对话、文件内容、命令输出都在抢同一块窗口。尤其是 MCP 工具定义和命令输出，很容易被忽略。

我的建议是把上下文按“加载频率”分层：

| 信息类型                           | 放在哪里         | 加载方式   | 判断标准                   |
| ---------------------------------- | ---------------- | ---------- | -------------------------- |
| 每次会话都必须遵守的项目契约       | `CLAUDE.md`      | 常驻       | 少了它就会反复犯错         |
| 某个目录、语言或文件类型的局部规则 | `.claude/rules/` | 按路径加载 | 只对一部分代码成立         |
| 低频但步骤固定的工作流             | Skills           | 按需加载   | 需要方法论，不需要常驻     |
| 大范围探索、长日志、并行审查       | Subagents        | 隔离加载   | 会产生大量中间输出         |
| 确定性检查和阻断                   | Hooks            | 不进主对话 | 不需要模型推理，只需要执行 |

`CLAUDE.md` 最忌讳写成团队 wiki。背景故事、完整 API 文档、长篇架构介绍、口号式原则都不适合放进去。Claude 可以自己读仓库推断的东西，也不要重复写。

真正值得写的是：

1. 怎么安装、怎么启动、怎么测试。
2. 哪些目录承担哪些职责。
3. 哪些文件或命令不能碰。
4. 提交前必须跑什么验证。
5. 上下文压缩后必须保留什么信息。

我会在 `CLAUDE.md` 里专门留一段压缩规则，比如要求保留架构决策、修改过的文件、当前验证状态、未解决风险和回滚线索。长会话压缩后最怕丢的不是命令输出，而是“为什么这么改”。

## 3. CLAUDE.md 要像合同，不要像说明书

一个好用的 `CLAUDE.md` 大概长这样：

```markdown
# Project Contract

## Build And Test
- Install: `npm install`
- Dev: `npm run dev`
- Test: `npm test`
- Build: `npm run build`

## Architecture Boundaries
- Content posts live in `src/content/posts/`
- Site identity lives in `src/site.config.ts`
- Do not put article-specific promotion links in global navigation

## Safety Rails
## NEVER
- Do not rewrite unrelated dirty files
- Do not deploy production without explicit approval
- Do not claim success without running verification

## ALWAYS
- Inspect `git status` before committing
- Keep article frontmatter valid
- Run `npm test` and `npm run build` before publishing

## Compact Instructions
Preserve:
1. User-approved scope and wording
2. Modified files and why they changed
3. Verification commands and pass/fail status
4. Remaining risks or rollback notes
```

这类内容越具体越好。比如“写高质量代码”没有用，“修改 API 时同步更新 contract test”才有用。

还有一个很实用的习惯：每次 Claude 犯了可以沉淀的错误，就让它更新自己的 `CLAUDE.md`。例如它忘了跑测试，就直接说：“把这条写进 CLAUDE.md，之后提交前必须先跑测试。”但这类规则也要定期清理，否则半年后会堆成新的噪声。

## 4. Skills 不是模板库，而是工作流

Skill 的价值在于“按需加载”。它的描述符会常驻上下文，所以描述要短，但要足够清楚地告诉 Claude 什么时候该用。

坏描述：

```yaml
description: help with backend
```

好描述：

```yaml
description: Use when reviewing API changes for compatibility, tests, and rollback risk.
```

一个稳定的 Skill 至少要写清楚四件事：

1. 触发条件：什么时候必须使用。
2. 输入范围：需要读哪些文件或拿到哪些证据。
3. 执行步骤：先查什么，再判断什么，最后输出什么。
4. 停止条件：遇到什么风险必须停下来问人。

不要把一个 Skill 写成 review、debug、deploy、incident、docs 全包的大杂烩。一个 Skill 只解决一类问题，效果会稳定很多。

如果 Skill 有明显副作用，比如迁移配置、发布版本、清理数据，就不要让模型自动决定是否执行。它必须被显式触发，并且要写好 dry-run、备份和回滚步骤。

## 5. 工具设计要让 Agent 少选错

给人用的 API 和给 Agent 用的工具不是一回事。人会读文档、会猜上下文、会主动问；Agent 更依赖工具名、参数名和返回结构。

好工具通常有几个特点：

1. 名称明确，比如 `github_pr_get`、`sentry_errors_search`，不要叫 `query`、`fetch`、`do_action`。
2. 参数语义明确，比如 `issue_key`、`project_id`、`response_format`，不要只给 `id`、`name`、`target`。
3. 默认返回精简信息，能支持 `concise` 和 `detailed` 两种格式。
4. 错误信息要告诉 Claude 下一步怎么修，而不是只返回一个 opaque code。
5. 高层任务尽量封装成一个工具，不要暴露一堆底层碎片让模型自己拼。

如果一件事本地 shell 可以稳定完成，就不一定要做成 MCP 工具。如果只是静态知识，也更适合写进 Skill 或文档。工具真正适合的是“需要和外部系统发生交互，并且交互结果会影响下一步决策”的动作。

## 6. Hooks 负责硬约束，不负责复杂推理

Hooks 最适合处理那些“必须执行，而且不需要 Claude 判断”的事情。

比如：

1. 编辑某类文件后自动格式化。
2. 修改关键文件前先阻断或提醒。
3. 每次任务完成后发通知。
4. 会话开始时注入当前分支、环境名、运行目录。
5. 对轻量检查输出做截断，避免污染上下文。

不适合放到 Hooks 的是复杂业务判断、长时间工作流、多步权衡和需要大量上下文的语义审查。这些应该交给 Skill 或 Subagent。

Hooks 的一个细节很重要：输出必须短。比如测试失败只保留关键失败信息，不要把几千行日志全部塞回 Claude 的上下文。Claude 需要的是“哪里挂了、下一步查什么”，不是完整噪声。

## 7. Subagents 的核心价值是隔离，不只是并行

很多人把 Subagent 当并行工具，其实它更重要的价值是隔离上下文。

适合派出去的任务：

1. 全仓库搜索某个行为的调用链。
2. 只读审查一组文件。
3. 分析一段很长的日志。
4. 比较两种实现方案。
5. 跑一组不会改变状态的验证。

不适合派出去的任务：

1. 子任务之间强依赖，需要频繁共享中间状态。
2. 权限边界说不清楚。
3. 输出格式不固定，主会话拿回来没法继续用。
4. 文件系统没有隔离，却让多个 agent 同时改同一块代码。

给 Subagent 的指令要明确：它能用哪些工具，不能用哪些工具，最多跑几轮，最后按什么格式汇报。需要改文件时，最好配合 git worktree 隔离。

## 8. Plan Mode 的意义是先读懂，再动手

复杂任务不要一上来就让 Claude 写代码。先让它进入只读探索，明确目标、边界、影响范围、验证方式，再开始实现。

适合先规划的场景：

1. 跨模块重构。
2. 数据迁移。
3. 发布流程调整。
4. 涉及生产环境的任务。
5. 需求本身还不够清楚的功能。

规划不是为了形式感，而是为了降低错误假设的成本。一个错误假设如果在读代码阶段发现，只浪费几分钟；如果写完、测完、发完才发现，代价就高多了。

我自己比较喜欢让一个 agent 写计划，再让另一个 agent 站在 reviewer 角度挑问题。AI 审 AI 不是万能，但对发现范围遗漏、验证缺口、危险默认值很有帮助。

## 9. 验证闭环才是 Agent 工程化的分水岭

Claude 说“我完成了”没有工程意义。工程上真正可用的完成，必须能被验证。

验证可以分层：

| 验证层级 | 典型手段                                         | 适合证明什么                   | 不足                   |
| -------- | ------------------------------------------------ | ------------------------------ | ---------------------- |
| 基础验证 | 命令退出码、lint、typecheck、unit test           | 代码能编译、核心逻辑没破       | 不能证明真实链路可用   |
| 集成验证 | 集成测试、contract test、截图、浏览器 smoke test | 页面、接口、跨模块行为符合预期 | 成本更高，需要准备环境 |
| 线上验证 | 生产日志、监控指标、真实页面、人工 review 清单   | 发布后真实用户路径可用         | 只能在发布或预览后确认 |

写 prompt 或 Skill 时，最好提前定义完成标准：

```markdown
## Definition of Done
- `npm test` passes
- `npm run build` passes
- Changed article appears in generated blog route
- No unrelated dirty files are staged
- Deployment only happens after explicit approval
```

不要把“看起来可以”当成完成。如果是前端页面，就看截图；如果是接口，就跑 contract test；如果是生产发布，就看真实健康检查和线上页面。

## 10. 高频命令和使用习惯

下面这些命令或习惯，在长期使用 Claude Code 时很有用：

`/context` 用来看上下文占用，尤其是 MCP、文件内容和命令输出有没有挤爆窗口。

`/clear` 用来重开同一任务。Claude 被你纠偏两次还跑偏，通常不如清空重新说。

`/compact` 用来进入新阶段，但最好配合 `CLAUDE.md` 里的压缩保留规则。

`/memory` 用来看哪些记忆和项目规则真的被加载了。

`/mcp` 用来检查 MCP server，闲置的就关掉，不要让工具定义常驻吃上下文。

`/permissions` 和 `/sandbox` 用来管理边界。自动化越强，权限越要收紧。

`claude --continue` 适合恢复当前目录最近会话。

`claude --resume` 适合从历史会话里挑一个继续。

`claude --worktree` 适合需要隔离文件系统的任务。

`claude -p --output-format json` 适合把 Claude 接进脚本或 CI。

还有一个朴素但很有用的习惯：长任务结束后，让 Claude 写一份 handoff，把当前进展、做过的尝试、已验证的结果、失败路径、下一步计划写清楚。新会话读 handoff，比赌自动压缩摘要靠谱。

## 11. 常见反模式

**把 CLAUDE.md 当 wiki。** 症状是每次加载一堆背景，关键指令反而被冲淡。修法是只保留契约，资料放到 rules、skills 或 docs。

**Skill 写成大杂烩。** 症状是触发不稳定，流程互相打架。修法是一个 Skill 只服务一种任务。

**工具太多且命名模糊。** 症状是 Claude 选错工具，或者工具 schema 把上下文挤满。修法是合并重叠工具，做清楚的命名空间和返回格式。

**没有验证闭环。** 症状是 Claude 觉得完成了，但你不知道能不能发。修法是给每类任务绑定明确 verifier。

**过度自治。** 症状是多个 agent 同时跑，权限全开，出了问题不知道谁改了什么。修法是最小权限、worktree 隔离、固定输出格式、限制轮数。

**已批准命令长期堆积。** 症状是 `settings.json` 里残留危险命令，某天被自动执行。修法是定期审查 allowed tools 和 permissions。

## 12. 我的建议：从最小可用配置开始

如果你刚开始认真用 Claude Code，不用一口气把 Skills、Hooks、Subagents、MCP 全配满。先做一个最小版本：

1. 写一个短的 `CLAUDE.md`，只放构建、测试、边界和禁止项。
2. 固定提交前验证命令。
3. 只接最常用的 MCP，别全开。
4. 给高风险流程写一个 Skill，比如发布、迁移、线上排障。
5. 给容易忘的检查加 Hook，比如格式化或轻量 lint。
6. 大范围搜索和审查再交给 Subagent。

用一段时间后，凡是你反复提醒 Claude 的东西，再沉淀到项目规则里。凡是只偶尔用一次的知识，不要让它常驻上下文。

## 13. 结语

Claude Code 的上限不只取决于模型，也取决于你给它设计的协作系统。

如果你只是把任务扔进去等结果，它像一个聪明但容易跑偏的实习生；如果你给它清晰的上下文边界、可调用的工具、强制执行的 Hook、隔离的 Subagent 和明确的 verifier，它就更像一个可以纳入工程流程的 Agent。

一句话总结：别只问“Claude Code 会不会写”，要问“它怎么知道该写什么、不能碰什么、写完怎么证明是对的”。

## 14. 最重要的事情

上面所有最佳实践都有一个前提：你得能稳定使用 Claude Code。账号经常异常、订阅不稳定、网络环境频繁跳变，工程化配置做得再漂亮，也会被最基础的可用性问题打断。

如果你还卡在 Claude 账号注册、订阅支付、代理环境、风控和稳定使用这些问题上，可以先看我这篇实操文，把基础环境跑稳：

[2026 最新 Claude 订阅防封全攻略：小白也能搞定的低成本防封实操](https://zhetengxia.com/blog/claude-subscription-anti-ban-guide)

先解决“能稳定订阅和使用 Claude Code”，再谈 `CLAUDE.md`、Skills、Hooks、Subagents 和验证闭环，顺序不要反过来。
