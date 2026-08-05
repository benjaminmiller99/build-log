# Cost-sizing a moderation queue: token math for user text and images in Node.js

If you just want the recommendation: estimate what each moderation item will cost in tokens inside your own Node.js process, before you classify anything, and let that number decide whether the model gets called at all. Count user text tokens locally, price images from their pixel dimensions instead of their file size, and pin the response shape with a JSON schema so the output half of the bill stays flat.

That's the whole design.

I run a small product where people post short notes with attachments, and the LLM moderation line landed second on my bill within a month of launch. No single call was expensive. I was classifying everything that moved — including four thousand near-identical submissions from an integration test somebody left running over a weekend, which is the kind of thing you only find out about on the invoice.

## How do I estimate the token cost of user text and images before I classify them?

An item lands, gets normalized (trimmed, whitespace collapsed, zero-width characters stripped, Unicode NFC), and hashed. If that hash sits in the recent-verdict cache, reuse the verdict and stop. If deterministic rules settle it — a known-good author, a banned domain, an exact phrase hit — stop there too. Only what survives both gets estimated: text tokens plus image tokens plus fixed prompt overhead on the input side, and a schema-bounded number on the output side. Multiply by the two rates, compare against a per-item ceiling, then decide: call now, route to a smaller model, or park it for a slower batch pass.

Exact text counts mean running the same BPE encoder the model family uses — in Node.js that's normally a WASM build you load once at boot. The four-characters-per-token rule of thumb survives English prose and collapses everywhere else. Emoji, CJK, and pasted base64 all cost far more tokens per character than their length suggests, and I measured a 34% undercount on a multilingual sample before I wired in the real encoder. The estimator itself is cheap: a few microseconds per item, no network, no state. That asymmetry is the point — the decision has to cost roughly nothing compared to the thing it's deciding about.

## The pre-flight check, in one file

```ts
// budget.ts — what will this item cost, before we spend anything on it
type Rates = { inPerMTok: number; outPerMTok: number };
type Image = { width: number; height: number };
type Item = { text: string; images: Image[] };

// Example values. Read yours off the provider's vision pricing table: most
// bill a flat base per image plus one charge per tile of the scaled image.
const IMAGE = { base: 85, perTile: 170, tile: 512, maxEdge: 1024 };
const PROMPT_TOKENS = 320;   // system prompt + policy text, encoded once
const SCHEMA_TOKENS = 24;    // measured ceiling of the structured response

export function countTextTokens(s: string): number {
  // Swap this for encoder.encode(s).length once the WASM encoder is loaded.
  let n = 0;
  for (const ch of s) n += (ch.codePointAt(0) as number) > 0x7f ? 1 : 0.27;
  return Math.ceil(n);
}

export function countImageTokens({ width, height }: Image): number {
  const scale = Math.min(1, IMAGE.maxEdge / Math.max(width, height));
  const cols = Math.ceil((width * scale) / IMAGE.tile);
  const rows = Math.ceil((height * scale) / IMAGE.tile);
  return IMAGE.base + IMAGE.perTile * cols * rows;
}

export function estimate(item: Item, rates: Rates) {
  const input = PROMPT_TOKENS + countTextTokens(item.text)
    + item.images.reduce((n, img) => n + countImageTokens(img), 0);
  const output = SCHEMA_TOKENS;
  return { input, output, cost: (input * rates.inPerMTok + output * rates.outPerMTok) / 1e6 };
}
```

The routing layer is smaller than the estimator, and it's the part that actually saves money:

```ts
const CEILING = 0.0008;      // per item, derived from the queue's daily budget

export async function route(item: Item, rates: Rates) {
  const key = fingerprint(item);            // NFC text + perceptual image hashes
  const cached = await verdicts.get(key);
  if (cached) return cached;

  const settled = rules(item);              // regex + allowlists, zero tokens
  if (settled) return settled;

  const { input, cost } = estimate(item, rates);
  if (cost > CEILING) return { label: "review", reason: "over ceiling", input };
  return classify(item, key);               // the only branch that spends tokens
}
```

Everything above the `classify` line runs in single-digit milliseconds. In my queue the cache plus the rules settle somewhere between 55% and 70% of items on a normal day, and close to 100% during a scripted flood, which is exactly when you most want the spending to stop.

## Why images and a JSON schema move the number more than the prompt does

Image billing keys off resolution, not bytes. A 4032×3024 phone photo and a 768px thumbnail of the same photo produce the same verdict from any model I've tried, and wildly different token counts, because the provider slices the scaled image into tiles and charges per tile. Downscaling attachments to a 1024px long edge before the call cut my image tokens by about four times, and the labels didn't move on the sample I check by hand.

The schema is the other lever, and it works on both sides of the bill.

Every property name and enum value in the schema is input tokens on every single call, so `label` beats `moderation_decision_category`. The response side is where the savings compound: an unconstrained model writes a paragraph, a strict schema writes a couple of dozen tokens. Field order matters more than it looks like it should — as far as I can tell, generation follows the property order in the schema, so a `reason` string declared before `label` means you pay for the model to think out loud on every item. Put the label first, cap free text to an enum, and add a real explanation field only for the items you sample.

```json
{
  "name": "verdict",
  "schema": {
    "type": "object",
    "properties": {
      "label": { "type": "string", "enum": ["ok", "review", "block"] },
      "policy": { "type": "string", "enum": ["none", "sexual", "violence", "harassment", "self_harm", "spam"] },
      "confidence": { "type": "number" }
    },
    "required": ["label", "policy", "confidence"],
    "additionalProperties": false
  },
  "strict": true
}
```

## What the pre-filter costs you

| Tier | What it settles | Marginal cost per item | Where it stops working |
| --- | --- | --- | --- |
| Normalized hash + verdict cache | Exact repeats, retries, floods | Memory and a hash | Any small mutation of the text |
| Deterministic rules | Contact-info leaks, banned domains, known-good authors | Sub-millisecond CPU | Sarcasm, context, anything visual |
| Model call with a strict schema | Nuanced policy judgement, images | Input tokens + a bounded output | Cost and tail latency under bursts |

The catch is that every item you don't send is an item you never measured, so a pre-filter quietly becomes an unaudited policy of its own. I sample 1% of the cached and rule-settled decisions into the same review queue as the model's, and order that queue with a reranker so the plausibly-wrong ones surface first. Without that, the filter's false negatives are invisible by construction.

Two boundaries worth stating plainly. If your policy is a fixed list of banned terms and domains, a model isn't suitable — stick with the rules engine, which is faster, auditable, and free. And if you need a versioned category taxonomy you can point a regulator at, a hosted moderation endpoint with published categories is a better fit than a prompt you wrote yourself, because your prompt's behaviour changes silently when the underlying model does. Audio and video don't belong in this pipeline at all until something has transcribed them; the transcript is what you estimate and classify.

## The cold start that doubled my bill

For six weeks the numbers looked fine: p50 around 180 ms, p99 around 400 ms, spend tracking the estimate within a few percent. Then a campaign landed and p99 went to 6.5 seconds while p50 barely moved. The cause wasn't the model at all — it was my own tokenizer. I'd been loading the WASM encoder lazily inside the request handler, and the ranks file is a couple of megabytes, so every cold instance paid that load before it could even decide whether to make a call. Under steady traffic almost nothing was cold. Under a burst, most instances were. The expensive part came next: the client timed out at 3 seconds and retried, the retry wasn't keyed on anything, and for about two hours I paid for every borderline item twice while my estimator cheerfully reported one call each. Loading the encoder at module scope, warming it on boot, and keying the model call on the same fingerprint the cache uses took p99 to roughly 900 ms and made the duplicate spend go away. I'm still not sure why the first version passed load testing — my best guess is that my test harness reused connections in a way real clients don't.

So the operational habits that came out of that, in the order I'd add them. Log estimated tokens next to the usage the response reports, and alert when the two drift more than 15% over an hour, because drift means either your prompt grew or your encoder no longer matches the model. Version the schema and store its version with every verdict, so a re-run is reproducible. Cap spend per user per day, not just per item, since the per-item ceiling is useless against ten thousand cheap items. Keep the estimator and the caller in one module so nobody updates the prompt without updating the constant. And load anything that touches WASM at module scope — off the request path, and off the event loop with a worker if it's big enough to notice.

## References

- JSON Schema specification — https://json-schema.org/specification
- tiktoken, the BPE tokenizer used by GPT-family models — https://github.com/openai/tiktoken
- WASM/JavaScript bindings for tiktoken — https://github.com/dqbd/tiktoken
- Node.js worker_threads documentation — https://nodejs.org/api/worker_threads.html
- Unicode Normalization Forms (UAX #15) — https://unicode.org/reports/tr15/
- Rerank overview, for ordering a human review queue — https://docs.cohere.com/docs/rerank-overview
- openai/whisper, for transcribing audio before it reaches a text pipeline — https://github.com/openai/whisper
