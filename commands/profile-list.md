---
name: profile-list
description: Show all configured AI provider profiles and their current status
allowed-tools: Bash, Read, Glob
---

# /profile-list

Display all configured Claude Code profiles and their details.

## Procedure

### Step 1: Discover profiles

Scan `~/.claude/profiles/` for all `*.json` files. Exclude non-profile files (ORCHESTRATION.md, SETUP-GUIDE.md, README.md, etc.).

### Step 2: Read each profile

For each profile JSON, extract:
- Profile name: filename minus `.json`
- Model: `env.ANTHROPIC_MODEL`
- Base URL: `env.ANTHROPIC_BASE_URL`
- Provider: infer from URL (e.g. `api.deepseek.com` → DeepSeek, `api.anthropic.com` → Claude Official, `api.z.ai` → GLM, etc.)
- Haiku model: `env.ANTHROPIC_DEFAULT_HAIKU_MODEL` (if set)
- Opus model: `env.ANTHROPIC_DEFAULT_OPUS_MODEL` (if set)

Also check if each profile has a corresponding shell alias/function.
Search by **profile file path** (e.g. `profiles/<name>.json` or `profiles\<name>.json`), not by guessing the shortcut name — users may use abbreviations (e.g. `claude-ds` for deepseek).

- Windows: search `$PROFILE` for any function/alias whose body contains the profile's JSON path
- macOS/Linux: search `~/.bashrc` and `~/.zshrc` for any alias whose body contains the profile's JSON path

If a match is found, extract the actual shortcut name from the function/alias definition and display it.

### Step 3: Display

Format as a table:

```
## Configured Profiles

| Name | Provider | Model | Shell Shortcut | Status |
|------|----------|-------|----------------|--------|
| deepseek | DeepSeek | deepseek-v4-pro | claude-ds | ✅ ready |
| glm | GLM | glm-4 | — | ⚠️ no shortcut |
| kimi | Kimi | kimi-k2.5 | claude-kimi | ✅ ready |
```

Status meanings:
- ✅ ready — profile file exists and shell shortcut is configured
- ⚠️ no shortcut — profile file exists but no shell alias found
- ❌ missing file — shortcut found but profile file missing

### Step 4: Summary

Below the table, show:
```
Total: 3 profiles across 3 providers
Commands: /profile-add <name> | /profile-activate <name> | /profile-list
Profiles stored in: ~/.claude/profiles/
```

### Edge cases

- **No profiles found**: Show an empty state message with a prompt to run `/profile-add` first.
- **Profile file exists but can't read**: Show ⚠️ with error reason.
