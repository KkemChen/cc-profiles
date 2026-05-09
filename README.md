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
/profile-list                  # See all profiles and their status
/profile-activate deepseek     # Enable as a helper for this session
```

Or just say it naturally: "add a deepseek profile", "activate kimi", "delegate to glm".

## Commands

| Command | What it does |
|---|---|
| `/profile-add [provider]` | Create a provider profile — collects API key, base URL, model |
| `/profile-list` | Show all profiles and their status |
| `/profile-activate [name]` | Enable a profile as a helper for the current session |

## How it works

**Adding a profile** collects your API key, base URL, and model name, clones your global Claude Code config, and saves it to `~/.claude/profiles/<name>.json`.

**Enabling a profile** makes it available as a helper — the AI can then offload subtasks via:
```
claude --settings ~/.claude/profiles/<name>.json -p "..."
```
It supports `--resume` for multi-step workflows — the AI decides when to continue a previous context.

**Listing profiles** scans the profiles directory and reports provider, model, and readiness for each.

## Supported Providers

DeepSeek, GLM, Kimi, Qwen, MiniMax, Doubao, OpenRouter, Ollama, and any Anthropic-compatible endpoint.

## Requirements

- Claude Code >= 1.0.61
