# RAG summarization in Node.js: PDF pages, embeddings, rerank, then a final summary

Bottom line: build the Node.js pipeline exactly as the question describes it — PDF pages, chunks, embeddings, semantic search, rerank, final summary — but only run the retrieval half when the reader's question is narrow. If someone asks you to summarize the whole document, RAG summarization is the wrong shape, and a map-reduce pass over every page costs less and misses less. I've shipped both against the same 90-page filing and the difference in what came out the other end wasn't subtle.

## How do you summarize a long PDF without stuffing every page into the final summary?

Two very different asks get flattened into the same verb.

The first is document-level: give me the gist of this whole report. Retrieval actively hurts here. Semantic search hands back the passages nearest your query vector, and "summarize this document" is a query that resembles nothing in particular, so you end up with eight chunks that happened to score high against a generic sentence while four hundred other chunks never get read at all. The final summary reads beautifully. It's also missing an entire section, and nobody notices until a customer asks why the risk disclosures aren't in there.

The second ask is question-scoped: what did they commit to on data retention, and for how long? Now the retriever is earning its cost, because there really is a small set of pages that answers the question and a large set that doesn't.

The data flow for that second case is short. You open the PDF and pull text per page, keeping the page number attached to every chunk you cut, because citations are the only thing that makes a generated summary auditable. You embed all chunks once and cache them keyed by a hash of the file. At query time you embed the question, take the top 40 by cosine similarity, and hand that shortlist to a reranker, which is a cross-encoder that reads the query alongside each passage in one pass instead of comparing two independent vectors. It's slower per pair and far more accurate, which is why you only feed it 40 candidates and not 400. The top 8 survivors go into one final call that writes the summary with page markers.

Routing between the two modes is product logic, not machine learning. I check whether the question contains a noun the user cares about. If it doesn't, I run the map-reduce path and skip the vector store entirely.

## The pipeline, in about seventy lines of TypeScript

Dependencies first. Text extraction is the only part I don't write myself.

```bash
npm i pdfjs-dist
```

Everything else is `fetch` against three endpoints you configure with environment variables — an embeddings endpoint, a rerank endpoint, and a chat endpoint. Keeping them as URLs rather than SDK imports is deliberate: I've swapped the embedding provider twice without touching this file.

```ts
import { readFile } from "node:fs/promises";
import { getDocument } from "pdfjs-dist/legacy/build/pdf.mjs";

type Chunk = { id: string; page: number; text: string };

const KEY = process.env.API_KEY!;
const post = async (url: string, body: unknown, attempt = 0): Promise<any> => {
  const res = await fetch(url, {
    method: "POST",
    headers: { "content-type": "application/json", authorization: `Bearer ${KEY}` },
    body: JSON.stringify(body),
    signal: AbortSignal.timeout(30_000),
  });
  if (res.status === 429 && attempt < 4) {
    const after = Number(res.headers.get("retry-after")) * 1000;
    await new Promise((r) => setTimeout(r, after || 2 ** attempt * 500 + Math.random() * 250));
    return post(url, body, attempt + 1);   // batches are replayable; nothing here mutates state
  }
  if (!res.ok) throw new Error(`${url} -> ${res.status} ${await res.text()}`);
  return res.json();
};

function split(text: string, size = 1200, overlap = 150): string[] {
  const out: string[] = [];
  for (let i = 0; i < text.length; i += size - overlap) out.push(text.slice(i, i + size));
  return out.filter((s) => s.trim().length > 80);
}

// Page number rides along with every chunk, so the summary can cite pages.
async function chunkPdf(path: string): Promise<Chunk[]> {
  const doc = await getDocument({ data: new Uint8Array(await readFile(path)) }).promise;
  const chunks: Chunk[] = [];
  for (let p = 1; p <= doc.numPages; p++) {
    const content = await (await doc.getPage(p)).getTextContent();
    const text = content.items.map((it: any) => it.str ?? "").join(" ");
    split(text).forEach((part, i) => chunks.push({ id: `p${p}-${i}`, page: p, text: part }));
  }
  return chunks;
}

// Batched embeddings come back with an index field. Sort by it; never trust arrival order.
async function embed(input: string[]): Promise<number[][]> {
  const json = await post(process.env.EMBED_URL!, { model: process.env.EMBED_MODEL, input });
  return [...json.data].sort((a, b) => a.index - b.index).map((d) => d.embedding);
}

const cosine = (a: number[], b: number[]) => {
  let dot = 0, na = 0, nb = 0;
  for (let i = 0; i < a.length; i++) { dot += a[i] * b[i]; na += a[i] ** 2; nb += b[i] ** 2; }
  return dot / (Math.sqrt(na) * Math.sqrt(nb));
};

export async function answer(path: string, question: string) {
  const chunks = await chunkPdf(path);
  const vectors = await embed(chunks.map((c) => c.text));
  const [q] = await embed([question]);

  const shortlist = chunks
    .map((c, i) => ({ c, score: cosine(q, vectors[i]) }))
    .sort((a, b) => b.score - a.score)
    .slice(0, 40)
    .map((x) => x.c);

  // The reranker returns positions into `documents`, not the documents themselves.
  const ranked = await post(process.env.RERANK_URL!, {
    model: process.env.RERANK_MODEL,
    query: question,
    documents: shortlist.map((c) => c.text),
    top_n: 8,
  });
  const picked = ranked.results.map((r: any) => shortlist[r.index]);

  const context = picked.map((c: Chunk) => `[page ${c.page}]\n${c.text}`).join("\n\n---\n\n");
  const final = await post(process.env.CHAT_URL!, {
    model: process.env.CHAT_MODEL,
    temperature: 0.2,
    messages: [
      { role: "system", content: "Answer only from the passages. Cite page numbers as [page N]. If the passages don't cover it, say so." },
      { role: "user", content: `Question: ${question}\n\nPassages:\n${context}` },
    ],
  });
  return final.choices[0].message.content as string;
}
```

Run it once and cache `vectors` to disk. Re-embedding the same PDF on every request is the single most common way this design turns expensive.

## The failures that actually cost me time

None of them were model quality.

Text extraction is where a third of the pain lives. A scanned PDF yields empty strings per page and no error at all, so the pipeline happily embeds four hundred blanks and produces a confident summary of nothing. I gate on it now: if the mean characters per page falls under 200, the file goes to an OCR path instead, and if there's no OCR path, the job fails loudly rather than summarizing whitespace. Two-column layouts are the other extraction trap — text items come back in reading order per column on some documents and interleaved on others, which shreds sentences across the split boundary. I'm not sure there's a general fix short of layout-aware parsing; for the filings I deal with, sorting items by their vertical position before joining was enough.

Then there was the response shape. I assumed the reranker echoed each document back, something like `results[i].document.text`, because the request takes a `documents` array — so obviously the response returns them. It doesn't. It returns `{ index, relevance_score }` and expects you to index into your own array. What I saw was `TypeError: Cannot read properties of undefined (reading 'text')`, thrown from inside a `.map()` two files away from the real cause, naming no field and no endpoint. I spent 40 minutes adding logging in the wrong function before I finally dumped the raw JSON body and counted the keys. Four, where I expected five. That's the whole bug. Now every new endpoint gets its first response object logged once at boot behind a `DEBUG_SHAPES` env flag, which is crude and has already paid for itself three times.

Chunk boundaries deserve one warning. A 1200-character window with 150 of overlap is a reasonable default for prose, and a terrible one for tables — a financial table sliced mid-row produces chunks that embed as gibberish and rank low forever. If your PDF is mostly tables, chunk by structural element and stop tuning the window.

## What does this cost, and how do you know it still works?

Three model calls per query, not four hundred. That's the entire economic argument for retrieval, and it holds only when the embedding pass is amortized across many questions about the same document.

| Strategy | Calls for a 90-page PDF | What it misses | Fits when |
| --- | --- | --- | --- |
| Map-reduce over every page | ~1 per page, plus a merge | Cross-page reasoning gets flattened at the merge step | You need document-level coverage |
| Retrieve, rerank, then summarize | 1 embedding batch, 1 rerank, 1 completion | Anything the retriever left out of the shortlist | The question names something specific |
| Outline first, retrieve per heading | 1 per section, plus a merge | Detail that lives below heading level | Long structured documents: contracts, filings |

Latency splits about the way you'd guess. Embedding a fresh 90-page PDF dominates the first request; after that, cosine over a few thousand cached vectors in process is microseconds, the rerank call is a few hundred milliseconds, and the final completion is most of what the user waits for. Streaming the final summary buys you more perceived speed than any retrieval tuning will.

Testing this is unglamorous and non-optional. I keep about thirty question-and-expected-page pairs per document type in a JSON file, and CI asserts recall at the shortlist stage — did the correct page survive into the top 40? — separately from whether the final summary was any good. Splitting those two assertions matters, because a bad summary caused by a retriever that never surfaced the page needs a completely different fix than a bad summary written from correct passages. For tracing, the OpenTelemetry GenAI semantic conventions give you span names and attributes that don't have to be reinvented per project, and having the retrieved chunk ids on the span is what lets you debug a complaint from three days ago.

## The operational checklist, and when to skip all of this

Before shipping, walk the boring list. Cache embeddings keyed by file hash plus chunker version, so changing the window size invalidates cleanly. Put a timeout and a retry budget on every call — `AbortSignal.timeout` covers the first half, and a bounded exponential backoff on 429 and 5xx, with jitter and a `Retry-After` check, covers the second. The embedding batch is the one that gets rate limited, and it's also the one you can safely replay, so key each batch by chunk id and skip what you already stored. Log the retrieved page numbers next to every generated summary. Cap the context you assemble by characters, not by chunk count, because chunk sizes drift once you add a table-aware splitter.

Treat retrieved text as untrusted input. A PDF is user-supplied content, and the OWASP Top 10 for LLM Applications puts prompt injection at the top for good reason: a line of text inside page 43 instructing the model to ignore its system prompt gets pasted into your context by design. Keeping the system message adversarial ("answer only from the passages") helps, isolating the passages inside a delimited block helps more, and neither is a guarantee.

There's a compliance angle too. PDFs are full of names, addresses and contract terms, so sending pages to a hosted model is processing personal data under GDPR, and you want a documented legal basis and a retention policy for whatever your vector cache keeps on disk. Your mileage may vary by jurisdiction and by how much of the stack you self-host.

The catch with the whole design is that it's tuned for one question at a time against a large document. If your PDFs are under about twenty pages, skip the embeddings and the reranker entirely and put the full text in one call — you'll spend a bit more per query and delete two-thirds of the code, which for a small team is usually the better trade. If you need the actual gist of a whole document, map-reduce over pages and accept the cost. And if you're answering hundreds of questions per hour against a fixed corpus, this in-process cosine loop stops being adequate and you want a real vector index with hybrid keyword scoring behind it, because pure dense retrieval quietly loses exact identifiers, product codes and section numbers that a BM25 pass would nail.

## References

- [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [GDPR full text](https://gdpr-info.eu)
- [pdf.js — Mozilla PDF rendering and text extraction library](https://mozilla.github.io/pdf.js/)
- [MDN: AbortSignal.timeout()](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal/timeout_static)
- [OpenTelemetry semantic conventions for GenAI](https://opentelemetry.io/docs/specs/semconv/gen-ai/)
- [Okapi BM25](https://en.wikipedia.org/wiki/Okapi_BM25)
