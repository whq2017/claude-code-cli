# Agent 工程化学习路线图

这份路线图专门服务你的目标：根据 `Claude Code CLI`、`claw-code` 和你指定的外部参考主题，系统提升自己的 Agent 设计规范与工程化开发能力，重点面向 Python 实现。

## 1. 先建立一个正确的认知框架

你现在需要把“AI 编码能力”拆成三层，不要混在一起。

## 1.1 模型能力层

这是大家最容易关注的一层：

- 推理
- 代码生成
- 工具调用
- 多轮对话

但这不是工程能力本身。

## 1.2 运行时能力层

这是 Claude Code CLI 最强的部分：

- query 主循环
- Tool 协议
- Agent 递归
- permission / hook / classifier
- task lifecycle
- isolation

这决定系统能否在复杂环境下长期运行。

## 1.3 演进能力层

这是 claw-code 补上的部分：

- manifest
- parity audit
- bootstrap report
- session store
- 自检 CLI
- 测试工作台

这决定系统能否被持续重构、迁移、教学和扩展。

一个成熟的 Agent 工程师，必须同时理解第 2 层和第 3 层。

## 2. 你的学习顺序应该怎么排

## 第一步：学“运行时骨架”

先读：

- [01-overall-architecture.md](/Users/wuhuanqing/code/github/claude-code-cli/docs/analysis/01-overall-architecture.md)
- [02-agent-architecture.md](/Users/wuhuanqing/code/github/claude-code-cli/docs/analysis/02-agent-architecture.md)

这一阶段只回答五个问题：

1. 为什么 Agent 是 Tool，而不是另一个独立系统？
2. 为什么 query loop 必须是状态机？
3. 为什么任务系统必须和 Agent 绑定？
4. 为什么权限系统必须是执行协议的一部分？
5. 为什么隔离执行是一级概念？

如果这五个问题你能用自己的话讲清楚，说明你已经跨过“会调 API”的阶段了。

## 第二步：学“迁移与表达能力”

再读：

- [04-claude-cli-vs-claw-code.md](/Users/wuhuanqing/code/github/claude-code-cli/docs/analysis/04-claude-cli-vs-claw-code.md)

这一阶段重点不是理解某个函数，而是学：

- 如何对齐新旧系统表面
- 如何让迁移过程可度量
- 如何给团队提供 summary / audit / route / bootstrap 这类理解界面

这是很多技术人最缺的一环：会做系统，但不会做系统的“自解释层”。

## 第三步：把它落成 Python 蓝图

最后读：

- [03-python-agent-blueprint.md](/Users/wuhuanqing/code/github/claude-code-cli/docs/analysis/03-python-agent-blueprint.md)

然后按其中的 Phase 1 到 Phase 5 去做自己的实现。

## 3. 每个阶段你应该练什么

## 阶段 A：最小可运行 Agent Runtime

目标：先做出正确骨架，不追求功能多。

必须实现：

- `Tool` 协议
- `ToolRegistry`
- `ToolUseContext`
- `QueryLoop`
- `AgentRunner`
- `TaskState`

练习要求：

- 至少实现 3 个工具：读文件、搜索、shell
- 支持一个主 Agent 和一个子 Agent
- 子 Agent 复用同一个 query loop

达标标准：

- 你不需要写第二套“子 Agent 执行器”

## 阶段 B：权限与验证闭环

目标：把 Agent 从“会跑”升级到“可控”。

必须实现：

- `allow / ask / deny`
- 简单规则系统
- hook 插口
- 任务日志
- transcript 存储

练习要求：

- shell 工具必须能被规则阻止
- 文件写工具必须被记录
- 每个 turn 的 tool use / tool result 都能回放

达标标准：

- 你能回答“这个 Agent 刚刚到底干了什么”

## 阶段 C：隔离与后台执行

目标：把系统从 demo 提升为真正可扩展运行时。

必须实现：

- 后台 task
- 输出落盘
- 会话恢复
- 独立工作目录或临时 repo copy

练习要求：

- 一个长任务可在后台继续运行
- 主界面可查看任务输出
- 失败后能恢复最近 transcript

达标标准：

- 你的系统不再依赖“当前终端窗口一直活着”

## 阶段 D：多 Agent 协作

目标：从单 Agent 工具链提升到 agent system。

必须实现：

- leader / worker 拓扑
- worker 不能直接拿最终权限
- worker 的结果有独立任务视图

练习要求：

- 一个 planner agent 只读分析
- 一个 executor agent 执行代码修改
- 一个 verification agent 只做验证

达标标准：

- 你开始在系统层设计“角色分工”，而不是只追求大 prompt

## 4. 你最该形成的 10 条设计规范

下面这 10 条，建议你直接变成自己的工程守则。

1. Agent 一律复用统一 query loop，禁止主/子 Agent 两套核心执行流程。
2. Tool 必须声明输入、输出、并发性、只读性和权限需求。
3. 子 Agent 默认隔离，只有必要能力显式共享。
4. 后台任务必须可追踪、可恢复、可清理。
5. 权限系统必须区分执行拓扑，main agent 和 worker 不应同流。
6. 隔离执行不是附加功能，而是运行时一级能力。
7. transcript 和 task output 都要落盘，不要只存在内存。
8. verification 要独立成角色，不要把“测试一下”塞给实现 Agent 自己说了算。
9. 迁移/重构过程必须有 manifest、audit、summary 等自解释工具。
10. 任何复杂系统都应提供面向开发者的 inspection CLI，并为其写测试。

## 5. 你可以直接照做的项目任务清单

如果你现在就要开始做自己的 Python Agent 项目，建议按这个顺序做。

### Week 1

- 建 `Tool`、`AgentDefinition`、`ToolUseContext`、`TaskState`
- 建一个最小 `query_loop`
- 做 3 个工具：`read_file`、`grep`、`shell`

### Week 2

- 给工具加 schema 校验
- 加 permission rules
- 加 transcript store
- 加 session store

### Week 3

- 做 subagent runner
- 让子 Agent 复用主 loop
- 做 planner / executor 两个角色

### Week 4

- 做后台 task
- 做 output log
- 做任务查看命令

### Week 5

- 做 verification agent
- 做最小 worktree / temp workspace 隔离
- 做 bootstrap / summary / route 这类开发者命令

## 6. 你要刻意避免的几个误区

## 误区一：把大 prompt 当架构

Agent 系统的问题，很少能靠 prompt 变长解决。大多数问题都来自：

- 状态不清
- 上下文不隔离
- 没有任务系统
- 没有权限系统
- 没有验证闭环

## 误区二：过早追求全量 feature parity

这正是 claw-code 给你的重要提醒。先做：

- 能力地图
- 差距审计
- 渐进迁移

不要一开始就试图把完整系统硬拷出来。

## 误区三：只测业务，不测运行时界面

你自己的：

- `/summary`
- `/inspect-task`
- `/list-tools`
- `/session-load`

都应该被测试。

## 误区四：让实现 Agent 自己验证自己

实现与验证最好分角色，否则很容易出现：

- happy path 自证
- 不跑命令只读代码
- 说“应该可以”

这是最常见的伪工程化。

## 7. 你如何判断自己真的进步了

下面是比“写了多少代码”更重要的判断标准。

### 初级

- 会调模型
- 会写工具调用
- 会让模型改文件

### 中级

- 会写统一 query loop
- 会管理工具和上下文
- 会做 session / transcript / tasks

### 高级

- 会做权限分层
- 会做多 Agent 角色编排
- 会设计隔离执行
- 会做 parity audit / runtime summary / developer inspection surfaces

当你从“我能让 AI 写代码”进化到“我能设计 Agent Runtime 和它的工程控制面”，你的工程能力才算真正跨过门槛。

## 8. 最后的建议

不要把这次学习理解成“研究某个产品源码”，而应该理解成“建立自己的 Agent 工程方法论”。

你最终需要掌握的，不是某个仓库的目录，而是三件事：

- 如何让 Agent 正确执行
- 如何让 Agent 可控执行
- 如何让 Agent 系统能够持续演进

这三件事分别对应：

- Claude Code CLI 的运行时骨架
- Claude Code CLI 的权限/任务/隔离治理
- claw-code 的迁移、审计和自解释工作台思路

把这三者组合起来，你就会从“AI 编程工具用户”转向“Agent 工程系统设计者”。
