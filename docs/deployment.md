# Deployment Guide

本文档说明如何把当前仓库中的命令行客户端部署到其他机器。

先给结论：

- 如果你要一个更适合持续维护和标准化发布的版本，优先选择 `hare/`
- 如果你要一个尽量贴近恢复源码行为的版本，选择 `recovered-from-cli-js-map/`
- `python-mvp/` 更适合实验和架构验证，不建议优先作为正式交付版本

## 1. 推荐路线

### 推荐正式发布：`hare/`

选择 `hare/` 的原因：

- 已经有标准 `pyproject.toml`
- 已经定义了 CLI 入口 `hare`
- 更容易用 Python 常规方式安装、构建和分发
- 更适合继续补功能、修 bug 和做内部发布

### 推荐恢复验证：`recovered-from-cli-js-map/`

选择 `recovered-from-cli-js-map/` 的原因：

- 更贴近恢复出来的 JS/TS 结构
- 适合对照源码和验证恢复版行为
- 已有 Windows 启动脚本

局限：

- 目前没有看到正式的一键打包脚本
- 更适合作为运行环境整体交付，而不是单文件安装包

## 2. 环境准备

### `hare/` 部署环境

最低建议：

- Python 3.11 或更高
- `pip`
- 网络访问能力
- 有效的 `ANTHROPIC_API_KEY`，如果你要连接真实模型

可选：

- `build`，如果你要构建 wheel 包
- `PyInstaller`，如果你后续要尝试封装成 Windows `.exe`

### `recovered-from-cli-js-map/` 部署环境

最低建议：

- Windows
- [Bun](https://bun.sh/)
- 有效的 `ANTHROPIC_API_KEY`

目录交付时建议保留：

- `src/`
- `local-shims/`
- `vendor/`
- `bun.lock`
- `package.json`
- `start-recovered.ps1`
- `start-recovered.cmd`

## 3. 本地运行

### 运行 `hare/`

在仓库根目录执行：

```powershell
python -m pip install -e .\hare
hare --version
```

如果要启用真实 API：

```powershell
python -m pip install -e .\hare[anthropic]
$env:ANTHROPIC_API_KEY="your_api_key"
hare --version
```

如果你想直接进入交互模式：

```powershell
hare
```

如果你想跑一次非交互请求：

```powershell
hare -p "hello"
```

### 运行 `recovered-from-cli-js-map/`

```powershell
cd .\recovered-from-cli-js-map
bun install
.\start-recovered.ps1 --isolated-profile
```

如果更偏向 CMD：

```cmd
cd recovered-from-cli-js-map
bun install
start-recovered.cmd --isolated-profile
```

说明：

- `--isolated-profile` 会把运行配置放到项目目录下的 `.recovered-claude-home`
- 这样更适合测试和部署验证，避免直接污染默认用户配置

## 4. 打包与发布

### 路线 A：发布 `hare` wheel 包

这是当前最推荐的正式发布方式。

#### 4.1 安装构建工具

```powershell
python -m pip install build
```

#### 4.2 构建 wheel 和源码包

```powershell
python -m build .\hare
```

构建完成后，产物通常会出现在：

```text
hare\dist\
```

常见文件包括：

- `hare-<version>-py3-none-any.whl`
- `hare-<version>.tar.gz`

#### 4.3 发布方式

你可以选择：

- 直接把 wheel 文件发到目标机器
- 上传到内部 PyPI 源
- 在 CI 中构建后作为制品归档

### 路线 B：直接交付 `hare` 源码目录

如果还处在内部试用阶段，也可以不先构建 wheel，直接交付源码并在目标机执行：

```powershell
python -m pip install -e <path-to-hare>
```

优点：

- 快速
- 改动后容易热更新

缺点：

- 版本边界不清晰
- 不如 wheel 分发规范

### 路线 C：交付恢复版 JS/TS 目录

如果你的目标是恢复验证，而不是规范发布，可以直接交付整个 `recovered-from-cli-js-map/` 目录，在目标机执行：

```powershell
cd <path-to-recovered-from-cli-js-map>
bun install
.\start-recovered.ps1 --isolated-profile
```

这种方式本质上是“部署运行环境”，不是严格意义上的安装包发布。

## 5. 目标机器部署步骤

### 方式一：安装 `hare` wheel

1. 在构建机执行：

```powershell
python -m pip install build
python -m build .\hare
```

2. 把 `hare\dist\` 下的 wheel 文件复制到目标机

3. 在目标机执行：

```powershell
python -m pip install .\hare-<version>-py3-none-any.whl
```

4. 配置环境变量：

```powershell
$env:ANTHROPIC_API_KEY="your_api_key"
```

5. 验证安装：

```powershell
hare --version
hare -p "hello"
```

### 方式二：直接部署恢复版目录

1. 把整个 `recovered-from-cli-js-map/` 目录复制到目标机
2. 在目标机安装 Bun
3. 设置 `ANTHROPIC_API_KEY`
4. 执行：

```powershell
cd .\recovered-from-cli-js-map
bun install
.\start-recovered.ps1 --isolated-profile
```

## 6. 配置要求

### 必需环境变量

如果连接真实 Anthropic API，至少需要：

```powershell
$env:ANTHROPIC_API_KEY="your_api_key"
```

### 恢复版 CLI 常见附加行为

`start-recovered.ps1` 和 `start-recovered.cmd` 会自动设置一些环境：

- `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1`
- `DISABLE_AUTOUPDATER=1`

这样更适合可控测试环境。

## 7. 验证建议

### 对 `hare/`

在交付前建议至少验证：

```powershell
python test_imports.py
python -m pip install -e .\hare
hare --version
hare -p "hello"
```

### 对 `recovered-from-cli-js-map/`

在交付前建议至少验证：

```powershell
cd .\recovered-from-cli-js-map
bun install
.\start-recovered.ps1 --isolated-profile --bare-mode
```

`--bare-mode` 更适合先验证启动链路，因为它会跳过部分 OAuth 和 keychain 相关流程。

## 8. 常见问题

### `python -m build` 报错 `No module named build`

先安装：

```powershell
python -m pip install build
```

### `hare` 能安装，但调用真实模型时报错

优先检查：

- 是否安装了可选依赖：`python -m pip install -e .\hare[anthropic]`
- 是否设置了 `ANTHROPIC_API_KEY`
- 目标环境是否可以访问外部网络

### 恢复版 CLI 可以启动，但依赖不完整

优先检查：

- 是否在 `recovered-from-cli-js-map/` 目录执行过 `bun install`
- 是否完整保留了 `local-shims/` 与 `vendor/`
- Bun 版本是否可用

## 9. 现阶段建议

如果你现在就要交付一个客户端版本，建议优先走这条路径：

1. 先把 `hare/` 作为正式发布对象
2. 用 wheel 做内部发布
3. 把 `recovered-from-cli-js-map/` 保留为参考实现和行为对照环境

这样后续无论是继续开发、修复问题，还是接入自动化构建，成本都会更低。
