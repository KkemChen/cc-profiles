# Known Provider Base URLs

When the user says "add a [provider] profile", use this table to pre-fill the Base URL.

| Provider | Base URL | Models (snapshot) |
|---|---|---|
| **DeepSeek** | `https://api.deepseek.com/anthropic` | `deepseek-chat`, `deepseek-reasoner` |
| **GLM (global)** | `https://api.z.ai/api/anthropic` | `glm-4`, `glm-4-flash` |
| **GLM (China)** | `https://open.bigmodel.cn/api/anthropic` | `glm-4`, `glm-4-flash` |
| **Kimi (global)** | `https://api.moonshot.ai/anthropic` | `kimi-k2.5`, `kimi-k2-turbo` |
| **Kimi (China)** | `https://api.moonshot.cn/anthropic` | `kimi-k2.5` |
| **Qwen** | `https://coding-intl.dashscope.aliyuncs.com/apps/anthropic` | `qwen3-max` |
| **MiniMax** | `https://api.minimaxi.com/anthropic` | `MiniMax-M2.5` |
| **Doubao/Seed** | `https://ark.cn-beijing.volces.com/api/anthropic` | `doubao-seed-2-0-code` |
| **Claude Official** | `https://api.anthropic.com` | `claude-sonnet-4-5-20250929` |
| **OpenRouter** | `https://openrouter.ai/api/anthropic` | Any model on OpenRouter |
| **Ollama (local)** | `http://localhost:11434/anthropic` | Any local model (requires Ollama with Anthropic-compatible proxy) |

## Notes

- All URLs above are the Anthropic-compatible endpoints (not OpenAI format). These providers have built Anthropic API-compatible proxies.
- If the user uses a custom aggregator (OneAPI, NewAPI, LiteLLM), the base URL is their own deployment URL — do not override it.
- Model names change frequently. Ask the user to confirm or provide the model name they want.
