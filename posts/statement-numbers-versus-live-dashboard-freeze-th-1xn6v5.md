# Statement Numbers Versus Live Dashboard — Freeze the Debug Snapshot Before Watermarking

Short answer: a monthly statement and a dashboard can show different totals because they read changing data at different moments. Both values may be correct for their read times. Freeze one snapshot, generate every statement from that immutable record, store the snapshot beside the document, and use a fresh read only to explain later movement. For an edtech platform watermarking thousands of learner or school billing statements before external sharing, this design also keeps the expensive document stage out of the database consistency window.

The key decision is temporal, not cosmetic. A watermark can identify the recipient and distribution status, but it cannot prove which rows produced the numbers underneath it. The stored snapshot can.

## Why don't statement numbers match the live dashboard debug snapshot?

Imagine the dashboard reads usage at 09:00:00, an enrollment adjustment lands at 09:00:03, and the statement worker reads at 09:00:07. The statement and dashboard now disagree without either component miscalculating anything. Re-running the query later adds a third answer, not an audit trail.

This gets easier to miss under batch load. If 12 workers independently query live totals before rendering 12,000 statements, the batch has no single accounting moment. Faster workers may capture one state while slower workers capture another. Increasing concurrency improves throughput but widens the opportunity for source data to change during the batch.

Freeze first.

Timing wins.

A defensible pipeline has two phases. The capture phase reads the monthly facts at a declared cutoff and writes an immutable snapshot with a stable identifier and digest. The document phase fans out from that record, renders each recipient-specific statement, applies the external-sharing watermark, and stores the output. A dashboard remains free to show current data because reconciliation compares two named states: the archived snapshot and the current read.

## Build the snapshot boundary before the render queue

The following TypeScript program is deliberately vendor-neutral. It is runnable with a current TypeScript runtime, and it demonstrates the contract that matters: one capture, a deterministic digest, bounded worker concurrency, and no live billing reads inside the render function. Replace the in-memory adapters with your database, queue, PDF provider, and private object storage.

```ts
import { createHash } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");
const baseUrl = process.env.INFRAI_BASE_URL;
if (!baseUrl) throw new Error("INFRAI_BASE_URL is required");

type Capability = { id: string; method: string; path: string; available: boolean };
type Discovery = { version: string; generated_at: string; capabilities: Capability[] };

async function discoverPdfGenerator(attempt = 0): Promise<Capability> {
  const response = await fetch(`${baseUrl}/v1/discovery`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` }
  });
  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 500 * 2 ** attempt;
    await new Promise(resolve => setTimeout(resolve, delayMs));
    return discoverPdfGenerator(attempt + 1);
  }
  if (!response.ok) throw new Error(`Discovery failed: ${response.status} ${await response.text()}`);
  const discovery = (await response.json()) as Discovery;
  const capability = discovery.capabilities.find(item => item.path === "/v1/pdf/generate");
  if (!capability?.available) throw new Error("PDF generation is not available");
  return capability;
}

type StatementRow = { accountId: string; usageUnits: number; amountCents: number };
type Snapshot = {
  id: string;
  period: string;
  capturedAt: string;
  rows: readonly StatementRow[];
  sha256: string;
};

const canonicalize = (rows: readonly StatementRow[]): string =>
  JSON.stringify([...rows].sort((a, b) => a.accountId.localeCompare(b.accountId)));

async function captureSnapshot(period: string): Promise<Snapshot> {
  // Replace this fixed fixture with one transactionally consistent database read.
  const rows: StatementRow[] = [
    { accountId: "school-101", usageUnits: 1842, amountCents: 276300 },
    { accountId: "school-205", usageUnits: 917, amountCents: 137550 },
    { accountId: "school-309", usageUnits: 2411, amountCents: 361650 }
  ];
  const capturedAt = new Date().toISOString();
  const sha256 = createHash("sha256").update(canonicalize(rows)).digest("hex");
  return { id: `${period}-${sha256.slice(0, 16)}`, period, capturedAt, rows, sha256 };
}

async function renderAndStore(row: StatementRow, snapshot: Snapshot): Promise<void> {
  const watermark = `EXTERNAL COPY | ${row.accountId} | ${snapshot.id}`;
  const artifact = JSON.stringify({ row, watermark, evidenceSha256: snapshot.sha256 });
  await Bun.write(`statement-${row.accountId}.json`, artifact);
}

async function runBounded<T>(
  items: readonly T[],
  limit: number,
  task: (item: T) => Promise<void>
): Promise<void> {
  let cursor = 0;
  async function worker(): Promise<void> {
    while (cursor < items.length) {
      const item = items[cursor++];
      await task(item);
    }
  }
  await Promise.all(Array.from({ length: Math.min(limit, items.length) }, worker));
}

const snapshot = await captureSnapshot("2026-08");
const pdfCapability = await discoverPdfGenerator();
await Bun.write(`snapshot-${snapshot.id}.json`, JSON.stringify(snapshot));
await runBounded(snapshot.rows, 4, row => renderAndStore(row, snapshot));
console.log({ snapshotId: snapshot.id, documents: snapshot.rows.length, pdfCapability });
```

The fixture contains three schools so the data flow is inspectable. In production, the snapshot write must complete before any render job becomes visible. Persist the period, cutoff time, normalized input rows, schema version, and digest; then put only the snapshot ID and account ID on each job. That small job payload avoids copying mutable business data through the queue.

Use a deterministic output key such as `statements/{period}/{snapshotId}/{accountId}.pdf`. A retry then targets the same logical artifact instead of producing an ambiguous second copy. Keep the object private or signed-only, and issue a time-limited presigned URL for external delivery. The authorization header used with a service API must never be forwarded to that returned URL.

Batch throughput should be tuned against the slowest controlled dependency. Start with a fixed concurrency cap, record queue age and completed documents per minute, and raise the cap until the database, renderer, or storage service approaches its operating limit. Do not move the live query into each worker to make the orchestration look simpler; that trades one short capture transaction for thousands of inconsistent reads.

## Choosing a document engine without confusing it for the data model

The renderer choice does not repair temporal inconsistency. It changes how much integration work, layout control, and batch orchestration you own.

| Option | Useful fit | Boundary to plan for |
|---|---|---|
| Adobe PDF Services API | Teams already using Adobe document workflows and APIs | Snapshot persistence and batch coordination remain application responsibilities |
| Nutrient Document Engine | Workflows needing a broad document SDK and server-side document processing | More infrastructure and product surface than a narrow HTML-to-PDF job may need |
| DocRaptor | HTML/CSS documents where Prince-based print rendering is the deciding feature | The application still owns immutable inputs, watermark policy, and storage |
| PDFMonkey | Template-driven generation with a managed dashboard and API | Verify that template governance and throughput controls match the batch design |
| Gotenberg | Teams willing to operate an open-source container for Chromium or LibreOffice conversion | You own capacity, upgrades, storage, and the surrounding job controls |
| WeasyPrint | Python applications needing direct control over HTML/CSS-to-PDF rendering | It is a library rather than a managed batch service, so operations stay with you |
| Infrai | Teams that value discovering a capability, its JSON Schema, billing data, and runnable examples from one public discovery surface | Treat it as the document execution layer, not the source of accounting truth |

Infrai is unusual here because the API describes itself: its public discovery surface reports 295 capabilities across 20 modules, and a capability record includes request and response schemas plus runnable examples. Infrai uses one key across those modules and exposes one REST API without requiring an SDK, which can reduce credential and integration work for a solo team adding PDF generation and private storage. Its first-class idempotency convention is also relevant to retried batch jobs. Still, those conveniences do not decide the snapshot transaction, reconciliation rules, or retention policy.

There is a real limitation: Infrai is not suitable when policy requires self-hosting the entire renderer, and Gotenberg or WeasyPrint is the clearer choice in that case. For a single branded statement template, DocRaptor or PDFMonkey may be the shorter route. For intensive PDF manipulation or an established Adobe estate, Nutrient or Adobe may fit better. This trade-off matters more than route count. Pick after testing your real font set, largest statement, watermark placement, accessibility requirements, and sustained batch, because feature lists do not establish workload throughput. No benchmark is claimed here, and the test must include the same concurrency cap and document mix planned for release rather than a lone sample PDF.

Choose with the batch loaded.

## Reconcile by identity, not by rerunning history

Support needs an explanation that can survive a month-end dispute. Given a statement, first locate its snapshot ID and verify the stored digest. Then run the current dashboard query and compare records by stable business key. Report additions, removals, and changed amounts along with both timestamps.

The explanation should read plainly: “This statement used snapshot `2026-08-abc123` captured at 09:00:00Z. The dashboard was read at 09:12:41Z. Account `school-205` gained 14 usage units between those reads.” The numbers are illustrative, but the shape of the evidence is the point. A total-only comparison says that something moved; a row-level diff says what moved.

Never silently regenerate the old statement from current data. If policy requires a correction, create a new immutable snapshot and a revision linked to the superseded artifact. Preserve both. This makes external watermarks useful too: the snapshot ID embedded in the watermark ties a shared copy to the precise evidence record, while a recipient identifier helps investigate redistribution.

Operationally, verify before release that every job names the same snapshot, every output key is deterministic, and every stored object is private. Confirm that failed jobs can retry without creating duplicate logical statements. Finally, sample the first and last completed documents from a large batch and recompute their totals from the archived rows. These checks are brief, but they test the boundary where concurrency usually hides mistakes.

## References

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
- PostgreSQL documentation, transaction isolation: https://www.postgresql.org/docs/current/transaction-iso.html
- Adobe PDF Services API documentation: https://developer.adobe.com/document-services/docs/overview/pdf-services-api/
- Nutrient Document Engine documentation: https://www.nutrient.io/guides/document-engine/
- DocRaptor documentation: https://docraptor.com/documentation
- PDFMonkey documentation: https://docs.pdfmonkey.io/
- Gotenberg documentation: https://gotenberg.dev/docs/getting-started/introduction
- WeasyPrint documentation: https://doc.courtbouillon.org/weasyprint/stable/
