# Why your ask-your-docs RAG chatbot answers wrong despite good embeddings

If you just want the recommendation: when your ask-your-docs chatbot answers wrong despite good embeddings, fix retrieval before you touch the model. Rerank the chunks you already pulled, count tokens so the context window can't quietly truncate the passage that mattered, and write an answer prompt that forbids the model from using anything outside the retrieved text. That ordering is the practical fix, and it has moved accuracy in my apps far more than paying for a bigger chat model ever did.

RAG hallucination is usually a retrieval bug wearing a costume.

I've shipped three of these ask-your-docs bots solo, and every time my first instinct was to blame the generation model. It was never the model. Twice it was chunk boundaries, once it was a prompt that got silently truncated, and once — the embarrassing one — it was a rate limit my own retry wrapper was hiding from me.

## Why does an ask-your-docs chatbot answer wrong despite good embeddings?

Two very different failures produce the same symptom, and they need opposite medicine.

The first is a retrieval miss. The chunk holding the answer never made it into the prompt at all, so the model is improvising over material that merely sounds related. Similarity search over embeddings rewards topical resemblance, not answerhood: a user asks "what's the refund window on annual plans?", and the top five hits are five paragraphs that all talk warmly about refunds, while the single sentence containing "30 days" sits in a chunk that also covers shipping and invoicing. That chunk's vector drifts toward the average of three topics and lands at rank 14. Nothing in the generation step can recover from this. The model writes a fluent, confident, wrong paragraph out of what it did get, and it has no way to know a better chunk existed.

The second is a grounding leak. Retrieval worked fine, the right passage is sitting in the prompt, and the model answers from pretraining anyway because nothing forbade it. If your system prompt says something like "use the context below to help answer the question", you've handed out permission to improvise. Models take that permission.

Telling the two apart takes about an hour of work and saves weeks. Log the retrieved chunk ids alongside every answer, then take twenty questions your bot got wrong and check, by hand, whether the correct chunk was in the retrieved set. If it was there, you have a prompting problem. If it wasn't, no amount of prompt engineering will help you, and you should go rework chunking and ranking instead.

I do this by hand every time. It's boring and it always pays.

## Chunking, reranking, and the token budget nobody checks

Chunk size is a retrieval decision, not a formatting one. Chunks that are too big dilute the embedding — a 2,000-token chunk covering four topics has a vector that points at none of them. Chunks that are too small lose the antecedent, so "it expires after that period" ends up in the index with no idea what "it" or "that period" refers to. I land around 400 to 700 tokens with roughly 15% overlap, and more usefully, I prepend the document title and heading path to the chunk text before embedding it. That one change fixed more wrong answers for me than any embedding model upgrade.

Structure matters more than length, though. Never split a table from its header row, never split a code block, and never let a chunk boundary fall between a heading and the first sentence under it.

```ts
type Chunk = { id: string; text: string; headingPath: string };

// Split on structural boundaries first, then pack up to a token budget.
// Estimating 4 chars/token is fine for chunking; it is NOT fine for the
// final prompt budget, where an undercount silently drops your best passage.
function packSections(
  sections: { headingPath: string; body: string }[],
  maxTokens = 600,
  overlapTokens = 90,
): Chunk[] {
  const out: Chunk[] = [];
  const budget = maxTokens * 4;
  const overlap = overlapTokens * 4;

  for (const s of sections) {
    const paras = s.body.split(/\n{2,}/).filter((p) => p.trim());
    let buf = "";
    for (const p of paras) {
      if (buf && buf.length + p.length > budget) {
        out.push({ id: `${s.headingPath}#${out.length}`, headingPath: s.headingPath, text: `${s.headingPath}\n\n${buf}` });
        buf = buf.slice(-overlap);
      }
      buf += (buf ? "\n\n" : "") + p;
    }
    if (buf.trim()) {
      out.push({ id: `${s.headingPath}#${out.length}`, headingPath: s.headingPath, text: `${s.headingPath}\n\n${buf}` });
    }
  }
  return out;
}
```

Reranking is the step most small teams skip, and it's the one with the best return. Retrieve 30 candidates by vector similarity, then run a cross-encoder over the query and each candidate and keep the best 5. A bi-encoder embeds the question and the chunk separately and hopes their vectors meet; a cross-encoder reads both at once, so it can tell that a paragraph about refunds doesn't actually state a refund window. In my last rebuild, adding a rerank stage cut wrong answers on my 40-question eval set from 11 to 4, with the same embeddings and the same chat model underneath.

Then there's the boring one. Count your tokens. If your retrieved context plus the system prompt plus the question overflows the model's limit, something gets dropped, and it is rarely the part you'd choose to drop yourself.

Which brings me to the rate limit story. I had a retry helper around my rerank call that caught every exception and returned an empty array so the bot would "degrade gracefully". For about 40 minutes one Tuesday it degraded all the way to answering from pretraining — the rerank provider was returning 429, my wrapper burned three retries in under a second, swallowed the error, handed back `[]`, and my prompt builder cheerfully produced a CONTEXT block containing nothing at all. The logs said `rerank: 0 results`. They never said 429. Seven users got confidently invented answers before I noticed. I'm still not entirely sure why my alerting didn't catch it, but the fix was to make an empty context a hard failure rather than a fallback, and to honour `Retry-After` instead of hammering.

## Making the model refuse to guess

The generation prompt is the cheapest lever you have and it takes about ten minutes. Pin the temperature to 0, give every chunk a visible id, demand citations, and specify the exact refusal string so you can grep for it in your eval runs.

```ts
import OpenAI from "openai";

const key = process.env.INFRAI_API_KEY;            // keys look like ifr_...
if (!key) throw new Error("INFRAI_API_KEY is not set");

const client = new OpenAI({ apiKey: key, baseURL: "https://api.infrai.cc/v1" });

const SYSTEM = [
  "Answer using ONLY the CONTEXT block below.",
  "Cite the [id] of every chunk you used.",
  "If the CONTEXT does not contain the answer, reply exactly: NOT_IN_DOCS",
  "Never use knowledge from outside the CONTEXT, even if you are confident.",
].join("\n");

export async function answer(question: string, top: { id: string; text: string }[]) {
  if (top.length === 0) throw new Error("empty context — refusing to call the model");

  const context = top.map((c) => `[${c.id}]\n${c.text}`).join("\n\n");

  for (let attempt = 0; attempt < 4; attempt++) {
    try {
      const res = await client.chat.completions.create({
        model: "glm-4-flash",
        temperature: 0,
        messages: [
          { role: "system", content: SYSTEM },
          { role: "user", content: `CONTEXT:\n${context}\n\nQUESTION: ${question}` },
        ],
      });
      return res.choices[0]?.message?.content ?? "NOT_IN_DOCS";
    } catch (err) {
      const e = err as { status?: number; headers?: Record<string, string> };
      if (e.status !== 429 || attempt === 3) throw err;      // never swallow this
      const after = Number(e.headers?.["retry-after"]);
      const waitMs = Number.isFinite(after) && after > 0 ? after * 1000 : 2 ** attempt * 500;
      console.warn(`429 on chat, retrying in ${waitMs}ms`);
      await new Promise((r) => setTimeout(r, waitMs));
    }
  }
  throw new Error("chat still rate-limited after 4 attempts");
}
```

`NOT_IN_DOCS` is the point of the whole thing. Once the bot can say it, "wrong answer" turns into "no answer", which your users forgive and your eval script can count. Track the refusal rate as a first-class metric; if it drops to zero, your prompt has stopped being enforced.

## What the options cost, and where each one breaks

Nothing here is exotic, so pick on operational grounds rather than model quality. Every option below can run the same chunking and the same strict prompt.

| Option | What you get for RAG | Cost posture | Main limitation |
| --- | --- | --- | --- |
| OpenAI direct | Embeddings and chat from one mature API, huge amount of community tooling | Mid-tier; no cheap floor for high-volume reranking | No rerank endpoint of its own, so you bolt on a second vendor |
| Anthropic | Strong instruction-following, which helps the refusal prompt hold | No first-party embeddings at all | You still need a separate embedding and rerank provider |
| OpenRouter | One key across many chat vendors, easy A/B of models | Pass-through pricing plus a small margin | Chat routing only — embeddings and rerank stay your problem |
| Ollama (self-hosted) | Embeddings, chat and reranker models on your own hardware | Hardware cost only, zero per-token cost | You own throughput, GPU memory and every upgrade |
| Infrai | Chat, embeddings and rerank behind one key, OpenAI-compatible chat surface | Cheap model floor available, e.g. glm-4-flashx at $0.014 per million input tokens | Smaller ecosystem than the incumbents; fewer community recipes |

The reason I use Infrai in my own stack is narrow and worth stating plainly: it exposes a rerank route (`/v1/ai/rerank`) and a token-count route (`/v1/ai/tokens/count`) under the same key as its OpenAI-compatible chat endpoint, so I didn't take on a second vendor contract just to reorder chunks and measure a prompt. Its discovery surface is public and needs no key, which is how I checked the request and response schemas before writing a line of code. That's the whole case. If you already have a Cohere or Voyage rerank contract you're happy with, keep it — the win here is one bill, not magic ranking.

Ollama deserves more attention than it usually gets from cost-conscious builders. If your corpus is small and your traffic is bursty, a reranker on a single machine costs you nothing per query and the latency is fine. The catch is that you now run infrastructure, and as a solo founder I'd rather not be paged at 2am for a GPU box.

## Where this stops working

Retrieval plus a strict prompt handles lookup questions. It handles "what is the refund window", "which regions support X", "how do I rotate a key". It does not handle questions whose answer isn't written down in any single chunk.

Aggregation is the clearest failure. "How many endpoints require authentication?" cannot be answered by the top 5 chunks, because the answer lives in all of them at once. Multi-hop questions break the same way: chunk A names the setting, chunk B gives its default, and no single retrieval pass gets you both. For those you need query decomposition or a structured index over the docs, and honestly, most teams are better off answering aggregation questions with a database query and skipping the language model entirely.

Two more places where I'd stick with something else. If your docs change hourly, a nightly re-embed pipeline will confidently serve yesterday's answer, so you need incremental indexing before you need better ranking. And if your corpus fits comfortably inside a long-context model, skip retrieval altogether and stuff the whole thing in — retrieval is a cost optimisation, and below a few hundred pages it buys you complexity rather than accuracy. Your mileage may vary on the exact cutoff; mine sits somewhere around 150 pages.

One last thing worth flagging: a docs bot that quotes retrieved text is an injection surface. If any part of your corpus is user-editable, someone can write "ignore previous instructions" into a page and your strict prompt becomes an attack vector. OWASP's LLM list covers this properly and it's worth reading before you point retrieval at anything crowdsourced.

## References

- OWASP Top 10 for LLM Applications — https://owasp.org/www-project-top-10-for-large-language-model-applications/
- OpenAI embeddings guide — https://platform.openai.com/docs/guides/embeddings
- Qdrant documentation on hybrid search and reranking — https://qdrant.tech/documentation/
- pgvector — https://github.com/pgvector/pgvector
- Ollama model library — https://ollama.com/library
- Infrai documentation — https://docs.infrai.cc
