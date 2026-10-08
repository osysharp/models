# Osysharp.Models.Gemini — the Google Gemini API as a model client

The class that talks to Gemini, **written in Osy#**. Register it on the models it serves and every call to those
models — an agent turn, `Llm.Extract`, a `stream<T>` — goes through it. The platform names no provider anywhere.

## Use case

You want to reach a model. The usual answer is that your platform ships support for some providers and you wait for the
rest — which means the provider released last week, the gateway your company routes everything through, and the model
you host yourself are all "not supported yet".

This kit is the answer instead: **a provider is a class.** It is ordinary Osy# over `Http.*` and `JsonSerializer`,
using nothing an app cannot use, so you can read all of it and copy it. What it implements —
`Osysharp.Llm.ILlmClient`, two members — is the only thing the platform knows.

It is also the kit that proves the seam is not Anthropic's shape with other names on it. Gemini diverges more than any
of the three, and all of it stays inside this class:

| Gemini | the seam |
|---|---|
| roles are `user` / `model` | `"user"` / `"assistant"` |
| the system prompt is a separate `system_instruction` | a `SystemPrompt` / `SystemSegments` on the request |
| the model id travels in the URL **path** | a `Model` on the request |
| a tool result is keyed by the function's **name** — there are no call ids | a `ToolUseId` pairing a call with its result |
| a whole function call arrives in one part, with no argument deltas | `ToolUseStart` → `ToolInputDelta` → `ToolUseEnd` |

It has no user interface, so there is nothing to screenshot: it is where a model call goes.

## Install and use

```osy
// app.osy — a package gets no authority from being depended on, so the host is granted here or the app does not compile
app Support {
  model "**/*.osy";
  use Osysharp.Models.Gemini { egress "generativelanguage.googleapis.com"; }
}
```

```osy
// config.osy
using Osysharp.Models.Gemini;

app.Secrets = [ new Secret("Gemini") ];

app.Models = [
  new LlmConfig("Fast")   { Client = new GeminiClient { ApiKey = Secret.Gemini }, Model = "gemini-2.5-flash" },
  new LlmConfig("Strong") { Client = new GeminiClient { ApiKey = Secret.Gemini }, Model = "gemini-2.5-pro" },
];

app.DefaultModel = Llm.Fast;
```

Then `osy secret set Gemini`, and you are done. Which model serves which task is a separate decision —
see `Osysharp.LlmRouter`.

## Settings

| | |
|---|---|
| `ApiKey` | your Gemini key. Declared here so the app names its own secret; this package never learns which. |
| `BaseUrl` | the API root. Google's own unless you are pointed at a proxy that speaks the same wire. |
| `ApiVersion` | `v1beta` (the default) carries the tool surface this kit uses; `v1` is the stable subset. |
| `MaxTokens` | what to ask for when the request does not say. Gemini does not require a bound; this sends one anyway, because an unbounded answer is an unbounded bill. |

## What it does, and what it deliberately does not

- **A tool call wins over the finish reason.** Gemini answers `STOP` on a turn that asked for a tool, so reading the
  reason alone would tell the platform the model was finished while its answer is a tool call waiting to be run.
- **A tool result is wrapped when it must be.** Gemini requires an object at `functionResponse.response`: an object is
  passed through, an array is wrapped under `result`, and anything that is not JSON — a number, a sentence — is
  wrapped as a JSON string, which is the honest reading of it.
- **The cached prefix is not charged twice.** Gemini counts cached tokens INSIDE `promptTokenCount`, so the kit reports
  `promptTokenCount − cachedContentTokenCount` as input and the cached count beside it.
- **It does not choose a model.** `request.Model` arrives already decided by the app's `ILlmRouter`. A provider that
  second-guessed the decision would make every routing policy advisory.
- **It does not police spend.** The kill switch, the budget reservation, the metering row and the cost rollup all
  happen above the seam, from the token counts reported here. So it reports what Gemini reported, never an estimate.
- **No prompt-cache breakpoints**, because Gemini places none from the request: the structured system segments join in
  order. Those hints are hints, and a provider that cannot act on them is correct to ignore them.
- **No explicit context caching** (Gemini's `cachedContent` API). The implicit cache is read and reported; managing a
  cache resource is a separate feature with its own lifetime, and it is not something a provider should do behind an
  app's back.

## Source

`model/GeminiClient.osy` is the client; `model/GeminiWire.osy` is the request body as a typed class plus Gemini's
response JSON, with `[ExternalName]` on each field so the class names stay Osy#'s and the wire stays Google's.

Tested: the platform's own suite asserts the exact bytes of the request that reaches the wire, not just that the kit
compiles.
