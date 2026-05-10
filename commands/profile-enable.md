---
name: profile-enable
description: Enable one or more AI provider profiles as helpers for the current session
argument-hint: "[profile-name ...]"
allowed-tools: Bash, Read, Glob, AskUserQuestion
---

# /profile-enable

Enable specific provider profiles so the current session's AI knows it can delegate tasks to them. This does NOT switch your main model — it makes additional models available as helpers.

## Procedure

### Step 1: Determine target profiles

If the user provided names as arguments (e.g. `/profile-enable deepseek kimi`), use those. If no argument, scan `~/.claude/profiles/` and present available profiles for the user to pick from.

### Step 2: Validate each profile

For each profile name, read `~/.claude/profiles/<name>.json`. Verify:
- File exists and is valid JSON
- Has `env.ANTHROPIC_BASE_URL` and `env.ANTHROPIC_AUTH_TOKEN`
- Has at minimum `env.ANTHROPIC_MODEL` or `env.ANTHROPIC_DEFAULT_SONNET_MODEL`

Skip and warn about any profile that fails validation.

If the profile has not been smoke-tested in this session, suggest `/profile-test <name>` first — but proceed if the user wants to skip.

### Step 3: Build the enabling prompt

For each valid profile, extract:
- Profile name
- Provider: match `ANTHROPIC_BASE_URL` against `references/provider-urls.md`. **If no match, use the URL's host as the label** (e.g. `oneapi.example.com` → `oneapi.example.com`). Do not call it "Unknown" — custom aggregators are common.
- Primary model: `ANTHROPIC_MODEL` or `ANTHROPIC_DEFAULT_SONNET_MODEL`
- Fast model: `ANTHROPIC_DEFAULT_HAIKU_MODEL` (if set)
- Strong model: `ANTHROPIC_DEFAULT_OPUS_MODEL` (if set)
- Mode hint: if `enabledPlugins` is `{}` and `mcpServers` is `{}`, label as **Slim**; otherwise **Mirror**

Generate this awareness prompt and output it directly into the conversation:

````
## Profile Enabled: <name> (<Slim|Mirror>)

**Provider**: <provider> | **Model**: <model>
**Profile path**: `~/.claude/profiles/<name>.json` (Windows: `%USERPROFILE%\.claude\profiles\<name>.json`)

You can delegate tasks to this profile using the `Bash` tool. Choose the right pattern for the task.

---

#### Pattern A — Short single task (under ~30s, ≤3 turns)

```bash
claude --settings ~/.claude/profiles/<name>.json \
  -p "TASK" \
  --max-turns 5 \
  --output-format json \
  --allowedTools "Read,Grep,Glob"
```

Best for: lookups, formatting, single-file rewrites, "explain this regex" — anything where you don't need to watch progress and the answer fits in one turn.

`--allowedTools` whitelist keeps the system prompt smaller and prevents the delegate from going off into WebFetch / nested Agent calls. Drop the flag if you want the full toolbox.

---

#### Pattern B — Longer task with progress visibility

For multi-turn work (5+ tool calls, file edits, anything you'd want to watch), use streaming output redirected to a log file plus a Monitor tail:

```bash
# 1. Launch in background, stream every event to a log
LOG="delegate-$(date +%s).log"
touch "$LOG"
claude --settings ~/.claude/profiles/<name>.json \
  --output-format stream-json --verbose \
  --allowedTools "Read,Edit,Grep,Glob,Bash" \
  --max-turns 15 \
  -p "TASK" > "$LOG" 2>&1 &

# 2. Tail with a filter (run via the Monitor tool for live notifications)
tail -F "$LOG" | grep --line-buffered '"type":"' | while IFS= read -r line; do
  case "$line" in
    *'"type":"system"'*'init'*) echo "[init] $(echo "$line" | grep -oE '"session_id":"[^"]+"')" ;;
    *'"thinking"'*) echo "[think] ..." ;;
    *'tool_use'*)   echo "[call] $(echo "$line" | grep -oE '"name":"[A-Z][a-zA-Z]+","input":\{[^}]{0,200}')" ;;
    *'"text"'*)     echo "[say] $(echo "$line" | grep -oE '"text":"[^"]{1,240}')" ;;
    *'tool_result'*) echo "[ret]" ;;
    *'"type":"result"'*) echo "[done] $(echo "$line" | grep -oE '"total_cost_usd":[0-9.]+|"num_turns":[0-9]+|"is_error":(true|false)' | tr '\n' ' ')"; break ;;
  esac
done
```

Why: `--output-format json` (Pattern A) only emits at completion — long tasks look hung. `stream-json` emits one line per event, so the user sees `[call] Read foo.lua → [ret] → [call] Edit foo.lua → ...` in real time.

---

#### Pattern C — Multi-step workflow (use cache!)

The first delegation pays the full system-prompt cost (~$0.20–0.30 cold start). Every subsequent call that resumes the same session reads the prompt from cache (~$0.02). Use `--resume` aggressively for related tasks:

```bash
# First call: capture session_id
RESULT=$(claude --settings ~/.claude/profiles/<name>.json \
  -p "Read app/lua/pet.lua and summarize" \
  --max-turns 5 --output-format json)
SID=$(echo "$RESULT" | grep -oE '"session_id":"[^"]+"' | head -1 | cut -d'"' -f4)
echo "Session: $SID"

# Follow-up: pay only for new tokens
claude --settings ~/.claude/profiles/<name>.json \
  --resume "$SID" \
  -p "Now do the same for pickup.lua and compare" \
  --max-turns 5 --output-format json
```

Rule of thumb: for any chain of 3+ related delegations, resume saves 80%+ of the total cost. The `session_id` is stable across resumes — chain them indefinitely until the task changes domains.

---

#### Reporting back to the user

After every delegation, surface the cost and outcome — these come back free in the JSON:

- `total_cost_usd` — what this call cost
- `usage.cache_read_input_tokens` — non-zero means cache hit (great)
- `num_turns` — how many tool calls the delegate made
- `is_error` — false means success

Example one-liner: *"Done. <model> took 4 turns / 12.3s / $0.04 (cache hit on 43K tokens)."*

Cost transparency lets the user decide whether to keep delegating or switch tactics.

---

#### Session management — when to resume

Decide autonomously whether a delegation needs session continuity:

- **New session** (default): one-off tasks — code generation, formatting, lookup, review. No context needed from prior delegations.
- **Resume session**: multi-step workflows where the delegate has already built up context — e.g., first "read this codebase", then "refactor module X". Restarting would waste tokens re-explaining.

When resuming, extract `session_id` from the previous delegation's JSON output and pass it via `--resume`. Do not ask the user whether to resume — judge from the task's relationship to prior delegations.

---

[IF MULTIPLE PROFILES ENABLED, ADD A COMPARISON TABLE:]

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
- Custom aggregators (OneAPI/NewAPI/LiteLLM) → whatever the underlying model is — ask user

---

#### Safety
- Never include API keys in delegated prompts — they're in the profile JSON
- Always set `--max-turns` to prevent runaway loops (5 for lookup, 10 for simple, 15–30 for code)
- Always set `--allowedTools` for delegation — prevents WebFetch / nested Agent / unintended Bash. Drop only if the task genuinely needs the full toolbox.
- Verify delegated output before applying it — especially diffs and code edits
- Mirror profiles inherit `bypassPermissions` if the parent had it; double-check `permissions.defaultMode` for sensitive tasks
````

### Step 4: Proactive collaboration offer

After displaying the enabling prompt, ask proactively based on context:

If the user has an active task in the conversation:
> "I notice we're working on [task summary]. Want me to delegate [specific subtask] to **<name>** to speed things up?"

If no active task:
> "**<name>** is now available as a helper. Tell me what you'd like to delegate, or I'll suggest opportunities as we work."

### Step 5: Track enabled profiles

Maintain a mental note of which profiles are enabled in this session. When the user later asks to delegate, reference the already-enabled profiles rather than re-reading them. Track session_ids across delegations so you can `--resume` for follow-up tasks without re-asking the user.

## Error Handling

| Situation | Action |
|---|---|
| Profile not found | "No profile named `<name>`. Run `/profile-list` to see available profiles, or `/profile-add <name>` to create one." |
| Profile JSON invalid | Show the specific error, suggest re-creating with `/profile-add` |
| No profiles at all | "No profiles configured yet. Start with `/profile-add <provider>` to add your first one." |
| Profile lacks model field | Warn but proceed with whatever model info is available |
| Profile never smoke-tested | Suggest `/profile-test <name>` before delegating real work |

## Examples

```
/profile-enable deepseek
```
→ Enables DeepSeek profile, AI now knows the three delegation patterns and will pick the right one for each task.

```
/profile-enable deepseek kimi
```
→ Enables both, shows comparison table, suggests which profile suits which task type.

```
/profile-enable
```
→ No argument: scans profiles dir, lets user pick interactively.
