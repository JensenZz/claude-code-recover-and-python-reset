# Architecture Guide

本文档用于帮助快速理解当前仓库，尤其是 `hare/` 这条 Python 恢复线的整体结构。

先给结论：

- 这个仓库不是单一应用，而是一个恢复与迁移工作区
- 真正最值得阅读的主线是 `hare/`
- `hare/` 的核心不是某个命令，而是围绕 `QueryEngine` 组织起来的 agent runtime
- 当前代码已经有比较完整的运行骨架，但仍然存在不少简化实现和占位模块

## 1. 仓库结构概览

仓库中目前主要有三条线：

- `hare/`：更贴近恢复源码目录结构的 Python 端口，也是当前最值得继续推进的一条线
- `python-mvp/`：更偏最小可运行架构验证的 Python 版本
- `recovered-from-cli-js-map/`：从 sourcemap 恢复出来的 JS/TS 版本，用作行为和结构参照

如果你的目标是继续开发、修复、部署或梳理架构，建议优先看 `hare/`。

## 2. `hare/` 的整体分层

从职责划分上看，`hare/` 大致可以分成这几层：

1. CLI 入口层
2. 会话与主循环层
3. 命令系统
4. 工具系统
5. API / service 层
6. 类型与消息模型

这套分层基本延续了恢复出来的 TypeScript 版本思路，而不是简单写成一个脚本式 CLI。

## 3. 请求是怎么流动的

一条典型请求链路大致是：

1. 用户从命令行进入 `hare`
2. CLI 入口解析参数并做初始化
3. `QueryEngine` 创建当前会话上下文
4. `query()` 进入模型调用与工具执行循环
5. 模型返回文本或 `tool_use`
6. 工具被调度执行
7. 工具结果重新写回消息流
8. 循环继续，直到模型给出最终文本

这个链路说明项目的中心不是“命令菜单”，而是“消息驱动的 agent loop”。

## 4. CLI 入口层

最上层入口是：

- [hare/entrypoints/cli.py](D:/work/claude-code-recover-and-python-reset/hare/entrypoints/cli.py)
- [hare/main.py](D:/work/claude-code-recover-and-python-reset/hare/main.py)

### `hare/entrypoints/cli.py`

这个文件是 bootstrap 层，职责比较克制：

- 处理 `--version`
- 设置少量运行环境变量
- 处理 `--bare` 这类早期开关
- 最后再加载并运行 `hare.main.cli_main`

它的作用类似“轻量外壳”，避免一上来就把全部模块都加载进来。

### `hare/main.py`

这是实际 CLI 的主入口，主要负责：

- 参数解析
- 初始化工作目录和会话状态
- 执行 `setup()`
- 进入交互模式或非交互模式

这里目前分成两个主要执行路径：

- `_run_repl()`：交互模式
- `_run_print_mode()`：非交互模式

不管走哪条路径，最后都会装配：

- `commands`
- `tools`
- `QueryEngine`

所以 `main.py` 更像“运行装配层”。

## 5. 会话与主循环层

这是 `hare/` 最核心的一层。

关键文件：

- [hare/query_engine.py](D:/work/claude-code-recover-and-python-reset/hare/query_engine.py)
- [hare/query/__init__.py](D:/work/claude-code-recover-and-python-reset/hare/query/__init__.py)

### `QueryEngine`

`QueryEngine` 是会话级对象，负责保存一段对话的状态。它的职责包括：

- 保存消息历史
- 保存读取文件缓存
- 保存 usage 和成本统计
- 维护工具上下文
- 发起每一轮提交

一个重要设计点是：

- 一个 `QueryEngine` 对应一段会话
- `submit_message()` 不是一次性脚本调用，而是在同一会话里开启新 turn

这说明它是“持久会话引擎”，不是简单的“发一个 prompt 拿一个结果”。

### `query()`

`query()` 是真正的 agent loop。它负责：

- 准备消息
- 调模型
- 识别 `tool_use`
- 调工具
- 把工具结果回写成消息
- 再次进入下一轮

它在结构上已经很接近恢复出来的 TS 版主循环，虽然部分实现还是简化版。

这个文件最值得关注的几个点：

- 它已经以“消息流”而不是“字符串”来组织执行
- 工具调用是模型输出驱动的
- 中断、最大轮数、错误回填这些控制逻辑已经进入主循环

## 6. 命令系统

关键文件：

- [hare/commands.py](D:/work/claude-code-recover-and-python-reset/hare/commands.py)
- [hare/commands_impl/__init__.py](D:/work/claude-code-recover-and-python-reset/hare/commands_impl/__init__.py)

命令系统负责 slash commands，例如：

- `/help`
- `/clear`
- `/model`
- `/status`

### 当前结构

`commands.py` 负责命令注册与聚合，理论上会把这些来源合并起来：

- 内建命令
- skills
- plugins

但从当前实现看，skill / plugin 加载仍是 stub，返回的是空集合。这意味着：

- 命令系统的“接口设计”已经在
- 动态加载能力还没有真正补全

### 当前状态判断

命令系统现在是“框架已在、能力未完全接线”的典型部分。

## 7. 工具系统

关键文件：

- [hare/tool.py](D:/work/claude-code-recover-and-python-reset/hare/tool.py)
- [hare/tools.py](D:/work/claude-code-recover-and-python-reset/hare/tools.py)

这是另一个核心层。

### `hare/tool.py`

这里定义了工具的抽象协议，核心包括：

- `input_schema()`
- `call()`
- `check_permissions()`
- `description()`
- `prompt()`
- `map_tool_result_to_tool_result_block_param()`

同时还定义了 `ToolUseContext`，用于贯穿工具执行过程传递上下文。

这个设计的价值在于：

- 工具执行和主循环解耦
- 权限策略可以独立挂接
- 消息流、缓存、模型配置都能作为上下文传入

### `hare/tools.py`

这个文件更像工具注册中心，负责：

- 汇总所有内建工具
- 基于权限规则过滤工具
- 在 simple mode 下只暴露有限工具
- 与 MCP 工具池合并

这说明工具系统不是“写死的一组函数”，而是一个可组合、可裁剪的能力池。

## 8. 具体工具实现

`hare/tools_impl/` 目录下可以看到很多具体工具。

例如：

- `BashTool`
- `FileReadTool`
- `FileEditTool`
- `FileWriteTool`
- `GlobTool`
- `GrepTool`
- `AgentTool`

### `BashTool`

`BashTool` 已经有比较完整的最小实现：

- 接受命令输入
- 根据平台选择 shell
- 执行子进程
- 收集 stdout / stderr
- 返回工具结果

这部分已经能支撑基本 agent 行为。

### `AgentTool`

`AgentTool` 的方向也很清晰：通过新建 `QueryEngine` 来递归处理子任务。

不过它当前仍然是简化实现，更多体现的是“架构方向已确定”，而不是已经完整恢复全部子代理能力。

### 一个典型过渡点

从当前代码看，并不是所有工具都完成了统一收口。某些工具还保留着更函数式或过渡态的实现方式，这说明工具层仍在逐步整理中。

## 9. API 与服务层

关键文件：

- [hare/services/api/client.py](D:/work/claude-code-recover-and-python-reset/hare/services/api/client.py)
- [hare/services/session_memory.py](D:/work/claude-code-recover-and-python-reset/hare/services/session_memory.py)
- [hare/context.py](D:/work/claude-code-recover-and-python-reset/hare/context.py)

### API 客户端

`hare/services/api/client.py` 已经接上了 Anthropic Python SDK，并负责：

- 组装 API 请求
- 转换工具 schema
- 处理 streaming / non-streaming
- 提取 usage 信息

这说明真实模型调用链路已经不是纯占位。

### Session Memory

`hare/services/session_memory.py` 提供了 `CLAUDE.md` 形式的会话记忆文件逻辑，说明项目正在沿着原始 Claude Code 的长期记忆思路前进。

### Context

`hare/context.py` 用来构建系统上下文和用户上下文，比如：

- git 状态
- 当前日期

不过从当前实现看，这部分还是偏保守，功能没有完全展开。

## 10. 类型与消息模型

关键文件：

- [hare/types/message.py](D:/work/claude-code-recover-and-python-reset/hare/types/message.py)
- [hare/utils/messages.py](D:/work/claude-code-recover-and-python-reset/hare/utils/messages.py)

这部分很重要，因为整个系统本质上是消息驱动的。

消息模型目前区分了：

- `UserMessage`
- `AssistantMessage`
- `SystemMessage`
- `ProgressMessage`
- `AttachmentMessage`

这使得系统可以表达：

- 普通对话
- 工具结果
- 中断
- 预算或最大轮数限制
- 系统边界消息

这也是为什么这个项目比普通 CLI 更像一个 agent runtime。

## 11. 当前完成度判断

从代码整体看，当前状态可以概括成：

- 主骨架已经成型
- 关键路径已经可运行
- 很多细节仍是简化实现

### 已经比较成型的部分

- CLI 入口分层
- `QueryEngine` 会话模型
- `query()` 主循环
- 消息类型系统
- 工具注册抽象
- Anthropic API 基本接入

### 仍明显处于过渡态的部分

- skills / plugins 动态加载
- 部分命令的真实接线程度
- 一些工具实现风格还不完全统一
- 上下文、权限、压缩、MCP 等高级能力仍有不少简化

## 12. 如何阅读这套代码

如果你要从零开始读，建议顺序是：

1. [hare/entrypoints/cli.py](D:/work/claude-code-recover-and-python-reset/hare/entrypoints/cli.py)
2. [hare/main.py](D:/work/claude-code-recover-and-python-reset/hare/main.py)
3. [hare/query_engine.py](D:/work/claude-code-recover-and-python-reset/hare/query_engine.py)
4. [hare/query/__init__.py](D:/work/claude-code-recover-and-python-reset/hare/query/__init__.py)
5. [hare/tool.py](D:/work/claude-code-recover-and-python-reset/hare/tool.py)
6. [hare/tools.py](D:/work/claude-code-recover-and-python-reset/hare/tools.py)
7. 再按需看 `commands_impl/`、`tools_impl/`、`services/`

这个顺序能帮助你先掌握执行骨架，再下沉到具体能力。

## 13. 推荐的后续文档方向

如果后面要继续完善文档，最值得新增的可能是：

- 一份 `hare` 模块覆盖度清单
- 一份“哪些模块仍是 stub”的跟踪文档
- 一份“从 recovered TS 对照到 Python 端口”的映射文档

## 14. 总结

`hare/` 当前最有价值的地方，不是它已经完整恢复了所有 Claude Code 功能，而是它已经恢复出了一个清晰、可继续演进的架构骨架：

- 有入口
- 有会话引擎
- 有消息模型
- 有工具协议
- 有模型调用路径

这意味着它已经从“零散恢复文件”进入了“可以系统化继续开发”的阶段。
