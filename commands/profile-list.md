---
name: profile-list
description: Show all configured AI provider profiles and their current status
allowed-tools: Read, Glob
---

# /profile-list

Display all configured Claude Code profiles and their details.

## Procedure

### Step 1: Discover profiles

Scan `~/.claude/profiles/` for all `*.json` files. On Windows, `~` = `%USERPROFILE%`. Skip `.gitignore` and any other non-JSON files.

### Step 2: Read each profile

For each profile JSON, extract:
- **Name**: filename minus `.json`
- **Model**: `env.ANTHROPIC_MODEL` (fallback: `env.ANTHROPIC_DEFAULT_SONNET_MODEL`)
- **Base URL**: `env.ANTHROPIC_BASE_URL`
- **Provider**: match URL against `references/provider-urls.md`. **If no match, use the URL's host portion** (e.g. `oneapi.example.com` from `https://oneapi.example.com/v1/anthropic`). Do not label as "Unknown" — custom aggregators (OneAPI/NewAPI/LiteLLM) are common.
- **Fast model**: `env.ANTHROPIC_DEFAULT_HAIKU_MODEL` (only show if different from main)
- **Strong model**: `env.ANTHROPIC_DEFAULT_OPUS_MODEL` (only show if different from main)
- **Mode**: if `enabledPlugins` is `{}` (empty object) and `mcpServers` is `{}`, label **Slim**. Otherwise **Mirror**. Slim profiles cost ~$0.10 less per cold start.
- **Health**: derived only from file presence + JSON validity. Call `/profile-test <name>` for an actual API probe — `profile-list` does not spend money on network checks.

### Step 3: Display

Format as a table:

```
## Configured Profiles

| Name | Provider | Model | Mode | Fast | Strong |
|------|----------|-------|------|------|--------|
| deepseek | DeepSeek | deepseek-v4-pro | Slim | deepseek-v4-flash | — |
| glm | GLM | glm-4.6 | Slim | glm-4-flash | — |
| onepay | oneapi.example.com | claude-sonnet-4-5 | Mirror | — | — |

Total: 3 profiles
Profiles stored in: ~/.claude/profiles/
Commands: /profile-add | /profile-test | /profile-enable | /profile-list
```

After the table, if any profile is **Mirror** mode, add a one-line note:
> Mirror profiles inherit your global plugins/MCP. They cost ~$0.10 more per cold start than Slim. Recreate with `/profile-add <name>` to switch to Slim if you only use them for delegation.

### Edge cases

- **No profiles found**: "No profiles yet. Run `/profile-add <provider>` to create one."
- **Profile file can't be read**: Show the name with the specific error (permission denied, corrupted JSON, etc.) rather than hiding it — the user needs to know it exists but is broken.
- **Profile missing `env` block**: Show with mode `(invalid)` and a hint to recreate.
