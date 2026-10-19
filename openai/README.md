# Osysharp.Models.OpenAi — the OpenAI Chat Completions API as a model client

The class that talks to OpenAI, **written in Osy#** — and to anything else that speaks the same wire. Name it on the
models it serves and every `Llm.Complete`, `Llm.Stream`, `Llm.Extract<T>` and `Llm.Decide<T>` call to those models goes
through it. Osysharp.Llm names no provider anywhere.

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
  use Osysharp.Models.OpenAi@1 { egress "api.openai.com"; }
}
```

```osy
// config.osy
using Osysharp.Llm;
using Osysharp.Models.OpenAi;

app.Secrets = [ new Secret("OpenAi") ];

app.Llm = new LlmSetup {
  Models = [
    new LlmModel { Name = "Fast", Client = new OpenAiClient { ApiKey = Secret.OpenAi }, Model = "gpt-4o-mini",
                   Price = new LlmPrice { Input = 0.15m, Output = 0.60m } },
    new LlmModel { Name = "Strong", Client = new OpenAiClient { ApiKey = Secret.OpenAi }, Model = "gpt-4o",
                   Price = new LlmPrice { Input = 2.50m, Output = 10 } },
  ],
  Default = "Fast",
};
```

Then `osy secret set OpenAi`, and call it: `await Llm.Complete("Summarise this: " + note)`. Which model serves which call is a
separate decision — see [`Osysharp.LlmRouter`](../llm-router) and [`Osysharp.Llm`](../llm).

## An endpoint OpenAI never heard of

vLLM, llama.cpp, OpenRouter, a gateway of your own — anything that implements `/v1/chat/completions`. It is the same
class with two settings:

```osy
app Support {
  model "**/*.osy";
  use Osysharp.Llm@2;
  use Osysharp.Models.OpenAi@1 {
    egress "api.openai.com";
    egress "vllm.internal";     // YOUR host. This package cannot declare a host its consumer chooses.
  }
}
```

```osy
new LlmModel {
  Name = "Local",
  Client = new OpenAiClient { ApiKey = Secret.Local, BaseUrl = "http://vllm.internal:8000", LegacyMaxTokens = true },
  Model = "llama-3.3-70b",
}
```

This package's manifest says `egress consumer;` — *a host the consumer names* — so it declares the shape of what it
reaches without naming a host it cannot know. It hands over no authority: your grant on the `use` is the only thing that
names a host, and a package granted nothing reaches nothing.

## Settings

| | |
|---|---|
| `ApiKey` | your OpenAI key. Declared here so the app names its own secret; this package never learns which. |
| `BaseUrl` | the API root. OpenAI's own unless you are pointing at an endpoint that speaks the same wire. |
| `MaxTokens` | what to ask for when the request does not say. OpenAI does not require a bound; this sends one anyway, because an unbounded answer is an unbounded bill. |
| `LegacyMaxTokens` | send the legacy `max_tokens` instead of `max_completion_tokens`. False for OpenAI itself — its reasoning models refuse `max_tokens` outright. True for most compatible servers, which only implemented the legacy field. |

## What it does, and what it deliberately does not

- **Structured output.** When the request carries a schema (`LlmRequest.OutputSchemaJson`, which `Llm.Extract<T>` and
  `Llm.Decide<T>` set), it goes out as `response_format` of type `json_schema` with `strict: false`. Strict mode
  demands every property be `required`, and a schema written from a type leaves a nullable member out of `required`,
  which strict mode refuses; unstrict, the model is still given the schema and follows it. The answer comes back as the
  message's content — one text block whose text is the JSON.
- **Tool choice.** `LlmRequest.ToolChoice` is sent as `tool_choice`: `Any` is `"required"` (OpenAI's word for "any of
  them"), `None` is `"none"`, `Named` is `{"type":"function","function":{"name":…}}`; `Auto` sends nothing.
- **Streaming with usage.** `stream_options.include_usage` is always set when streaming, because without it the token
  counts never arrive and the last event of the stream — the one the meter prices from — carries zeros.
- **It always sends a `MessageComplete`**, even if the stream is cut short — not only when a `finish_reason` arrives —
  so a dropped connection still produces a call the meter can record.
- **The cached prefix is not charged twice.** OpenAI counts cached tokens INSIDE `prompt_tokens`, so the kit reports
  `prompt_tokens − cached_tokens` as input and the cached count beside it. The two add up to what OpenAI billed.
- **It does not choose a model.** `request.Model` arrives already decided by the app's router. A provider that
  second-guessed the decision would make every routing policy advisory.
- **It does not police spend.** The budget reservation, the meter row and the cost all come from the token counts
  reported here, so it reports what OpenAI reported, never an estimate.
- **No prompt-cache breakpoints**, because OpenAI has none: the structured system segments join in order. Those hints
  are hints, and a provider that cannot act on them is correct to ignore them.

## Source

`model/OpenAiClient.osy` is the client; `model/OpenAiWire.osy` is the request body as a typed class plus OpenAI's
response JSON, with `[ExternalName]` on each field so the class names stay Osy#'s and the wire stays OpenAI's.

## Tested

`kits/openai/tests` is a recipe box that reads recipes people paste in. Nothing is sent to OpenAI: `Http.Post.Stub`
stands in for it, so each test reads the exact request that would have gone out and decides what comes back. Five tests
cover: the address, bearer key and body of a plain question (and what is absent from it), with the answer read back and
the meter pricing the cached prefix apart from the fresh prompt; an endpoint that speaks the same wire, reached at its
own address with its own key and the legacy token limit; a typed value asked for through `response_format`; a tool call
read back, with `tool_choice` as sent for each choice; and a 429 answer surfacing as the provider's failure and landing
on the meter as a failed call.

```
osy test kits/openai/tests
```
