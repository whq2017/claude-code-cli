# Claude Code CLI 架构分析索引

本文档集基于当前仓库快照中的源码进行分析，重点服务两个目标：

1. 理解这个项目的代码框架、Agent 架构和工程设计规范。
2. 提炼出一套可迁移到 Python Agent 系统中的实现方法。

## 阅读顺序

1. `01-overall-architecture.md`
   先建立整仓运行时视角，理解入口、注册表、主循环、状态层、任务层的关系。
2. `02-agent-architecture.md`
   再深入 AgentTool、subagent、teammate、权限与安全、后台任务和隔离机制。
3. `03-python-agent-blueprint.md`
   最后把前两份报告抽象成一套 Python 版 Agent 框架设计蓝图。
4. `04-claude-cli-vs-claw-code.md`
   对照 `Claude Code CLI` 和 `claw-code`，理解“完整运行时”和“渐进式 Python 重写工作台”两种工程路线各自解决什么问题。
5. `05-agent-engineering-roadmap.md`
   把这些结论落成一条可执行的学习与实践路径，服务于你自己的 Python Agent 工程化能力提升。

## 分析覆盖的核心源码

- 启动与入口：`entrypoints/cli.tsx`、`main.tsx`
- 注册表：`commands.ts`、`tools.ts`
- 统一抽象：`Tool.ts`、`Task.ts`
- 主循环：`query.ts`
- Agent 核心：`tools/AgentTool/AgentTool.tsx`、`tools/AgentTool/runAgent.ts`
- Agent 定义：`tools/AgentTool/loadAgentsDir.ts`
- 任务与输出：`tasks/LocalAgentTask/LocalAgentTask.tsx`、`tasks/RemoteAgentTask/RemoteAgentTask.tsx`、`utils/task/framework.ts`、`utils/task/diskOutput.ts`
- 工具执行：`services/tools/toolOrchestration.ts`、`services/tools/toolExecution.ts`、`services/tools/StreamingToolExecutor.ts`
- 权限系统：`hooks/toolPermission/PermissionContext.ts`、`hooks/toolPermission/handlers/*.ts`、`utils/permissions/permissionSetup.ts`
- 状态层：`state/store.ts`、`state/AppStateStore.ts`、`state/AppState.tsx`
- 多 Agent 协作：`tools/shared/spawnMultiAgent.ts`、`utils/swarm/*`
- 对照项目：`/Users/wuhuanqing/code/github/claw-code/README.md`、`/Users/wuhuanqing/code/github/claw-code/PARITY.md`、`/Users/wuhuanqing/code/github/claw-code/src/*.py`、`/Users/wuhuanqing/code/github/claw-code/tests/test_porting_workspace.py`

## 两个阅读提醒

- 这是一个“源代码分析快照”，不是完整可构建发行版；仓库本身缺少完整包管理和构建元数据，所以应把它视为架构样本而不是完整交付仓。
- 文中凡是写“推断”或“可以合理判断”的地方，表示这是基于源码结构和注释做出的工程解释，而不是某个函数显式声明的事实。
- 你提供的知乎链接存在反爬限制，当前分析无法稳定抓取其完整正文；因此新增文档不会把未验证的原文表述写成事实，而是只吸收与两个代码库相互印证、且能公开验证的工程主题，例如隔离执行、意图控制、计划先行、验证闭环、把 AI 当成团队运行时成员而不是单点补全器。
