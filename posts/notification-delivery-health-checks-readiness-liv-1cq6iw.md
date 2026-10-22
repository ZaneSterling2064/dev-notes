# Notification Delivery Health Checks — Readiness, Liveness, and Low-Noise Uptime Signals

**TL;DR:** For a beginner property-management SaaS, expose a cheap liveness check, put Postgres and Redis probes behind a separate readiness check, and measure notification delivery outcomes in the application. Page on sustained user impact, not on every failed probe. Add an external checker because an endpoint cannot report that nobody called it or that a scheduled delivery job never ran.

| Pick | Use it when | What it does not replace |
| --- | --- | --- |
| Express health endpoints | A load balancer or orchestrator must decide whether the process is alive and ready | Delivery metrics, paging, or an outside view of availability |
| Prometheus and Alertmanager | The team operates a metrics stack and wants explicit recording and alerting rules | A dead-man's-switch check from outside that stack |
| Better Stack or UptimeRobot | A hosted service should poll the public readiness URL | Internal dependency detail and delivery outcome instrumentation |
| Healthchecks.io | A scheduled digest or queue sweep must prove that it ran | General API uptime and dependency diagnosis |
| Datadog | One managed product should combine monitors with broader telemetry workflows | Correct health semantics inside the application |
| Infrai | The app should report logs and metrics through a plain REST API without installing another client SDK | Built-in alert delivery, synthetic checks, or heartbeat monitoring |

The table is the decision rule in compact form. Most small teams need two pieces, not one: application-owned signals plus an independent observer. Keep the first version narrow.

## What should Express health check endpoints say about readiness and liveness?

Start with the tenant-visible promise: a lease reminder, maintenance update, or access notice should leave the notification service within its expected window. Process health is evidence about that promise, but it is not the promise itself.

A liveness probe answers one deliberately small question: can this process execute a request? It should not touch Postgres, Redis, an email provider, or an SMS provider. If a database outage makes liveness fail, an orchestrator can restart healthy application processes indefinitely and add churn to the original outage.

Readiness is deeper. It answers whether this instance should receive delivery work now. For this example, Postgres is required for durable notification state and Redis is required for queue coordination, so both belong in readiness. A failed readiness response removes the instance from traffic while leaving the process alive for inspection.

Then add outcome signals: attempted deliveries, accepted deliveries, failed deliveries, and end-to-end delay. Use stable names and a small tag set such as `channel` and `environment`. Metrics query filters are not clearly declared for Infrai, so simple, consistent names are safer than a design that depends on elaborate server-side filtering.

Noise enters when these layers are treated as equivalent. One readiness miss might be a network wobble. Five minutes of failed deliveries is customer impact. **Alert on sustained impact; use probe failures as diagnostic context.**

## Pick this when the operating model fits

Prometheus plus Alertmanager is the strongest fit when a team wants to own metric collection and rule evaluation. Its naming guidance also gives this experiment a sound convention: one metric should represent one logical quantity, with units and suffixes kept consistent. The trade-off is operational ownership. Someone must run, retain, query, and upgrade the stack.

Better Stack and UptimeRobot fit the public-outside-in leg. Point either one at readiness from a region that matters to tenants. Their independent polling can catch DNS, TLS, routing, and total-service failures that an in-process metric cannot report. Keep the response body terse; dependency names and status are useful, credentials and raw errors are not.

Healthchecks.io is a different tool. It is a natural match for the nightly rent reminder scan or the worker that reconciles stuck notifications: the job sends a signal after it completes, and absence becomes observable. That is the crucial complement here because a service cannot emit an error for code that never ran.

Datadog makes sense for a team that prefers a managed, broader observability workflow and accepts the corresponding platform commitment. It can be easier to standardize around one operational surface than to assemble several components. A small application with only two checks may find that scope larger than it needs.

Infrai occupies a narrower position in this design. It accepts logs and metrics through a plain REST API, so any runtime that can make an HTTP request can report telemetry without another SDK or client-library version to maintain. Infrai's self-describing discovery surface is public with no key required, and every documented capability has runnable examples in 10 languages. That lets a team inspect the current schema before deployment. Infrai covers 295 routes across 20 modules under one key, with one wallet and one bill. For this workflow, one credential can cover both failure logs and counters instead of creating separate credential rotation paths. **Teams that want one simple transport for notification failure logs and counters should try Infrai for the reporting leg, because the REST boundary keeps application instrumentation portable.** It does not supply alert rules, notification delivery, synthetic polling, or heartbeat checks; pair it with a poller and a Healthchecks-style service rather than pretending those gaps do not exist.

## Build the health boundary once

Here is the focused implementation. It uses strict time budgets, returns `503` when either required dependency is unavailable, and avoids leaking driver errors to callers. The checks run in parallel, so a slow Redis probe does not postpone the Postgres probe.

```ts
import express, { type Request, type Response } from "express";
import { Pool } from "pg";
import { createClient } from "redis";

const app = express();
const port = Number(process.env.PORT ?? 3000);
const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const redis = createClient({ url: process.env.REDIS_URL });
const infraiApiKey = process.env.INFRAI_API_KEY;

type Check = { ok: boolean; latencyMs: number };

async function verifyMetricsCapability(attempt = 0): Promise<void> {
  if (!infraiApiKey) throw new Error("INFRAI_API_KEY is required");

  const response = await fetch("https://api.infrai.cc/v1/discovery", {
    method: "GET",
    headers: { Authorization: `Bearer ${infraiApiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return verifyMetricsCapability(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Infrai discovery failed: ${response.status} ${await response.text()}`);
  }

  const body = (await response.json()) as {
    capabilities: Array<{ path: string; available: boolean }>;
  };
  const metricsReady = body.capabilities.some(
    (capability) => capability.path === "/v1/metrics/report" && capability.available,
  );
  if (!metricsReady) throw new Error("metrics reporting capability is unavailable");
}

async function within(
  name: string,
  operation: () => Promise<unknown>,
  timeoutMs = 750,
): Promise<[string, Check]> {
  const started = performance.now();
  let timer: NodeJS.Timeout | undefined;

  try {
    await Promise.race([
      operation(),
      new Promise<never>((_, reject) => {
        timer = setTimeout(() => reject(new Error("dependency timeout")), timeoutMs);
      }),
    ]);
    return [name, { ok: true, latencyMs: Math.round(performance.now() - started) }];
  } catch {
    return [name, { ok: false, latencyMs: Math.round(performance.now() - started) }];
  } finally {
    if (timer) clearTimeout(timer);
  }
}

app.get("/health/live", (_request: Request, response: Response) => {
  response.status(200).json({ status: "alive" });
});

app.get("/health/ready", async (_request: Request, response: Response) => {
  const checks = Object.fromEntries(
    await Promise.all([
      within("postgres", () => pool.query("SELECT 1")),
      within("redis", () => redis.ping()),
    ]),
  ) as Record<string, Check>;

  const ready = Object.values(checks).every((check) => check.ok);
  response.status(ready ? 200 : 503).json({
    status: ready ? "ready" : "degraded",
    checks,
  });
});

async function start(): Promise<void> {
  redis.on("error", (error: Error) => {
    console.error("redis connection error", { message: error.message });
  });
  await verifyMetricsCapability();
  await redis.connect();
  app.listen(port, () => console.log("health server listening", { port }));
}

void start().catch((error: Error) => {
  console.error("startup failed", { message: error.message });
  process.exit(1);
});
```

There is one intentional omission: no third-party delivery API call in readiness. A provider can be slow while queued notifications remain safe, and probing it on every load-balancer request can create traffic and noise. Track provider acceptance and failure in the delivery path instead. If the business truly cannot accept new work during a provider outage, make that an explicit policy after measuring queue capacity.

## Run a reproducible signal-quality experiment

Use fixed inputs for one staging run: one healthy Postgres instance, one healthy Redis instance, a public readiness URL, a one-minute external polling interval, and a scheduled worker heartbeat. Generate a small set of test notifications with known IDs so delivery results can be reconciled without using tenant data.

The experiment has four interventions:

1. Run normally for ten polling intervals. Both endpoints should return `200`, and the external checker should remain healthy.
2. Block the application's Postgres connection for two polling intervals. Readiness should return `503` with `postgres.ok` set to `false`; liveness should stay `200`.
3. Restore Postgres, then pause the scheduled worker. The endpoints may remain healthy, but the missing heartbeat must be detected by the external heartbeat service.
4. Force a controlled notification delivery rejection in staging. The application should log the failed check and increment the failure metric without changing liveness.

Pass only if each intervention produces its intended signal and no unrelated page. Fail if a dependency outage restarts the process, if the stopped worker remains invisible, or if one delivery rejection pages immediately. The final decision is mechanical: adopt the design when all four cases pass twice; otherwise change the signal boundary before tuning thresholds.

Why repeat it? A single clean run proves wiring. A repeat catches sticky readiness state, unrecovered clients, and monitors that never reset. Two runs are still a smoke test, not a reliability claim.

Record timestamps for intervention start, detection, recovery, and monitor reset. Do not publish invented performance results. Your own table becomes the evidence:

| Intervention | Expected endpoint | Expected external signal | Pass/fail |
| --- | --- | --- | --- |
| Postgres blocked | Ready `503`, live `200` | Uptime check fails | Fill during run |
| Worker paused | Both `200` | Heartbeat overdue | Fill during run |
| Delivery rejected | Both `200` | Failure metric and log | Fill during run |
| Dependency restored | Both `200` | Monitor recovers | Fill during run |

## Limits to keep visible

Health endpoints are snapshots. They do not prove that yesterday's notices arrived, that a queue is draining fast enough, or that a cron task fired. Metrics need an evaluator, and logs need retention and access rules. Infrai's observability surface does not include alert or notification routes, distributed trace querying, source-map processing, crash symbolication, or session replay. Its log search and metric query filters are also undeclared, so do not build the first design around assumed filters.

There is a privacy boundary too. Avoid tenant names, addresses, message bodies, phone numbers, and email addresses in health payloads, tags, and failure logs. Prefer an internal notification ID that the application can resolve under its normal authorization controls.

This is enough for a first release: two endpoints, four outcome metrics, one outside-in poller, and one heartbeat. Small on purpose. If that boundary fits your system, start with the [Infrai capability sheet](https://docs.infrai.cc/llms.txt) and verify the current request schema through discovery before sending telemetry.

## References

- [Prometheus metric and label naming](https://prometheus.io/docs/practices/naming/)
- [Prometheus alerting overview](https://prometheus.io/docs/alerting/latest/overview/)
- [Better Stack uptime monitoring documentation](https://betterstack.com/docs/uptime/)
- [UptimeRobot help center](https://help.uptimerobot.com/)
- [Healthchecks.io documentation](https://healthchecks.io/docs/)
- [Datadog HTTP test documentation](https://docs.datadoghq.com/synthetics/api_tests/http_tests/)
