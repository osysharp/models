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

## Examples

Taken from this package's own tests, which compile and run on every release.

```osy
using Osysharp.Llm;
using Osysharp.Http;
using PlantNursery;

[Test(Keys)]
void a_question_goes_out_in_geminis_shape_and_its_answer_and_usage_come_back() {
  Http.Post.Stub((url, body, headers) => {
    new Sent { Url = url, Body = body, Key = headers["x-goog-api-key"] };
    return Says("Once a week.", 5000, 1000, 200);
  });
  Assert.Equal("Once a week.", Answer("How often do I water a fig?"));

  var sent = Sent.Single();
  // The model travels in the address, and the key in a header — never in the address.
  Assert.Equal("https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent", sent.Url);
  Assert.Equal("g-test", sent.Key);
  Assert.DoesNotContain("\"model\"", sent.Body);
  // The system prompt is its own object, and the question a user turn.
  Assert.Contains("\"system_instruction\":{\"parts\":[{\"text\":\"You advise growers.\"}]}", sent.Body);
  Assert.Contains("How often do I water a fig?", sent.Body);
  Assert.DoesNotContain("toolConfig", sent.Body);
  Assert.DoesNotContain("responseMimeType", sent.Body);

  // Google counts the cached prefix inside the prompt; the meter is told them apart:
  // 4000 fresh at $0.30 a million, 1000 cached at $0.03, 200 out at $2.50 — 1200 + 30 + 500.
  var row = ModelCall.Single();
  Assert.Equal(4000, row.InputTokens);
  Assert.Equal(1000, row.CacheReadTokens);
  Assert.Equal(200, row.OutputTokens);
  Assert.Equal(1730, row.CostMicros);
}
```

From [`tests/gemini.test.osy`](tests/gemini.test.osy).

```osy
using Osysharp.Llm;
using Osysharp.Http;
using PlantNursery;

[Test(Keys)]
void a_typed_value_is_asked_for_as_geminis_response_schema_and_read_from_the_text() {
  Http.Post.Stub((url, body, headers) => {
    new Sent { Body = body };
    return Says("{\"Plant\":\"Ficus carica\",\"WaterEveryDays\":7,\"FullSun\":true}", 600, 0, 30);
  });
  var care = ReadLabel("Ficus carica — full sun, water weekly.");
  Assert.Equal("Ficus carica", care.Plant);
  Assert.Equal(7, care.WaterEveryDays);
  Assert.True(care.FullSun);

  var body = Sent.Single().Body;
  Assert.Contains("\"responseMimeType\":\"application/json\"", body);
  Assert.Contains("\"responseJsonSchema\":{", body);
  Assert.Contains("Days between waterings.", body);
}
```

From [`tests/gemini.test.osy`](tests/gemini.test.osy).

```osy
using Osysharp.Llm;
using Osysharp.Http;
using PlantNursery;

[Test(Keys)]
void a_tool_call_comes_back_as_one_and_a_forced_choice_is_sent_in_geminis_words() {
  Http.Post.Stub((url, body, headers) => {
    new Sent { Body = body, Seq = Sent.Count() };
    return new HttpResponse { StatusCode = 200, Body =
      "{\"candidates\":[{\"content\":{\"role\":\"model\",\"parts\":[{\"functionCall\":{\"name\":\"in_stock\",\"args\":{\"plant\":\"fig\"}}}]},"
      + "\"finishReason\":\"STOP\"}],\"usageMetadata\":{\"promptTokenCount\":150,\"candidatesTokenCount\":12,\"cachedContentTokenCount\":0}}" };
  });

  var auto = AskWithStock("Any figs left?", LlmToolChoice.Auto);
  // Gemini says STOP for a tool call too; the call itself is what says the model wants a tool run.
  Assert.Equal(LlmStopReason.ToolUse, auto.Response.StopReason);
  var call = auto.Response.Content.Single();
  Assert.Equal(LlmBlockKind.ToolUse, call.Kind);
  Assert.Equal("in_stock", call.ToolName);
  Assert.Contains("\"plant\":\"fig\"", call.InputJson!);

  AskWithStock("Any figs left?", LlmToolChoice.Any);
  AskWithStock("Any figs left?", LlmToolChoice.None);
  AskWithStock("Any figs left?", LlmToolChoice.Named);
  var bodies = Sent.OrderBy(s => s.Seq).ToList().Select(s => s.Body).ToList();
  Assert.DoesNotContain("toolConfig", bodies[0]);
  Assert.Contains("\"toolConfig\":{\"functionCallingConfig\":{\"mode\":\"ANY\"}}", bodies[1]);
  Assert.Contains("\"toolConfig\":{\"functionCallingConfig\":{\"mode\":\"NONE\"}}", bodies[2]);
  Assert.Contains("\"allowedFunctionNames\":[\"in_stock\"]", bodies[3]);
}
```

From [`tests/gemini.test.osy`](tests/gemini.test.osy).

## Learn more

- [The Osysharp.Models.Gemini reference](https://osysharp.com/reference/agent/kit-gemini/)
- [Osysharp.Models.Gemini in the package catalogue](https://osysharp.com/packages/#osysharp-models-gemini)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
