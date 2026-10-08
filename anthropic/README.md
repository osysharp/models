# Osysharp.Models.Anthropic — the Anthropic Messages API as a model client

The class that talks to Anthropic, **written in Osy#**. Register it on the models it serves and every call to those
models — an agent turn, `Llm.Extract`, a `stream<T>` — goes through it. The platform names no provider anywhere.

## Use case

You want to reach a model. The usual answer is that your platform ships support for some providers and you wait for the
rest — which means the provider released last week, the gateway your company routes everything through, and the model
you host yourself are all "not supported yet".

This kit is the answer instead: **a provider is a class.** It is ordinary Osy# over `Http.*` and `JsonSerializer`,
using nothing an app cannot use, so you can read all 250 lines of it, copy it, and point the copy at whatever speaks a
similar wire. What it implements — `Osysharp.Llm.ILlmClient`, two members — is the only thing the platform knows.

It has no user interface, so there is nothing to screenshot: it is where a model call goes.

## Install and use

```osy
// app.osy — a package gets no authority from being depended on, so the host is granted here or the app does not compile
app Support {
  model "**/*.osy";
  use Osysharp.Models.Anthropic { egress "api.anthropic.com"; }
}
```

```osy
// config.osy
using Osysharp.Models.Anthropic;

app.Secrets = [ new Secret("Anthropic") ];

app.Models = [
  new LlmConfig("Fast")   { Client = new AnthropicClient { ApiKey = Secret.Anthropic }, Model = "claude-haiku-4-5" },
  new LlmConfig("Strong") { Client = new AnthropicClient { ApiKey = Secret.Anthropic }, Model = "claude-sonnet-5" },
];

app.DefaultModel = Llm.Fast;
```

Then `osy secret set Anthropic`, and you are done. Which model serves which task is a separate decision —
see `Osysharp.LlmRouter`.

## Settings

| | |
|---|---|
| `ApiKey` | your Anthropic key. Declared here so the app names its own secret; this package never learns which. |
| `BaseUrl` | the API root. Anthropic's own unless you are pointed at a proxy that speaks the same wire. |
| `MaxTokens` | what to ask for when the request does not say. Anthropic REQUIRES a number, so there is always one. |

## What it does, and what it deliberately does not

- **Prompt caching.** A breakpoint is placed at the end of each cacheable stability tier of the system prompt, and on
  the last tool when the request asks for it. Both are HINTS on the seam — a provider without prompt caching ignores
  them correctly — and this one honours them.
- **It does not choose a model.** `request.Model` arrives already decided by the app's `ILlmRouter`. A provider that
  second-guessed the decision would make every routing policy advisory.
- **It does not police spend.** The kill switch, the budget reservation, the metering row and the cost rollup all
  happen above the seam, from the token counts reported here. So it reports what Anthropic reported, never an estimate.
- **Sampling parameters are dropped for models that removed them** (`temperature`, `top_p`). The list of models that
  still accept them is an ALLOWLIST, not a denylist, and the direction is the point: the set is closed and shrinking,
  while new models keep arriving without them. A denylist is wrong the day a model ships, and wrong in the worst
  direction — every call 400s.

## Source

`model/AnthropicClient.osy` is the client; `model/AnthropicWire.osy` is the request body as a typed class plus
Anthropic's response JSON, with `[ExternalName]` on each field so the class names stay Osy#'s and the wire stays
Anthropic's.

Tested: the platform's own suite asserts the exact bytes of the request that reaches the wire, not just that the kit
compiles.
