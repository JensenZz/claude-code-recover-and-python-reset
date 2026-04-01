# claude-code-recover-and-python-reset

这个仓库用于整理和推进一组与 Claude Code 恢复、源码对照、以及 Python 迁移相关的实验性工作。

目前它不是单一应用，而是一个工作区，里面并排放着三条主要线索：

- `recovered-from-cli-js-map/`：从 sourcemap 恢复出来的 Claude Code CLI JavaScript/TypeScript 版本，便于对照原始结构和行为。
- `python-mvp/`：一个更偏 MVP 的 Python 端口，README 已经整理了核心控制流、已实现范围和未完成项。
- `hare/`：另一个更贴近恢复源码目录结构的 Python 端口，包名为 `hare`，当前版本号为 `2.1.88`。

## 当前仓库状态

- 根目录 README 原本几乎为空，这里补充了项目说明，方便后续继续恢复和迁移。
- `hare/pyproject.toml` 将项目描述为 “Python port of Claude Code CLI (recovered from sourcemap v2.1.88)”。
- `hare/entrypoints/cli.py` 与 `hare/main.py` 表明 `hare` 提供了一个可执行 CLI 入口，并区分交互模式与 print mode。
- 根目录提供了一个简单验证脚本 `test_imports.py`，会递归导入 `hare` 下所有 Python 模块。
- 当前执行 `python test_imports.py` 的结果是：
  - Python 文件数：319
  - 代码总行数：18226
  - import 错误数：0

这说明 `hare` 这条线至少在“模块可导入”这一层面上已经处于一个相对完整、可继续迭代的状态。

## 目录说明

### `hare/`

这是当前仓库里最完整的一条 Python 恢复线，特点是尽量贴近恢复出来的源码布局。

你可以从这些文件快速进入：

- `hare/pyproject.toml`：包元数据与入口脚本定义
- `hare/entrypoints/cli.py`：CLI 启动入口
- `hare/main.py`：参数解析、REPL / print mode 分流
- `hare/commands_impl/`：slash commands 的实现
- `hare/tools_impl/`：工具系统实现
- `hare/services/`：会话、紧凑化、LSP、插件、MCP 等服务层代码

## `python-mvp/`

这是另一个 Python 版本，目标更偏“先把 Claude Code 代理循环跑通”的 MVP。它自己的 README 已经写得比较完整，适合快速理解整体 agent loop：

- 用户输入进入 REPL / CLI
- `QueryEngine` 组织消息
- 模型返回文本或 `tool_use`
- 工具执行结果回填到消息流
- 循环持续到产出最终文本

如果你想先理解“最小可工作的 Python 版本”，这里通常比 `hare/` 更容易上手。

## `recovered-from-cli-js-map/`

这是从 sourcemap 中恢复出来的 JS/TS 版本，用来做结构参考、行为比对和迁移依据。

目录里可以看到：

- `src/`：恢复后的源码
- `package.json`：依赖与版本信息
- `start-recovered.ps1` / `start-recovered.cmd`：本地启动脚本
- `local-shims/` 与 `vendor/`：恢复运行时需要的补丁与本地依赖

如果你要确认某个 Python 实现是否忠实于恢复版本，这个目录就是主要参照物。

## 快速验证

在仓库根目录运行：

```powershell
python test_imports.py
```

如果你想单独查看 `hare` 的包定义：

```powershell
Get-Content .\hare\pyproject.toml
```

如果你想先从 MVP 版本开始看：

```powershell
Get-Content .\python-mvp\README.md
```

## 如果你要打包部署一个客户端

当前仓库还没有统一的桌面客户端安装包流程，更适合按“命令行客户端”来理解和部署。

推荐顺序：

1. `hare/`：最适合标准化打包发布
2. `recovered-from-cli-js-map/`：最适合作为恢复版运行环境直接部署
3. `python-mvp/`：更适合实验，不建议优先正式交付

快速结论：

- 如果你要持续维护和标准化发布，优先选 `hare/`
- 如果你要最大程度贴近恢复源码行为，选 `recovered-from-cli-js-map/`
- 如果你只是做最小闭环验证，再考虑 `python-mvp/`

完整部署说明见 [docs/deployment.md](D:/work/claude-code-recover-and-python-reset/docs/deployment.md)。

## 适合下一步做什么

- 继续补齐根 README 中缺失的运行说明和依赖安装步骤
- 以 `recovered-from-cli-js-map/` 为参照，逐步核对 `hare/` 的模块覆盖率
- 在 `hare/` 和 `python-mvp/` 之间明确各自定位，避免两条 Python 线长期职责重叠

## 备注

这个仓库当前更像恢复与迁移工作区，而不是已经打磨完成的单产品仓库。阅读时建议先确定你的目标是：

- 看恢复出来的原始结构
- 看最小可运行的 Python MVP
- 还是继续推进 `hare` 这一条更完整的 Python 端口
