# 面向 Python 的 Agent 框架蓝图

这份文档不是对 TypeScript 源码的重复解释，而是把前两份报告中的结构，抽象成一个更适合 Python 实现的工程蓝图。

目标不是“复刻 Claude Code”，而是借它的设计思想，做出一套更稳、更可扩展、更安全的 Python Agent 系统。

## 1. 先定设计目标

建议你的 Python Agent 框架至少满足以下目标：

1. 同一套主循环支持主 Agent 与子 Agent。
2. Tool 是协议，不是散函数。
3. Agent 运行时具备权限控制、任务追踪、日志落盘、取消传播。
4. 支持同步执行、后台执行、隔离执行。
5. 后续可以无痛接入 Web UI、队列系统、远程 worker、MCP 类外部工具。

如果你从一开始就把这些目标写进设计文档，后面很多重构可以省掉。

## 2. 推荐的 Python 目录结构

```text
agent_runtime/
  entrypoints/
    cli.py
    worker.py
  core/
    query_loop.py
    message_types.py
    context.py
    state_store.py
    events.py
  tools/
    base.py
    registry.py
    execution.py
    permissions.py
    builtins/
  agents/
    definitions.py
    loader.py
    runner.py
    builtins.py
    fork_context.py
  tasks/
    models.py
    runtime.py
    disk_output.py
    notifications.py
  permissions/
    policy.py
    decision.py
    rules.py
    handlers.py
  memory/
    transcript_store.py
    summary.py
  integrations/
    mcp.py
    remote_worker.py
    plugins.py
  ui/
    terminal.py
    api.py
```

重点不是目录名，而是边界要清楚：

- `core` 不知道具体有哪些工具。
- `tools` 不知道 UI 如何展示。
- `agents` 不直接操作全局状态，而是通过 context。
- `tasks` 不关心模型细节，只关心生命周期。

## 3. 四个必须先建好的核心数据结构

## 3.1 Tool

```python
from dataclasses import dataclass
from typing import Any, Protocol

class Tool(Protocol):
    name: str

    def is_enabled(self) -> bool: ...
    def is_read_only(self, input: Any) -> bool: ...
    def is_concurrency_safe(self, input: Any) -> bool: ...
    async def check_permissions(self, input: Any, ctx: "ToolUseContext"): ...
    async def call(self, input: Any, ctx: "ToolUseContext"): ...
```

不要一开始就把 Tool 写成继承大基类的重量对象。优先使用 `Protocol` 或轻量抽象，保留实现自由度。

## 3.2 ToolUseContext

```python
@dataclass
class ToolUseContext:
    app_state_store: "StateStore"
    tool_registry: "ToolRegistry"
    messages: list["Message"]
    abort_signal: "AbortSignal"
    model_name: str
    query_source: str
    agent_id: str | None = None
    agent_type: str | None = None
```

后续你会继续加字段，但一开始至少要把“执行环境”从全局变量中拿出来。

## 3.3 AgentDefinition

```python
@dataclass
class AgentDefinition:
    agent_type: str
    when_to_use: str
    system_prompt: str | None = None
    tools: list[str] | None = None
    disallowed_tools: list[str] | None = None
    model: str | None = None
    permission_mode: str | None = None
    max_turns: int | None = None
    background: bool = False
    isolation: str | None = None
    required_capabilities: list[str] | None = None
```

核心思想是：AgentDefinition 表示“运行模板”，不是“立即执行的对象”。

## 3.4 Task

```python
@dataclass
class TaskState:
    id: str
    type: str
    status: str
    description: str
    output_file: str
    start_time: float
    end_time: float | None = None
    metadata: dict[str, Any] | None = None
```

如果你的 Python 项目一开始就没有 Task 抽象，后面加后台 Agent、远程 Agent、恢复功能时会很痛苦。

## 4. Python 版主循环应该怎么写

不要写成这种容易失控的形式：

```python
async def agent(messages):
    resp = await llm(messages)
    if resp.tool_calls:
        ...
        return await agent(messages + [resp, tool_result])
```

建议写成显式状态机：

```python
async def query_loop(params: QueryParams):
    state = QueryState.from_params(params)

    while True:
        state = await maybe_compact(state)
        assistant_msg = await call_model(state)
        yield assistant_msg

        tool_calls = extract_tool_calls(assistant_msg)
        if not tool_calls:
            return build_terminal_result(state, assistant_msg)

        async for tool_msg in run_tools(tool_calls, state.tool_context):
            yield tool_msg
            state.messages.append(tool_msg)
```

这样以后你才能自然加入：

- token budget
- prompt too long 恢复
- 流式工具执行
- stop hooks
- synthetic tool_result

## 5. AgentRunner 的职责边界

推荐你单独做一个 `AgentRunner`，职责只包括：

1. 解析 AgentDefinition。
2. 根据 definition 构造 agent-specific context。
3. 决定 sync / async / isolated / remote。
4. 调用统一 `query_loop()`。
5. 记录 transcript 和 task progress。
6. 做清理。

它不应该直接承担：

- 具体 UI 绘制
- HTTP 路由
- 具体的工具实现

这和 Claude Code 的 `AgentTool` + `runAgent` 拆分是同一种思路。

## 6. 子 Agent 上下文必须默认隔离

强烈建议复刻 `createSubagentContext()` 的设计精神。

子 Agent 默认应当：

- 拥有自己的取消信号
- 拥有自己的消息视图
- 拥有自己的文件缓存
- 不能直接修改父 UI 状态
- 不能默认弹权限对话框

只有少数场景才共享：

- 父级 state 更新函数
- 某些 telemetry callback
- 统一 response metrics

### 一个简单实现

```python
def create_subagent_context(
    parent: ToolUseContext,
    *,
    share_state: bool = False,
    share_abort: bool = False,
    agent_id: str | None = None,
    agent_type: str | None = None,
) -> ToolUseContext:
    return ToolUseContext(
        app_state_store=parent.app_state_store if share_state else parent.app_state_store.fork_readonly(),
        tool_registry=parent.tool_registry,
        messages=list(parent.messages),
        abort_signal=parent.abort_signal if share_abort else parent.abort_signal.child(),
        model_name=parent.model_name,
        query_source=parent.query_source,
        agent_id=agent_id,
        agent_type=agent_type,
    )
```

重点不是代码，而是“共享必须显式”。

## 7. Tool 执行层的推荐做法

## 7.1 工具并发能力要声明式化

不要默认所有工具都可并发。推荐按下面规则调度：

- 连续只读、并发安全工具可并发执行
- 写工具、外部副作用工具串行执行
- 同一工作目录下可能冲突的 shell 工具串行

## 7.2 执行结果要统一为消息

不要让一部分工具返回字符串，一部分直接改状态，一部分抛异常。统一产出：

- progress event
- tool_result message
- optional context patch

比如：

```python
@dataclass
class ToolExecutionUpdate:
    message: "Message | None" = None
    context_modifier: Callable[[ToolUseContext], ToolUseContext] | None = None
```

这样 query loop 才能自然消费。

## 8. 权限系统不要只做“allow / deny”

最容易被低估的是权限系统。

推荐至少区分：

- `allow`
- `deny`
- `ask`

同时保留 reason：

- rule
- hook
- classifier
- user
- system policy

### 推荐的裁决顺序

```text
静态规则
  -> hook
  -> classifier
  -> user interaction / leader approval
  -> 最终 decision
```

### 为什么这么设计

- 静态规则快，适合第一层过滤。
- hook 可插入企业策略。
- classifier 可做语义安全判断。
- 用户交互是最后兜底。

## 9. 多 Agent 协作在 Python 中怎么落地

如果你打算做多 Agent，不建议一开始就上分布式。可以分三阶段。

### 阶段一：单进程内多任务

- 一个进程
- 多个 TaskState
- 子 Agent 共享同一模型客户端
- UI 只做只读观察

### 阶段二：本地多 worker

- `multiprocessing` / `asyncio subprocess`
- mailbox 或 SQLite/Redis 队列通信
- leader 统一处理权限与用户输入

### 阶段三：远程 worker

- 任务进入消息队列
- 远端 worker 拉取执行
- transcript / output 存储到共享存储

你可以直接借鉴这个仓库的“leader 决策、worker 执行”理念，而不是让每个 worker 都拥有全权限。

## 10. transcript 与 task output 的建议实现

至少做两套存储：

### 10.1 transcript store

记录结构化消息：

- user
- assistant
- tool_use
- tool_result
- progress

### 10.2 task output log

记录 append-only 文本输出，供：

- 后台 UI
- 任务恢复
- 审计
- debug

不要只把信息放进内存。后台 Agent 一多，内存状态和恢复状态一定会分叉。

## 11. 你最应该复刻的 8 个工程习惯

1. Tool 默认 fail-closed，不要默认可并发、可写、可自动批准。
2. Agent 只是统一 query loop 的配置化实例。
3. 子 Agent 上下文默认隔离，只有必要字段共享。
4. 权限系统拓扑敏感，main agent 和 worker 不同流。
5. Task 是一等公民，不要把后台执行藏在裸协程里。
6. 所有长运行对象都要有 cleanup。
7. 把输出写到磁盘或 durable store，而不是只存在内存。
8. 让“为什么这样设计”的注释写在临界路径旁边。

## 12. 一个可执行的迭代路线

如果你要在 Python 中从零开始，我建议按下面顺序做，而不是一步到位。

### Phase 1: 单 Agent 可工具调用

- `Tool`
- `ToolRegistry`
- `ToolUseContext`
- `query_loop`
- 基础消息模型

### Phase 2: 后台任务与恢复

- `TaskState`
- `TaskRuntime`
- `disk_output`
- transcript 持久化

### Phase 3: 子 Agent

- `AgentDefinition`
- `AgentRunner`
- `create_subagent_context`
- sync / async 切换

### Phase 4: 权限治理

- rules
- hooks
- classifier
- interactive approval

### Phase 5: 多 Agent

- leader / worker 模型
- mailbox
- remote worker
- worktree / temp workspace isolation

## 13. 最后给你的判断标准

你做的 Python Agent 框架，如果满足下面这些条件，就已经进入“工程化”而不是“脚本化”阶段了：

- 主 Agent 与子 Agent 走同一执行主循环。
- 工具是协议对象，不是散落函数。
- 权限可以针对不同执行位置切换流程。
- 后台任务可以追踪、恢复、通知、清理。
- 子 Agent 的 transcript 和输出可以持久化。
- 你可以很自然地再加一个 remote worker，而不是推倒重写。

这就是从这个项目里最该学到的东西。
