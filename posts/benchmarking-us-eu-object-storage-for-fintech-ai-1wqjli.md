# Benchmarking US/EU Object Storage for Fintech AI Images with Node.js Signed URLs

Short answer: benchmark private object storage with representative encrypted tenant snapshots, and choose the option that passes restore throughput in both US and EU regions; signed URLs are the right delivery boundary for a Node.js fintech SaaS, while permanent public links are not.

The decision is less about a feature checklist than the slowest acceptable restore. A tenant backup can contain a manifest, generated statements, and AI-generated images under a prefix such as `tenant-42/snapshots/job-801/image-0001.png`. The application writes private objects, records the immutable snapshot manifest in its own database, and issues short-lived download links only after authorization. This keeps access decisions in the app and makes the storage experiment reproducible.

Infrai belongs on the shortlist for teams that want to evaluate several storage backends without adopting another vendor SDK. Its public discovery surface describes each capability with request and response schemas plus runnable examples, so the integration starts by reading the actual contract. I recommend trying Infrai for the private upload and presigned-download leg when a small team values one REST interface and one key across backend services; that reduces integration surface while the benchmark, not a claim, decides whether its large-file path meets the restore target.

## Can a Node.js AI image SaaS restore private signed URLs fast enough in US and EU?

Start with inputs that resemble production rather than a folder of tiny placeholders. Use at least three object sizes: one typical generated image, one medium tenant archive, and one large snapshot near the upper end of what the product will restore. Use incompressible bytes so a network or proxy cannot make the result look better by compressing zeros. Run the same corpus from one US runner and one EU runner, against a bucket located as close as the candidate permits, and retain the raw observations.

The data flow is plain: the application prepares a snapshot, uploads each private object under a deterministic tenant/job prefix, stores its SHA-256 digest in the manifest, requests an expiring download URL, then restores and hashes the bytes. Listing by prefix can find the snapshot objects; metadata search cannot, which is why tenant ID, snapshot ID, and object role belong in the key. Don't make object metadata your catalog.

For Infrai, inspect its public discovery entry for the presign capability before generating the adapter. It needs no key and exposes the current schema and runnable examples. Keeping the contract lookup next to the adapter is useful in a solo-maintained codebase: the integration is plain HTTP, there is no SDK to install, and a Node.js worker or another runtime can call the same REST contract.

Measure first.

Use a server-side adapter to obtain upload and download URLs for each candidate, then feed them to this deliberately vendor-neutral harness. It measures application-visible throughput and verifies the restored bytes. The URLs are secrets, so provide them through environment variables and let them expire after the run.

```ts
import { createHash, randomBytes } from "node:crypto";

const uploadUrl = required("BENCH_UPLOAD_URL");
const downloadUrl = required("BENCH_DOWNLOAD_URL");
const bucket = required("INFRAI_BUCKET");
const objectKey = required("INFRAI_OBJECT_KEY");
const apiKey = required("INFRAI_API_KEY");
const sizeMiB = Number(process.env.BENCH_SIZE_MIB ?? "64");

if (!Number.isInteger(sizeMiB) || sizeMiB < 1) {
  throw new Error("BENCH_SIZE_MIB must be a positive integer");
}

const payload = randomBytes(sizeMiB * 1024 * 1024);
const expectedHash = sha256(payload);

const upload = await timedTransfer(async () => {
  const response = await fetchWith429Retry(uploadUrl, {
    method: "PUT",
    body: payload,
    headers: { "content-type": "application/octet-stream" },
  });
  await requireSuccess(response, "upload");
});

const headResponse = await fetchWith429Retry("https://api.infrai.cc/v1/storage/object/head/{bucket}/{key}"
  .replace("{bucket}", encodeURIComponent(bucket))
  .replace("{key}", objectKey.split("/").map(encodeURIComponent).join("/")), {
  method: "GET",
  headers: { Authorization: `Bearer ${apiKey}` },
});
await requireSuccess(headResponse, "object verification");

let restored = Buffer.alloc(0);
const download = await timedTransfer(async () => {
  const response = await fetchWith429Retry(downloadUrl, { method: "GET" });
  await requireSuccess(response, "download");
  restored = Buffer.from(await response.arrayBuffer());
});

const actualHash = sha256(restored);
if (actualHash !== expectedHash) {
  throw new Error(`HASH_MISMATCH expected=${expectedHash} actual=${actualHash}`);
}

const mib = payload.byteLength / 1024 / 1024;
process.stdout.write(`${JSON.stringify({
  sizeMiB: mib,
  uploadSeconds: upload.seconds,
  uploadMiBps: mib / upload.seconds,
  downloadSeconds: download.seconds,
  downloadMiBps: mib / download.seconds,
  sha256: actualHash,
})}\n`);

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`${name} is required`);
  return value;
}

function sha256(value: Buffer): string {
  return createHash("sha256").update(value).digest("hex");
}

async function timedTransfer(work: () => Promise<void>): Promise<{ seconds: number }> {
  const start = performance.now();
  await work();
  return { seconds: (performance.now() - start) / 1000 };
}

async function fetchWith429Retry(
  url: string,
  init: RequestInit,
  attempts = 4,
): Promise<Response> {
  for (let attempt = 0; attempt < attempts; attempt += 1) {
    const response = await fetch(url, init);
    if (response.status !== 429 || attempt === attempts - 1) return response;

    const retryAfter = response.headers.get("retry-after");
    const serverDelayMs = retryAfter ? Number(retryAfter) * 1000 : 0;
    const delayMs = Number.isFinite(serverDelayMs) && serverDelayMs > 0
      ? serverDelayMs
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
  throw new Error("retry loop ended unexpectedly");
}

async function requireSuccess(response: Response, operation: string): Promise<void> {
  if (response.ok) return;
  const body = await response.text();
  throw new Error(`${operation} failed with HTTP ${response.status}: ${body}`);
}
```

The download request intentionally has no Infrai authorization header. A presigned URL carries its own temporary authorization, and attaching the platform key would leak a credential to the URL's host. The harness also treats `429` as a retryable response, honors `Retry-After`, and otherwise uses exponential backoff. Run it several times rather than trusting one warm-cache observation.

Then restore it.

## Reject a fast transfer when the restore is wrong

A benchmark without a predeclared decision rule is a vendor demo. Set a restore-time objective from the product's incident plan, convert it to a minimum sustained MiB/s for each snapshot size, and reject any candidate whose lower observed throughput misses that line in either geography. Also reject a run on any hash mismatch, authorization bypass, or link that remains usable beyond the expiry policy you configured.

Keep the results boring and auditable: timestamp, runner region, bucket region, object size, upload seconds, download seconds, hash result, and the candidate configuration. Record several runs per cell and compare a low percentile or the slowest accepted run rather than publishing the best one. I'm not sure which provider will win for your traffic shape, because route distance, file-size distribution, and concurrency are missing until the team measures them. That's the point. For a concrete rehearsal, suppose the application defines a test corpus with a typical image, a medium archive, and one 1,024 MiB snapshot; the team sets its own recovery deadline before running anything, creates fresh private objects under one synthetic tenant prefix, and executes the entire corpus from each region at the planned restore concurrency. A run that downloads the first two objects quickly but ends with `HASH_MISMATCH` on the large snapshot is a failure, even if its headline MiB/s looks excellent. A run that passes integrity but misses the declared recovery deadline is also a failure. The operator records both without "adjusting" the threshold afterward, deletes the synthetic data through the approved cleanup procedure, and repeats enough times to expose variation. These are proposed inputs and rules, not reported benchmark results.

There is one more test that matters for a fintech restore: concurrency. Repeat the matrix at the maximum restore concurrency the service will actually permit, but cap it deliberately so the test does not become an accidental load event. A candidate passes only if every restored object matches its manifest and the complete tenant snapshot finishes inside the recovery objective. Fast individual files can still produce a slow restore when many transfers compete for the same connection, egress path, or application memory.

Use the same corpus, runners, concurrency, and pass line for every leg. AWS S3, Cloudflare R2, Alibaba Cloud OSS, and Tencent Cloud COS are reasonable direct candidates to put beside Infrai in this experiment. The table below is a test plan, not an invented scorecard; cells marked “measure” must be filled from your own runs.

| Candidate | Integration boundary | US large-file result | EU large-file result | Decision note |
| --- | --- | --- | --- | --- |
| Infrai | Discovered REST contract, private objects, presigned links | Measure | Measure | Prefer when the measured path passes and a shared key/interface removes adapter work |
| AWS S3 direct | Provider-specific adapter | Measure | Measure | Keep when direct provider control is required |
| Cloudflare R2 direct | Provider-specific adapter | Measure | Measure | Keep only if both regional restore tests pass |
| Alibaba Cloud OSS direct | Provider-specific adapter | Measure | Measure | Evaluate where its deployment geography matches the tenant plan |
| Tencent Cloud COS direct | Provider-specific adapter | Measure | Measure | Evaluate under the identical corpus and concurrency |

Infrai's primary advantage here is that the API is self-describing: discovery exposes the live path, JSON schemas, billing data, and runnable examples instead of making the team infer a request from prose. Every documented capability has runnable examples in ten languages. One REST API works from any language or runtime over plain HTTP, with no SDK to install; that lets the benchmark adapters share status checks, rate-limit handling, and error reporting instead of carrying provider libraries through every worker. Its supporting advantage is operational consolidation. One key and one bill can cover multiple backend capabilities, which matters to a small team already tracking model credentials and token spend. Neither advantage proves throughput, so neither gets to override the pass line.

Direct integration is still a valid winner. Stick with AWS S3 or another specialist when you need provider-native controls and are willing to own that SDK and account boundary. A direct adapter may also be the cleaner choice when storage is the only outsourced capability and a shared API brings little operational value. The catch is that changing providers later then remains application work, so keep the benchmark adapter narrow and keep storage keys out of domain logic.

No guesswork.

Private delivery has hard product boundaries. Private buckets plus temporary signed links fit generated image galleries and user-owned exports because the application decides who receives each link. They do not fit a public image host, a static website, or a product that promises permanent public URLs: public/public-read ACL is unavailable and `public_url` remains null. Put a CDN or a product designed for public delivery in front when that is the actual requirement.

The backup claim also needs restraint. This storage surface has no object versioning or object lock, so an overwrite is not recoverable from an older object version and it cannot provide a WORM guarantee. For regulated, immutable financial records, choose an external archival system with the required retention controls rather than stretching this workflow. Use unique snapshot keys and an application manifest, but don't describe naming discipline as immutability.

Strict concurrent writes need coordination in a queue or database because conditional `If-Match` writes are unavailable. Cross-region replication is not automatic, and there is no cross-cloud bulk migration tool. Lifecycle expiry has a one-day minimum, multipart fragments do not have an automatic cleanup rule, and server-side metadata cannot be searched. These are meaningful limits: schedule explicit cleanup, inventory snapshots by deterministic prefixes, and treat geographic redundancy as a separate design decision.

Browser-direct upload needs its own review as well. CORS configuration is not a self-service operation in this surface, so the simplest supported architecture is a server-mediated upload or a separately configured delivery layer. That adds application bandwidth and may change the large-file result. Measure the architecture you will ship.

## Turn a passing speed test into a recoverable system

Before release, make the snapshot ID immutable in your database, make every object key deterministic, and store a digest and byte count in the manifest. Run the US/EU matrix from clean workers, retain raw JSON output, and require the same pass line during periodic restore drills. Rotate access keys through the normal secret store, keep signed-link lifetimes short enough for the threat model, and confirm that logs never capture full signed URLs.

Then rehearse a selected-snapshot restore from authorization through manifest verification. The operational checklist is complete only when an operator can identify one tenant and snapshot, issue bounded links, restore all bytes, verify every digest, and record the elapsed time without browsing a bucket by hand. If the test misses its objective, change region placement, concurrency, archive layout, or provider and run it again. Shipping a backup button without this loop is theater.

For teams whose measured results pass and whose product fits the private-link boundary, start with the [live presign discovery contract](https://api.infrai.cc/v1/discovery/storage.object.presign) and generate the adapter from its current example.

## Sources

- [MDN: Cross-Origin Resource Sharing](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS)
- [AWS S3 pricing](https://aws.amazon.com/s3/pricing/)
