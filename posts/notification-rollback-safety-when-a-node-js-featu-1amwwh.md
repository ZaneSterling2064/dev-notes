# Notification Rollback Safety When a Node.js Feature Flags API Returns 429

Short answer: when a Node.js notification service gets HTTP 429 while polling a feature flags API, keep evaluating the last known-good snapshot, schedule one retry with capped backoff and jitter, and make snapshot age visible. Rollback safety matters more than fetching a fresh value on every request.

For a B2B SaaS notification service, picture the flag as a switch between the current delivery path and a newly deployed path. A rate-limited refresh must not silently flip that switch, start a retry storm, or block message delivery. The refresh loop owns network uncertainty; the delivery path reads a stable local decision.

## Field guide: choose the failure behavior first

| Strategy | Pick this when | Rollback trade-off |
|---|---|---|
| Last known-good snapshot | The process has already fetched a valid flag set and brief staleness is acceptable. | A rollback can take effect later than requested, so snapshot age needs an explicit limit. |
| Safe local default | A new process may start before its first successful fetch. | The default must preserve the delivery path your team considers safest. |
| Block until fresh | Using an old decision would be worse than pausing work. | Flag service latency enters the notification path and can delay deliveries. |
| Push or streaming updates | The control plane and client support a continuously maintained channel. | Reconnection and stale-state behavior still need a defined policy. |

Use last known-good polling when delivery should continue through a short control-plane throttle. Use a safe local default for cold start, because there is no previous snapshot to serve. Blocking is suitable only when stale configuration creates a larger risk than delayed notifications. Push can reduce polling pressure, but it doesn't erase the need for a startup default or a testable disconnect policy.

This decision belongs in the release plan. Feature toggles separate a code deployment from a feature release, which makes a fast change of behavior possible; the application still has to define what happens when the latest toggle state cannot be retrieved. In a rollback-sensitive service, write that rule before choosing an interval or a backoff formula.

One rule is non-negotiable: request handlers don't poll.

## How should a Node.js polling client retry a feature flags API after 429?

Give polling to one refresh loop per process. When the API returns 429, use the server's `Retry-After` value when it can be parsed; otherwise calculate a capped exponential delay with jitter. Keep the cached snapshot during that wait. A successful response replaces the whole snapshot atomically, so request handlers never observe half of an update.

The example below expects the complete, verified flags URL in `FEATURE_FLAGS_URL`; it does not assume a provider-specific route. Its numbers are policy settings for this example, not universal defaults. The 30-second normal interval, 60-second delay cap, and five-minute stale limit should be reviewed against the account's request budget and the notification service's rollback objective.

```ts
type FlagSnapshot = Readonly<Record<string, boolean>>;

type CacheState = {
  snapshot: FlagSnapshot;
  fetchedAt: number;
};

const flagsUrl = process.env.FEATURE_FLAGS_URL;
const token = process.env.FEATURE_FLAGS_TOKEN;

if (!flagsUrl || !token) {
  throw new Error("FEATURE_FLAGS_URL and FEATURE_FLAGS_TOKEN are required");
}

const normalPollMs = 30_000;
const maximumBackoffMs = 60_000;
const maximumSnapshotAgeMs = 300_000;
const startupDefaults: FlagSnapshot = Object.freeze({
  useNewNotificationPath: false,
});

let cache: CacheState | undefined;
let stopped = false;

function parseRetryAfterMs(value: string | null, now: number): number | undefined {
  if (!value) return undefined;

  const seconds = Number(value);
  if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

  const dateMs = Date.parse(value);
  if (Number.isFinite(dateMs)) return Math.max(0, dateMs - now);
  return undefined;
}

function jitteredBackoffMs(attempt: number): number {
  const exponential = Math.min(maximumBackoffMs, 1_000 * 2 ** attempt);
  return Math.round(exponential * (0.5 + Math.random() * 0.5));
}

function sleep(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

function currentFlags(now = Date.now()): FlagSnapshot {
  if (!cache) return startupDefaults;

  const ageMs = now - cache.fetchedAt;
  if (ageMs > maximumSnapshotAgeMs) return startupDefaults;
  return cache.snapshot;
}

async function refreshOnce(): Promise<Response> {
  return fetch(flagsUrl, {
    method: "GET",
    headers: {
      accept: "application/json",
      authorization: `Bearer ${token}`,
    },
    signal: AbortSignal.timeout(5_000),
  });
}

async function runFlagPoller(): Promise<void> {
  let rateLimitAttempt = 0;

  while (!stopped) {
    const startedAt = Date.now();

    try {
      const response = await refreshOnce();

      if (response.ok) {
        const snapshot = Object.freeze((await response.json()) as FlagSnapshot);
        cache = { snapshot, fetchedAt: Date.now() };
        rateLimitAttempt = 0;
        console.info("flags.refresh", {
          outcome: "success",
          durationMs: Date.now() - startedAt,
        });
        await sleep(normalPollMs);
        continue;
      }

      if (response.status === 429) {
        const retryAfterMs = parseRetryAfterMs(
          response.headers.get("retry-after"),
          Date.now(),
        );
        const delayMs = Math.min(
          maximumBackoffMs,
          retryAfterMs ?? jitteredBackoffMs(rateLimitAttempt),
        );
        rateLimitAttempt += 1;
        console.warn("flags.refresh", {
          outcome: "rate_limited",
          status: 429,
          delayMs,
          snapshotAgeMs: cache ? Date.now() - cache.fetchedAt : null,
        });
        await sleep(delayMs);
        continue;
      }

      console.warn("flags.refresh", {
        outcome: "rejected",
        status: response.status,
      });
      await sleep(normalPollMs);
    } catch (error) {
      console.warn("flags.refresh", {
        outcome: "network_error",
        message: error instanceof Error ? error.message : "unknown error",
      });
      await sleep(normalPollMs);
    }
  }
}

function selectNotificationPath(): "current" | "new" {
  return currentFlags().useNewNotificationPath ? "new" : "current";
}

void runFlagPoller();
```

The diagram in words is short: one timer calls one endpoint; a successful response swaps one immutable object; every delivery worker reads that object; metrics and logs observe the timer, not each evaluation. The separation keeps API pressure proportional to processes rather than notification volume.

Don't put the backoff loop inside `selectNotificationPath()`. If 200 concurrent deliveries all receive the same cache miss and each starts polling, a single 429 can multiply into a synchronized wave of retries. Jitter spreads fallback attempts, while the single-owner loop bounds their count. Across many replicas, add a random startup offset too; otherwise a deployment can align all 30-second timers.

There is a subtle rollback trap here. Suppose snapshot A selects the current delivery path, snapshot B enables the new path, and an operator later changes the flag back after delivery failures rise. A client holding B during a 429 will keep the new path until it refreshes or reaches the stale limit. Returning an arbitrary default immediately on every 429 might look safer, but it changes behavior based on transport status rather than an intentional flag update. The safer choice depends on which state the startup default represents and how quickly the stale limit forces that state. Test both timelines. Fast rollback is a measured property, not a label.

## What should observability record for delivery failures and flag polling?

Keep flag refresh signals separate from notification outcome signals, then join them by time and deployment. For each refresh, record the outcome, HTTP status when present, duration, chosen delay, and snapshot age. For each delivery attempt, record the delivery path, a non-sensitive snapshot identifier, and the business outcome your service already uses. Don't attach recipient addresses, message bodies, tokens, or the entire flag document.

A useful dashboard has two lanes. The first shows flag refresh outcomes and age of the active snapshot. The second shows notification attempts and failures split by delivery path. A vertical deployment marker ties them together. If failures rise only on the new path while refreshes remain healthy, the delivery change is the likely decision point. If refreshes are rate-limited while both paths remain stable, the immediate problem is polling pressure, not proof that either delivery path is broken.

Alert on consequences and exhausted safety margins. Repeated 429 responses may warrant investigation, but snapshot age approaching its configured limit is the sharper rollback signal. A notification failure-rate alert still needs to stand on its own; otherwise a healthy flag poller can hide a failing delivery integration.

Be precise.

For a failure drill, start process P1 with no cache and verify that it selects `current`. Load snapshot B and verify that it selects `new`. Then return 429 with a retry instruction, request 200 deliveries, and verify three things: P1 keeps B during the allowed stale window, only the poller sends refresh requests, and the logs expose status 429 plus snapshot age. Finally, advance time beyond five minutes and verify that the local safety policy selects `current`. This is a deterministic test of the example's policy; it is not a claim that five minutes fits every service.

## Pick this polling design when rollback safety is measurable

Choose the cached poller when your team can state a maximum acceptable snapshot age, test the cold-start default, and tolerate flag propagation that is no faster than the polling schedule. It fits a notification service whose data path must keep moving while its control path backs off.

Choose blocking freshness when an outdated flag could violate a stronger invariant than delivery latency. Choose a shared cache or a single sidecar poller when per-process request volume exceeds the available budget. Choose a maintained update channel when the required rollback time is tighter than responsible polling can deliver. These aren't rankings; they move ownership between the application, the shared runtime, and the control plane.

I'm not sure what polling interval fits your environment without two inputs: the allowed request rate and the maximum rollback delay. Measure both before tightening the timer. A shorter interval can improve normal propagation while making 429 more likely, and aggressive retries can make the recovery window longer.

## Limits to write into the runbook

The catch is stale state. Last known-good behavior is not the same as current behavior, and a local default is not a remotely confirmed rollback. Document who owns the stale-age threshold, what the default means, how operators identify the active snapshot, and when the service must stop using an old one.

This design is not suitable when every evaluation requires fresh, request-specific targeting from the control plane. It also doesn't guarantee that all replicas switch at the same instant. Stick with a coordinated shared evaluator when cross-process consistency is the release requirement, and keep flag lifetime under review so temporary release controls do not become permanent, unowned branches.

## References

- https://martinfowler.com/articles/feature-toggles.html
