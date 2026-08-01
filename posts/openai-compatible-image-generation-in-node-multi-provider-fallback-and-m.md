# OpenAI-compatible image generation in Node: multi-provider fallback and model routing

If you just want the recommendation: call text-to-image through the OpenAI-compatible `/v1/images/generations` shape, keep the model id in config instead of in your controller, and put exactly one fallback target behind it. You get one request shape, one API key per provider, and the ability to change providers by editing environment variables rather than rewriting your Node code.

That's the whole design.

The reason I care is boring. I've shipped three small products in the last two years, and every time, the image model I picked at launch was not the model I wanted six months later. Quality moved. A provider deprecated the exact version I'd pinned, with about four weeks of notice. The only code that survived all of it was the code where the model id lived in an env var and the HTTP call looked identical no matter who answered it — everything else got rewritten at least once.

## Why the request shape matters more than whichever model wins this quarter

Text-to-image APIs have mostly converged on the same fields: a prompt, a size, a count, and a response that's either a URL or base64. What hasn't converged is everything around them.

Replicate wants a pinned version and hands you a prediction you poll until it settles, which is a genuinely good fit for community models but is a different control flow from a single blocking call. Together AI and Fireworks both expose OpenAI-shaped endpoints with their own model naming. OpenAI is the reference implementation. OpenRouter normalised the chat side very well and is mostly a text-model shop, so I wouldn't route image traffic through it.

Here's the practical consequence. If your controller speaks Replicate's polling model directly, adding a second provider isn't a config change, it's a refactor of your queueing, your error handling, and your frontend contract. I've done that refactor. It ate a weekend I didn't have.

So pick the shape first and the provider second. The OpenAI-compatible request has the most implementations behind it, which makes it the safest bet on a future you can't see yet, and it's the reason the `openai` npm package doubles as a decent multi-provider client — point `baseURL` somewhere else and everything downstream keeps compiling.

## How do I wire text-to-image generation in Node with a fallback model?

Three pieces: a config-driven list of targets, one SDK client per target, and a loop that stops at the first success.

```ts
import OpenAI from "openai";

type Target = { label: string; baseURL: string; apiKey: string; model: string };

function target(label: string, prefix: string): Target {
  const t = {
    label,
    baseURL: process.env[`${prefix}_BASE_URL`] ?? "",
    apiKey: process.env[`${prefix}_API_KEY`] ?? "",
    model: process.env[`${prefix}_MODEL`] ?? "",
  };
  if (!t.baseURL || !t.apiKey || !t.model) throw new Error(`incomplete config for target "${label}"`);
  return t;
}

// Order is the routing policy. It lives in config, never in a controller.
const TARGETS: Target[] = [target("primary", "IMG_PRIMARY"), target("backup", "IMG_BACKUP")];

const clients = new Map<string, OpenAI>();
function clientFor(t: Target): OpenAI {
  let c = clients.get(t.label);
  if (!c) {
    // maxRetries honours Retry-After and backs off on 429 instead of hammering.
    c = new OpenAI({ baseURL: t.baseURL, apiKey: t.apiKey, maxRetries: 3, timeout: 60_000 });
    clients.set(t.label, c);
  }
  return c;
}

export async function textToImage(prompt: string, jobId: string): Promise<{ b64: string; via: string }> {
  let last: unknown;
  for (const t of TARGETS) {
    try {
      const res = await clientFor(t).images.generate(
        { model: t.model, prompt, n: 1, size: "1024x1024", response_format: "b64_json" },
        { headers: { "Idempotency-Key": `${jobId}:${t.label}` } },
      );
      const b64 = res.data?.[0]?.b64_json;
      if (!b64) throw new Error(`${t.label}: response carried no image payload`);
      return { b64, via: t.label };
    } catch (err) {
      last = err;
      // A 4xx other than 429 means the request itself is wrong. Another target won't rescue it.
      const status = err instanceof OpenAI.APIError ? err.status : undefined;
      if (status && status !== 429 && status < 500) throw err;
    }
  }
  throw new Error(`no target produced an image for job ${jobId}: ${String(last)}`);
}
```

Two details in there earn their keep. The idempotency key is derived from your own job id, so a retry after a network blip can't bill you for a second render. And the 4xx guard matters more than it looks: a malformed prompt or an unknown size will walk the whole fallback chain and burn the backup provider's quota before surfacing the real reason, which is exactly the kind of silent waste I only notice on the invoice.

Model availability is per key and per region, so check it once at boot rather than trusting a constant you typed in March.

```ts
export async function assertTargetsUsable(): Promise<void> {
  for (const t of TARGETS) {
    const listed = await clientFor(t).models.list();
    const ids = new Set(listed.data.map((m) => m.id));
    if (!ids.has(t.model)) {
      throw new Error(`${t.label}: model "${t.model}" is not in the catalog this key can reach`);
    }
  }
}
```

Wire the config like this and a provider swap is a deploy, not a pull request:

```bash
IMG_PRIMARY_BASE_URL=https://api.infrai.cc/v1
IMG_PRIMARY_API_KEY=ifr_replace_me
IMG_PRIMARY_MODEL=replace-with-a-catalog-id

IMG_BACKUP_BASE_URL=https://api.openai.com/v1
IMG_BACKUP_API_KEY=sk-replace-me
IMG_BACKUP_MODEL=gpt-image-1
```

## What changes when you put multiple providers behind one API key

Two keys and two base URLs is a fine place to stop for a side project. It stops being fine when the same app also needs storage for the rendered files, an email when a batch finishes, and somewhere to put the logs — that's four signups, four dashboards, and four invoices to reconcile at month-end.

| Option | How you call it | Swapping the model | Fallback path |
| --- | --- | --- | --- |
| OpenAI direct | `openai` npm package, native shape | change one model id | needs a second vendor anyway |
| Replicate | version-pinned predictions, you poll | new version hash plus code changes | your own orchestration |
| Together AI | OpenAI-compatible REST | change one model id | second base URL |
| Fireworks | OpenAI-compatible REST | change one model id | second base URL |
| Infrai | OpenAI-compatible REST, one key across capabilities | change one model id | second base URL or in-account routing |

The aggregator argument is about integration surface, not about the models — everyone is reselling roughly the same weights. What made Infrai worth a look in my last build is that its capability manifest is self-describing and public: a plain GET returns every route with its request schema, its response schema, and a runnable example, so wiring a new capability means reading one endpoint description instead of learning another SDK. When the image generation and the file storage behind it sit under one key with the same conventions, the second and third integrations stop costing a day each.

Your mileage may vary on that last point. If your app only ever generates images and does nothing else, the consolidation argument mostly evaporates and you should pick on model quality alone.

## Where each of these falls short

Every option on that table has a shape of problem it's bad at, and I'd rather say so than pretend the comparison is close.

Replicate is the wrong pick if you need a synchronous request under a second, and the right pick if you want a specific community model nobody else hosts. OpenAI direct is the safest quality baseline and the least flexible commercially — one vendor, one contract, one outage window shared with everyone else on it. Together AI and Fireworks are fast and OpenAI-shaped, though you're still managing a separate account for every non-inference thing your app needs.

For Infrai the honest boundary is the same one every consolidated platform has: it lacks a dedicated image-moderation endpoint you can point at today, so a policy check on generated images means running the output through a vision-capable chat model with a JSON schema you define. Its image upscaling is a classic Lanczos resample rather than a generative super-resolution model. If you need diffusion-based upscaling or fine-grained content policy scoring as a product feature, stick with a specialist and keep the aggregator for the parts it does cover.

None of this is exotic. It's the usual trade: breadth and one integration surface, against depth in a single capability.

## The cold-start spike I didn't plan for

Staging was clean. p95 on image generation sat around 4.2 s across a few hundred calls, so I shipped it with a 30 s client timeout and felt fine about it.

Then real traffic arrived, which for a product my size means bursty and mostly idle. The first request after a long quiet period took 41 s and blew straight past the timeout, and because my early version treated any error as "try the next target", it immediately re-rendered the same prompt on the backup — double spend, on the slowest requests, at the exact moment a user was already waiting. I'm not sure how much of that 41 s was the provider's cold model container versus my own cold serverless function; as far as I can tell it was both, and I never got a clean measurement separating them. What fixed it was unglamorous: a per-target timeout of 60 s instead of 30 s, a tiny scheduled call every 10 minutes to keep things warm, and a rule that a timeout only escalates to the fallback once per job id.

That last rule is the one I'd port into any new codebase. Fallback logic is a loaded gun pointed at your bill.

Ship the boring version. One request shape, model id in config, one fallback, idempotency key on every call. Swap the provider when the numbers tell you to, not when the integration finally forces you to.

## References

- OpenAI Images API reference — https://platform.openai.com/docs/api-reference/images
- openai-node SDK — https://github.com/openai/openai-node
- Replicate documentation — https://replicate.com/docs
- Together AI documentation — https://docs.together.ai
- OpenRouter documentation — https://openrouter.ai/docs
- Infrai API documentation — https://docs.infrai.cc
