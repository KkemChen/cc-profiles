---
name: profile-add
description: Add a new AI provider profile to your Claude Code team
argument-hint: "[provider-name]"
allowed-tools: Bash, Read, Write, Edit, AskUserQuestion
---

# /profile-add

Create an isolated Claude Code settings profile for switching between AI providers via `--settings`.

## Step 0: Detect a pasted JSON config (Full mode shortcut)

**Before asking anything else**, scan the user's invocation arguments and any prior message in this turn for a JSON object containing `env.ANTHROPIC_BASE_URL` (or `"ANTHROPIC_BASE_URL"` as a key). If found:

1. Treat this as **Full mode** — the user is providing a complete settings JSON.
2. Skip Steps 1–5; ask only for the **profile name** (one short question), then jump to Step 7 (Validate) and Step 8 (Save) using the pasted JSON verbatim.
3. Do **not** merge or modify the pasted JSON — the user gave you what they want.

This auto-detection prevents a common mistake: users paste a JSON in answer to "what should this profile be called?" and the workflow misroutes it as a name.

## Step 1: Collect profile name

If the user provided a name as argument (e.g. `/profile-add deepseek`), use it. Otherwise ask:

> What should this profile be called? (e.g. `deepseek`, `glm`, `kimi`, `my-custom`)

Normalize the name: lowercase, alphanumeric + hyphens only.

## Step 2: Check for known provider

If the name matches a known provider, pre-fill the base URL. The full list is in `references/provider-urls.md` — read it and match. If no match, treat as custom.

## Step 3: Collect required information

Ask these questions together (use `AskUserQuestion` with multiple fields):

1. **API Key** — the auth token for this provider
2. **Base URL** — pre-filled if known provider, otherwise ask
3. **Main model** — the model to use for most tasks (maps to `ANTHROPIC_MODEL` and `ANTHROPIC_DEFAULT_SONNET_MODEL`)
4. **Mode preference** — pick one:
   - **Slim** ⭐ (recommended, default) — minimal profile: env + permissions only. Best for delegation: ~33% smaller system prompt → ~$0.10 cheaper per cold start, faster startup, no plugin side-effects.
   - **Mirror** — clone your global plugins, MCP servers, statusLine, marketplace config. Use this if you intend to launch this profile as your primary harness (`claude --settings ...`), not just for delegation.
   - **Full** — paste a complete JSON config. (Auto-selected in Step 0 if a JSON was already pasted.)

If Full mode, jump to Step 7 (Validate).

## Step 4: Collect optional model overrides (Slim and Mirror modes)

Ask optionally (user can skip):

5. **Haiku-equivalent model** — smaller/faster model (defaults to main model)
6. **Opus-equivalent model** — strongest model (defaults to main model)
7. **Small/fast model** — for quick tasks (defaults to Haiku-equivalent)

## Step 5: Build the profile JSON

### Step 5a — Slim mode (default)

Build this minimal structure. **Do not** read or merge `~/.claude/settings.json`:

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
  "permissions": {
    "defaultMode": "bypassPermissions"
  },
  "hasCompletedOnboarding": true,
  "skipDangerousModePermissionPrompt": true,
  "enabledPlugins": {},
  "mcpServers": {}
}
```

**Why empty `enabledPlugins` / `mcpServers`** — these are explicit empty objects (not omitted) so the delegated session does not inherit them from the parent harness's defaults. This is the single biggest cost saver: plugins and MCP server schemas can add 20K+ tokens to every cold start.

### Step 5b — Mirror mode

Read `~/.claude/settings.json` (Windows: `%USERPROFILE%\.claude\settings.json`). Extract:
- `permissions`, `enabledPlugins`, `theme`, `statusLine`
- `mcpServers`, `extraKnownMarketplaces`, `hasCompletedOnboarding`
- `skipDangerousModePermissionPrompt`, `agentPushNotifEnabled`, `model`

Merge with the env block from Step 5a. Result:

```json
{
  "env": { "...same as Slim..." },
  "...all cloned fields from global settings..."
}
```

If `~/.claude/settings.json` does not exist, fall back to Slim mode and warn the user.

### Critical env var note

`ANTHROPIC_DEFAULT_SONNET_MODEL` is the most important field. Claude Code defaults to Sonnet for most tasks, so setting this routes those requests to your chosen model. Always set it equal to your main model.

## Step 6: (reserved — formerly Step 6 logic moved into Step 5)

## Step 7: Validate

- Check JSON is valid (parses without error)
- Check `env` object exists with at minimum `ANTHROPIC_BASE_URL` and `ANTHROPIC_AUTH_TOKEN`
- Warn if any model field is empty (but don't block)
- For Slim mode: confirm `enabledPlugins` and `mcpServers` are present as `{}` (not missing) — this is the cost-saving feature

## Step 8: Save

1. Create directory `~/.claude/profiles/` if it doesn't exist (Windows: `%USERPROFILE%\.claude\profiles\`).
2. **Auto-create `.gitignore` inside the profiles directory** if missing:
   ```bash
   # Write to ~/.claude/profiles/.gitignore
   *
   !.gitignore
   ```
   This protects against accidental commits if the parent `~/.claude/` is ever git-tracked. Do this every time even if just one profile already exists — the file is idempotent.
3. Write the profile JSON to `~/.claude/profiles/<name>.json`.
4. If a file with the same name already exists, ask the user whether to overwrite.

## Step 9: Confirm

Tell the user:
- Profile saved at: `~/.claude/profiles/<name>.json`
- Mode used: **Slim** / **Mirror** / **Full**
- **Suggest a smoke test**: `/profile-test <name>` — verifies the API key, base URL, and model name actually work before they spend real money on a long task.
- To enable as a delegation helper: `/profile-enable <name>`
- To use as primary harness: `claude --settings ~/.claude/profiles/<name>.json`

## Step 10: Security reminder

After saving, remind the user:

> **Security**: Profile files contain API keys in plain text. The plugin auto-creates a `.gitignore` inside `~/.claude/profiles/` to block accidental commits *within* that directory, but if you ever commit your entire `~/.claude/` directory or copy profiles elsewhere, that protection won't follow. For belt-and-suspenders safety, also add to your global gitignore:
> ```bash
> git config --global core.excludesFile ~/.gitignore_global
> echo '.claude/profiles/' >> ~/.gitignore_global
> ```

## Error Handling

| Situation | Action |
|---|---|
| Profile name already exists | Ask to overwrite |
| `~/.claude/settings.json` missing (Mirror mode) | Fall back to Slim mode, warn |
| JSON invalid after merge | Show the error, let user fix |
| Cannot write to profiles dir | Show permissions error |
| User pasted JSON without `env.ANTHROPIC_BASE_URL` | Treat as malformed Full input — ask user to provide a valid settings JSON |

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
