# osysharp/models

Osy# packages, one per folder. Each folder is a package of its own, with its own README, tests and releases (tagged `<folder>/v<version>`).

| Folder | Package | What it is |
|---|---|---|
| [`anthropic/`](anthropic/) | `Osysharp.Models.Anthropic` | the Anthropic Messages API as a model provider for Osysharp.Llm — complete and stream, with prompt caching, tool choice and structured output. |
| [`gemini/`](gemini/) | `Osysharp.Models.Gemini` | the Google Gemini API as a model provider for Osysharp.Llm — complete and stream, tools included. |
| [`openai/`](openai/) | `Osysharp.Models.OpenAi` | the OpenAI Chat Completions API as a model provider for Osysharp.Llm — and any endpoint that speaks its wire. |

## Learn more

- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
