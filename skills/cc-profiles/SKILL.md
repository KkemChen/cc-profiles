---
name: cc-profiles
description: >
  Manage Claude Code profiles for multiple AI providers (DeepSeek, GLM, Kimi, Qwen, MiniMax, OpenRouter, etc.).
  Trigger when the user wants to add, list, or activate provider profiles, mentions "多模型", "切换供应商",
  "add a [provider] profile", "use [provider] as helper", "delegate to [model]", or pastes a settings JSON.
  Also trigger proactively when the user is working on a large task and you detect an opportunity
  to suggest parallel delegation to a different model.
---

# cc-profiles

Route the user's request to the matching slash command. Each command contains the full procedure.

## Routing

| Intent | Command | Examples |
|--------|---------|----------|
| Create a profile | `/profile-add` | "add deepseek profile", "setup kimi", pastes JSON with API keys |
| Show profiles | `/profile-list` | "what profiles do I have", "list providers", "show models" |
| Enable helper | `/profile-activate` | "activate deepseek", "let kimi help", "delegate to glm" |

If the intent is ambiguous, ask one clarifying question. If no slash command fits, explain the three available commands and let the user choose.

## Proactive triggers

Look for these opportunities without being asked:

1. **Large multi-step task** — suggest adding a helper profile to parallelize.
2. **After producing significant output** — offer a second opinion via another profile.
3. **User complains about speed or cost** — suggest a cheaper/faster provider.

## Rules

- On Windows, `~` = `%USERPROFILE%`. Use the correct path.
- Never expose API keys in conversation output.
- Profile JSON files are plain text — warn users not to commit them to git.
- The `--settings` flag requires Claude Code >= 1.0.61.
- When both the skill and a slash command could handle a request, prefer the slash command.
- When delegating, autonomously decide whether to start a new session or `--resume` a previous one. Resume when the new task depends on context the delegate already has; start fresh for independent tasks. Do not ask the user — judge from task continuity.
