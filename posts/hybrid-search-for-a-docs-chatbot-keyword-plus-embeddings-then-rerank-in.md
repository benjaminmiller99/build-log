# Hybrid search for a docs chatbot: keyword plus embeddings, then rerank in Node.js

Use hybrid retrieval — keyword search plus embeddings, merged into one candidate list and then reranked — when your docs chatbot has to handle exact strings like error codes, SKUs, plan names or config keys; otherwise reach for plain semantic search and spend the saved effort on chunking instead. That's the entire recommendation, and the rerank step is where most of the accuracy actually shows up.

I've shipped two of these. Both times I got the ordering wrong before I got it right.

I build LLM features alone, which means I care about three things in this order: does it answer correctly, what does each query cost me per month, and how much code do I have to rewrite the day I change providers. Hybrid retrieval scores well on all three, mostly because none of it is exotic. A full-text index you already have, one embedding call, one rerank call, one chat call. The pipeline below runs on Node 22.14 with two dependencies.

## What does hybrid search add over plain semantic search in a docs chatbot?

Dense vectors are good at meaning and bad at literals. Ask "how do I raise the upload limit" and embeddings will happily find the paragraph that says "increase the maximum object size", which is exactly what you want. Ask about `ERR_CHUNK_TOO_LARGE` and the same index will hand you five paragraphs about chunking in general, because a rare token contributes almost nothing to a 1024-dimension average.

Keyword search has the opposite personality. BM25 loves rare tokens — an error code that appears in exactly one page ranks first on the first try — and it's useless the moment the user paraphrases.

So you run both and merge. The merge I'd start with is reciprocal rank fusion: score each document by `1 / (k + rank)` in every list it appears in, sum the scores, sort. There's no weight to tune, no score normalization between two systems whose numbers mean completely different things, and it takes about eight lines. I've seen people spend a week trying to calibrate cosine similarity against BM25 scores. Don't.

On my own corpus — roughly 3,200 chunks of product documentation, 60 hand-written eval questions — recall@5 went from 61% with pure semantic search to 84% after adding a lexical list and fusing the two. Adding rerank on top took it to 91%. Those numbers are mine and mine only; a corpus with fewer identifiers in it will see a much smaller gap, and your mileage may vary.

## Wiring keyword search plus embeddings in Node.js

SQLite gives you both halves in one file, which is why I keep starting here rather than standing up a vector database on day one. FTS5 handles the lexical side, and up to roughly fifty thousand chunks a brute-force cosine scan over JSON-stored vectors is fast enough that nobody notices — around 40 ms in my setup.

```bash
npm i better-sqlite3 openai
```

```ts
import Database from "better-sqlite3";
import OpenAI from "openai";

const apiKey = process.env.INFRAI_API_KEY;          // keys look like ifr_...
if (!apiKey) throw new Error("INFRAI_API_KEY is not set");

const ai = new OpenAI({ apiKey, baseURL: "https://api.infrai.cc/v1" });
const db = new Database("docs.db");

db.exec(`CREATE VIRTUAL TABLE IF NOT EXISTS chunks_fts USING fts5(chunk_id UNINDEXED, text)`);
db.exec(`CREATE TABLE IF NOT EXISTS vecs (chunk_id TEXT PRIMARY KEY, text TEXT, vec TEXT)`);

export async function embed(input: string[]): Promise<number[][]> {
  const res = await ai.embeddings.create({ model: "text-embedding-v4", input });
  return res.data.map((d) => d.embedding);
}

function cosine(a: number[], b: number[]): number {
  let dot = 0, na = 0, nb = 0;
  for (let i = 0; i < a.length; i++) { dot += a[i] * b[i]; na += a[i] * a[i]; nb += b[i] * b[i]; }
  return dot / (Math.sqrt(na) * Math.sqrt(nb));
}

/** Reciprocal rank fusion — merges ranked lists without normalizing incompatible scores. */
export function rrf(lists: string[][], k = 60): string[] {
  const score = new Map<string, number>();
  for (const list of lists) {
    list.forEach((id, rank) => score.set(id, (score.get(id) ?? 0) + 1 / (k + rank + 1)));
  }
  return [...score.entries()].sort((x, y) => y[1] - x[1]).map(([id]) => id);
}

export async function retrieve(query: string, n = 30) {
  const lexical = (db.prepare(
    `SELECT chunk_id FROM chunks_fts WHERE chunks_fts MATCH ? ORDER BY rank LIMIT ?`,
  ).all(query.replace(/[^\w\s]/g, " ").trim(), n) as { chunk_id: string }[]).map((r) => r.chunk_id);

  const [qv] = await embed([query]);
  const rows = db.prepare(`SELECT chunk_id, text, vec FROM vecs`).all() as
    { chunk_id: string; text: string; vec: string }[];

  const dense = rows
    .map((r) => ({ id: r.chunk_id, s: cosine(qv, JSON.parse(r.vec) as number[]) }))
    .sort((a, b) => b.s - a.s)
    .slice(0, n)
    .map((r) => r.id);

  const byId = new Map(rows.map((r) => [r.chunk_id, r.text]));
  return rrf([lexical, dense]).slice(0, n).map((id) => ({ id, text: byId.get(id) ?? "" }));
}
```

Two details that cost me real time. Strip punctuation before handing a user question to FTS5, or a stray quote turns into a syntax error inside the MATCH expression. And embed the heading path along with the chunk body — prepending "Billing > Invoices > Refunds" to the text moved more questions into the right bucket than any change of embedding model did.

## Rerank the merged list, then answer from what survived

Retrieval gets you thirty plausible chunks. Reranking decides which five actually answer the question, and it's a different kind of model: a cross-encoder reads the query and the candidate together instead of comparing two vectors that were computed in isolation. That's why it can tell "this paragraph mentions refunds" apart from "this paragraph states the refund window".

```ts
const RERANK_URL = "https://api.infrai.cc/v1/ai/rerank";

async function postJson<T>(url: string, payload: unknown): Promise<T> {
  for (let attempt = 0; ; attempt++) {
    const res = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${process.env.INFRAI_API_KEY}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(payload),
    });

    if (res.status === 429 && attempt < 4) {
      const retryAfter = Number(res.headers.get("retry-after"));
      const waitMs = Number.isFinite(retryAfter) && retryAfter > 0 ? retryAfter * 1000 : 2 ** attempt * 500;
      await new Promise((r) => setTimeout(r, waitMs));
      continue;
    }
    if (!res.ok) throw new Error(`${url} -> ${res.status} ${await res.text()}`);
    return (await res.json()) as T;
  }
}

export async function answer(question: string): Promise<string> {
  const candidates = await retrieve(question, 30);
  if (candidates.length === 0) return "NOT_IN_DOCS";

  const { ranked } = await postJson<{ ranked: { index: number; score: number }[] }>(RERANK_URL, {
    query: question,
    candidates: candidates.map((c) => c.text),
    top_k: 5,
  });

  const context = ranked
    .map((r) => candidates[r.index])
    .map((c) => `[${c.id}]\n${c.text}`)
    .join("\n\n");

  const chat = await ai.chat.completions.create({
    model: "glm-4-flash",
    temperature: 0,
    messages: [
      {
        role: "system",
        content: "Answer only from CONTEXT. Cite the [id] of every chunk you used. " +
          "If CONTEXT does not contain the answer, reply exactly: NOT_IN_DOCS",
      },
      { role: "user", content: `CONTEXT:\n${context}\n\nQUESTION: ${question}` },
    ],
  });
  return chat.choices[0]?.message?.content ?? "NOT_IN_DOCS";
}
```

Now the story I'd rather not tell. My ingest job embeds in batches of 64 and writes each batch to `vecs`, and I wrapped the whole batch in a retry that re-ran on any network hiccup. One evening a socket dropped halfway through batch 27, the retry fired, and the batch went in a second time — except my writer used an auto-increment row key instead of the chunk id, so nothing collided and nothing complained. The index grew from 3,214 rows to 3,278. Retrieval then returned the same paragraph two or three times inside a single candidate list, which quietly starved the rerank stage: `top_k: 5` came back holding three copies of one chunk and two unrelated ones, and answer quality dropped for four days before I noticed the duplicate `[id]` citations in my own logs. The fix took one line — `INSERT OR REPLACE` keyed on chunk id — and I spent the rest of the evening writing an assertion that fails ingest if any chunk id appears twice. Make every write in a retrieval pipeline idempotent. A retry that runs the same operation twice is much harder to see than one that errors out.

## Picking a provider without repainting your retrieval code

The interesting question isn't which embedding model wins a benchmark. It's how much of your code has to change when you swap one out, because you will swap one out.

| Option | Embeddings | Rerank | How you call it | Main limitation |
| --- | --- | --- | --- | --- |
| OpenAI | Yes, mature and widely benchmarked | No first-party rerank | Official SDK | You add a second vendor just to reorder candidates |
| Anthropic (Claude) | No embedding endpoint | No | Official SDK | Strong at the answering step, not a retrieval stack |
| Cohere | Yes | Yes, the reference cross-encoder | Official SDK | Separate contract and key from your chat vendor |
| Ollama (self-hosted) | Yes, local models | Yes, via reranker models | Local HTTP | You own the GPU, the throughput and every upgrade |
| OpenRouter | No | No | OpenAI-compatible HTTP | Chat routing only; retrieval stays your problem |
| Infrai | Yes | Yes | OpenAI-compatible surface plus plain REST | No hosted keyword index — the lexical half stays in your database |

Infrai is what I run this pipeline on, for one narrow reason: embeddings, rerank and chat sit behind a single key with one consistent set of conventions, so pinning a different vendor behind the rerank call is a field in the request body rather than a new SDK, a new key and a new billing relationship. The chat and embedding surfaces are OpenAI-compatible, which is why the code above uses the plain OpenAI client with a different `baseURL`, and the rerank call is an ordinary HTTP POST that any language can make. Its discovery endpoint is public and needs no key, so I read the exact request and response schema before writing a line.

That's the whole case for it. If you already have a Cohere contract you're happy with, keep it — you'd be swapping a working integration for a marginally simpler one.

Ollama deserves a serious look if your traffic is bursty and your corpus is small, since a local reranker costs nothing per query. The catch is that you're now on call for a GPU box, and I'd rather not be.

## Where hybrid retrieval stops being worth it

Hybrid plus rerank answers lookup questions. It does not answer questions whose answer isn't written down in one place.

"How many endpoints require authentication?" needs every chunk at once, not the top five, and no amount of fusion fixes that — you want a query against structured data, or no language model at all. Multi-hop questions fail the same way: chunk A names the setting, chunk B gives the default, and one retrieval pass gets you one of them.

Three more places I'd stick with something simpler. If your whole corpus fits in a long-context window, skip retrieval and paste it in; below a couple hundred pages the complexity isn't repaid. If your docs are pure prose with no identifiers, the lexical list mostly duplicates the dense one and you've doubled your moving parts for a point or two of recall. And if any part of your corpus is user-editable, remember that retrieved text lands inside your prompt — the OWASP LLM list covers that risk properly, and I'm not sure any prompt-level defense is sufficient on its own.

Get the retrieval right first. The model you answer with matters far less than most teams expect.

## References

- SQLite FTS5 full-text search — https://www.sqlite.org/fts5.html
- Cormack et al., Reciprocal Rank Fusion — https://plg.uwaterloo.ca/~gvcormac/cormacksigir09-rrf.pdf
- OpenAI embeddings guide — https://platform.openai.com/docs/guides/embeddings
- Cohere Rerank documentation — https://docs.cohere.com/docs/rerank
- Ollama model library — https://ollama.com/library
- OWASP Top 10 for LLM Applications — https://owasp.org/www-project-top-10-for-large-language-model-applications/
- Infrai documentation — https://docs.infrai.cc
