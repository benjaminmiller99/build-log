# Email Signatures Failing After DKIM Rotation: 3 States (Before Key Retirement)

TL;DR: Treat a half-finished DKIM rotation as a three-state investigation: what the sender signs with, what DNS publishes for that selector, and what the receiving side can authenticate. For an edtech onboarding flow, domain ownership is not proven merely because a DNS lookup returns something. Completion should wait for deliverability evidence tied to the exact tenant, selector, and message under test. Keep the old key available until traffic signed with it has drained; publish and observe the new path before making it authoritative.

This framing resolves the main trade-off. A fast cutover shortens the period with two keys, but it also removes the evidence needed to distinguish stale DNS from a sender that never switched. A deliberate overlap costs a little operational complexity and buys a reversible transition. I would take that trade for onboarding mail, where a lost verification message can strand a school administrator before the account is usable.

## Why are email signatures failing after a half-completed DKIM rotation?

A DKIM result depends on a chain, not a single dashboard state. The outbound message carries a signature that identifies a signing domain and selector. The verifier uses that information to find public key material in DNS, then evaluates the signature. DMARC can consume the resulting DKIM authentication result, but it additionally cares about identifier alignment with the domain visible in the message's From field. RFC 7489 distinguishes authentication from alignment, which is why a message can have a valid DKIM result yet still fail the DKIM side of DMARC.

The useful unit of evidence is therefore one received message. Capture its From domain, DKIM signing domain, selector, authentication result, and observation time. Do not collapse those fields into a green or red tenant flag. If failures affect only messages carrying the retired selector, the signing fleet and DNS publication are in different states. If the new selector authenticates but is not aligned, rotation exposed a domain-choice problem rather than a key problem. If observations differ by receiving path, resolver visibility is still part of the investigation.

3 states are enough to organize the incident: old selector still signing and still published; new selector signing and published; or one side changed without the other. The dangerous state is the last one.

That is the trap.

## Trace one message before changing anything

The following TypeScript keeps the check intentionally small. It does not claim that a TXT record proves delivery or verifies a cryptographic signature. It turns a message observation into a stable DNS question and records all returned strings so the result can be attached to an onboarding attempt. The injected resolver also makes the decision code testable without binding it to a commercial mail service.

```ts
import { promises as dns } from "node:dns";

type RotationProbe = {
  tenantId: string;
  fromDomain: string;
  signingDomain: string;
  selector: string;
  observedAt: string;
};

type TxtResolver = (name: string) => Promise<string[][]>;

async function inspectSelector(
  probe: RotationProbe,
  resolveTxt: TxtResolver = dns.resolveTxt,
) {
  const dnsName = `${probe.selector}._domainkey.${probe.signingDomain}`;
  const records = (await resolveTxt(dnsName)).map((chunks) => chunks.join(""));
  const aligned =
    probe.fromDomain === probe.signingDomain ||
    probe.signingDomain.endsWith(`.${probe.fromDomain}`);

  return { ...probe, dnsName, records, aligned };
}

const sample: RotationProbe = {
  tenantId: "district-042",
  fromDomain: "notices.school.example",
  signingDomain: "notices.school.example",
  selector: "onboarding-b",
  observedAt: new Date().toISOString(),
};

inspectSelector(sample).then((result) => console.log(JSON.stringify(result, null, 2)));
```

Run the probe from the same operational environment used for the onboarding check, but preserve evidence from the received message as the authority for what was actually signed. A current DNS answer cannot rewrite a message already sent. That timing distinction matters.

One message is enough to begin.

For each test message, store the probe beside the receiver's authentication result. Avoid storing message content when the selector, domains, timestamp, tenant identifier, and result are sufficient; onboarding mail may contain student or administrator data that has no place in a rotation log.

## Use deliverability evidence as the gate

A DNS publication check answers one narrow question: can this resolver retrieve material at the selector name now? It does not establish that every sender uses the selector, that a receiver validated the signature, or that the authenticated identifier aligns for DMARC. The onboarding gate should reflect those boundaries.

DNS is not delivery.

| Evidence | What it supports | What it cannot prove |
|---|---|---|
| Selector lookup | Key material is visible to the queried resolver | The message was signed with that selector |
| Received-message authentication result | A receiver evaluated this particular message | Every sending path has switched |
| Aligned DKIM result | The DKIM result can satisfy the DKIM side of DMARC alignment | Future delivery to every receiver |
| Repeated tests across sending paths | The observed fleet is converging | Unobserved traffic is already drained |

**The acceptance rule should require message-level evidence, not DNS presence alone.** For example, keep ownership in a pending state until a fresh onboarding message carries the intended selector and the receiver reports an aligned DKIM pass. If the application has more than one outbound path, sample each path. This rule is stricter than checking a record, but it measures the outcome the administrator needs: mail that can be authenticated under the tenant's domain.

There is no honest universal wait time in the available evidence. Record timestamps and continue testing until the old-signing observations disappear under the traffic coverage your system actually has. Inventing a fixed delay would hide uncertainty rather than manage it.

## Separate the failure before retrying

Start with the failed message. Compare its selector with the intended rotation state, query that exact selector name, and inspect the receiver's DKIM and DMARC results. This produces 4 practical branches. A missing old selector with an old signature points toward retirement before signing traffic drained. A missing new selector with a new signature points toward signing before publication was observable. A DKIM pass paired with failed alignment points toward the relationship between signing and From domains. Mixed selectors across fresh messages point toward incomplete sender rollout. For a district with separate transactional and bulk sending paths, do not let one successful onboarding message stand in for both paths: label the source of each test, compare selectors, and hold completion if either path still emits the retired identity. The point is coverage, not message count.

Do not respond to all four by republishing records and retrying blindly. That erases the transition history while leaving the signing path untouched. It can also generate repeated onboarding messages, adding noise to the deliverability evidence and unnecessary mail volume. First classify, then change one boundary.

For retry policy, distinguish transient lookup failures from a stable negative result and from an authentication failure on a received message. Backoff is reasonable for a transient observation; it is not a repair for a sender that continues to select the old key. Cap automated retries and surface the recorded selector, signing domain, and observation time to the operator. The useful alert says which state disagrees, not merely that DKIM failed.

This is also where cost discipline helps. Retain compact structured evidence, not full raw messages by default. Sample normal successes, preserve failures through the rotation window, and aggregate by tenant plus selector. Those dimensions show whether one tenant is misconfigured or a shared sending path has not converged, without turning routine onboarding into an unbounded logging bill.

## Close the cutover without losing the trail

The operational checklist is short enough to keep in prose. Before switching signers, publish the new selector and confirm that the intended resolver can retrieve it. Send a controlled onboarding message, record the selector it actually used, and retain the receiver's authentication result. Move each outbound path to the new selector, then watch the selector distribution rather than trusting deployment completion. Keep the old public key available while old-signed traffic is still observed. Remove it only after the defined coverage shows no remaining use, and retain the compact audit record that explains why the rotation was closed.

**A completed rotation is an evidence state, not a deployment event.** For edtech domain onboarding, that means a fresh message authenticated with the intended, aligned identity across every known sending path. DNS visibility is necessary, but the received message closes the loop.

## References

- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
