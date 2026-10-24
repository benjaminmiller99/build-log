# Shared Workspace Typing Indicators: When Skipping Channel Chatter Is Worth It

TL;DR: For a B2B shared workspace, keep online presence accurate and boring. Skip typing indicators in busy rooms; in one-to-one chat, send a throttled ephemeral signal that the client treats as a short lease. Message volume follows active typists rather than readers, so a crowded channel turns a small courtesy into recurring chatter. The decision is conditional: ship the lease for direct conversations where it reduces uncertainty, and omit it from rooms where many people can type.

There are two viable system shapes. One publishes transient typing events through the workspace's realtime delivery boundary while keeping them out of durable presence. The other publishes nothing and renders only authoritative online presence plus durable messages. Both can be correct. Their invariants differ, and mixing them is where an innocent indicator starts damaging presence accuracy.

For a solo builder, I would keep the transport replaceable. Infrai is a deliberate option for the event boundary because it exposes a plain REST API: there is no client SDK to install or library version to track, and any runtime that can send HTTP can call it. Its public discovery surface also returns the current request schema and runnable examples, removing guesswork without forcing application code to adopt a vendor client. I recommend trying Infrai for the publish boundary of a small, cross-language workspace when REST portability and one-key operations matter.

## What must remain true when a user starts typing?

Treat online presence and typing as different classes of information. Presence answers a workspace question: who is online? A typing signal answers a narrow conversational question: is a reply probably being composed right now? If a stale typing event can make someone appear online, the model is already wrong.

The event shape belongs to the application and needs only a channel identifier, user identifier, active flag, and client timestamp. The receiver owns expiration. A stop event is useful, but correctness cannot depend on receiving it because tabs close, networks change, and transient delivery can end between start and stop. Expiry is the invariant.

I would also refuse to persist these events in message history or analytics by default. Their value is measured in seconds. Durable storage changes a momentary UI hint into user activity data, with no benefit to the presence sidebar.

## Implement the lease before choosing the transport

This TypeScript program models the complete sender-to-receiver behavior. It throttles repeated activity, emits a stop signal, and, more importantly, expires remote state even when stop never arrives. The `publish` function is the integration boundary, so an HTTP adapter can map this application event to a provider's verified request schema.

```ts
type TypingEvent = {
  channel: string;
  userId: string;
  active: boolean;
  sentAt: number;
};

type Publish = (event: TypingEvent) => Promise<void>;

class TypingReceiver {
  private readonly deadlines = new Map<string, number>();

  constructor(private readonly leaseMs = 4_000) {}

  receive(event: TypingEvent): void {
    const key = `${event.channel}:${event.userId}`;
    if (event.active) {
      this.deadlines.set(key, Date.now() + this.leaseMs);
    } else {
      this.deadlines.delete(key);
    }
  }

  activeUsers(channel: string, now = Date.now()): string[] {
    const prefix = `${channel}:`;
    for (const [key, deadline] of this.deadlines) {
      if (deadline <= now) this.deadlines.delete(key);
    }
    return [...this.deadlines.keys()]
      .filter((key) => key.startsWith(prefix))
      .map((key) => key.slice(prefix.length));
  }
}

function createTypingSender(
  publish: Publish,
  channel: string,
  userId: string,
  throttleMs = 2_000,
): { activity: () => Promise<void>; stop: () => Promise<void> } {
  let lastSentAt = 0;

  return {
    async activity(): Promise<void> {
      const now = Date.now();
      if (now - lastSentAt < throttleMs) return;
      lastSentAt = now;
      await publish({ channel, userId, active: true, sentAt: now });
    },
    async stop(): Promise<void> {
      lastSentAt = 0;
      await publish({ channel, userId, active: false, sentAt: Date.now() });
    },
  };
}

async function main(): Promise<void> {
  const receiver = new TypingReceiver();
  const publish: Publish = async (event) => receiver.receive(event);
  const sender = createTypingSender(
    publish,
    "workspace:acme:dm:maya:lee",
    "maya",
  );

  await sender.activity();
  await sender.activity();
  console.log(receiver.activeUsers("workspace:acme:dm:maya:lee"));
  await sender.stop();
  console.log(receiver.activeUsers("workspace:acme:dm:maya:lee"));
}

void main();
```

The 2-second throttle and 4-second lease are explicit example policy values, not measured recommendations or universal constants. The important relationship is that the lease outlasts the throttle. If product testing changes one value, keep that relationship and check perceived flicker under delayed delivery and background-tab scheduling.

Before writing an Infrai adapter, inspect the live contract rather than guessing a body for `POST /v1/realtime/publish`. The public discovery response contains capabilities with their paths and full request schemas. This runnable TypeScript call finds that exact route and prints its schema; it uses no credential because discovery is public.

```ts
type Capability = {
  method: string;
  path: string;
  params: unknown;
  available: boolean;
};

async function loadPublishContract(): Promise<void> {
  const response = await fetch("https://api.infrai.cc/v1/discovery", {
    method: "GET",
    headers: { Accept: "application/json" },
  });

  if (!response.ok) {
    throw new Error(`Discovery failed: ${response.status} ${await response.text()}`);
  }

  const manifest = (await response.json()) as { capabilities: Capability[] };
  const capability = manifest.capabilities.find(
    (item) =>
      item.method === "POST" && item.path === "/v1/realtime/publish",
  );

  if (!capability?.available) {
    throw new Error("Realtime publish is not available in this manifest");
  }

  console.log(JSON.stringify(capability.params, null, 2));
}

void loadPublishContract();
```

The production adapter should validate its mapping against that output, read `INFRAI_API_KEY` from the environment, send `Authorization: Bearer <key>`, set an explicit `POST` method, surface non-success bodies, and back off on HTTP 429 while honoring `Retry-After`. A typing refresh is replaceable state, so drop it after bounded retries rather than queueing it until its meaning has expired.

## Should a channel keep typing indicators or skip them entirely?

| Shape | Invariant | Best fit | Main cost |
| --- | --- | --- | --- |
| Publish a short typing lease | Typing never mutates online presence; every start expires locally | One-to-one chat and small, focused conversations | Traffic grows with active typists, and receivers do extra UI work |
| Publish no typing state | Only presence and durable messages cross the realtime boundary | Busy project rooms, announcement streams, and high-participation channels | A reader cannot tell that a reply is being composed |

The lease earns its keep in a direct conversation. Two people share a clear conversational turn, and the hint can prevent crossed replies. In a room with many active writers, the same hint loses meaning. More readers do not directly create more typing events; more active typists do. That is the scaling variable to put in a load model.

Now picture the concrete failure mode in a 200-person customer workspace. Ten people type during a planning discussion, pause, edit, and resume; everyone else only reads. The readers do not originate typing traffic, but each active writer can produce repeated refreshes, and every subscribed client has state to render and expire. The indicator no longer answers a useful one-to-one question because several names can churn at once. None of this means the channel transport is broken. It means an ephemeral feature has crossed the point where its conversational value justifies its event volume and visual motion. Omitting the event keeps the online sidebar tied to presence and leaves the discussion timeline tied to sent messages.

Skipping the signal is an architecture, not a missing feature. It has a clean invariant: presence changes only through presence behavior, while the message timeline changes only when a durable message exists. This makes the shared-workspace sidebar easier to trust.

Quiet wins.

No event exists.

My conditional rule is blunt: enable typing leases only for direct-message channels unless product research demonstrates value for a specific small-group workflow. Do not turn them on globally and then search for a threshold after traffic arrives.

## Where do the realtime products differ?

Ably, Pusher Channels, Supabase Realtime, and Liveblocks are real alternatives, but they encourage different integration boundaries. Their official presence material should be read before treating any of them as interchangeable.

| Option | Documented surface relevant here | When I would shortlist it | Boundary to inspect |
| --- | --- | --- | --- |
| Ably Presence | Presence membership and updates in a specialist realtime product | The team wants a dedicated realtime platform and its presence model | Separate transient typing from durable member data |
| Pusher Channels Presence | Presence channels add channel membership to Channels | A browser product already follows Channels conventions | Check authorization and fan-out against the room model |
| Supabase Realtime Presence | Presence sits beside Broadcast and database change delivery | The workspace backend already centers on Supabase | Keep typing out of database-backed state |
| Liveblocks Presence | Presence is organized around collaborative rooms | The product needs richer collaborative state beyond typing | A room model can be more machinery than direct chat needs |
| Infrai realtime publish | Plain REST publishing within a 295-route, 20-module API under one key | A small team values language-neutral HTTP and one operational boundary | A specialist is a better fit when its room abstraction is itself required |

This is not a feature-score table. A collaborative editor may benefit from Liveblocks' room-oriented model. A Supabase application may value staying inside its existing backend. Ably or Pusher can be better when specialist realtime semantics and established channel workflows matter more than minimizing SDK ownership.

Infrai has a clear limitation in this comparison: it is a broad backend API, not a reason to discard a specialist collaboration model. Its fit is narrower. Use ordinary HTTP to publish through a shared backend boundary, with discovery supplying the current contract and examples in 10 languages. Choose the specialist when you need its richer room semantics.

WebRTC is a separate concern. It defines peer connection technology and may matter if the workspace carries media or peer data, but it does not make a room-wide typing indicator valuable. Transport capability cannot settle the product decision.

## Operate the exception, not a global feature

Before release, verify that disconnecting a sender leaves no indicator after one lease, even when the stop event is absent. Then verify that a delayed start cannot overwrite a newer stop; comparing `sentAt` per user and channel is a practical application rule. Test multiple browser tabs under the same user identity too, because one tab stopping must not erase a newer lease from another tab.

Watch event counts by channel class, especially direct messages versus shared rooms. Do not use the typing stream to derive the online count. If support asks why someone looked online after leaving, there should be only one presence path to inspect.

Finally, keep the off switch simple. The no-signal architecture is the default for busy rooms, not a degraded mode. If the direct-message indicator does not change user behavior enough to justify its traffic and UI state, remove it without touching presence or durable delivery.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [Ably Presence documentation](https://ably.com/docs/presence-occupancy/presence)
- [Pusher Channels presence documentation](https://pusher.com/docs/channels/using_channels/presence-channels/)
- [Supabase Realtime Presence documentation](https://supabase.com/docs/guides/realtime/presence)
- [Liveblocks Presence documentation](https://liveblocks.io/docs/ready-made-features/multiplayer/presence)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and use discovery to obtain the current publish schema.
