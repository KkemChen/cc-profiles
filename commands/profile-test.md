---
name: profile-test
description: Run a quick connectivity probe against a profile to verify auth, base URL, and model
argument-hint: "[profile-name]"
allowed-tools: Bash, Read, Glob
---

# /profile-test

Verify a profile actually works by sending a 1-turn no-tool probe. Use this:
- Right after `/profile-add` to catch typos in API key / base URL / model name
- Whenever a delegation hangs or behaves weirdly
- Before kicking off a long workflow on an unfamiliar gateway

A probe takes ~5–10 seconds and ~$0.10–0.30 (one cold start) per run.

## Procedure

### Step 1: Determine target profile

If the user provided a name, use it. Otherwise scan `~/.claude/profiles/` for `*.json` files:
- One profile → use it
- Multiple profiles → ask which one (or accept `--all` to test each in sequence)
- None → tell user to run `/profile-add` first

### Step 2: Validate the profile file before spending money

Read `~/.claude/profiles/<name>.json`. Bail out with a clear error if any of these are missing:
- Valid JSON syntax
- `env.ANTHROPIC_BASE_URL`
- `env.ANTHROPIC_AUTH_TOKEN`
- `env.ANTHROPIC_MODEL` OR `env.ANTHROPIC_DEFAULT_SONNET_MODEL`

These checks are free and catch the most common breakage (truncated copy/paste, wrong field names).

### Step 3: Run the probe

Use Bash with a hard 90-second timeout (longer than any healthy gateway should ever take to respond to a 1-token completion):

```bash
claude --settings ~/.claude/profiles/<name>.json -p "ok" --max-turns 1 --output-format json
```

If the call exits with non-zero status or hits the 90s timeout, jump to Step 5 (diagnose).

### Step 4: Parse and report

Extract from the JSON output:
- `is_error`, `result`
- `duration_ms`, `duration_api_ms`
- `usage.input_tokens`, `usage.output_tokens`, `usage.cache_read_input_tokens`
- `total_cost_usd`
- `modelUsage[<model>].contextWindow`, `.maxOutputTokens`

Format like this:

```
## Profile Test: <name> ✅

- **Provider**: <provider> (<base_url_host>)
- **Model**: <model> — context <ctxWindow> / max output <maxOut>
- **Latency**: <duration_ms>ms total (API: <duration_api_ms>ms)
- **Tokens**: <input> in / <output> out
- **Cost (this probe)**: $<cost>
- **Status**: <result excerpt>
```

⚠️ **If `usage.input_tokens` > 50,000**, append a warning:
> Profile carries plugin/MCP/statusLine clones — adds ~$0.10 per cold start.
> Recreate with default slim mode: `/profile-add <name>`.

### Step 5: Diagnose failures

When the probe fails, classify by symptom and tell the user the likely fix:

| Symptom | Likely cause | Fix |
|---|---|---|
| Hits 90s timeout, no output | Wrong base URL, blocked by firewall, or model name invalid (some gateways hang silently on 404 instead of erroring) | Verify base URL is reachable: `curl -I <base_url>`. Re-check model name against provider docs. |
| HTTP 401 / 403 / `Invalid API Key` in result | API key wrong, expired, or wrong tier | Edit profile JSON, replace `ANTHROPIC_AUTH_TOKEN` |
| HTTP 404 / `model not found` | Model name not supported by this gateway | Check provider docs; aggregators (OneAPI/NewAPI) often use Anthropic-style aliases like `claude-sonnet-4-5` |
| HTTP 429 / `rate_limit` | Provider quota exceeded | Wait or check dashboard |
| `is_error: true` with non-zero `total_cost_usd` | Model errored mid-response (gateway charged anyway) | Read `result` for the upstream error message |
| Empty `result` but `is_error: false` | Gateway returned success with empty body — likely a routing issue | Try a different model name on the same profile |

For any failure, also print the exact command you ran so the user can retry manually with `--verbose --output-format stream-json` for deeper diagnostics.

### Step 6: Suggest next steps

- ✅ **Pass** → `/profile-enable <name>` to start using it as a delegation target
- ❌ **Fail (auth/network)** → `/profile-add <name>` to recreate with corrected fields
- ⚠️ **Pass but bloated** → `/profile-add <name>` and choose **Slim** mode this time

## Examples

```
/profile-test deepseek
```
→ One profile, probe runs, reports OK with cost.

```
/profile-test
```
→ No argument: picks the most recently modified profile, or asks if there are several.

```
/profile-test deepseek glm kimi
```
→ Probes each in sequence. Useful after a marathon `/profile-add` session to catch any typos.
