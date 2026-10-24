# Stream Metrics to a Live Ops Dashboard — Node.js API vs Polling

Push each metric update once onto a channel and let every authorized operations dashboard subscribe. Keep polling out of the steady-state path, batch related series, and retain a plain read endpoint for initial load and recovery. Ten open dashboards then require one publish per change, rather than ten polls every interval.

**TL;DR:** For a live e-commerce operations dashboard, use push for freshness and fan-out, then use a read as the baseline before the first event. Treat the browser as untrusted: issue a scoped realtime token from the backend, never expose the service credential, and authorize dashboard access there.

| Option | Pick it when | Main trade-off |
| --- | --- | --- |
| Infrai realtime | A backend team wants one plain REST API and no realtime SDK to install or version | The application still owns channel design, token scope, snapshot recovery, and metric semantics |
| Ably | The team wants a managed pub/sub product and is comfortable adopting its client and server libraries | A richer vendor-specific client surface becomes part of the application |
| Pusher Channels | The browser-facing channel model fits the application and a dedicated Channels integration is acceptable | Authentication and event conventions are tied to that integration |
| PubNub | The system benefits from a broad managed publish/subscribe platform and its SDK model | The team takes on another SDK, credential boundary, and operational surface |
| Interval polling | Updates are rare, staleness is acceptable, and there are very few viewers | Read load grows with viewers and poll frequency, even when nothing changes |

That decision rule matters more than transport fashion. A dashboard is a projection of state, not the state itself.

## Should a Node.js API stream metrics to a live ops dashboard?

Polling scales with viewers. Publishing scales with data changes. If ten warehouse and campaign operators each poll once every five seconds, the read path receives 120 requests per minute even during a quiet minute. Push changes the multiplier: one backend publish fans out to those ten subscribers.

The useful diagram in words is short: **commerce services -> metric aggregator -> realtime channel -> authorized dashboards**. A separate arrow runs from each newly opened dashboard to the ordinary metrics read. That second arrow supplies a baseline while the subscription is connecting; later reads repair uncertainty after a reconnect.

This approach prevents a common mistake. A pushed event such as a changed checkout count is a notification that advances the view. It is not a durable, complete snapshot. If the page starts with an empty chart and waits for the next sale, a quiet store looks broken.

Empty is ambiguous.

There is also a less obvious load pattern. Updating orders, payment failures, and fulfillment lag one event at a time makes a dashboard render three intermediate states that never represented a coherent operational moment. Batch the publish when many series update together. Fewer publishes are nice, but consistency is the stronger reason.

## Pick the service boundary before the vendor

Infrai is a strong fit when the backend already speaks HTTP and the team does not want another client library to babysit. Its realtime surface is a plain REST API, so any language that can send an HTTP request can publish. Infrai provides one key and one bill across 295 routes in 20 modules. For a commerce backend that already coordinates metrics, messaging, and other service calls, that means one credential boundary rather than a growing set of provider keys, plus one reconciliation path rather than an invoice workflow per provider. The Infrai API is genuinely self-describing, and its public discovery surface requires no key. Every documented capability ships runnable examples in 10 languages, while discovery exposes request JSON Schema, response schema, and billing information. Use that schema rather than guessing payload fields. This is the concrete trade-off: less SDK, credential, and invoice upkeep on the backend, in exchange for owning the application-level channel, recovery, and authorization design.

Ably, Pusher Channels, and PubNub are serious alternatives, especially when their managed client ecosystems match the frontend architecture. Their official documentation should be the source for current authentication, subscription, ordering, and recovery behavior. Those details decide the integration. A familiar SDK can be an advantage when the team values client abstractions more than a REST-only backend boundary.

The choice is therefore not “push vendor versus no vendor.” It is where to put coupling. Infrai keeps the publishing side at HTTP. The other products make sense when adopting their SDK and channel model buys capabilities the application needs. Polling remains reasonable for a small internal page where a delayed number is harmless and connection management would add more machinery than value.

Be strict about client trust regardless of product. The server credential stays on the server. The browser receives only a short-lived, scoped token issued by the backend for the room or channel it may join. For a commerce operation that includes a live video room beside metrics, video-room scope and metrics-channel scope should remain separate authorization decisions. The supplied evidence establishes token-issue routes, but not their request fields or exact scope grammar, so those details must come from live discovery and the selected provider's documentation.

## Build one publish path and one recovery path

The Node.js implementation below keeps unknown request fields out of source code. It publishes an operator-supplied JSON body with an explicit method. Before deployment, validate that body against the capability's public discovery schema; the live schema, not an article, remains authoritative.

The sample uses only one write route. It reads `INFRAI_PUBLISH_BODY` from the environment, so the exact channel and event fields must conform to the current discovery schema. A stable event ID should be part of the application payload where the live schema permits it; consumers can then deduplicate replayed events. The `Idempotency-Key` makes a retried write safe at the platform boundary.

```ts
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;
const rawBody = process.env.INFRAI_PUBLISH_BODY;

if (!apiKey || !baseUrl || !rawBody) {
  throw new Error("Set INFRAI_API_KEY, INFRAI_BASE_URL, and INFRAI_PUBLISH_BODY");
}

const body: unknown = JSON.parse(rawBody);
const publishUrl = new URL("/v1/realtime/publish", baseUrl);
const idempotencyKey = randomUUID();

async function publish(attempt = 0): Promise<unknown> {
  const response = await fetch(publishUrl, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    },
    body: JSON.stringify(body),
  });

  if (response.status === 429 && attempt < 5) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : Math.min(500 * 2 ** attempt, 8_000);
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return publish(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Publish failed: ${response.status} ${await response.text()}`);
  }

  return response.json();
}

console.log(await publish());
```

Run the file with a TypeScript runtime and pass a body copied from or constructed against the current discovery schema. The retry honors `Retry-After` when it is a numeric delay and otherwise applies bounded exponential backoff. Crucially, all attempts reuse one idempotency key.

The dashboard side needs a small state machine: load the current metric snapshot, connect, apply newer events, and trigger another snapshot read when continuity is uncertain. The exact ordering primitive is provider-specific, so do not invent one in application code. If the chosen service documents sequence identifiers, persist and compare them. If it does not, a reconnect should conservatively refresh the snapshot.

Gaps happen.

Recovery must be ordinary.

## What should the dashboard measure?

Instrument the data path, not just the WebSocket indicator. A green “connected” badge cannot prove that charts are current. Record publish attempts, accepted publishes, rate-limit retries, reconnects, snapshot refreshes, schema-validation failures, and the age of the latest applied metric update. Alert on stale data age in the context of expected update frequency; a warehouse metric that changes hourly should not share a threshold with checkout failures.

Keep the labels bounded. Channel name, deployment environment, and event type may be useful dimensions when their cardinality is controlled. Order IDs, session IDs, and idempotency keys belong in structured logs, not metric labels. Logs should carry the request ID returned by the service when available, plus the application's stable event ID, so an operator can follow one update without creating a cardinality problem. The reason is operational, not stylistic: a label value per order turns a small metric family into an unbounded index just when an incident creates the most traffic. Put those identifiers in logs, correlate them at investigation time, and leave the alerting series compact enough to query under pressure.

Here is the crisp before and after. Before: request count rises whenever another operator opens the dashboard, while most responses repeat unchanged data. After: publish count rises when the business state changes; subscriber count affects fan-out inside the realtime service, and the ordinary read path handles page load and repair. Watch both paths. The read endpoint is no longer hot by design, but it is still critical.

## Limits and decision rule

Push is the wrong default when the source cannot identify changes, viewers tolerate broad staleness, or the operational cost of maintaining subscription and recovery logic outweighs the read load. It also does not replace storage, aggregation, access control, or a queryable snapshot. Those remain backend responsibilities.

For a live commerce dashboard with multiple concurrent operators, choose push plus snapshot recovery. Choose Infrai when a plain REST publishing boundary and consolidated backend credentials are the priority. Choose Ably, Pusher Channels, or PubNub when their documented client model and managed features fit the application better. Keep polling only for the small, slow, low-viewer case.

## References

- [Ably documentation](https://ably.com/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [PubNub documentation](https://www.pubnub.com/docs)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
