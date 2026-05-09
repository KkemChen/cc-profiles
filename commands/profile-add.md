---
name: profile-add
description: Add a new AI provider profile to your Claude Code team
argument-hint: "[provider-name]"
allowed-tools: Bash, Read, Write, Edit, AskUserQuestion
---

# /profile-add

Create an isolated Claude Code settings profile for switching between AI providers via `--settings`.

## Procedure

### Step 1: Collect profile name

If the user provided a name as argument (e.g. `/profile-add deepseek`), use it. Otherwise ask:

> What should this profile be called? (e.g. `deepseek`, `glm`, `kimi`, `my-custom`)

Normalize the name: lowercase, alphanumeric + hyphens only.

### Step 2: Check for known provider

If the name matches a known provider, pre-fill the base URL. The full list is in `references/provider-urls.md` — read it and match. If no match, treat as custom.

### Step 3: Collect required information

Ask these questions together (use `AskUserQuestion` with multiple fields):

1. **API Key** — the auth token for this provider
2. **Base URL** — pre-filled if known provider, otherwise ask
3. **Main model** — the model to use for most tasks (maps to `ANTHROPIC_MODEL` and `ANTHROPIC_DEFAULT_SONNET_MODEL`)
4. **Mode preference**:
   - **Quick** (recommended) — just the 3 fields above, everything else cloned from your current global config
   - **Full** — you'll paste a complete JSON config

If Full mode, the user will paste a complete JSON — jump to Step 7 (Validate).

### Step 4: Collect optional model overrides (Quick mode only)

Ask optionally (user can skip):

5. **Haiku-equivalent model** — smaller/faster model (defaults to main model)
6. **Opus-equivalent model** — strongest model (defaults to main model)
7. **Small/fast model** — for quick tasks (defaults to Haiku-equivalent)

### Step 5: Clone global config (Quick mode only)

Read `~/.claude/settings.json`. On Windows, `~` = `%USERPROFILE%`.

Extract these fields from the global config (do NOT copy `env`):
- `permissions`, `enabledPlugins`, `theme`, `statusLine`
- `extraKnownMarketplaces`, `hasCompletedOnboarding`
- `skipDangerousModePermissionPrompt`, `agentPushNotifEnabled`
- `model`

If `~/.claude/settings.json` does not exist, warn the user and proceed with minimal defaults:
```json
{
  "permissions": { "defaultMode": "acceptEdits" },
  "hasCompletedOnboarding": true
}
```

### Step 6: Build the profile JSON

Merge into this structure:

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "<base_url>",
    "ANTHROPIC_AUTH_TOKEN": "<api_key>",
    "ANTHROPIC_MODEL": "<main_model>",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "<haiku_or_main>",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "<main_model>",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "<opus_or_main>",
    "ANTHROPIC_SMALL_FAST_MODEL": "<fast_or_haiku>"
  },
  "...all cloned fields..."
}
```

The critical env var is `ANTHROPIC_DEFAULT_SONNET_MODEL` — Claude Code defaults to Sonnet for most tasks, so setting this "tricks" it into routing those requests to your chosen model.

### Step 7: Validate

- Check JSON is valid (`JSON.parse` equivalent)
- Check `env` object exists and has at minimum `ANTHROPIC_BASE_URL` and `ANTHROPIC_AUTH_TOKEN`
- Warn if any model field is empty (but don't block)

### Step 8: Save

Create directory `~/.claude/profiles/` if needed, then write to `~/.claude/profiles/<name>.json`.

If a file with the same name already exists, ask the user whether to overwrite.

### Step 9: Add shell integration

**Ask the user for confirmation before modifying any shell profile file.** Show exactly what will be appended and to which file. If the user declines, skip this step and show the manual command instead.

**Windows (PowerShell):**
```powershell
function claude-<name> { claude --settings "$env:USERPROFILE\.claude\profiles\<name>.json" @args }
```
Check if `$PROFILE` exists. If not, create the directory and file.
Before appending, search `$PROFILE` for the profile's JSON path (e.g. `profiles\<name>.json`) — if a function already references this profile (even under a different name like `claude-ds`), skip and report the existing shortcut name.

**macOS / Linux (bash/zsh):**
```bash
alias claude-<name>='claude --settings ~/.claude/profiles/<name>.json'
```
Check both `~/.bashrc` and `~/.zshrc`. Before appending, search for the profile's JSON path — if an alias already references it, skip and report the existing shortcut name.

### Step 10: Confirm

Tell the user:
- Profile saved at: `~/.claude/profiles/<name>.json`
- Shell shortcut: `claude-<name>`
- To activate: restart terminal or run `source ~/.zshrc` / `. $PROFILE`
- Test with: `claude-<name> --version`

### Step 11: Security reminder

After saving, remind the user:

> **Security**: Profile files contain API keys in plain text. Do not commit `~/.claude/profiles/` to version control. Consider adding it to your global `.gitignore`:
> ```
> git config --global core.excludesFile ~/.gitignore_global
> echo '.claude/profiles/' >> ~/.gitignore_global
> ```

## Error Handling

| Situation | Action |
|---|---|
| Profile name already exists | Ask to overwrite |
| `~/.claude/settings.json` missing | Warn, use minimal defaults |
| JSON invalid after merge | Show the error, let user fix |
| Cannot write to profiles dir | Show permissions error |
| Shell profile not found | Create it |

## Provider Quick Reference

For the full list, read `references/provider-urls.md`. Common providers:

| Name | Base URL |
|---|---|
| deepseek | `https://api.deepseek.com/anthropic` |
| glm | `https://api.z.ai/api/anthropic` |
| glm-cn | `https://open.bigmodel.cn/api/anthropic` |
| kimi | `https://api.moonshot.ai/anthropic` |
| kimi-cn | `https://api.moonshot.cn/anthropic` |
| qwen | `https://coding-intl.dashscope.aliyuncs.com/apps/anthropic` |
| minimax | `https://api.minimaxi.com/anthropic` |
| doubao | `https://ark.cn-beijing.volces.com/api/anthropic` |
| openrouter | `https://openrouter.ai/api/anthropic` |
| claude | `https://api.anthropic.com` |
| ollama | `http://localhost:11434/anthropic` |
