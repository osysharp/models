# Osysharp.Models.Gemini — the Google Gemini API as a model client

The class that talks to Gemini, **written in Osy#**. Name it on the models it serves and every `Llm.Complete`,
`Llm.Stream`, `Llm.Extract<T>` and `Llm.Decide<T>` call to those models goes through it. Osysharp.Llm names no provider
anywhere.

## Use case

You want to reach a model. The usual answer is that your tooling ships support for some providers and you wait for the
rest — which means the provider released last week, the gateway your company routes everything through, and the model
you host yourself are all "not supported".

This kit is the answer instead: **a provider is a class.** It is ordinary Osy# over `Http.*` and `JsonSerializer`,
using nothing an app cannot use, so you can read all of it and copy it. What it implements —
`Osysharp.Llm.ILlmClient`, two members — is the only thing the Llm kit knows.

It is also the kit that shows the contract is not Anthropic's shape with other names on it. Gemini diverges more than
any of the three, and all of it stays inside this class:

| Gemini | the contract |
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
  use Osysharp.Llm@2;
  use Osysharp.Models.Gemini@1 { egress "generativelanguage.googleapis.com"; }
}
```

```osy
// config.osy
using Osysharp.Llm;
using Osysharp.Models.Gemini;

app.Secrets = [ new Secret("Gemini") ];

app.Llm = new LlmSetup {
  Models = [
    new LlmModel { Name = "Fast", Client = new GeminiClient { ApiKey = Secret.Gemini }, Model = "gemini-2.5-flash",
                   Price = new LlmPrice { Input = 0.30m, Output = 2.50m } },
    new LlmModel { Name = "Strong", Client = new GeminiClient { ApiKey = Secret.Gemini }, Model = "gemini-2.5-pro",
                   Price = new LlmPrice { Input = 1.25m, Output = 10 } },
  ],
  Default = "Fast",
};
```

Then `osy secret set Gemini`, and call it: `await Llm.Complete("Summarise this: " + note)`. Which model serves which call is a
separate decision — see [`Osysharp.LlmRouter`](../llm-router) and [`Osysharp.Llm`](../llm).

## Settings

| | |
|---|---|
| `ApiKey` | your Gemini key. Declared here so the app names its own secret; this package never learns which. |
| `BaseUrl` | the API root. Google's own unless you are pointed at a proxy that speaks the same wire. |
| `ApiVersion` | `v1beta` (the default) carries the tool surface this kit uses; `v1` is the stable subset. |
| `MaxTokens` | what to ask for when the request does not say. Gemini does not require a bound; this sends one anyway, because an unbounded answer is an unbounded bill. |

## What it does, and what it deliberately does not

- **Structured output.** When the request carries a schema (`LlmRequest.OutputSchemaJson`, which `Llm.Extract<T>` and
  `Llm.Decide<T>` set), it goes out as `generationConfig.responseMimeType` (`application/json`) with
  `generationConfig.responseJsonSchema`. The answer comes back as the candidate's text, which is the JSON.
- **Tool choice.** `LlmRequest.ToolChoice` is sent as `toolConfig.functionCallingConfig`: `Any` is mode `ANY`, `None` is
  mode `NONE`, and `Named` is mode `ANY` with `allowedFunctionNames` holding the one function; `Auto` sends nothing.
- **A tool call wins over the finish reason.** Gemini answers `STOP` on a turn that asked for a tool, so reading the
  reason alone would tell the caller the model was finished while its answer is a tool call waiting to be run.
- **A tool result is wrapped when it must be.** Gemini requires an object at `functionResponse.response`: an object is
  passed through, an array is wrapped under `result`, and anything that is not JSON — a number, a sentence — is
  wrapped as a JSON string, which is the honest reading of it.
- **The cached prefix is not charged twice.** Gemini counts cached tokens INSIDE `promptTokenCount`, so the kit reports
  `promptTokenCount − cachedContentTokenCount` as input and the cached count beside it.
- **It does not choose a model.** `request.Model` arrives already decided by the app's router. A provider that
  second-guessed the decision would make every routing policy advisory.
- **It does not police spend.** The budget reservation, the meter row and the cost all come from the token counts
  reported here, so it reports what Gemini reported, never an estimate.
- **No prompt-cache breakpoints**, because Gemini places none from the request: the structured system segments join in
  order. Those hints are hints, and a provider that cannot act on them is correct to ignore them.
- **No explicit context caching** (Gemini's `cachedContent` API). The implicit cache is read and reported; managing a
  cache resource is a separate feature with its own lifetime, and it is not something a provider should do behind an
  app's back.

## Source

`model/GeminiClient.osy` is the client; `model/GeminiWire.osy` is the request body as a typed class plus Gemini's
response JSON, with `[ExternalName]` on each field so the class names stay Osy#'s and the wire stays Google's.

## Tested

`kits/gemini/tests` is a plant nursery that answers growers. Nothing is sent to Google: `Http.Post.Stub` stands in for
it, so each test reads the exact request that would have gone out and decides what comes back. Four tests cover: the
address (model in the path, key in a header), system instruction and body of a plain question, with the answer read
back and the meter pricing the cached prefix apart from the fresh prompt; a typed value asked for through
`responseMimeType` and `responseJsonSchema`; a tool call read back as one, with `toolConfig` as sent for each choice;
and a 503 answer surfacing as the provider's failure and landing on the meter as a failed call.

```
osy test kits/gemini/tests
```
