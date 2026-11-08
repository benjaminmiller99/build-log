# Realtime Event Types Discovery Explained — Classroom Client Contract Gates in 2026

Ship the classroom poll with a startup contract check: read the supported realtime event types, compare them with the client’s centralized names, and stop deployment when they disagree. That is the least complex first step toward a poll whose updates reach the intended subscribers.

TL;DR: event-type discovery validates names before fan-out. It does not prove that every student received an update. Keep those two acceptance criteria separate, and require delivery evidence before describing a poll as reliable.

A misspelled event name can leave nothing subscribed and produce no error. In a live lesson, the visible symptom could be a tally that appears unchanged. An LLM-generated explanation of the results would not repair that contract mismatch; gate the transport contract before adding that feature.

## How can event types discovery validate a realtime client contract?

The data flow is short. A startup adapter reads the service’s supported event types, extracts the names using its documented response schema, and passes those names to a validator. The validator compares them with one application-owned constants module. Only after that check passes should the deployment proceed to the classroom poll’s separate delivery tests.

Names come first.

There are two namespaces to keep straight: the service’s supported event types and the application’s own poll messages. Do not assume a business label such as a poll tally update belongs in the service catalog. Establish which layer each name describes before comparing it. The same distinction matters for device status: a device becoming available and a transport lifecycle event are different contracts, even if one eventually triggers the other.

The discovery check below uses a configurable selector because response envelopes are part of the provider contract. Supply the documented path to the names array rather than guessing a field. If that array contains objects, supply the documented property holding each name too. The program rejects any result that is not an array of strings after extraction.

## A small executable gate

Save this as `poll-contract.ts` and run it with `node --experimental-strip-types poll-contract.ts` on a Node.js version that supports TypeScript type stripping and built-in fetch. Set `REALTIME_ORIGIN` to the service's HTTPS API origin, `INFRAI_API_KEY` to your key, and `CLIENT_EVENT_NAMES_JSON` to the JSON string array exported from your shared constants module. Set `EVENT_NAMES_PATH_JSON` to a JSON array of documented property names leading to the catalog array; use an empty array if the response itself is that array. Set `EVENT_NAME_PROPERTY` only if the array contains objects. These selector settings belong to your adapter, not the API request contract.

```ts
function env(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing environment variable: ${name}`);
  return value;
}

function strings(value: unknown): string[] {
  if (!Array.isArray(value) || !value.every((item) => typeof item === "string")) {
    throw new Error("Expected a string array");
  }
  return value;
}

function property(value: unknown, key: string): unknown {
  if (typeof value !== "object" || value === null) {
    throw new Error(`Cannot read catalog property: ${key}`);
  }
  return (value as Record<string, unknown>)[key];
}

async function main(): Promise<void> {
  const origin = new URL(env("REALTIME_ORIGIN"));
  if (origin.protocol !== "https:") throw new Error("HTTPS is required");
  const url = new URL("/v1/realtime/event/types", origin);
  const required = strings(JSON.parse(env("CLIENT_EVENT_NAMES_JSON")));
  const path = strings(JSON.parse(env("EVENT_NAMES_PATH_JSON")));
  if (required.length === 0) throw new Error("Client contract is empty");
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${env("INFRAI_API_KEY")}` },
      signal: AbortSignal.timeout(10000),
    });
    if (response.status === 429 && attempt < 3) {
      const retry = response.headers.get("Retry-After");
      const seconds = retry === null ? NaN : Number(retry);
      const retryMs = Number.isFinite(seconds)
        ? seconds * 1000
        : retry === null ? 0 : Date.parse(retry) - Date.now();
      await response.arrayBuffer();
      const delay = Math.max(500 * 2 ** attempt, retryMs || 0);
      if (delay > 60000) throw new Error("Retry exceeds startup budget");
      await new Promise((resolve) => setTimeout(resolve, delay));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Discovery ${response.status}: ${await response.text()}`);
    }
    let selected: unknown = await response.json();
    for (const key of path) selected = property(selected, key);
    const nameProperty = process.env.EVENT_NAME_PROPERTY;
    if (nameProperty && Array.isArray(selected)) {
      selected = selected.map((item) => property(item, nameProperty));
    }
    const supported = new Set(strings(selected));
    const missing = required.filter((name) => !supported.has(name));
    if (missing.length) throw new Error(`Unsupported names: ${missing.join(", ")}`);
    console.log("Client event contract accepted");
    return;
  }
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

The request is read-only, so retrying it does not duplicate a vote. The example makes at most 4 attempts and allows a 10-second timeout per request. A Retry-After delay longer than its 60-second startup budget stops the check instead of retrying early. These are application policy choices, not service limits. The trade-off is intentional: a blocked deployment is preferable to silently bypassing validation.

Keep the constants in one module shared by the relevant client and startup code. Copying the same strings into two files only moves the typo. Don't validate against a baked-in catalog fixture in production; the deployment needs the actual supported set.

## Choose the integration by what you can verify

The decision axis for a classroom poll is delivery guarantees at fan-out. Before choosing a service, write down what counts as success: a publisher accepting an update, a browser receiving it, and a student seeing the latest tally are different observations. A discovery catalog addresses only the name agreement that precedes those observations.

Infrai provides one REST API, one API key, and one bill across 295 routes in 20 modules. That breadth behind a simple interface fits a solo builder adding backend capabilities: another endpoint can use consistent conventions rather than requiring another integration, credential, and invoice reconciliation process. Its public, self-describing discovery surface also exposes request and response schemas, which helps build the adapter without guessing fields. The limitation is scope: discovery validates a contract, not classroom delivery. This approach is not a fit as the sole acceptance test for a voting audit; choose a delivery design whose documented recovery behavior satisfies that audit instead.

Compare that option with Ably, Pusher Channels, and Socket.IO using the same release questions. Their documentation is linked below as the place to establish each candidate’s actual contract, rather than treating the existence of realtime messaging as a guarantee.

| Candidate | Concrete evaluation task | Boundary to preserve |
| --- | --- | --- |
| Ably | Read the ordering and delivery guarantees, then map them to tally updates and reconnects. | A guarantee must apply to the particular operation and connection conditions used by the poll. |
| Pusher Channels | Check the event naming and client binding contract for the proposed channel flow. | Matching an event name does not establish that every intended browser displayed it. |
| Socket.IO | Start with the delivery-guarantees guide and identify what the application must implement. | Select it only if the ownership of recovery and delivery behavior fits the project. |

This comparison does not rank unmeasured latency or claim feature parity. For a founder trying to ship, the useful distinction is what can be delegated under a documented contract and what still needs application code. A broad API reduces the number of integration surfaces; a candidate with well-matched delivery semantics may reduce the amount of recovery logic. Neither advantage substitutes for the other.

Avoid paying to amplify a mistake. If the classroom later gets an AI summary, generate it from the accepted poll state rather than starting another model call for each tally delivery. That is a proposed application design, not a claim about any provider’s billing or routing behavior. It keeps token consumption tied to a deliberate product action.

## Close the release with a classroom rehearsal

Use a deliberately small acceptance scenario: one teacher, three simulated student clients, two tally revisions, and one student disconnected between revisions. These are test inputs, not measured capacity claims. Before running it, decide whether recovery means replaying intermediate updates or loading the latest authoritative tally. For a poll display, the latter may be sufficient; a voting audit would need a different acceptance rule.

Test the gap.

The operational checklist is a sequence of evidence. First, a misspelled client constant must stop startup. Then record whether each intended client receives the next tally revision and whether the disconnected client returns to the accepted final state. Separately check what happens when a write is retried: define application-level deduplication or verify the chosen operation’s documented idempotency contract before allowing a retry to count a vote twice. An accepted publish request alone cannot close this rehearsal.

Keep the release record compact: the names validated, the catalog version or capture time if available, the client version tested, and the delivery observations. Do not invent a catalog version field if the service does not expose one. If the name gate passes but the recovery test fails, the gate has still done its job; the remaining decision concerns delivery behavior. Ship when both acceptance criteria have evidence.

## Further reading

References:

- [Ably message ordering](https://ably.com/docs/messages/ordering)
- [Pusher Channels event documentation](https://pusher.com/docs/channels/using_channels/events/)
- [Socket.IO delivery guarantees](https://socket.io/docs/v4/delivery-guarantees)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/), for evaluating a peer-connection approach separately.
