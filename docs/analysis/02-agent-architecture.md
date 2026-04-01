# Claude Code CLI Agent 架构深度分析

## 1. 总结先行

这个项目的 Agent 架构有一个非常强的中心思想：

```text
Agent 不是单独的一套引擎
Agent 是一类特殊的 Tool
Agent 的执行仍复用统一 query() 主循环
```

这意味着：

- Agent 可以被权限系统治理。
- Agent 可以被任务系统追踪。
- Agent 可以像其他工具一样被注册、过滤、禁用和观测。
- Agent 与主线程共享同一套消息协议、工具协议和生命周期协议。

这比“主 Agent + 子 Agent 两套不同框架”高明很多。

## 2. Agent 的定义模型

## 2.1 AgentDefinition 不是 prompt 字符串，而是一份运行配置

`tools/AgentTool/loadAgentsDir.ts` 中的 `AgentDefinition` 包含的不是简单 prompt，而是一份比较完整的运行定义：

- `agentType`
- `whenToUse`
- `tools`
- `disallowedTools`
- `skills`
- `mcpServers`
- `hooks`
- `model`
- `effort`
- `permissionMode`
- `maxTurns`
- `background`
- `initialPrompt`
- `memory`
- `isolation`
- `requiredMcpServers`
- `omitClaudeMd`

这说明作者把 Agent 当成“可配置执行模板”，而不是“系统 prompt 别名”。

## 2.2 三类 Agent 来源

源码里区分了：

- built-in agent
- custom agent
- plugin agent

这三类共享统一接口，但来源不同：

- built-in：代码内置，通常有动态 prompt 逻辑
- custom：用户 / 项目 / policy settings 中加载
- plugin：插件提供

这套模型非常适合 Python 里做成：

- 内置 Python 类
- 本地 YAML/Markdown/frontmatter 定义
- pip plugin entrypoints

## 2.3 Built-in Agent 的职责切分很清晰

从几个内置 Agent 可以看出作者的分工思路。

### `general-purpose`

- 默认通用子 Agent
- 强调搜索、分析、多步任务
- 工具开放度高

### `Explore`

- 只读
- 快速
- 禁止编辑
- 鼓励并行搜索

### `Plan`

- 只读
- 面向架构与实施方案
- 明确要求输出关键文件和分步计划

### `verification`

- 默认后台
- 不是“确认正确”，而是“主动尝试把实现打坏”
- 强制命令证据和 VERDICT 输出

这不是简单的 prompt engineering，而是在做“角色能力收敛”。每个 Agent 都是一个小型 operating mode。

## 3. AgentTool 的路由决策

`tools/AgentTool/AgentTool.tsx` 本身像一个调度器，而不是执行器。它要先决定这次调用属于哪种路径。

## 3.1 调度分支

可以把它概括为：

```text
AgentTool.call()
  -> 如果是 team spawn: 走 teammate 路径
  -> 否则解析 subagent_type / fork path
  -> 校验权限和 MCP 前置条件
  -> 决定 isolation(worktree/remote/none)
  -> 决定 async 还是 sync
  -> 组装 runAgent 参数
  -> 执行或注册后台任务
```

## 3.2 多 Agent 团队路径

当 `team_name` 和 `name` 同时存在时，不再是普通 subagent，而是 spawn teammate：

- 可能是 tmux pane / iTerm pane / in-process teammate
- 可在 team roster 中被寻址
- 与 `SendMessage`、mailbox、权限同步配合

项目还明确阻止：

- teammate 再 spawn teammate，避免扁平 team roster 混乱
- in-process teammate 再开后台 agent

这些约束说明作者不是“功能越多越好”，而是非常重视拓扑清晰度。

## 3.3 普通 subagent 路径

普通 subagent 会先解决两个问题：

1. 用哪个 AgentDefinition。
2. 是否是 fork path。

fork path 的本质是：复用父上下文和精确工具集，以最大化 prompt cache 命中。这不是“另一个 Agent 类型”，而是一种特殊上下文继承策略。

## 3.4 MCP 前置条件检查

某些 Agent 依赖特定 MCP server。AgentTool 在真正运行前会：

- 等待 required MCP server 的 pending 连接完成
- 校验是否真的有工具可用
- 不满足时直接失败

这是很值得借鉴的设计：依赖满足性不应该留到 Agent 运行途中才发现。

## 3.5 isolation 路径

当前源码里有三类运行隔离语义：

- 无隔离
- `worktree`
- `remote`

### worktree

- 创建独立 git worktree
- 将 Agent 的文件系统操作定向到该工作区
- 执行后检查是否有改动
- 无改动则自动回收，有改动则保留

### remote

- 把任务升级为远端 session
- 本地只保留 task 追踪和结果观察

这个设计对 Python 特别有启发：Agent 隔离不一定只是 sandbox，也可以是“工作区级隔离”和“执行位置级隔离”。

## 4. sync Agent 与 async Agent 的区别

## 4.1 何时强制 async

源码中的 async 不只是 `run_in_background=true`。

还包括：

- agent 定义 `background: true`
- coordinator mode
- fork subagent experiment
- assistant mode
- proactive mode

说明“是否后台化”是一个系统策略问题，而不是用户显式参数问题。

## 4.2 sync Agent 也可以中途转后台

这是很精彩的一点。

同步 Agent 启动时会先注册 foreground task。如果运行时间变长，用户或系统可以把它 background 化，然后：

- 结束当前 foreground 迭代器
- 重新以 async 形态继续执行
- 返回 `async_launched`

也就是说，前台与后台不是两套不同任务，而是同一执行单元的两种观察模式。

这对 Python 的启发很大：不要把“同步调用”和“后台任务”设计成两个完全不兼容的 API，最好允许运行时迁移。

## 5. runAgent() 真正做了什么

`tools/AgentTool/runAgent.ts` 是 Agent 执行核心。它做的事情比“调用 query”多得多。

## 5.1 Agent 启动阶段

启动时会处理：

- preload skills
- agent-specific MCP servers
- tool 集合解析和去重
- agent options 构造
- subagent context 构造
- transcript sidechain 初始化
- metadata 持久化

这里最重要的是两件事。

### 5.1.1 Agent 自己有 MCP server 生命周期

AgentDefinition 可以声明自己的 MCP servers。`runAgent()` 会：

- 建立 parent MCP clients + agent MCP clients 的合并视图
- 仅清理该 Agent 新建的 client
- 不误伤父级共享 MCP client

这是一种非常成熟的资源所有权设计。

### 5.1.2 Agent 不直接共享父上下文，而是生成 subagent context

这是 `createSubagentContext()` 的职责。

## 6. createSubagentContext() 的真正价值

`utils/forkedAgent.ts` 里这部分非常值得逐行学习。

## 6.1 默认是隔离，不是共享

默认行为包括：

- 克隆 `readFileState`
- 新建 nested memory trigger 集合
- 克隆或新建 contentReplacementState
- 新建 child abort controller
- `setAppState` 默认 no-op
- 子 Agent 默认 `shouldAvoidPermissionPrompts = true`

也就是说，子 Agent 默认假定自己不可直接操纵父 UI，不可污染父可变状态。

## 6.2 共享是显式 opt-in

只有在 sync agent 等场景下，才通过 override 显式共享：

- `shareSetAppState`
- `shareSetResponseLength`
- `shareAbortController`

这是一种非常好的工程纪律：共享必须被声明。

## 6.3 prompt cache 被当成架构约束

`CacheSafeParams` 和相关注释说明，作者把 prompt cache 命中率当成重要的系统指标。为了不破坏缓存，fork child 需要保持：

- system prompt
- user/system context
- tools
- model
- messages prefix
- thinking config

这非常关键。很多 Python Agent 框架完全不考虑“子 Agent 是否能复用父上下文缓存”，导致成本和延迟都偏高。

## 7. Agent 的消息、进度和任务管理

## 7.1 LocalAgentTask 是可观测子 Agent 容器

`tasks/LocalAgentTask/LocalAgentTask.tsx` 管的不是业务逻辑，而是 Agent 的“可见生命周期”：

- 任务注册
- abort controller
- progress tracker
- recent activities
- transcript 保留与面板展示
- pending message
- evict after

它把 Agent 从“后台协程”提升成“系统可管理对象”。

## 7.2 progress tracking 很工程化

源码跟踪：

- tool use 数量
- token 数量
- 最近活动
- activity description
- read/search 分类

这说明系统既关心最终结果，也关心中间过程的可解释性。

## 7.3 Disk output 解决了异步任务可恢复和可审计问题

`utils/task/diskOutput.ts` 的价值在于：即使某个 Agent 不在当前 UI 前台，也有稳定输出落盘。

它处理：

- 会话隔离的输出目录
- 大文件写入上限
- 不跟随 symlink
- flush 队列和 GC 压力控制

这很适合 Python 借鉴为：

- 每个 task 一个 append-only log
- UI/HTTP API/恢复逻辑都从 log 读取

## 8. 权限系统如何嵌入 Agent

这套系统最大的强项之一，是权限不是外挂，而是内嵌在 Agent runtime 中。

## 8.1 PermissionContext 统一裁决上下文

`hooks/toolPermission/PermissionContext.ts` 把一次 tool permission 判断封装成对象，里面有：

- tool
- input
- toolUseContext
- assistantMessage
- toolUseID
- persist / cancel / resolve / hooks / classifier 等方法

这比散落的 if/else 强很多，因为它把“权限检查”从流程代码中抽成了一个小状态机。

## 8.2 三种权限路径

### interactive main-agent

- dialog 与用户交互
- classifier 和 hooks 与用户操作竞争
- bridge / channel 回调也可参与

### coordinator worker

- 先串行跑 hooks
- 再跑 classifier
- 最后才落到交互式 dialog

### swarm worker

- 先试 classifier
- 不行则把权限请求通过 mailbox 发给 leader
- leader 返回允许或拒绝

这说明权限系统是“拓扑敏感”的：不同执行位置，有不同决策策略。

## 8.3 auto mode 的危险规则剥离

`utils/permissions/permissionSetup.ts` 里对危险规则做了显式识别，例如：

- Bash wildcard
- PowerShell 执行器
- Agent allow rule

这里体现出的原则是：自动模式下，不允许存在能绕过安全分类器的粗粒度白名单。

这是 Python Agent 系统非常应该抄的地方。不要把“自动批准”当成简单开关，必须先检查规则本身是否能让攻击面逃逸。

## 9. 多 Agent 协作的工程实现

## 9.1 teammate backend 是策略模式

`utils/swarm/backends/registry.ts` 负责检测并选择 backend：

- tmux
- iTerm2
- in-process fallback

backend 不直接塞在业务代码里，而是通过 registry 延迟注册和检测。这是典型策略模式。

## 9.2 spawn 时继承 CLI flags 与环境变量

`tools/shared/spawnMultiAgent.ts` 和 `utils/swarm/spawnUtils.ts` 会把关键状态传给 teammate：

- permission mode
- model
- settings path
- inline plugins
- chrome flag
- proxy / CA / remote env vars

这说明“多 Agent 协作”不是简单起个子进程，而是要复制足够的运行上下文，保证子 Agent 行为与父会话一致。

## 9.3 swarm worker 通过 mailbox 向 leader 请求权限

这部分很有代表性：

- worker 不自己拍板
- leader 才是最终交互和授权入口
- worker 通过 mailbox 发请求
- 本地 UI 显示 pending

这是一个很成熟的控制面/执行面分离模型。Python 做多 Agent 时，非常推荐把：

- scheduler / leader
- worker / executor

拆成两个角色，而不是让每个 worker 都直接和用户打交道。

## 10. 这套 Agent 架构的强项

## 10.1 强项

### 单一执行主循环

没有主 Agent / 子 Agent 两套完全不同逻辑，降低了心智负担和 bug 面积。

### 生命周期完整

从 spawn、progress、background、notification、resume 到 cleanup，都有明确落点。

### 安全与权限内建

权限不是外部守卫，而是执行协议的一部分。

### 对长期运行和失败路径友好

大量注释都在讨论 race condition、cleanup、恢复、缓存一致性，说明系统是按真实复杂环境设计的。

## 10.2 代价

### 上下文对象很大

`ToolUseContext` 功能强，但也意味着理解门槛高。

### feature flags 很多

这增强了可裁剪性，也提高了局部阅读难度。

### AgentTool 调度职责较重

它同时做路由、校验、任务注册和执行模式选择，后续继续演进时要注意复杂度失控。

## 11. 面向 Python 的关键迁移建议

如果你想把这套思想移植到 Python，请优先复刻下面 6 个点。

### 11.1 让 Agent 复用同一 query loop

不要为 subagent 另写一套推理管线。

### 11.2 用显式 Context 管理依赖

避免全局单例乱飞，尤其是：

- state
- permission policy
- tool registry
- abort / cancellation
- message history

### 11.3 先做 TaskRuntime，再做多 Agent

没有任务系统，多 Agent 一定会很快失控。

### 11.4 权限系统要跟执行拓扑绑定

main agent、background worker、distributed worker 不应共享完全相同的批准流程。

### 11.5 隔离要是一级概念

至少设计：

- 无隔离
- 临时工作目录 / repo copy
- 远程 worker

### 11.6 记录 sidechain transcript

子 Agent 的消息不要只存在内存里，要可以：

- 恢复
- 审计
- 汇总
- 重放

## 12. 总结

Claude Code 的 Agent 架构，本质上是一套“受控的递归执行框架”：

```text
模型可以通过 AgentTool 递归地产生新的执行单元
但每个执行单元都必须：
  1. 有明确的定义来源
  2. 运行在显式上下文中
  3. 受权限系统约束
  4. 受任务系统追踪
  5. 受日志与输出系统记录
  6. 在结束时被清理和通知
```

这正是很多 Python Agent 项目缺失的一环。它们通常能“调用另一个 Agent”，但缺少：

- 统一生命周期
- 权限拓扑
- cache 一致性
- 隔离语义
- 任务可观测性

如果你把这份报告中的结构真正迁移过去，你写出来的就不再只是“会调 LLM 的脚本”，而是一套能长期演进的 Agent runtime。
