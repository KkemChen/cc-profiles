---
name: profile-list
description: Show all configured AI provider profiles and their current status
allowed-tools: Read, Glob
---

# /profile-list

Display all configured Claude Code profiles and their details.

## Procedure

### Step 1: Discover profiles

Scan `~/.claude/profiles/` for all `*.json` files. On Windows, `~` = `%USERPROFILE%`.

### Step 2: Read each profile

For each profile JSON, extract:
- Profile name: filename minus `.json`
- Model: `env.ANTHROPIC_MODEL`
- Base URL: `env.ANTHROPIC_BASE_URL`
- Provider: infer from URL (e.g. `api.deepseek.com` → DeepSeek, `api.anthropic.com` → Claude, `api.z.ai` → GLM, etc.)
- Fast model: `env.ANTHROPIC_DEFAULT_HAIKU_MODEL` (if different from main)
- Strong model: `env.ANTHROPIC_DEFAULT_OPUS_MODEL` (if different from main)

### Step 3: Display

Format as a table:

```
## Configured Profiles

| Name | Provider | Model | Fast | Strong |
|------|----------|-------|------|--------|
| deepseek | DeepSeek | deepseek-v4-pro | deepseek-v4-flash | — |
| glm | GLM | glm-4 | glm-4-flash | — |

Total: 2 profiles across 2 providers
Profiles stored in: ~/.claude/profiles/
Commands: /profile-add <name> | /profile-activate <name> | /profile-list
```

### Edge cases

- **No profiles found**: "No profiles yet. Run `/profile-add <provider>` to create one."
- **Profile file can't be read**: Show name with error reason.
