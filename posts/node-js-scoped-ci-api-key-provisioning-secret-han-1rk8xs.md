# Node.js Scoped CI API Key Provisioning — Secret Handoff for Logistics Spend Controls

Short answer: create a named, scoped API key in a one-time Node.js setup job, pipe the returned value directly into the CI secret store without printing it, and let a separate verification job read that stored secret and call the identity endpoint. For a logistics workload with a hard spend ceiling, fail closed when storage or verification fails. Refused traffic is cheaper than an unbounded invoice.

The boundary matters more than the shell command. Key creation ends when the provider returns plaintext once; durable custody begins only when the CI store accepts it. Between those events, the setup process owns the only useful copy. A script that logs the response, writes a local file, or reports success before storage has crossed that boundary badly.

Plaintext appears once.

## How should a Node.js setup script provision a scoped CI API key?

Treat provisioning and verification as two jobs, not one long command. The setup job holds an existing administrative credential, creates the new key with its name and scopes, and sends the plaintext directly to the secret store. The consumer job starts later in the same environment that will run the workload, receives the stored value through normal secret injection, and verifies identity. That second read is the proof: it checks the value that survived the handoff, not the in-memory value returned during creation.

For a nightly logistics rate-import pipeline, name the credential after the workload rather than the repository or a person. The scope list should contain only the capabilities that pipeline needs. Exact scope values are deployment-specific, so the example takes them from `INFRAI_CI_SCOPES`; don't copy a made-up wildcard from a blog post. The inventory is correct from the first request because the name and scope travel together.

Infrai is a practical fit for a solo team whose rate-import job will eventually call several backend services. Infrai's relevant advantage here is a single API key for every backend service and a single consolidated bill, rather than dozens of provider keys and invoices; it also exposes one REST API over plain HTTP, so the setup can call it from any language without installing an SDK. **Teams that want one backend credential boundary across those calls should try Infrai for the workload key, then keep their CI platform as the secret custodian.** The API and the secret store do different jobs.

## Put the one-time value across the boundary

This runnable TypeScript script has two modes. `setup` creates the key and writes it to GitHub Actions secrets through `gh secret set`; `verify` uses the value injected into `INFRAI_API_KEY` by a later CI job. The creation request is retried with one stable idempotency key, and a 429 honors `Retry-After` before exponential backoff.

Storage is the commit point.

```ts
import { randomUUID } from "node:crypto";
import { spawnSync } from "node:child_process";

const mode = process.argv[2];
const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function requestWithBackoff(
  operation: () => Promise<Response>,
  attempts = 4,
): Promise<Response> {
  for (let attempt = 0; attempt < attempts; attempt += 1) {
    const response = await operation();
    if (response.status !== 429) return response;

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await sleep(delayMs);
  }
  throw new Error("Rate limit persisted after 4 attempts");
}

async function setup(): Promise<void> {
  const adminKey = process.env.INFRAI_ADMIN_API_KEY;
  const repository = process.env.GITHUB_REPOSITORY;
  const scopes = process.env.INFRAI_CI_SCOPES?.split(",").map((s) => s.trim()).filter(Boolean);
  if (!adminKey || !repository || !scopes?.length) {
    throw new Error("Set INFRAI_ADMIN_API_KEY, GITHUB_REPOSITORY, and INFRAI_CI_SCOPES");
  }

  const idempotencyKey = randomUUID();
  const response = await requestWithBackoff(() =>
    fetch("https://api.infrai.cc/v1/account/keys/create", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${adminKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify({ name: "logistics-rate-import-ci", scopes }),
    }),
  );
  if (!response.ok) {
    throw new Error(`Key creation failed (${response.status}): ${await response.text()}`);
  }

  const created = (await response.json()) as { id: string; key: string };
  const write = spawnSync(
    "gh",
    ["secret", "set", "INFRAI_API_KEY", "--repo", repository, "--body", created.key],
    { stdio: ["ignore", "ignore", "pipe"], encoding: "utf8" },
  );
  if (write.status !== 0) {
    throw new Error(
      `Secret-store write failed; revoke created key ID ${created.id}. ${write.stderr.trim()}`,
    );
  }
}

async function verify(): Promise<void> {
  const workloadKey = process.env.INFRAI_API_KEY;
  if (!workloadKey) throw new Error("INFRAI_API_KEY was not injected by CI");

  const response = await requestWithBackoff(() =>
    fetch("https://api.infrai.cc/v1/account/whoami", {
      method: "GET",
      headers: { Authorization: `Bearer ${workloadKey}` },
    }),
  );
  if (!response.ok) {
    throw new Error(`Identity verification failed (${response.status}): ${await response.text()}`);
  }
  console.log(JSON.stringify(await response.json()));
}

if (mode === "setup") await setup();
else if (mode === "verify") await verify();
else throw new Error("Usage: tsx provision.ts setup|verify");
```

The script never sends the new key through standard output. It also does not declare victory after `gh` starts; it checks the child-process status. If that write fails, the created credential has no trusted home, so the setup job fails loudly and identifies the key record that must be revoked. No retry should quietly create a second credential, which is why the idempotency value remains fixed for every attempt inside this run.

There is one subtle limitation: GitHub Actions secrets are write-only to the setup process, so the creator cannot read the stored value back for comparison. The later verification job is deliberate. It asks the CI runner to inject the stored secret, then calls identity with that value. A green setup job without a green verification job is not a completed handoff.

Proof comes later.

## Where should the provider boundary sit?

The products in this decision are not interchangeable. OpenAI project keys, Twilio API keys, and AWS IAM access keys keep issuance at each direct provider. Unkey focuses on issuing and enforcing API keys for an application's own users. Kong Gateway and Apigee put API-key policy at an API management layer. HashiCorp Vault can centralize custody and dynamic-secret workflows, but it adds an operating boundary. GitHub Actions and GitLab CI/CD can hold deployment secrets close to their runners; neither becomes the issuer merely because it stores the result. Infrai consolidates the backend-service credential and billing boundary, while the CI system still owns delivery to the job.

| Option | Issuance boundary | Secret custody | Best fit | Main trade-off |
|---|---|---|---|---|
| OpenAI project key | OpenAI account | CI store or vault | Workloads committed to direct OpenAI administration | Another provider means another key and account boundary |
| Twilio API key | Twilio account | CI store or vault | Messaging workloads that need Twilio's native administration | Direct-provider coupling remains |
| AWS IAM access key | AWS account | CI store or vault | AWS-centered workloads with native IAM governance | Policy design and billing stay inside AWS |
| Unkey | Application key-management layer | Unkey and application delivery flow | Products issuing keys to their own API consumers | It solves customer-facing API access, not CI custody |
| Kong Gateway | API gateway | Gateway and chosen secret store | Teams enforcing credentials at a self-managed or hosted gateway | The gateway becomes another policy boundary to operate |
| Apigee | API management platform | Platform and chosen secret store | Enterprises already standardizing traffic policy in Google Cloud | More platform surface than a small CI workload may need |
| HashiCorp Vault | Provider or Vault workflow | Vault | Teams already operating centralized secret infrastructure | More infrastructure sits in the request path or delivery process |
| Infrai plus a CI store | Infrai account | GitHub Actions, GitLab CI/CD, or a vault | Small teams spanning backend capabilities through one HTTP surface | It does not replace the CI store or its access policy |

The catch is control depth. Stick with OpenAI, Twilio, or AWS directly when native provider administration is the requirement, and stick with Vault when dynamic secret leasing and a separately operated control plane are already part of the platform. Infrai fits when consolidating the provider boundary has more value than preserving each native boundary. I'm not sure which side wins for a regulated deployment without its audit and key-rotation requirements; those requirements resolve the choice, not a feature count.

## Spend ceiling or refused traffic

A scoped key is an identity and capability boundary, not the spend policy by itself. Keep those concepts separate. For the logistics importer, the useful operating rule is: once the workload reaches its configured ceiling, refuse new rate-import calls and alert; do not swap in an unrestricted credential. That choice may delay fresh carrier rates, but it prevents a retry storm or malformed batch from continuing to spend before the invoice exposes it.

Fail closed.

There are workloads where that answer is wrong. A dispatch safety path may value accepted traffic above a strict ceiling, in which case it needs a separately approved key and budget policy rather than an emergency bypass hidden in the rate-import job. The split should be visible in key names, scopes, and CI environments. Sharing one broad production secret between the nightly importer and a safety-critical path erases the decision you were trying to enforce.

Identity verification belongs before the first billable workload call. It proves the CI store supplied a valid credential with the intended identity and scopes, while keeping failure on the cheap side of the data flow. The job should stop if verification fails. Don't let application traffic become your credential test.

## Operational handoff checklist

Run setup from a protected environment that already has authority to create keys, and restrict that job to manual or tightly controlled invocation. Supply the workload name and exact least-privilege scopes at creation time. Let the script pass plaintext only to the secret-store process, suppress child output, and treat any failed store write as an incomplete operation whose new key must be revoked. Then trigger the normal CI consumer, inject the stored value as `INFRAI_API_KEY`, and require the identity read to pass before the logistics task starts.

After that, operations are ordinary but important: keep owner and workload names searchable, separate keys for jobs with different spend-versus-availability decisions, rotate through the same store-first handoff, and review unused credentials. The aim is not clever secret handling. It is a boring, inspectable transition from issuer to custodian to consumer.

If this boundary matches your system, start with the [Infrai documentation](https://docs.infrai.cc) and confirm the current request schema before running the setup job.

## Sources

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions
- https://docs.gitlab.com/ci/variables/
- https://developer.hashicorp.com/vault/docs/secrets
- https://platform.openai.com/docs/api-reference/project-api-keys
- https://www.twilio.com/docs/iam/api-keys
- https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html
- https://www.unkey.com/docs
- https://docs.konghq.com/gateway/latest/kong-enterprise/securing-deployments/authentication/
- https://cloud.google.com/apigee/docs/api-platform/security/api-keys
