---
name: profile-activate
description: Enable one or more AI provider profiles as helpers for the current session
argument-hint: "[profile-name ...]"
allowed-tools: Bash, Read, Glob, AskUserQuestion
---

# /profile-activate

Enable specific provider profiles so the current session's AI knows it can delegate tasks to them. This does NOT switch your main model — it makes additional models available as helpers.

## Procedure

### Step 1: Determine target profiles

If the user provided names as arguments (e.g. `/profile-activate deepseek kimi`), use those. If no argument, scan `~/.claude/profiles/` and present available profiles for the user to pick from.

### Step 2: Validate each profile

For each profile name, read `~/.claude/profiles/<name>.json`. Verify:
- File exists and is valid JSON
- Has `env.ANTHROPIC_BASE_URL` and `env.ANTHROPIC_AUTH_TOKEN`
- Has at minimum `env.ANTHROPIC_MODEL` or `env.ANTHROPIC_DEFAULT_SONNET_MODEL`

Skip and warn about any profile that fails validation.

### Step 3: Build the activation prompt

For each valid profile, extract:
- Profile name
- Provider: infer from `ANTHROPIC_BASE_URL` (e.g. `api.deepseek.com` → DeepSeek, `api.anthropic.com` → Claude Official, `api.z.ai` → GLM)
- Primary model: `ANTHROPIC_MODEL` or `ANTHROPIC_DEFAULT_SONNET_MODEL`
- Fast model: `ANTHROPIC_DEFAULT_HAIKU_MODEL` (if set)
- Strong model: `ANTHROPIC_DEFAULT_OPUS_MODEL` (if set)

Generate this awareness prompt and output it directly into the conversation:

```
## Profile Enabled: <name>

**Provider**: <provider> | **Model**: <model>
**Profile path**: `~/.claude/profiles/<name>.json` (Windows: `%USERPROFILE%\.claude\profiles\<name>.json`)

You can delegate tasks to this profile using the `bash` tool:

**Single task (no context needed):**
```bash
claude --settings ~/.claude/profiles/<name>.json -p "TASK" --max-turns 30 --output-format json
```

**Resume a previous session (preserve context):**
```bash
claude --settings ~/.claude/profiles/<name>.json --resume <session-id> -p "FOLLOW-UP TASK" --max-turns 30 --output-format json
```

Key flags:
- `-p "..."` — non-interactive single task
- `--max-turns N` — cap tool calls (30 for code, 10 for simple, 5 for lookup)
- `--output-format json` — structured output (response includes `session_id` for future resume)
- `--resume <id>` — resume a previous session to retain context

[IF MULTIPLE PROFILES ACTIVATED, ADD A COMPARISON TABLE:]
| Profile | Best for |
|---------|----------|
| <name1> (<model>) | <suggested use> |
| <name2> (<model>) | <suggested use> |

Suggested use is inferred from:
- Claude Official → architecture review, code review, security audit
- DeepSeek → bulk code generation, fast iteration
- GLM/Kimi → general coding, Chinese context
- MiniMax → cost-efficient bulk tasks
- OpenRouter → model experimentation
- Ollama → offline/local tasks

### Session management

Decide autonomously whether a delegation needs session continuity:

- **New session** (default): one-off tasks — code generation, formatting, lookup, review. No context needed from prior delegations.
- **Resume session**: multi-step workflows where the delegate has already built up context — e.g., first "read this codebase", then "refactor module X". Restarting would waste tokens re-explaining.

When resuming, extract `session_id` from the previous delegation's JSON output and pass it via `--resume`. Do not ask the user whether to resume — judge from the task's relationship to prior delegations.

### Safety
- Never include API keys in delegated prompts — they're in the profile JSON
- Always set `--max-turns` to prevent runaway loops
- Verify delegated output before applying it
```

### Step 4: Proactive collaboration offer

After displaying the activation prompt, ask proactively based on context:

If the user has an active task in the conversation:
> "I notice we're working on [task summary]. Want me to delegate [specific subtask] to **<name>** to speed things up?"

If no active task:
> "**<name>** is now available as a helper. Tell me what you'd like to delegate, or I'll suggest opportunities as we work."

### Step 5: Track activated profiles

Maintain a mental note of which profiles are activated in this session. When the user later asks to delegate, reference the already-activated profiles rather than re-reading them.

## Error Handling

| Situation | Action |
|---|---|
| Profile not found | "No profile named `<name>`. Run `/profile-list` to see available profiles, or `/profile-add <name>` to create one." |
| Profile JSON invalid | Show the specific error, suggest re-creating with `/profile-add` |
| No profiles at all | "No profiles configured yet. Start with `/profile-add <provider>` to add your first one." |
| Profile lacks model field | Warn but proceed with whatever model info is available |

## Examples

```
/profile-activate deepseek
```
→ Enables DeepSeek profile, AI now knows it can offload to deepseek-v4-pro.

```
/profile-activate deepseek kimi
```
→ Enables both, shows comparison table, suggests which profile suits which task type.

```
/profile-activate
```
→ No argument: scans profiles dir, lets user pick interactively.
