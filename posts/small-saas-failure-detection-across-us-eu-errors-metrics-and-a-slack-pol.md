# Small SaaS Failure Detection Across US/EU: Errors, Metrics, and a Slack Poller

**Start with exception alerts, add metrics only for rate thresholds, and use one tiny poller to deliver Slack or email notifications.**

For a small SaaS, I would not begin with an all-in-one observability rollout. I would capture code crashes first, poll grouped errors on a schedule, and send a notification only when the observed state changes. Logs come next when an engineer needs request context. Metrics earn their place when the question becomes "is the failure rate rising?" rather than "did this code crash?"

There is one important gap: a polling worker cannot prove that it ran. If missing heartbeats or outside-in uptime matter, pair this stack with an external healthcheck service. Keep that separate and obvious.

Keep it boring.

## Before: one alert stack trying to do three jobs

The messy version looks familiar. An application emits errors, logs, and metrics. Each stream grows its own rule language. Then Slack delivery, email delivery, retries, and escalation get attached to whichever product happened to be configured first. Soon, changing the signal source also changes notification code. I teach alerting for a living, and this is where a small setup starts feeling much larger than the app it watches.

My after picture is a diagram in words: **application -> signal store -> polling decision -> notification destination**. The application records facts. The polling worker decides whether those facts deserve attention. Slack or email carries the decision to a human. Four boxes. Three arrows. Done.

This split matters because the signals answer different questions. An error event says an exception occurred. A metric says a count, rate, or gauge crossed a threshold. A log supplies the request and execution context needed to investigate. Treating all three as interchangeable usually creates either noisy alerts or clever queries nobody wants to own at 2 a.m.

I once had a poller run every 60 seconds while its retry loop quietly swallowed 17 consecutive `429` responses. The job logged a cheerful completion line after each attempt because the retry helper returned an empty result instead of raising the response. Meanwhile, the comparison step interpreted that empty value as "no new groups." I noticed only after sending a test exception and watching Slack stay quiet through several polling cycles. The fix was conceptual before it was technical: a rate limit is an unknown alert state, never an empty alert state. That changed my default. Retries must be bounded, exponential, and aware of `Retry-After`; any other unsuccessful response must surface its body; and the stored baseline must remain untouched until a real response arrives. Silence is not success.

The catch is that a cron poller is another moving part. It is still a good trade for a small team when notification routing is absent, because the boundary stays easy to inspect and replace. For a staffed operations team that already owns routing policy, on-call schedules, and escalation, keeping alert evaluation inside its existing platform may be simpler.

## Should a small SaaS alert on errors, logs, or metrics?

Use errors first when the requirement is "tell me when code crashes." Grouped exceptions give the polling decision a much cleaner input than a free-form log search. Add metrics for rate-based conditions such as a spike in server-error responses. Use logs when the responder needs richer request context, but expect more work to turn that context into a dependable alert condition.

Here is how I would shortlist real options. This is a workflow comparison, not a claim that every row has identical storage or query features.

| Option | Best fit in this design | Trade-off I would accept or avoid |
| --- | --- | --- |
| Sentry | Exception-first workflow | Choose it when error triage is the center of the job; assess notification and surrounding telemetry needs separately. |
| Datadog | A team already standardizing alert operations in one established platform | More platform than a tiny service may need on day one. |
| Grafana Cloud | Metrics-led teams comfortable owning alert rules and dashboards | Requires more observability decisions than a crash-first setup. |
| Amazon CloudWatch | Workloads already operated around AWS telemetry | Log ingestion and related usage have explicit pricing dimensions to track. |
| Better Stack | Teams wanting hosted monitoring and incident workflow together | Check whether its workflow matches the signals and regions you actually need. |
| Infrai | A thin REST contract across backend capabilities | It has no native notifier, alert routing, or escalation, so a poller is mandatory. Its useful edge here is contract stability: one key and one REST API let the code stay put while the provider behind a capability changes. |

I would pick the last approach when I value a small HTTP surface and want vendor swapping behind the capability to leave application code unchanged. I would stick with Sentry when exception investigation is the product-shaped center of the workflow, or with Datadog, Grafana Cloud, or CloudWatch when the team already operates its alert rules there. Your mileage may vary; existing operational muscle often beats a cleaner diagram.

## A copyable cron poller for Slack notifications

This TypeScript worker makes deliberately few assumptions. It fetches the verified grouped-error route, hashes the returned JSON, and notifies Slack only when the representation changes after the initial baseline. It does not invent filters or response fields. That makes it coarse, but honest. In production, I would refine the decision only after reading the response schema from discovery and choosing an explicit severity policy.

The sample uses a local state file, so run it from a cron host with persistent storage. A serverless deployment should replace those two file operations with its durable key-value store. The polling interval belongs in the scheduler, not in an endless loop.

No guesswork.

```ts
import { createHash } from "node:crypto";
import { readFile, writeFile } from "node:fs/promises";

const apiBase = required("OBSERVABILITY_API_BASE").replace(/\/$/, "");
const apiKey = required("INFRAI_API_KEY");
const slackWebhook = required("SLACK_WEBHOOK_URL");
const statePath = process.env.ALERT_STATE_PATH ?? "./error-groups.sha256";

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing environment variable: ${name}`);
  return value;
}

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return seconds * 1_000;
    const dateDelay = Date.parse(value) - Date.now();
    if (dateDelay > 0) return dateDelay;
  }
  return 500 * 2 ** attempt;
}

async function getGroups(): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(`${apiBase}/v1/errors/groups`, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 4) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelay(response, attempt)),
      );
      continue;
    }
    if (!response.ok) {
      throw new Error(`Error query failed (${response.status}): ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("Rate-limit retry budget exhausted");
}

async function previousDigest(): Promise<string | undefined> {
  try {
    return (await readFile(statePath, "utf8")).trim();
  } catch (error) {
    if ((error as NodeJS.ErrnoException).code === "ENOENT") return undefined;
    throw error;
  }
}

async function notifySlack(payload: unknown): Promise<void> {
  const response = await fetch(slackWebhook, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      text: `Error groups changed:\n${JSON.stringify(payload, null, 2)}`,
    }),
  });
  if (!response.ok) {
    throw new Error(`Slack delivery failed (${response.status}): ${await response.text()}`);
  }
}

const groups = await getGroups();
const digest = createHash("sha256").update(JSON.stringify(groups)).digest("hex");
const before = await previousDigest();

if (before && before !== digest) await notifySlack(groups);
await writeFile(statePath, `${digest}\n`, "utf8");
```

Set `OBSERVABILITY_API_BASE`, `INFRAI_API_KEY`, and `SLACK_WEBHOOK_URL`, then schedule the compiled worker. Short code is not the same as careless code — the status checks and rate-limit behavior are the important half of this example.

## What changes for US and EU workloads?

Region requirements should be a selection input, not a footnote added after launch. The public discovery response exposes a `regions` field for capabilities, so inspect the exact capability you plan to call and verify that its declared regions match the workload. I would not infer residency, replication, or legal coverage from a region label alone. Those are separate questions, and I'm not sure why teams so often collapse them into one checkbox.

The poller boundary helps here. Keep the signal call behind one small adapter and keep Slack or email delivery behind another. If a team later changes the provider serving the signal capability, the decision code can retain the same contract. If policy requires notifications to take another route, the signal collection does not need to move with it. This is the crisp before/after I want engineers to remember: before, provider details leak through capture, evaluation, and delivery; after, the replaceable edges meet at plain JSON.

Still, don't promise more than the interface establishes. Confirm where telemetry is accepted, processed, and retained with the provider before sending production data. Logs deserve extra scrutiny because this capability has no per-user deletion interface, no bulk export or subscription interface, and no exposed configuration entry point for retention or cold storage. For a product with strict erasure workflows, that is not suitable; choose a log system whose deletion and lifecycle controls meet the policy.

Also decide which data crosses each boundary. An exception notification usually needs an identifier and a concise summary, not a complete request body. Slack should point responders toward investigation without becoming a second telemetry archive. Less payload means fewer accidental surprises.

## Where does this simple failure stack stop fitting?

Two objections come up every time I teach this design. First: why not alert from logs? You can, but logs are best here as investigation context. Their search filters are not declared in the discovery parameters, so I would not build a critical rule around guessed query fields. A metric query has the same undeclared-filter caveat. Start with grouped errors, then add a rate threshold only after the exact query contract is verified.

Second: can the cron poller cover every kind of failure? No. It cannot detect its own missing execution, and the observability capability does not provide synthetic checks or heartbeat monitoring. Use a service such as Healthchecks for "the task should have run but did not." That extra tool is not architecture failure. It closes a different detection gap.

There are firmer limits. This setup has no native alert routing or escalation, no distributed-trace query or span tree, and no source-map decoding, crash symbolication, Electron minidump parsing, or Session Replay. Logs can carry `trace_id` and `span_id` for correlation, but correlation fields are not a tracing backend. If responders need full trace navigation, rich client replay, or symbolicated native crashes, choose a platform built for those workflows instead of stretching this poller.

Keep the escalation threshold practical too. A founder and two engineers may be well served by one Slack destination and one email fallback. A larger rotation needs ownership, schedules, deduplication, acknowledgements, and escalation policy. At that point, stick with the alert-management system the on-call team already trusts, even if the tiny worker remains useful for a narrow signal. Simple is contextual.

## References

- OpenTelemetry observability primer: https://opentelemetry.io/docs/concepts/observability-primer/
- Sentry alerts documentation: https://docs.sentry.io/product/alerts/
- Datadog monitors documentation: https://docs.datadoghq.com/monitors/
- Grafana Cloud alerting documentation: https://grafana.com/docs/grafana-cloud/alerting-and-irm/alerting/
- Amazon CloudWatch pricing: https://aws.amazon.com/cloudwatch/pricing/
- Healthchecks documentation: https://healthchecks.io/docs/
- Slack incoming webhooks: https://api.slack.com/messaging/webhooks
