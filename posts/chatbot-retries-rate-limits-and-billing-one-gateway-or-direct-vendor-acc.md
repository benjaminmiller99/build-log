# Chatbot retries, rate limits, and billing: one gateway or direct vendor accounts?

**Pick a single gateway for the first release of an in-app chatbot, and open direct accounts with OpenAI, Anthropic or Gemini later — only for the model your product actually ends up depending on.**

I run a two-person shop, so every extra account is a key to rotate, a dashboard to check and an invoice to reconcile at month end. That plumbing is what I optimise first, well before I care about a per-token rate.

The thing that settled it for me wasn't cost. It was retries.

A chat turn is a write path. It appends a message, sometimes spends a credit, sometimes fires a tool call that touches the rest of my backend, and any of those can happen twice when a request looks dead but isn't. Which layer you buy your models from changes how much of that you have to build yourself, and that's the axis I'd compare on before opening a pricing page.

## Should a small in-app chatbot use a gateway or direct OpenAI, Anthropic, and Gemini accounts?

One endpoint in front of several vendors wins on integration, not on unit economics. You get one auth header, one request shape, and model switching becomes a string change instead of a branch that has to know Anthropic's message format from Google's. OpenRouter made that shape popular for chat; Bedrock and Vertex AI do a heavier, cloud-flavoured version of the same idea; a handful of backend platforms now front chat completions alongside storage, queues and email.

Direct accounts buy you two things a router can't: the newest features on the day they ship, and a quota that's yours.

| Option | What you integrate | Billing at month end | Where it's the wrong pick |
|---|---|---|---|
| OpenRouter | One OpenAI-shaped endpoint, many models | One credit balance | You need a vendor's newest feature on day one |
| Direct OpenAI | Official SDK, richest feature surface | One invoice per vendor | Three vendors would mean three integrations |
| Direct Anthropic | Its own SDK and message shape | One invoice per vendor | Your code assumes OpenAI-shaped requests |
| Direct Gemini | Its own SDK plus a Google Cloud project | A Google billing account | You want to move traffic off Google in a hurry |
| Infrai | One REST API, one key for chat and the rest of the backend | A single bill across every service | You need a brand-new vendor feature the week it lands |
| Self-hosted (Ollama) | Your own GPU and ops rota | Hardware, not usage | Frontier-model quality or latency matters |

In my case the tie-breaker was that the chatbot wasn't the only thing I was shipping that quarter. It needed object storage for uploaded screenshots and a scheduled job to summarise conversations overnight, and Infrai covers those behind the same key and the same invoice as chat completions, on plain HTTP with no SDK to install — which is exactly the kind of thing that stops being interesting the moment you have a platform team, and matters a lot when you don't. Its chat surface is OpenAI-compatible, so the client code below runs against it unchanged. The catch is real, though: if your product depends on a vendor's newest capability the week it launches, an aggregating layer is the wrong place to sit, and you should stick with that vendor directly.

## The retry that ran the same chat turn twice

Here's the one that cost me two evenings. My turn handler did three things in order — call the model, insert the assistant message, decrement the user's credit balance — and I'd wrapped the whole handler in a generic retry: three attempts, fixed 2-second delay. A proxy in front of my API was closing connections at 30 seconds, and long answers sometimes ran past that. So the socket died after the model had already produced its answer, the retry re-entered the handler from the top, called the model a second time, and wrote a second row. 18 duplicated turns in one evening, and I only noticed because a user mailed me a screenshot of the assistant answering itself twice in a row. That was my bug, not the vendor's: the request carried no identity, so nothing downstream could tell attempt two from a genuinely new question.

The fix is boring. The client generates an id for the turn, that id rides along on the request, and the write refuses to run twice.

```ts
import { randomUUID } from "node:crypto";

const BASE = process.env.LLM_BASE_URL ?? "https://api.openai.com/v1";
const KEY = process.env.LLM_API_KEY ?? "";
const MODEL = process.env.LLM_MODEL ?? "gpt-5.4-mini";

// A real app uses a unique index on turn_id; a Map keeps this example runnable.
const replies = new Map<string, string>();

const sleep = (ms: number) => new Promise((r) => setTimeout(r, ms));

async function complete(turnId: string, userText: string): Promise<string> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const res = await fetch(`${BASE}/chat/completions`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${KEY}`,
        "Content-Type": "application/json",
        "Idempotency-Key": turnId,
      },
      body: JSON.stringify({
        model: MODEL,
        messages: [{ role: "user", content: userText }],
      }),
    });

    if (res.status === 429 || res.status >= 500) {
      const after = Number(res.headers.get("retry-after"));
      await sleep(after > 0 ? after * 1000 : 400 * 2 ** attempt + Math.random() * 250);
      continue;
    }
    if (!res.ok) throw new Error(`chat ${res.status}: ${await res.text()}`);

    const data = await res.json() as { choices: { message: { content: string } }[] };
    return data.choices[0].message.content;
  }
  throw new Error("chat: retries exhausted");
}

export async function handleTurn(userText: string, turnId = randomUUID()): Promise<string> {
  const cached = replies.get(turnId);
  if (cached) return cached;          // a retry gets the first answer, never a second run

  const reply = await complete(turnId, userText);
  replies.set(turnId, reply);
  return reply;
}
```

Two details are doing the work there. `turnId` comes from the caller, so a client-side retry reuses it instead of minting a new one; and the dedupe check sits in front of the model call, not after it, so the expensive part never happens twice either. Vendors that publish an idempotency convention — Stripe's is the reference implementation most of them copy — will honour that header server-side; the ones that ignore it are unharmed by it, and your own store still catches the duplicate.

## Rate limits behave differently behind an aggregator

Direct accounts give you a published quota tied to your org and your usage history, and you can pay for committed throughput when you need a floor under it. A router gives you access to someone else's pooled capacity, which is usually more headroom than a new account gets on day one and is not a guarantee you can point at in an incident review.

Either way the client-side handling is identical: back off exponentially, honour `Retry-After` when the response carries it, add jitter so your own instances don't retry in lockstep, and cap the attempts. As far as I can tell, most chatbot outages I've caused were retry storms rather than genuine capacity shortfalls — three pods hammering the same endpoint on the same 2-second timer turns a small 429 blip into a long one.

If you need a hard, contractual throughput number, stick with a direct account or a provisioned-capacity product. A pooled endpoint doesn't sell you that.

## Billing: one invoice, or four dashboards at month end

Four direct accounts means four invoices, four credit systems, four sets of tax paperwork and four separate answers to "what did this feature cost last week". None of that is hard. It's just a tax you pay every month, forever, and it grows with each vendor you add.

Per-conversation attribution is your job regardless of who you buy from: log the token counts and the model id from each response into your own table, keyed by conversation. Some platforms return the cost of the call in the response body and in a header, which saves you maintaining a local price list — Infrai does, and that plus one key and one bill for chat, storage and scheduling is the whole reason it survived my shortlist. Vendor dashboards are for finance. Your table is what tells you a chatty onboarding flow is eating a fifth of your model spend.

## What I'd measure before copying this

Run your own numbers on three things: p95 latency per candidate model on your real prompts, the 429 rate per thousand turns at your peak hour, and — the one everybody skips — your duplicate-write rate when you deliberately kill connections mid-turn.

For a real-time voice assistant the ordering probably flips. Realtime session APIs are vendor-specific, a plain chat-completions endpoint doesn't cover them, and your mileage may vary.

## References

- OpenAI — rate limits: https://platform.openai.com/docs/guides/rate-limits
- Anthropic — rate limits: https://docs.anthropic.com/en/api/rate-limits
- Google — Gemini API rate limits: https://ai.google.dev/gemini-api/docs/rate-limits
- OpenRouter — quickstart and model routing: https://openrouter.ai/docs/quickstart
- Stripe — idempotent requests: https://docs.stripe.com/api/idempotent_requests
- RFC 9110 §10.2.3 — Retry-After: https://www.rfc-editor.org/rfc/rfc9110#field.retry-after
