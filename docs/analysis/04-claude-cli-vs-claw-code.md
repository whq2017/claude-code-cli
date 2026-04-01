# Claude Code CLI 与 claw-code 对照学习报告

这份文档的目标不是判断谁“更强”，而是帮助你分清两类完全不同但都很有学习价值的 Agent 工程路线：

- `Claude Code CLI`：完整运行时型架构
- `claw-code`：渐进式 Python 重写 / 镜像 / 教学型架构

如果你想提升自己的 Agent 工程化能力，这两类项目都值得学，但学习重点不同。

## 1. 两个项目分别在解决什么问题

## 1.1 Claude Code CLI 在解决“如何把 Agent 变成产品级运行时”

从当前仓库源码可以看到，它关心的是：

- 多入口 CLI 启动性能
- 工具调用协议
- Agent 递归执行
- 权限与安全治理
- 后台任务和远程任务
- 多 Agent 协作
- UI、状态、消息、任务、遥测的一致性

也就是说，它要回答的是：

```text
一个真实可长期运行的 Agent CLI 系统应该怎样组织？
```

## 1.2 claw-code 在解决“如何把复杂系统迁移成 Python 工作台”

`claw-code` 当前主分支的 `README.md`、`src/main.py`、`src/query_engine.py`、`src/port_manifest.py`、`src/runtime.py`、`tests/test_porting_workspace.py` 显示，它更像一个 Python-first 的 porting workspace：

- 用 manifest 清点当前 Python 表面
- 用 snapshot 镜像原始 command/tool surface
- 用 parity audit 评估差距
- 用一个轻量 QueryEnginePort 演示 turn loop、session store、transcript
- 用 CLI 子命令把“工作区理解”和“架构对照”产品化

它要回答的是：

```text
当你还无法一次性重写一个大型 Agent 系统时，如何把迁移过程工程化？
```

这两者不是竞争关系，而是两个不同阶段的问题。

## 2. 两个项目最值得学的地方

## 2.1 从 Claude Code CLI 学什么

你应该重点学：

- Tool 协议化
- ToolUseContext 上下文注入
- `query()` 主循环状态机
- Agent 作为 Tool 的递归执行
- 任务系统和输出落盘
- 权限系统与执行拓扑耦合
- worktree / remote / teammate 这类隔离与协作机制

一句话：学“运行时设计”。

## 2.2 从 claw-code 学什么

你应该重点学：

- 迁移过程不要黑盒化，要显式化
- 用 `PortManifest`、`PortingModule`、`PortingBacklog` 把“进度”建模
- 用 snapshot 保留原系统表面，避免迁移时丢功能地图
- 用 `parity-audit` 持续评估缺口
- 用 `tests/` 保证工作台命令和摘要能力一直可运行
- 用 `RuntimeSession` 把 setup、route、execution、history、persist 串成教学闭环

一句话：学“渐进式工程迁移与教学化表达”。

## 3. 架构取向的核心差异

## 3.1 运行时优先 vs 工作台优先

### Claude Code CLI

- 优先保证真实运行
- 主循环是真调用模型、真执行工具、真处理权限
- 状态、消息、任务、UI 都是生产级 concern

### claw-code

- 优先保证结构可理解、可迁移、可验证
- 很多 command/tool 是 mirrored metadata，而不是完整执行器
- 更像一套 architecture workbook

这意味着：

- 如果你想学“怎么把 Agent 系统跑起来并长期维护”，优先看 Claude Code CLI。
- 如果你想学“怎么把大系统拆开、标记、镜像、迁移并持续验证”，优先看 claw-code。

## 3.2 强执行语义 vs 强可观测教学语义

### Claude Code CLI

核心对象是真正参与执行的：

- `Tool`
- `ToolUseContext`
- `TaskState`
- `AgentDefinition`
- `StreamingToolExecutor`

### claw-code

核心对象更偏教学与迁移管理：

- `PortManifest`
- `PortingModule`
- `PortingBacklog`
- `RoutedMatch`
- `RuntimeSession`

你能从中学到一个很重要的工程原则：

```text
不同阶段的系统，核心数据模型是不一样的
```

在产品阶段，你建模执行与副作用。
在迁移阶段，你建模表面、差距、路径和进度。

## 4. claw-code 对你做 Python Agent 的特殊价值

很多人看完 Claude Code 这类系统后，会直接试图“一次性复刻整套运行时”，最后容易失败。`claw-code` 给出的启发是：

## 4.1 先做可映射的表面，再做完整语义

`src/commands.py` 和 `src/tools.py` 先加载快照，构造 mirrored command/tool surface，而不是直接声称“已经有完整运行时”。

这非常重要，因为 Agent 系统重写最容易犯的错就是：

- 一开始就追求全量行为等价
- 没有“当前覆盖了什么、没覆盖什么”的显式地图

正确做法是先把表面建立起来，再逐步替换内核。

## 4.2 先做摘要型 query loop，再做真实 query runtime

`src/query_engine.py` 的 `QueryEnginePort` 很轻，但它已经包含了几个正确方向：

- `QueryEngineConfig`
- `TurnResult`
- transcript store
- compact threshold
- max turns
- usage summary
- structured output retry

它还不是 Claude Code 的真实 `query.ts`，但它是一个很好的 Python 训练场：你可以先在这里把 state machine 概念建立起来，再逐步替换成真实模型调用和工具编排。

## 4.3 先把 session persistence 做起来

`src/session_store.py` 和 `src/transcript.py` 虽然简单，但很有代表性：

- session 可持久化
- transcript 可 replay / compact / flush

这提醒你：Python Agent 项目不要等到功能复杂以后才想 session 持久化。应当从一开始就把“turn 结果会不会丢”“后台任务怎么恢复”纳入设计。

## 4.4 测试不只是测功能，也测学习界面

`tests/test_porting_workspace.py` 测的不只是业务逻辑，还包括：

- `python -m src.main summary`
- `parity-audit`
- `commands`
- `tools`
- `bootstrap`
- `turn-loop`

这很值得学，因为它体现出一个工程观：

```text
面向开发者的分析/对照/诊断界面，本身也应该被测试
```

对你做自己的 Agent 框架来说，这意味着：

- `debug summary`
- `tool registry list`
- `session inspect`
- `permission report`

这些命令也应该有测试。

## 5. 与你给的知乎参考相互印证的主题

由于该知乎链接存在反爬限制，当前无法稳定获取完整正文，因此这里不直接转述未经验证的原句，而只总结与两个项目可以相互印证、且对工程实践真正有价值的主题。

## 5.1 把 AI 当成团队运行时成员，而不是补全插件

无论是 Claude Code CLI 的：

- verification agent
- teammate / swarm
- task notification
- permission relay

还是 claw-code README 中对 OmX 的描述：

- `$team`
- `$ralph`
- architect-level verification

都指向同一个结论：

```text
AI Coding 的工程化关键，不是“模型写了多少代码”，而是“系统如何把 AI 纳入团队分工与治理”
```

## 5.2 计划、隔离、验证，比“会写代码”更重要

这三点在两个项目里都很强：

- Claude Code 有 `Plan`、`Explore`、`verification`、`worktree`、`remote`
- claw-code 的 README 与 PARITY.md 不断强调 porting workspace、audit、verification、runtime branching

这说明你做 Python Agent 时，优先级应当是：

1. 计划能力
2. 隔离能力
3. 验证闭环
4. 然后才是更强的代码生成

## 5.3 技术债不在“能不能生成”，而在“能不能持续治理”

Claude Code CLI 通过：

- permission setup
- classifier
- task lifecycle
- cleanup
- telemetry

治理复杂度。

claw-code 通过：

- manifest
- parity audit
- tests
- runtime session report

治理迁移复杂度。

本质一样：都不是让 AI 自由发挥，而是让系统暴露结构并接受约束。

## 6. 你应该怎样把两者合并成自己的方法论

推荐你采用“两阶段学习法”。

## 阶段 A：向 Claude Code CLI 学运行时骨架

你要理解：

- 为什么 Agent 是 Tool
- 为什么要有 ToolUseContext
- 为什么要有任务系统
- 为什么权限系统必须内嵌
- 为什么多 Agent 需要 leader / worker 结构

产出目标：

- 设计你自己的 Python `Tool` / `AgentContext` / `TaskRuntime` / `QueryLoop`

## 阶段 B：向 claw-code 学渐进迁移与工程表达

你要理解：

- 如何先做 manifest 和 backlog
- 如何用 snapshot 镜像旧系统能力面
- 如何写 parity audit
- 如何把 bootstrap / route / turn-loop 做成开发者命令

产出目标：

- 为你的 Python Agent 项目补一组“自我解释命令”和“迁移/差距测试”

## 7. 推荐的组合策略

如果你要自己做一个 Python Agent 框架，最实用的组合不是二选一，而是：

### 内核层学 Claude Code CLI

- Tool 协议
- Query Loop
- Permission
- Task
- Agent Runner

### 外层工程壳学 claw-code

- manifest
- parity audit
- runtime summary
- bootstrap report
- CLI inspection commands

这样你会得到一个更平衡的系统：

- 内核不空洞
- 外壳不黑盒
- 迁移不失控
- 学习成本更低

## 8. 对你当前学习目标的直接建议

你说目标是“学习 Agent 设计规范，提高工程化开发能力”。那就不要把这两个项目都当成“代码仓库”，而要把它们当成两种范式。

### 范式一：产品级 Agent Runtime

代表：Claude Code CLI

学习问题：

- 工具怎么治理
- Agent 怎么递归
- 后台任务怎么跑
- 权限如何分层
- 状态和 UI 如何解耦

### 范式二：可教学、可迁移、可审计的 Python 工作台

代表：claw-code

学习问题：

- 如何逐步重构
- 如何表达当前覆盖范围
- 如何让架构学习过程本身可测试、可展示、可回放

## 9. 结论

把这两个项目放在一起看，你会得到一个比单看任一项目都更完整的认知：

```text
Claude Code CLI 告诉你：Agent 系统在“运行时”层面应该怎样工程化
claw-code 告诉你：复杂 Agent 系统在“迁移与学习”层面应该怎样工程化
```

这两者叠加起来，才是你真正需要的能力：

- 既能设计运行时
- 又能控制重写路径
- 既能做系统
- 又能做系统的自解释层

这正是从“会用 AI 写代码”进阶到“会做 AI 工程系统”的关键一步。
