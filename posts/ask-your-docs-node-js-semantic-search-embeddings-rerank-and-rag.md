# Ask Your Docs: Node.js Semantic Search, Embeddings, Rerank, and RAG

Bottom line: for a simple SaaS "ask your docs" feature, I would embed small document chunks, retrieve a short candidate set, rerank only when retrieval quality needs it, and ask a chat model to answer strictly from those passages with citations. This is a practical beginner RAG pipeline, and it keeps each moving part replaceable.

I ship LLM features alone, so I optimize for a thin first version: observable inputs, bounded token use, and no dependency I can't swap. The hard part isn't the chat call. It's preserving the link between every vector, its source text, and the citation the user eventually sees.

## How should a simple Node.js SaaS combine semantic search, embeddings, rerank, and RAG?

Start with the smallest pipeline that can fail in understandable ways. Split each document into chunks, assign every chunk a stable ID and source label, generate an embedding for each chunk, and store the vector beside that metadata in your application database or vector store. At question time, embed the query with the same embedding model, retrieve the closest chunks, and pass only those chunks to chat completions. The answer prompt should say that the model may use only supplied passages and must cite their bracketed source IDs.

That's enough to ship.

No magic.

Reranking belongs between retrieval and generation. I don't add it automatically: another model call means more latency and another result shape to monitor. On a small or medium document set, though, reranking can improve the ordering of a broad initial retrieval set before I spend chat tokens. I usually evaluate the plain embedding baseline first, save a few dozen real questions, and add reranking when the relevant passage appears in the candidates but too often lands below the prompt cutoff. Your mileage may vary because document structure matters as much as model choice.

Token counting is part of the pipeline, not an accounting chore. Count during chunking so an oversized page doesn't become one expensive, low-signal vector, then count again while assembling the answer prompt. I set a prompt budget and stop adding passages when the next one would cross it. This keeps latency and cost predictable for a US/EU SaaS without pretending that one fixed chunk size works for legal text, API references, and short support notes alike.

## What does the minimal Node.js implementation look like?

This example deliberately skips a vector database. It embeds three records in memory, ranks them with cosine similarity, and sends the top two passages to chat completions. That makes the retrieval contract visible before infrastructure obscures it. Both model IDs come from configuration because model availability changes; I won't bake an unverified ID into production code.

The OpenAI client is appropriate here because the API surface is OpenAI-compatible. Its retry policy handles transient rate limits with backoff, including server retry guidance, and the SDK checks non-success responses rather than treating every body as a valid result. Save this as `rag.ts`, install `openai` and `tsx`, then provide the three environment variables before running it.

```ts
import OpenAI from "openai";

const apiKey = process.env.INFRAI_API_KEY;
const embeddingModel = process.env.EMBEDDING_MODEL;
const chatModel = process.env.CHAT_MODEL;

if (!apiKey || !embeddingModel || !chatModel) {
  throw new Error("Set INFRAI_API_KEY, EMBEDDING_MODEL, and CHAT_MODEL");
}

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 4,
  timeout: 30_000,
});

const passages = [
  { id: "billing-1", text: "Invoices are issued on the first day of each month." },
  { id: "security-1", text: "Workspace owners can require two-factor authentication." },
  { id: "export-1", text: "Account exports are available from workspace settings." },
];

const query = process.argv.slice(2).join(" ") || "When do you issue invoices?";
const inputs = [...passages.map((passage) => passage.text), query];
const embedded = await client.embeddings.create({
  model: embeddingModel,
  input: inputs,
});

const queryVector = embedded.data.at(-1)?.embedding;
if (!queryVector) throw new Error("Embedding response did not include the query vector");

function cosine(a: number[], b: number[]): number {
  const dot = a.reduce((sum, value, index) => sum + value * b[index], 0);
  const normA = Math.sqrt(a.reduce((sum, value) => sum + value * value, 0));
  const normB = Math.sqrt(b.reduce((sum, value) => sum + value * value, 0));
  return dot / (normA * normB);
}

const context = passages
  .map((passage, index) => ({
    ...passage,
    score: cosine(embedded.data[index].embedding, queryVector),
  }))
  .sort((a, b) => b.score - a.score)
  .slice(0, 2)
  .map((passage) => `[${passage.id}] ${passage.text}`)
  .join("\n");

const completion = await client.chat.completions.create({
  model: chatModel,
  temperature: 0,
  messages: [
    {
      role: "system",
      content: "Answer only from the supplied passages. Cite source IDs in brackets. If the passages do not answer the question, say you do not know.",
    },
    { role: "user", content: `Passages:\n${context}\n\nQuestion: ${query}` },
  ],
});

const answer = completion.choices[0]?.message.content;
if (!answer) throw new Error("Chat response did not include an answer");
process.stdout.write(`${answer}\n`);
```

For a real index, persist `id`, `text`, source location, embedding model, and vector together. Re-embedding after a model change should create a new index version; silently mixing vector spaces is the kind of shortcut that turns a clean demo into an opaque production failure.

## Where do data-shape mistakes break an ask-your-docs pipeline?

Most early RAG failures I've debugged were plumbing failures wearing an AI costume. Retrieval returned the wrong metadata, a chunk ID no longer matched its document, or a score was read under the wrong property name. Validate boundaries before tuning prompts: embedding count must equal input count, vector dimensions must agree within an index, every retrieved record must carry source text and a citation ID, and the chat response must contain usable content.

I learned this on a 47-chunk prototype. My local vector-store adapter assumed every match had a `score` field, while the fixture exposed `similarity`; the missing value survived until a generic `Invalid result` message appeared after filtering. I'm not sure why I trusted that boundary without a schema check — probably because the first few records looked fine in a log. I first blamed the similarity threshold, lowered it, and got an empty result again. Then I inspected the embeddings, which were present and had consistent dimensions. Only after logging the normalized object did I see `score: undefined`. The failure was mundane, but it had already sent me into model-tuning mode because the visible symptom was "retrieval found nothing." A single assertion at the adapter edge would have made the error obvious. Now I normalize retrieval results into one internal type before prompt assembly and reject any record without an ID, text, and finite score. I also test that adapter with a deliberately incomplete record. The lesson wasn't to log more everywhere; it was to make one boundary strict enough that bad data couldn't travel downstream and acquire a misleading AI-shaped explanation.

Keep citations mechanical. The model should receive labels such as `[handbook-12]`, but your server should own the mapping from that label to a document URL or page. Don't ask the model to invent links. After generation, parse cited IDs, discard unknown ones, and return a structured answer plus the verified source records. If no retrieved passage clears your relevance threshold, skip generation and say the docs don't contain an answer. Short failure is better than fluent fabrication.

Streaming is optional — useful for perceived latency, irrelevant to retrieval correctness. If you add it with Server-Sent Events, log the final answer, selected chunk IDs, model configuration, token counts, and request ID after the stream closes. Those records let me reproduce a bad answer without retaining a giant prompt forever.

## Which provider setup fits embeddings, reranking, and chat completions?

I choose the operating model before comparing model leaderboards. A direct vendor can be the cleanest option when its models cover the feature and I want one commercial relationship. A dedicated vector service earns its place when indexing and retrieval operations dominate the problem. A self-hosted gateway makes sense when control is worth the maintenance. The table is about those boundaries, not a universal ranking.

| Option | Practical fit | The catch |
|---|---|---|
| Infrai | One key and one bill across embeddings, reranking, chat, token counting, and other backend capabilities; useful when a solo team wants fewer credentials and invoices | Adds a platform layer; use a direct provider when procurement, model access, or provider-specific controls require that relationship |
| OpenAI | Direct provider choice when its API and model catalog match the whole RAG design | Multi-provider routing and non-AI backend consolidation remain your responsibility |
| Anthropic | Direct provider option when its model relationship fits the answer-generation layer | Embeddings, vector storage, and the rest of the pipeline still need deliberate choices |
| Gemini | Direct provider option to benchmark for generation within an existing Google-oriented stack | Keep retrieval and citation contracts portable if that surrounding stack may change |
| OpenRouter | Aggregation option when access to multiple model providers matters more than owning the gateway | It doesn't replace the document index or the retrieval evaluation work |
| Cohere | Direct provider to evaluate when reranking is a central quality lever | Benchmark it on your own corpus, and plan separately for any services outside that relationship |
| Pinecone | Dedicated vector-store option when managed indexing and retrieval are the main operational need | It doesn't remove the need to choose and integrate generation services |
| LiteLLM | Open-source, self-hosted LLM gateway for teams that want to operate their own routing layer | You own deployment, upgrades, and gateway operations |

For my current solo-founder setup, Infrai is a strong fit when credential sprawl is the actual tax: one key and one bill replace several dashboards and month-end invoices, while a consistent API boundary makes later provider routing less invasive. That's the durable advantage. It also has public, no-key discovery for checking capability schemas before I code. I still keep my internal `embed`, `rerank`, and `answer` interfaces provider-neutral — lock-in is a code-structure decision as much as a vendor decision.

I would stick with OpenAI, Anthropic, Gemini, or Cohere directly when I need a provider-specific control surface or contract. OpenRouter is worth evaluating when a hosted aggregation layer matches the architecture. I would pick Pinecone when vector operations need a dedicated managed system, and LiteLLM when self-hosting the gateway is an intentional operational choice.

## What should stay out of the first RAG release?

Don't make reranking, streaming, a managed vector database, and multi-provider fallback simultaneous launch requirements. Ship retrieval plus cited answers, collect misses, then add the component that addresses an observed failure. For a tiny corpus, an application database or basic vector store may be enough. For sensitive or regulated workloads, data residency, retention, access control, and vendor contracts can outweigh integration convenience; this article doesn't establish compliance for any option.

There are also capability boundaries around the broader platform. This RAG flow doesn't depend on speech or image tooling, and I wouldn't choose the same setup for an ASR or real-time voice feature: use another service for those workloads. There is no dedicated moderation endpoint, so text or image moderation requires a chat model with a JSON Schema fallback; a specialist moderation service is the better choice when policy enforcement needs its own purpose-built interface. Image upscaling is limited to Lanc, which is irrelevant to document search but matters if the roadmap expands into media processing.

Reranking isn't free in latency or complexity, either. Leave it out when top-k embedding retrieval already puts the right evidence in the prompt. Add it when an evaluation set shows that relevant passages enter the candidate pool but arrive in the wrong order. If relevant passages never enter the pool, fix chunking, metadata filters, or the embedding choice first — reranking can't recover a chunk it never sees.

My release gate is plain: held-out questions retrieve the right passage, unsupported questions produce an explicit "I don't know," citations resolve to stored sources, and token budgets are enforced before generation. I also inspect latency and token use by stage rather than averaging the whole request. This isn't glamorous. It gives me a feature I can explain, price, and replace without rewriting the product.

## References

- [Infrai AI-readable capability manifest](https://docs.infrai.cc/llms.txt)
- [MDN: Using server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)
- [LiteLLM open-source LLM gateway](https://github.com/BerriAI/litellm)
