# cc-profiles

Claude Code plugin for managing isolated multi-model profiles and orchestrating tasks across different AI providers.

## Install

```bash
claude plugin install KkemChen/cc-profiles
```

## What you get

### Slash Commands

| Command | What it does |
|---|---|
| `/profile-add [provider]` | Create a new provider profile (API key + base URL + model) |
| `/profile-list` | Show all configured profiles and their status |
| `/profile-activate [name]` | Activate a profile as a helper for the current session |

### Auto-triggered Skill

The plugin includes a `cc-profiles` skill that automatically activates when you mention:
- "add profile for [provider]" / "setup [provider]"
- "activate [name]" / "let [name] help" / "delegate to [name]"
- "多模型" / "切换供应商"

### How it works

**`/profile-add`** creates `~/.claude/profiles/<name>.json`:
- Quick mode: give API key + base URL + model, everything else cloned from your global config
- Full mode: paste a complete JSON
- Auto-adds shell shortcut for one-command launch (`claude-ds`, `claude-glm`, etc.)

**`/profile-activate`** injects delegation awareness into your session:
- Teaches the current AI how to offload tasks via `claude --settings <profile> -p "..."`
- Supports session continuity (`--resume`) for multi-step workflows
- AI decides autonomously whether to start fresh or resume context

**`/profile-list`** scans your profiles directory and shows status:
- Detects shell shortcuts by matching profile file paths (not name guessing)
- Shows provider, model, and readiness status

## Supported Providers

DeepSeek, GLM, Kimi, Qwen, MiniMax, Doubao, OpenRouter, Ollama, and any custom Anthropic-compatible endpoint.

## Requirements

- Claude Code >= 1.0.61 (for `--settings` support)

## Structure

```
cc-profiles/
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── commands/
│   ├── profile-add.md
│   ├── profile-list.md
│   └── profile-activate.md
├── skills/
│   └── cc-profiles/
│       └── SKILL.md
├── references/
│   └── provider-urls.md
└── README.md
```
