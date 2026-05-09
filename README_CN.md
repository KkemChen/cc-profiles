# cc-profiles

在 **Claude Code 内部** 运行任意 Anthropic 兼容模型 —— 完整保留插件、hooks、MCP 服务器和 Agent Teams。无需切换工具，体验不打折。

[English](README.md) | [中文](README_CN.md)

## 为什么选择 cc-profiles？

Claude Code 是目前最强的 AI 编程终端 —— 插件、权限、hooks、MCP、Agent Teams。但当额度紧张、或者想把任务委派给其他模型时，常见方案都有硬伤：

- **全局环境变量（cc-switch）**：直接改变你的身份 —— 同时只能用一种模型，切换需要重启。
- **外部工具（OpenCode、Codex）**：不同的运行环境，你在 Claude Code 里的一切配置都带不过去。

cc-profiles 让你创建**隔离的配置文件**，让其他模型*通过* Claude Code 运行。每个 profile 拥有一份独立的 settings JSON。你留在 Claude Code 里，它们帮你分担任务。

### 方案对比

| | cc-profiles | 全局环境变量（cc-switch） | 外部工具 |
|---|---|---|---|
| **运行环境** | Claude Code | Claude Code | 各自的 harness |
| **插件与 hooks** | 全部保留 | 全部保留 | 不可用 |
| **多模型并行** | 支持 — 隔离配置 | 不支持 — 一次一个 | 独立进程 |
| **切换成本** | 零 — 配置共存 | 需要重启 | 上下文切换 |
| **会话续接** | `--resume` 跨委派保持上下文 | 不支持 | 不支持 |
| **配置隔离** | 独立 JSON 文件 | 修改全局环境变量 | 独立配置体系 |

> **一句话**：cc-switch 改变的是*你是谁*，cc-profiles 给你的是*并肩作战的队友* —— 全部跑在 Claude Code 的轨道上。

## 安装

```bash
claude plugin marketplace add KkemChen/cc-profiles
claude plugin install cc-profiles@cc-profiles
```

## 快速上手

```bash
/profile-add deepseek          # 创建配置（API key、base URL、模型）
/profile-list                  # 查看所有配置及状态
/profile-activate deepseek     # 激活为当前会话助手
```

也支持自然语言触发：说 "添加 deepseek 配置"、"激活 kimi"、"委派给 glm" 即可。

## 命令

| 命令 | 功能 |
|---|---|
| `/profile-add [provider]` | 创建供应商配置 —— 收集 API key、base URL、模型名 |
| `/profile-list` | 查看所有配置、快捷命令及就绪状态 |
| `/profile-activate [name]` | 激活配置，让 AI 能向其委派任务 |

## 工作原理

**添加配置**时，收集你的 API key、base URL 和模型名，克隆全局 Claude Code 配置，保存至 `~/.claude/profiles/<name>.json`，并自动创建 shell 快捷命令（`claude-ds`、`claude-glm` 等），可从终端直接调用。

**激活配置**时，向当前会话注入委派指令，AI 可通过以下方式分发子任务：
```
claude --settings ~/.claude/profiles/<name>.json -p "..."
```
支持 `--resume` 保持上下文连续性 —— AI 自主判断何时续接之前的对话。

**列出配置**时，扫描 profiles 目录，按文件路径匹配检测 shell 快捷方式，报告各项就绪状态。

## 支持的供应商

DeepSeek、GLM、Kimi、Qwen、MiniMax、Doubao、OpenRouter、Ollama，以及任何 Anthropic 兼容端点。

## 要求

- Claude Code >= 1.0.61
