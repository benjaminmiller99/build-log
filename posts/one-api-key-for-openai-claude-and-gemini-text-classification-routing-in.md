# One API key for OpenAI, Claude, and Gemini: text classification routing in Node.js

Use an OpenAI-compatible gateway — OpenRouter, a self-hosted LiteLLM proxy, or fifty lines of your own routing code — when you want one API key sitting in front of OpenAI, Claude, and Gemini for text classification; otherwise reach for the Vercel AI SDK, keep three provider keys, and switch models in application code. Those are the two shapes that survive contact with production. Which one fits you depends on whether your pain is billing and ops, or type safety inside the app.

I tag support tickets and changelog entries for a product I run alone — roughly 40,000 short documents a month, no streaming, no agents, nothing exotic. I've moved that pipeline across all three vendors twice. The second migration took an afternoon instead of a week, and the difference wasn't some clever abstraction I'd discovered. It was that I had stopped writing provider-specific code at all.

Here's what actually changed.

## Should I replace direct OpenAI, Claude, and Gemini calls with one API key?

For a classification workload, yes, and it's less work than the migration horror stories suggest. Classification is the friendliest job you can hand a router. The prompt is short, the output is a single label, and there's no streaming, no tool calling, no image input, no 200k-token context window to babysit. All of it fits inside the chat completions request shape, which every major vendor now speaks: Anthropic publishes an OpenAI SDK compatibility layer, Google exposes an OpenAI-compatible path on the Gemini API, and Groq, Mistral and most of the smaller hosts shipped the same thing. Pointing the official `openai` Node package at a different base URL is a supported deployment, not a hack you have to apologise for in code review.

The catch is where it always is. Billing.

One gateway means one invoice, one rate-limit budget, and one credential to rotate — which matters more than it sounds when you're solo and every extra vendor is another dashboard, another payment method, and another quota you find out about at 2am. It also means a middleman on your critical path, a markup on every token, and an outage you can't do anything about except refresh a status page. In my setup a gateway hop added somewhere around 60–90ms over a direct OpenAI call from a us-east box, which is invisible for a background tagger and would be unacceptable if I were streaming into a text box a user is staring at. If you already live inside one cloud, Amazon Bedrock and Vertex AI hand you most of the multi-vendor benefit through credentials you already have, with a data-residency story your enterprise buyers will accept without a security review. I'd take either of those over a third-party gateway in a regulated setting and not argue much about it.

Model routing is the piece that gets oversold. Automatic "pick the best model per request" sounds great, and as far as I can tell it isn't measurable on a classification task where three decent models all land within a couple of points of each other on the same eval set. What routing actually buys me is failover and price tiering — run the small cheap model first, escalate when confidence comes back low, and cut over to a different vendor when one starts returning 500s. That's a decision tree I want to own rather than delegate to someone else's heuristic.

## Where each option actually hurts

All five of these work. They fail in different places, and the failure mode is what should drive the choice.

| Approach | One key? | What it costs you | Reach for it when |
| --- | --- | --- | --- |
| Hosted gateway (OpenRouter and similar) | yes | token markup, an extra hop, uneven feature support per model | a vendor swap should be a config change, not a deploy |
| Self-hosted LiteLLM proxy | yes | you operate it, and you get paged for it | you want one key plus your own budgets, logging and key scoping |
| Vercel AI SDK | no | three keys, three quotas, three sets of rate limits | you want typed results and don't mind swaps living in code |
| Amazon Bedrock / Vertex AI | cloud creds | menu limited to that cloud, plus per-region model availability | procurement, VPC and residency drive the decision |
| Direct vendor SDKs | no | three integrations to keep current | you depend on vendor-only features like the Batch API |

That last row is the one people skim past. A gateway generally doesn't support the batch endpoints, and for a tagging workload batching is the single largest cost lever there is — OpenAI runs the same chat completions requests through its Batch API at half price with a 24-hour completion window. My tagger doesn't need an answer in 200ms. It needs one by tomorrow morning. So the architecture I actually run is unglamorous: direct-to-vendor batch for the bulk nightly job, gateway for the live single-document path that fires when a ticket comes in. Two routes, and I've made my peace with that.

Ollama sits in a corner of this I keep meaning to revisit. For short-text classification a small local model is plausible and the marginal cost per document is zero, but I haven't done the eval work to say anything trustworthy about accuracy, so I won't pretend otherwise.

## Writing a classifier that survives a model swap

The portable core is structured output. A classifier that returns free text will have you writing per-model parsing hacks forever; one that returns a JSON object validated against a schema turns a model swap into a string change.

```ts
import OpenAI from "openai";

// One key, one base URL. Swap the model string to change vendor.
const router = new OpenAI({
  baseURL: "https://openrouter.ai/api/v1",
  apiKey: process.env.OPENROUTER_API_KEY,   // see the footgun below
});

const schema = {
  type: "object",
  properties: {
    label: { type: "string", enum: ["billing", "bug", "feature_request", "other"] },
    confidence: { type: "number" },
  },
  required: ["label", "confidence"],
  additionalProperties: false,
} as const;

export async function classify(text: string, model: string) {
  const res = await router.chat.completions.create({
    model,
    temperature: 0,
    messages: [
      { role: "system", content: "Classify the support ticket. Use the schema." },
      { role: "user", content: text },
    ],
    response_format: {
      type: "json_schema",
      json_schema: { name: "ticket_label", strict: true, schema },
    },
  });
  return JSON.parse(res.choices[0].message.content ?? "{}");
}
```

Two notes on that snippet. `strict: true` is what turns the schema into a hard constraint instead of a suggestion, and it demands `additionalProperties: false` plus every property named in `required` — omit either and the request comes back as a 400 with a message that does at least tell you which rule you broke. The second note is less comfortable: strict schema support is not uniform across a gateway's whole model catalog. Some smaller and older models accept the parameter and then return JSON that doesn't validate anyway. I treat a validation failure as a routing signal, exactly like a 5xx, which keeps the retry logic in one place.

```ts
const CHAIN = [
  "openai/gpt-4o-mini",
  "anthropic/claude-3.5-haiku",
  "google/gemini-2.0-flash-001",
];

export async function classifyWithFallback(text: string) {
  const failures: string[] = [];
  for (const model of CHAIN) {
    const started = Date.now();
    try {
      const out = await classify(text, model);
      record({ model, ms: Date.now() - started, label: out.label, conf: out.confidence });
      return out;
    } catch (err) {
      failures.push(`${model}: ${(err as Error).message}`);
    }
  }
  throw new Error(`no model produced a valid label -> ${failures.join(" | ")}`);
}
```

Gateway model slugs do drift as vendors deprecate versions, so read the chain from config and check it against the live model list on deploy rather than hardcoding it three files deep like I did the first time.

## The config footgun that ate an afternoon

Here's the one that got me, and it's embarrassingly small. I switched the classifier over to a gateway by adding `baseURL` to the client constructor and moving on, because that's the whole migration, right? Except I never passed `apiKey`. The Node SDK falls back to `process.env.OPENAI_API_KEY` when you don't give it one, that variable was still sitting in my `.env` from the direct integration, so the client happily sent a perfectly valid OpenAI key to a completely different host. What came back was a 401 saying no auth credentials were found. Not "invalid key" — *no credentials*, which sent me hunting for a stripped Authorization header through my proxy config for about forty minutes before I logged the outgoing request and saw my own `sk-proj-` prefix going to the wrong domain. The same class of mistake shows up on Bedrock, where model access is granted per region and a client pinned to `us-east-1` gets an AccessDeniedException for a model you definitely enabled — in `us-west-2`. Auth and region errors are not honest about which of the two they are.

So: pass the key explicitly, always, even when the default would work.

## Batching, cost, and the observability you'll wish you'd added

The batch path is worth the extra code if your volume is anywhere near mine. You write a JSONL file where each line is a full request with a `custom_id` you can join back to your own row, upload it, poll, and download results. Half price, one file, one job to monitor.

```json
{"custom_id":"ticket-8812","method":"POST","url":"/v1/chat/completions","body":{"model":"gpt-4o-mini","temperature":0,"messages":[{"role":"system","content":"Classify the support ticket. Use the schema."},{"role":"user","content":"card was declined twice today"}],"response_format":{"type":"json_schema","json_schema":{"name":"ticket_label","strict":true,"schema":{"type":"object","properties":{"label":{"type":"string","enum":["billing","bug","feature_request","other"]},"confidence":{"type":"number"}},"required":["label","confidence"],"additionalProperties":false}}}}}
```

The observability part is three fields. Log the model id, the latency, and the token counts on every call, then chart the label distribution per model per day. That last chart is the one that earns its keep — when I rotated the cheap tier to a different vendor, the share of documents landing in `other` jumped from 6% to 19% overnight, and nothing else in the system had a way to notice. Accuracy regressions from a model swap are silent by default. A distribution chart makes them loud.

Keep a small hand-labelled set too. Mine is 300 tickets I annotated over two evenings, and it's the only reason I can say a swap is safe rather than hope so.

**One key is a billing and ops decision, not an accuracy one.** Adopt a gateway because you're tired of managing three vendor relationships, and keep a direct path open for the batch and caching features gateways don't expose. Stick with per-vendor SDKs if you're already running well on one provider and multi-vendor is a hypothetical — the abstraction only pays for itself the day you need it.

## References

- OpenAI Structured Outputs guide — https://platform.openai.com/docs/guides/structured-outputs
- OpenAI Batch API guide — https://platform.openai.com/docs/guides/batch
- Anthropic OpenAI SDK compatibility — https://docs.anthropic.com/en/api/openai-sdk
- Gemini API OpenAI compatibility — https://ai.google.dev/gemini-api/docs/openai
- OpenRouter quickstart — https://openrouter.ai/docs/quickstart
- LiteLLM proxy quick start — https://docs.litellm.ai/docs/proxy/quick_start
- Vercel AI SDK docs — https://sdk.vercel.ai/docs
