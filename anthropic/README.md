# Osysharp.Models.Anthropic — the Anthropic Messages API as a model client

The class that talks to Anthropic, **written in Osy#**. Name it on the models it serves and every `Llm.Complete`,
`Llm.Stream`, `Llm.Extract<T>` and `Llm.Decide<T>` call to those models goes through it. Osysharp.Llm names no provider
anywhere.

## Use case

You want to reach a model. The usual answer is that your tooling ships support for some providers and you wait for the
rest — which means the provider released last week, the gateway your company routes everything through, and the model
you host yourself are all "not supported".

This kit is the answer instead: **a provider is a class.** It is ordinary Osy# over `Http.*` and `JsonSerializer`,
using nothing an app cannot use, so you can read all of it, copy it, and point the copy at whatever speaks a similar
wire. What it implements — `Osysharp.Llm.ILlmClient`, two members — is the only thing the Llm kit knows.

It has no user interface, so there is nothing to screenshot: it is where a model call goes.

## Install and use

```osy
// app.osy — a package gets no authority from being depended on, so the host is granted here or the app does not compile
app Support {
  model "**/*.osy";
  use Osysharp.Llm@2;
  use Osysharp.Models.Anthropic@1 { egress "api.anthropic.com"; }
}
```

```osy
// config.osy
using Osysharp.Llm;
using Osysharp.Models.Anthropic;

app.Secrets = [ new Secret("Anthropic") ];

app.Llm = new LlmSetup {
  Models = [
    new LlmModel { Name = "Fast", Client = new AnthropicClient { ApiKey = Secret.Anthropic }, Model = "claude-haiku-5-5",
                   Price = new LlmPrice { Input = 0.10m, Output = 0.50m } },
    new LlmModel { Name = "Strong", Client = new AnthropicClient { ApiKey = Secret.Anthropic }, Model = "claude-opus-5-5",
                   Price = new LlmPrice { Input = 4, Output = 20 } },
  ],
  Default = "Fast",
};
```

Then `osy secret set Anthropic`, and call it: `await Llm.Complete("Summarise this: " + note)`. Which model serves which call is
a separate decision — see [`Osysharp.LlmRouter`](../llm-router) and [`Osysharp.Llm`](../llm).

## Settings

| | |
|---|---|
| `ApiKey` | your Anthropic key. Declared here so the app names its own secret; this package never learns which. |
| `BaseUrl` | the API root. Anthropic's own unless you are pointed at a proxy that speaks the same wire. |
| `MaxTokens` | what to ask for when the request does not say. Anthropic REQUIRES a number, so there is always one. |
| `NativeStructuredOutput` | `true` (the default): Anthropic holds the model to the schema itself. `false`: the schema goes into the system prompt and the model is asked to follow it — for a schema Anthropic's own check refuses (a `maxLength`, a `minimum`, a class that contains itself). |

## What it does, and what it deliberately does not

- **Prompt caching.** A breakpoint is placed at the end of each cacheable stability tier of the system prompt, and on
  the last tool when the request asks for it. Both are hints on the request — a provider without prompt caching ignores
  them correctly — and this one honours them.
- **Structured output.** When the request carries a schema (`LlmRequest.OutputSchemaJson`, which `Llm.Extract<T>` and
  `Llm.Decide<T>` set), it goes out as `output_config.format` of type `json_schema`. The answer comes back as one text
  block whose text is the JSON.
- **Tool choice.** `LlmRequest.ToolChoice` is sent as `tool_choice`: `Any` is `{"type":"any"}`, `None` is
  `{"type":"none"}`, `Named` is `{"type":"tool","name":…}`; `Auto` sends nothing, which is Anthropic's default. The
  newest Anthropic models refuse a forced call (`Any`, `Named`) with a 400; that reaches the caller as the provider's
  failure (`ExternalServiceException`), so ask those models with `Auto` and say in the prompt which tool to use.
- **It does not choose a model.** `request.Model` arrives already decided by the app's router. A provider that
  second-guessed the decision would make every routing policy advisory.
- **It does not police spend.** The budget reservation, the meter row and the cost all come from the token counts
  reported here, so it reports what Anthropic reported, never an estimate.
- **Sampling parameters are dropped for models that removed them** (`temperature`, `top_p`). The list of models that
  still accept them is an ALLOWLIST, not a denylist, and the direction is the point: the set is closed and shrinking,
  while new models keep arriving without them. A denylist is wrong the day a model ships, and wrong in the worst
  direction — every call 400s.

## Source

`model/AnthropicClient.osy` is the client; `model/AnthropicWire.osy` is the request body as a typed class plus
Anthropic's response JSON, with `[ExternalName]` on each field so the class names stay Osy#'s and the wire stays
Anthropic's.

## Tested

`kits/anthropic/tests` is a ferry line's desk that answers travellers. Nothing is sent to Anthropic: `Http.Post.Stub`
stands in for it, so each test reads the exact request that would have gone out and decides what comes back. Five tests
cover: the address, key header and body of a plain question (and what is absent from it), with the answer read back and
the meter's tokens and cost; a typed value asked for through `output_config.format`, and through the system prompt with
`NativeStructuredOutput = false`; a tool call read back, with `tool_choice` as sent for `Auto`, `Any`, `None` and
`Named`; and a 529 answer surfacing as the provider's failure and landing on the meter as a failed call.

```
osy test kits/anthropic/tests
```
