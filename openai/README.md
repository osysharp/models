# Osysharp.Models.OpenAi — the OpenAI Chat Completions API as a model client

The class that talks to OpenAI, **written in Osy#** — and to anything else that speaks the same wire. Register it on
the models it serves and every call to those models — an agent turn, `Llm.Extract`, a `stream<T>` — goes through it.
The platform names no provider anywhere.

## Use case

You want to reach a model. The usual answer is that your platform ships support for some providers and you wait for the
rest — which means the provider released last week, the gateway your company routes everything through, and the model
you host yourself are all "not supported yet".

This kit is the answer instead: **a provider is a class.** It is ordinary Osy# over `Http.*` and `JsonSerializer`,
using nothing an app cannot use, so you can read all of it, copy it, and point the copy at whatever speaks a similar
wire. What it implements — `Osysharp.Llm.ILlmClient`, two members — is the only thing the platform knows.

It has no user interface, so there is nothing to screenshot: it is where a model call goes.

## Install and use

```osy
// app.osy — a package gets no authority from being depended on, so the host is granted here or the app does not compile
app Support {
  model "**/*.osy";
  use Osysharp.Models.OpenAi { egress "api.openai.com"; }
}
```

```osy
// config.osy
using Osysharp.Models.OpenAi;

app.Secrets = [ new Secret("OpenAi") ];

app.Models = [
  new LlmConfig("Fast")   { Client = new OpenAiClient { ApiKey = Secret.OpenAi }, Model = "gpt-4o-mini" },
  new LlmConfig("Strong") { Client = new OpenAiClient { ApiKey = Secret.OpenAi }, Model = "gpt-4o" },
];

app.DefaultModel = Llm.Fast;
```

Then `osy secret set OpenAi`, and you are done. Which model serves which task is a separate decision —
see `Osysharp.LlmRouter`.

## An endpoint OpenAI never heard of

vLLM, llama.cpp, OpenRouter, a gateway of your own — anything that implements `/v1/chat/completions`. It is the same
class with two settings, and that pair is the whole of what a platform-level "OpenAI-compatible provider" ever meant:

```osy
app Support {
  model "**/*.osy";
  use Osysharp.Models.OpenAi {
    egress "api.openai.com";
    egress "vllm.internal";     // YOUR host. This package cannot declare a host its consumer chooses.
  }
}
```

```osy
app.DefaultModel = new LlmConfig {
  Client = new OpenAiClient { ApiKey = Secret.Local, BaseUrl = "http://vllm.internal:8000", LegacyMaxTokens = true },
  Model  = "llama-3.3-70b",
};
```

⭐ **Grant your host on the `use`, and that grant is the only thing that names it:**

```osy
use Osysharp.Models.OpenAi { egress "vllm.internal"; }
```

This package's manifest says `egress consumer;` — *a host the consumer names* — so it declares the SHAPE of what it
reaches without naming a host it cannot know. It hands over no authority: your grant is still the only thing that
names a host, and a package granted nothing reaches nothing.

## Settings

| | |
|---|---|
| `ApiKey` | your OpenAI key. Declared here so the app names its own secret; this package never learns which. |
| `BaseUrl` | the API root. OpenAI's own unless you are pointing at an endpoint that speaks the same wire. |
| `MaxTokens` | what to ask for when the request does not say. OpenAI does not require a bound; this sends one anyway, because an unbounded answer is an unbounded bill. |
| `LegacyMaxTokens` | send the legacy `max_tokens` instead of `max_completion_tokens`. False for OpenAI itself — its reasoning models refuse `max_tokens` outright. True for most compatible servers, which only implemented the legacy field. |

## What it does, and what it deliberately does not

- **Streaming with usage.** `stream_options.include_usage` is always set when streaming, because without it the token
  counts never arrive and the last event of the stream — the one the platform bills from — carries zeros.
- **It always sends a `MessageComplete`**, even if the stream is cut short. The C# adapter this replaces sent it only
  when a `finish_reason` arrived, so a dropped connection produced a call the platform could not meter.
- **The cached prefix is not charged twice.** OpenAI counts cached tokens INSIDE `prompt_tokens`, so the kit reports
  `prompt_tokens − cached_tokens` as input and the cached count beside it. The two add up to what OpenAI billed.
- **It does not choose a model.** `request.Model` arrives already decided by the app's `ILlmRouter`. A provider that
  second-guessed the decision would make every routing policy advisory.
- **It does not police spend.** The kill switch, the budget reservation, the metering row and the cost rollup all
  happen above the seam, from the token counts reported here. So it reports what OpenAI reported, never an estimate.
- **No prompt-cache breakpoints**, because OpenAI has none: the structured system segments join in order. Those hints
  are hints, and a provider that cannot act on them is correct to ignore them.

## Source

`model/OpenAiClient.osy` is the client; `model/OpenAiWire.osy` is the request body as a typed class plus OpenAI's
response JSON, with `[ExternalName]` on each field so the class names stay Osy#'s and the wire stays OpenAI's.

Tested: the platform's own suite asserts the exact bytes of the request that reaches the wire, not just that the kit
compiles.
