# cc-profiles

Run any Anthropic-compatible model **inside Claude Code** — with full access to plugins, hooks, MCP servers, and agent teams. No context switching, no harness downgrade.

[English](README.md) | [中文](README_CN.md)

## Why cc-profiles?

Claude Code is the best harness out there — plugins, permissions, hooks, MCP, agent teams. But when you run low on credits or want to delegate to a different model, the usual options aren't great:

- **Global env vars (cc-switch)**: changes your identity entirely — one model at a time, restart required.
- **External tools (OpenCode, Codex)**: different harness, none of your Claude Code setup comes along.

cc-profiles lets you spin up **isolated profiles** for other models that run *through* Claude Code. Each profile gets its own settings JSON. You stay in Claude Code; they work alongside you.

### Comparison

| | cc-profiles | Global env vars (cc-switch) | External tools |
|---|---|---|---|
| **Harness** | Claude Code | Claude Code | Their own |
| **Plugins & hooks** | All retained | All retained | Not available |
| **Multi-model parallel** | Yes — isolated profiles | No — one at a time | Separate process |
| **Switching cost** | Zero — profiles coexist | Restart required | Context switch |
| **Session resume** | `--resume` across delegations | N/A | N/A |
| **Config isolation** | Per-profile JSON | Global env mutation | Separate config |

> **In short**: cc-switch changes *what model you are*. cc-profiles gives you *helpers that work alongside you* — all on Claude Code's rails.

## Install

**From terminal:**
```bash
claude plugin marketplace add KkemChen/cc-profiles
claude plugin install cc-profiles@cc-profiles
```

**From inside Claude Code:**
```
/plugin marketplace add KkemChen/cc-profiles
/plugin install cc-profiles@cc-profiles
```

## Quick Start

```bash
/profile-add deepseek          # Set up a profile (API key, base URL, model)
/profile-test deepseek         # 5-second probe to verify it actually works
/profile-list                  # See all profiles and their status
/profile-enable deepseek       # Enable as a helper for this session
```

Or just say it naturally: "add a deepseek profile", "enable kimi", "delegate to glm".

## Commands

| Command | What it does |
|---|---|
| `/profile-add [provider]` | Create a provider profile — collects API key, base URL, model. Defaults to **Slim** mode (cheap, delegation-optimized). Use **Mirror** mode if you want to launch it as a primary harness. |
| `/profile-test [name]` | Send a 1-turn probe to verify auth, base URL, and model. Catches "it just hangs" issues in 5 seconds. |
| `/profile-list` | Show all profiles, their mode (Slim / Mirror), provider, and models |
| `/profile-enable [name]` | Enable a profile as a helper for the current session, with three delegation patterns (single / streaming / cached resume) |

## How it works

**Adding a profile** collects your API key, base URL, and model name, then saves a minimal "Slim" settings JSON to `~/.claude/profiles/<name>.json`. Slim profiles ship with empty `enabledPlugins` / `mcpServers`, which keeps each delegation's cold-start cost ~33% lower than a full clone of your global config. Pick **Mirror** mode at creation time if you actually want all your plugins available in the delegate.

**Testing a profile** runs `claude --settings ... -p "ok" --max-turns 1` and reports latency, tokens, cost, and any auth/network errors. Always test new profiles before relying on them.

**Enabling a profile** makes it available as a helper — the AI then picks one of three delegation patterns based on task shape:
- Pattern A — short single task with `--output-format json`
- Pattern B — long task with `--output-format stream-json --verbose` redirected to a log + Monitor tail for live progress
- Pattern C — `--resume <session_id>` for multi-step workflows that benefit from prompt-cache reuse (~80% cost reduction on follow-up calls)

The AI surfaces `total_cost_usd` after every delegation so you can see what each call cost.

**Listing profiles** scans the profiles directory and reports provider, model, mode, and readiness for each. Custom aggregators (OneAPI/NewAPI/LiteLLM) are labeled by their host name rather than "Unknown".

## Supported Providers

DeepSeek, GLM, Kimi, Qwen, MiniMax, Doubao, OpenRouter, Ollama, and any Anthropic-compatible endpoint.

## Requirements

- Claude Code >= 1.0.61
