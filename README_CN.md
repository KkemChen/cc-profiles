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

**终端命令行：**
```bash
claude plugin marketplace add KkemChen/cc-profiles
claude plugin install cc-profiles@cc-profiles
```

**Claude Code 内部：**
```
/plugin marketplace add KkemChen/cc-profiles
/plugin install cc-profiles@cc-profiles
```

## 快速上手

```bash
/profile-add deepseek          # 创建配置（API key、base URL、模型）
/profile-test deepseek         # 5 秒探针，验证配置真的能通
/profile-list                  # 查看所有配置及状态
/profile-enable deepseek       # 启用为当前会话的助手
```

也支持自然语言触发：说 "添加 deepseek 配置"、"激活 kimi"、"委派给 glm" 即可。

## 命令

| 命令 | 功能 |
|---|---|
| `/profile-add [provider]` | 创建供应商配置 —— 收集 API key、base URL、模型名。默认使用 **Slim** 模式（精简，适合委派）。如需把 profile 当主 harness 启动，选 **Mirror** 模式。 |
| `/profile-test [name]` | 发送 1 轮探针验证 API key、base URL、模型名是否真能通。能在 5 秒内定位"卡住没响应"问题。 |
| `/profile-list` | 列出所有配置，显示模式（Slim / Mirror）、供应商、模型 |
| `/profile-enable [name]` | 启用配置作为当前会话的助手，提供三种委派模式（单次 / 流式 / 缓存续接） |

## 工作原理

**添加配置**时，收集你的 API key、base URL 和模型名，保存一份精简的 "Slim" settings JSON 到 `~/.claude/profiles/<name>.json`。Slim profile 自带空的 `enabledPlugins` / `mcpServers`，这让每次委派的冷启动成本比克隆全局配置低 ~33%。如果你确实需要插件全套都对委派模型可见，创建时选 **Mirror** 模式。

**测试配置**会跑 `claude --settings ... -p "ok" --max-turns 1`，回报延迟、token 用量、费用，以及任何认证/网络错误。新建配置后请务必先测一下再用。

**启用配置**让该 profile 成为可用的助手 —— AI 会根据任务形态自动选择三种委派模式之一：
- 模式 A —— 短任务用 `--output-format json`
- 模式 B —— 长任务用 `--output-format stream-json --verbose` 重定向到日志，配合 Monitor 工具实时看进度
- 模式 C —— 多步骤任务用 `--resume <session_id>` 复用 prompt cache（后续调用成本降低约 80%）

每次委派结束后，AI 会向你回报 `total_cost_usd`，让成本透明。

**列出配置**时扫描 profiles 目录，报告各 profile 的供应商、模型、模式（Slim/Mirror）及就绪状态。自定义聚合器（OneAPI/NewAPI/LiteLLM）会按其 host 名称标注，而不是显示 "Unknown"。

## 支持的供应商

DeepSeek、GLM、Kimi、Qwen、MiniMax、Doubao、OpenRouter、Ollama，以及任何 Anthropic 兼容端点。

## 要求

- Claude Code >= 1.0.61
