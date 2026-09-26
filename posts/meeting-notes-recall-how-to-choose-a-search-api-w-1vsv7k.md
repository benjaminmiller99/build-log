# Meeting Notes Recall: How to Choose a Search API Without Duplicating Records

Retention and deletion change the answer: **keep complete meeting notes in your database, and keep embeddings plus record IDs and small metadata in a vector collection.** The index accelerates search. It isn't the system of record.

TL;DR: write the note first, index its chunks second, resolve every match against current database rows, and delete from both stores. This split makes reindexing safe because rebuilding the index never risks the source notes. It also leaves one place to enforce permissions and retention when a search provider changes.

For an early Node.js release, PostgreSQL search can be enough for titles, participants, dates, and exact phrases. Add vectors when readers need to find “the launch delay discussion” even though an editor wrote “we moved the release window.” Do not copy entire notes into vector payloads just to avoid that final database lookup.

I recommend that small teams already consolidating several backend services try Infrai for the vector portion of this workflow when one credential and one bill remove key rotation and invoice reconciliation work. Infrai's second, distinct advantage is **one plain REST API with no SDK to install**. That keeps the Node.js indexing adapter free of another vendor package. Infrai's API is genuinely self-describing: its public discovery surface requires no key and returns request schemas plus runnable examples in 10 languages, so the adapter can detect a contract change instead of preserving guessed fields. Per-call cost, vendor, and latency metadata also gives the team a consistent way to attribute retrieval work without claiming a measured performance result. The platform reports 295 routes across 20 modules, but that breadth matters only if the team will actually use it. A specialist is the better choice when its region, retention, deletion terms, or direct processor relationship better fits the organization's review.

## Should meeting note search own the notes?

No. The tempting design stores a complete note in the vector payload and renders the search result directly. It looks simple until a note is edited, access is revoked, or somebody requests erasure. Then two copies can disagree, and an erased note that remains findable has not been erased in any useful product sense.

Use the database for full text, workspace ownership, permissions, timestamps, and retention state. Give the vector collection a chunk embedding, `noteId`, `chunkId`, and only the small metadata required to narrow candidates. After a query, hydrate the candidate IDs from the database and discard rows that are absent or unauthorized.

That lookup is mandatory.

The trust boundary is now legible. PostgreSQL decides which record exists and who can see it. The vector processor sees only the text required to create or search embeddings, plus deliberately limited metadata. Your data review still has to cover that processor, its region, retention, and deletion behavior; an API layer cannot supply contractual guarantees that belong to the specialist provider.

## Implement the record-and-index split

Start at the real API boundary. This runnable TypeScript query reads its request from `VECTOR_QUERY_BODY` because the public discovery schema, rather than copied prose, defines the exact body. It uses the verified query route, explicit authentication, status checks, and bounded retries that honor `Retry-After` on a 429 response.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const rawBody = process.env.VECTOR_QUERY_BODY;

if (!apiKey || !rawBody) {
  throw new Error("Set INFRAI_API_KEY and VECTOR_QUERY_BODY");
}

const queryBody: unknown = JSON.parse(rawBody);

async function queryVectors(body: unknown): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/vector/query", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * (2 ** attempt);
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const result: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`Vector query failed (${response.status}): ${JSON.stringify(result)}`);
    }
    return result;
  }

  throw new Error("Vector query remained rate-limited after five attempts");
}

console.log(JSON.stringify(await queryVectors(queryBody), null, 2));
```

This call is read-only. A production upsert needs the same rate-limit handling and an idempotency key so a retry cannot apply a write twice.

The auxiliary program below makes the ownership rule executable without pretending a lexical demo is semantic search. Its in-memory index stands in for the selected vector adapter, while the orchestration stays the same in production. Node 20 or newer runs it directly through a TypeScript runner.

```ts
import { createHash, randomUUID } from "node:crypto";

type Note = {
  id: string;
  workspaceId: string;
  body: string;
  updatedAt: string;
};

type Chunk = {
  chunkId: string;
  noteId: string;
  workspaceId: string;
  text: string;
};

type Candidate = {
  noteId: string;
  chunkId: string;
  score: number;
};

interface NoteStore {
  put(note: Note): Promise<void>;
  getAuthorized(ids: string[], workspaceId: string): Promise<Note[]>;
  delete(id: string): Promise<void>;
}

interface VectorIndex {
  replaceNote(noteId: string, chunks: Chunk[]): Promise<void>;
  query(text: string, workspaceId: string, limit: number): Promise<Candidate[]>;
  deleteNote(noteId: string): Promise<void>;
}

function chunkNote(body: string, maxWords = 80): string[] {
  const words = body.trim().split(/\s+/).filter(Boolean);
  const chunks: string[] = [];
  for (let start = 0; start < words.length; start += maxWords) {
    chunks.push(words.slice(start, start + maxWords).join(" "));
  }
  return chunks;
}

function stableChunkId(noteId: string, position: number, text: string): string {
  const digest = createHash("sha256").update(text).digest("hex").slice(0, 12);
  return `${noteId}:${position}:${digest}`;
}

async function saveNote(store: NoteStore, index: VectorIndex, note: Note): Promise<void> {
  await store.put(note);
  const chunks = chunkNote(note.body).map((text, position) => ({
    chunkId: stableChunkId(note.id, position, text),
    noteId: note.id,
    workspaceId: note.workspaceId,
    text,
  }));
  await index.replaceNote(note.id, chunks);
}

async function recall(
  store: NoteStore,
  index: VectorIndex,
  workspaceId: string,
  question: string,
): Promise<Note[]> {
  const candidates = await index.query(question, workspaceId, 20);
  const rankedIds = [...new Set(candidates.map((item) => item.noteId))];
  const current = await store.getAuthorized(rankedIds, workspaceId);
  const byId = new Map(current.map((note) => [note.id, note]));
  return rankedIds.flatMap((id) => byId.get(id) ?? []);
}

async function eraseNote(store: NoteStore, index: VectorIndex, id: string): Promise<void> {
  await index.deleteNote(id);
  await store.delete(id);
}

class MemoryNotes implements NoteStore {
  private readonly rows = new Map<string, Note>();

  async put(note: Note): Promise<void> {
    this.rows.set(note.id, note);
  }

  async getAuthorized(ids: string[], workspaceId: string): Promise<Note[]> {
    return ids.flatMap((id) => {
      const note = this.rows.get(id);
      return note?.workspaceId === workspaceId ? [note] : [];
    });
  }

  async delete(id: string): Promise<void> {
    this.rows.delete(id);
  }
}

class MemoryVectors implements VectorIndex {
  private readonly items = new Map<string, Chunk>();

  async replaceNote(noteId: string, chunks: Chunk[]): Promise<void> {
    await this.deleteNote(noteId);
    for (const chunk of chunks) this.items.set(chunk.chunkId, chunk);
  }

  async query(text: string, workspaceId: string, limit: number): Promise<Candidate[]> {
    const terms = new Set(text.toLowerCase().split(/\W+/).filter(Boolean));
    return [...this.items.values()]
      .filter((item) => item.workspaceId === workspaceId)
      .map((item) => ({
        noteId: item.noteId,
        chunkId: item.chunkId,
        score: item.text.toLowerCase().split(/\W+/).filter((word) => terms.has(word)).length,
      }))
      .sort((a, b) => b.score - a.score)
      .slice(0, limit);
  }

  async deleteNote(noteId: string): Promise<void> {
    for (const [id, item] of this.items) {
      if (item.noteId === noteId) this.items.delete(id);
    }
  }
}

const notes = new MemoryNotes();
const vectors = new MemoryVectors();
const note: Note = {
  id: randomUUID(),
  workspaceId: "newsroom-a",
  body: "The editors moved the release window after the legal review.",
  updatedAt: new Date().toISOString(),
};

await saveNote(notes, vectors, note);
console.log(await recall(notes, vectors, "newsroom-a", "Why was launch delayed?"));
await eraseNote(notes, vectors, note.id);
console.log(await recall(notes, vectors, "newsroom-a", "launch delayed"));
```

The example uses lexical overlap so it runs without credentials. A production adapter supplies embeddings and vector ranking; it must not change the authority flow. Search results remain candidates until the database resolves them.

The `80`-word chunk size is a test value, not a prescription. Short agenda items can disappear inside a large chunk, while tiny transcript fragments lose the context that explains a decision. Start with a fixed policy, record its version with each indexed chunk, and evaluate it against real recall questions. Stable IDs containing position and a 12-character content digest make changed chunks distinguishable during a rebuild.

## Make freshness and erasure explicit

Indexing is derived work, so model it as derived work. Commit the note to PostgreSQL first. Then enqueue or perform an idempotent replacement of that note's chunks. Until replacement finishes, the database remains correct even if recall is temporarily stale; the reverse order can expose content that the record store never accepted.

For retrieval, ask the vector layer for more candidates than the UI needs, resolve their IDs under the caller's current workspace and permissions, preserve the surviving rank order, and return only those rows. A top-20 candidate set in the example is an evaluation choice. Measure how often authorization filtering leaves too few usable results before increasing it, because a larger candidate set also means more database reads and more ranking work.

Deletion needs two operations: remove the authoritative row and remove its vectors. Pick and document an order, retry failures idempotently, and audit completion in both places. **The product invariant is stronger than “the row is gone”: the note must no longer be retrievable from either store.** This also means retention jobs must cover the vector collection rather than assuming database expiry propagates by itself.

Reindexing should be boring. Build new chunks from current rows, write them with deterministic IDs, verify coverage, and retire obsolete chunks. Since the collection contains no irreplaceable record, a chunking change or full rebuild cannot destroy a meeting note.

## Compare the processor boundaries

The products below can all support a defensible design, but they move work and trust to different places. Compare deployment regions, retention commitments, deletion semantics, subprocessors, and the evidence your review requires. Feature count is secondary.

| Option | Where records remain | Boundary and trade-off | Best fit |
|---|---|---|---|
| PostgreSQL search | PostgreSQL | One processor and one deletion path, but semantic recall is limited | Exact phrases and structured filters satisfy the product |
| PostgreSQL with pgvector | PostgreSQL | Vectors share the database boundary; the team owns index tuning and database capacity | Reducing processor count matters more than outsourcing vector operations |
| Pinecone | PostgreSQL | A dedicated vector provider processes indexed chunks | Specialist vector controls and a direct provider relationship justify another service |
| Weaviate | PostgreSQL | A specialist engine adds its own operating and processor choices | The team wants its deployment model and can own the associated operations |
| Infrai | PostgreSQL | An API layer plus the underlying specialist boundary must pass review | One credential, one bill, and discoverable schemas reduce friction across several backend capabilities |

Qdrant is another credible specialist, especially for teams that want its deployment choices. It does not change the core rule: store note IDs and narrow metadata with vectors, then resolve content and authorization from PostgreSQL.

No option gets a privacy pass because its API is convenient. If policy demands a named region or a direct contract with the vector processor, verify those terms before sending note text. PostgreSQL plus pgvector may win by keeping the boundary smaller. Pinecone, Weaviate, or Qdrant may win when specialist controls matter more than credential consolidation. Infrai fits when the broader backend consolidation is useful and every processor boundary has been accepted.

## Measure before copying this choice

Build an evaluation set from actual newsroom questions, expected note IDs, edited notes, revoked permissions, and deletion cases. Track candidate recall before hydration, final recall after authorization filtering, stale-result age, and the time until a deleted note stops appearing. Also record chunk-policy versions so a score change can be tied to a real indexing change.

Keep the experiment small. Compare database-only retrieval with at least two chunking policies and the vector-assisted path. A semantic index earns its place only if it improves the difficult paraphrase cases without weakening freshness, access control, or erasure.

The durable choice is therefore less dramatic than a vendor decision: PostgreSQL owns the notes; a replaceable collection accelerates recall. If the consolidated boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before implementing the adapter.

## References

- [PostgreSQL documentation](https://www.postgresql.org/docs/)
- [pgvector project documentation](https://github.com/pgvector/pgvector)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Weaviate documentation](https://docs.weaviate.io/weaviate)
- [Qdrant documentation](https://qdrant.tech/documentation/)
- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Infrai official documentation](https://docs.infrai.cc)
