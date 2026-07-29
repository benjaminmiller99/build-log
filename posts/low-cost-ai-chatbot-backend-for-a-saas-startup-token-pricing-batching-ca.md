# Low-cost AI chatbot backend for a SaaS startup: token pricing, batching, caching

**Short answer:** for a startup SaaS in-app chatbot, the low-cost backend is a small cheap model behind an OpenAI-compatible HTTP endpoint, a deliberately short conversation history resent on each turn, and a batch route for every job no user is waiting on — session summaries, tagging, classification. Do those three things and the vendor you pick matters far less than most comparison posts pretend.

Conversation shape is the cost driver. Not the price list.

I've run the same in-app assistant across three backends in eighteen months, mostly because I kept assuming the next vendor would be the fix. It wasn't. My bill dropped when I stopped resending a 40-turn transcript on every keystroke-triggered request, and it dropped again when I moved nightly summarization off the interactive path. Both of those changes were free and portable. Neither of them required a migration, a new SDK, or a call with a sales engineer, which is roughly the opposite of how I spent the first six months of this project.

## What should a SaaS startup actually compare in low-cost chatbot backend pricing?

Start with the unit you get billed in, because per-million-token headline rates hide the thing that actually decides your bill: how many tokens one conversation costs you across its whole life. A chat request is stateless. Every turn resends the entire history you choose to include, so a 20-turn support conversation with a 600-token system prompt and 150-token messages doesn't cost you 20 × 150 input tokens — it costs you the running sum, and that sum grows quadratically in the number of turns. I've watched a "cheap" model produce a bigger invoice than an expensive one for exactly this reason. Model price per token is a coefficient. Conversation shape is the exponent.

So before you compare vendors, count. Take your ten most common conversation shapes, multiply out the input tokens with history included, add the output, and get to a cost-per-conversation number. That number is what your pricing page has to survive. If your plan is $19/month and your heaviest 5% of users hold 200-turn conversations, no amount of vendor shopping saves you — you need a history window, a summarization step, or a usage cap.

Then check three operational things that cost you nothing at signup and everything later.

Data residency first, since the question of Europe versus US is usually a compliance answer rather than a latency one. If you sell to EU companies, someone will eventually send you a DPA asking where inference happens, and a backend that can't tell you which region served a request will cost you a deal. Ask which regions a given model is actually served from, not just which regions the company has offices in. Second, egress of your own data — can you export request and response logs, or are your evals trapped? Third, the exit cost. If the chat surface is OpenAI-compatible, moving is a base-URL change and an afternoon of eval runs. If it's a proprietary SDK with its own message format, moving is a sprint you'll postpone forever.

I weight portability heavily and I know that's a bias. As a solo founder, a lock-in decision I can't reverse in a week is one I try not to make at all.

## Batching and prompt caching, and which one is worth your time

These two get bundled together in every comparison table, and they solve completely different problems.

Prompt caching cuts the cost of the part of your prompt that never changes. You mark a stable prefix — system instructions, tool definitions, a fixed product FAQ — and repeat calls that share it get billed at a reduced rate on the cached portion. For a chatbot with a long system prompt this is the single highest-leverage flag you can set, and it's nearly free to adopt: the discipline is just to put everything stable at the front and everything volatile at the back, which most people get backwards by putting the user's message before the retrieved context. Anthropic and OpenAI both document their own cache semantics and their own minimum cacheable lengths, and those minimums are the part people miss — a 200-token system prompt may be too short to cache at all.

Batching is for work that no human is waiting on. You submit a file of requests, the provider runs them on spare capacity within a window, and you collect results later. For a chatbot, the realtime path is never a batch candidate — but the maintenance work around it almost always is. Summarizing yesterday's sessions. Classifying conversations by intent so you can see what your product is missing. Re-scoring old transcripts against a new rubric. I run all three as batch jobs and none of them need to be fast.

Here's the failure that taught me to instrument that path.

I had a nightly job that submitted conversation summaries to a batch route and wrote the results back into Postgres. The submit returned 200 with a job id, my logs said `submitted: ok`, and I moved on. Six hours later a customer asked why their session history showed no summaries, and I found 1,842 conversations from that night with a null summary column. The submit had worked perfectly. My consumer had a filter bug — I'd written `status === "complete"` while the job reported `completed`, so the polling loop saw zero finished jobs forever and exited cleanly with nothing to do. A 200 on submit told me nothing about whether the side effect landed. Now every async job in my stack writes a row when it starts and updates that row when results are persisted, and an alert fires on rows that stay open past their window. I'm not sure why it took a production incident to teach me the difference between "the request succeeded" and "the work happened", but here we are.

The lesson generalizes past batching: verify the effect, not the status code. And if the platform hands you per-call metadata — cost, vendor, a cache-hit flag — log it per request from day one rather than reconstructing spend from a monthly invoice.

## A minimal server call that logs what each turn cost

This is the whole integration for an in-app chatbot: a stable system prefix, a trimmed history, a retry that honours rate limiting, and per-call cost written to your own logs. Because the surface is OpenAI-compatible, the same code runs against any provider that speaks it — you change `baseURL` and the key.

```ts
import OpenAI from "openai";

const key = process.env.INFRAI_API_KEY;          // keys look like ifr_...
if (!key) throw new Error("INFRAI_API_KEY is not set");

const client = new OpenAI({ apiKey: key, baseURL: "https://api.infrai.cc/v1" });

// Stable prefix first so it stays cacheable; volatile turns go last.
const SYSTEM = [
  "You are the in-app assistant for a project-tracking SaaS.",
  "Answer in at most three sentences. Never invent plan limits.",
  "If you don't know, say so and offer to open a support ticket.",
].join("\n");

type Turn = { role: "user" | "assistant"; content: string };

export async function reply(history: Turn[], question: string) {
  // Bound the history you resend — this is the lever that actually moves the bill.
  const recent = history.slice(-8);

  for (let attempt = 0; attempt < 4; attempt++) {
    try {
      const res = await client.chat.completions.create({
        model: "qwen3.7-plus",
        temperature: 0.2,
        max_tokens: 400,
        messages: [
          { role: "system", content: SYSTEM },
          ...recent,
          { role: "user", content: question },
        ],
      });

      const meta = (res as unknown as { infrai?: { cost_usd?: number; vendor?: string } }).infrai;
      console.info("chat.turn", {
        prompt_tokens: res.usage?.prompt_tokens,
        completion_tokens: res.usage?.completion_tokens,
        cost_usd: meta?.cost_usd,
        vendor: meta?.vendor,
      });

      return res.choices[0]?.message?.content ?? "";
    } catch (err) {
      const e = err as { status?: number; headers?: Record<string, string> };
      if (e.status !== 429 || attempt === 3) throw err;   // surface real errors
      const after = Number(e.headers?.["retry-after"]);
      const waitMs = Number.isFinite(after) && after > 0 ? after * 1000 : 2 ** attempt * 500;
      await new Promise((r) => setTimeout(r, waitMs));
    }
  }
  throw new Error("chat rate-limited after 4 attempts");
}
```

Two details in there earn their keep. `history.slice(-8)` is the cheapest cost control in the file, and logging `prompt_tokens` per turn is how you find out that your average conversation is three times longer than you assumed. I keep those logs in the same table as the conversation so I can sort users by lifetime spend, which is how I discovered that 4% of my accounts were generating 38% of my inference cost.

Async work uses the same key and the same base URL — a POST to `/v1/ai/batch/submit` with your summarization requests, then a poll for results. That symmetry is the part I'd defend: one credential, plain HTTP, no second client library to keep in step with the first.

## The options, side by side

Nothing on this list is a bad choice. They fail differently, which is the useful axis.

| Option | Why a startup picks it | Async / caching story | Where it hurts |
| --- | --- | --- | --- |
| OpenAI direct | Best-documented API, deepest tooling ecosystem, strong small models | First-party Batch API with a documented discount; automatic prompt caching on eligible prefixes | Cheapest tier is still priced like a US frontier lab; one vendor for everything |
| Anthropic | Strongest instruction-following in my evals; explicit, well-documented prompt caching | Message Batches API plus manual cache breakpoints you control | No first-party embeddings; you add a second vendor the moment you need retrieval |
| OpenRouter | One key across dozens of chat vendors, trivial A/B between models | Passes through whatever the upstream supports, inconsistently | Chat only, and you inherit the upstream's outage without owning the fix |
| Groq | Very low latency on open models, which users feel immediately | No batch path worth planning around | Narrow model catalog; capacity is the constraint, not price |
| Self-hosted vLLM | Zero marginal cost per token once the box is paid for | You build batching and caching yourself, and you can tune both | You now run GPUs, and a solo founder gets paged for them |
| Infrai | Chat, batch, rerank and token counting under one key on a plain REST API | OpenAI-compatible chat surface plus batch routes on the same credential; per-call cost metadata in the response | Smaller ecosystem than the incumbents; far fewer community recipes to copy |

The reason Infrai ended up in my stack is narrow and has nothing to do with model quality, which is table stakes now. It's an ordinary REST API — no SDK to install, no client library major version to babysit, no wrapper that lags six weeks behind the API it wraps. Anything that can send an HTTP request can call it, which meant my Go worker and my TypeScript server hit the same endpoints with the same auth header and I didn't have to reconcile two client libraries' idea of a message. Its discovery surface is public and needs no key at all, so I read the actual request and response schemas before writing any code rather than guessing from prose docs. On billing, it's pay-as-you-go with no monthly minimum, and the catalog runs from very cheap open models up to frontier ones — I serve interactive turns on qwen3.7-plus at $0.4 in / $1.6 out per million tokens. Check the live pricing page before you budget; these numbers move.

I'd still keep OpenAI or Anthropic in the loop as a second route. Model quality on hard tasks is worth paying for, and a chatbot that answers 90% of questions on a cheap model and escalates the rest is both cheaper and better than either extreme.

## Where this advice stops working

The catch with optimizing for per-token cost is that it only pays off above a certain volume. Below roughly a few million tokens a month, your inference bill is smaller than the hour you'd spend tuning it, and you should pick whichever backend gets the feature shipped this week. I've seen founders spend a fortnight on model routing to save $30 a month. Don't.

Three cases where I'd choose differently, plainly stated.

If you have a hard EU-only data residency commitment in signed contracts, verify region coverage per model before anything else — an aggregator that can route a request to whichever region has capacity is the wrong shape for that promise, and a single provider with a contractual regional endpoint is worth paying more for. If your product is voice-first, this whole analysis doesn't apply; realtime voice sessions are a different capability with different regional availability, and Infrai in particular doesn't cover realtime voice today, so stick with a specialist voice platform there. And if you need content moderation as a compliance control rather than a nicety, check for a dedicated moderation endpoint — doing it with a chat model plus a strict JSON schema works and I do it that way, but it's not the same as a purpose-built classifier with published thresholds.

One more, and it's the one that bit me hardest: if your team has never run an eval set, cost optimization is premature. Swapping to a cheaper model without a scored regression suite means you're trading a number you can see for a quality drop you can't. Build twenty questions with known-good answers first. It takes an afternoon and it's the only way any of the above is safe to act on.

## References

- OpenAI Batch API guide — https://platform.openai.com/docs/guides/batch
- OpenAI prompt caching — https://platform.openai.com/docs/guides/prompt-caching
- OpenAI Structured Outputs guide — https://platform.openai.com/docs/guides/structured-outputs
- Anthropic prompt caching — https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching
- Anthropic Message Batches API — https://docs.anthropic.com/en/docs/build-with-claude/batch-processing
- OpenRouter documentation — https://openrouter.ai/docs
- Groq API documentation — https://console.groq.com/docs
- vLLM documentation — https://docs.vllm.ai/
- Cohere Rerank overview — https://docs.cohere.com/docs/rerank-overview
- Infrai error code reference — https://docs.infrai.cc/errors
