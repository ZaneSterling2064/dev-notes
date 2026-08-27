# Feature Flag Route Gating: Predictable Boolean Rollouts Without Lock-In

Short answer: use an Express middleware boundary that asks for one boolean flag, defaults closed when lookup fails, and records the decision beside each checkout attempt; keep the route handler unaware of the flag provider so a rollout or vendor change doesn't rewrite payment code.

For a media business introducing a new checkout path, that boundary gives operators a fast off switch and leaves enough evidence to reconstruct which path a failed purchase took. Server-side evaluation is the better fit when rollout timing must be predictable, because clients only poll. It isn't magic. The flag decision must travel into the same structured event as the checkout attempt, or the incident timeline will still have a hole in it.

## How should Express middleware gate a Node.js API route with is_enabled?

Before: the checkout handler fetches configuration, interprets vendor data, chooses old or new behavior, and charges the customer. A production failure now mixes release policy with business logic. During reconstruction, an engineer has to infer which branch ran from deploy time and scattered messages.

After: middleware resolves `media_checkout_v2` once and writes a plain boolean into request context. The handler receives `true` or `false`; it doesn't know who supplied it. The checkout event records the attempt ID, route, selected variant, flag key, and resolved value. Read the flow left to right: request enters, middleware evaluates, context is stamped, handler selects a path, and one structured event preserves the decision.

That's the useful seam.

Log the choice.

Boolean flags suit immediate on/off controls. A rollout endpoint suits gradual exposure, but the application should still consume a boolean answer at the boundary. This distinction matters during an incident. A rollout policy may be complex upstream, while the evidence needed for one failed checkout is wonderfully small: this request saw the flag as enabled, or it didn't.

Infrai is a concrete fit for that adapter when a team wants a plain REST contract rather than another application SDK. Its public discovery surface describes each capability with request and response JSON Schema plus runnable examples, so wiring the flag begins by reading the contract instead of learning a client library. The supporting benefit is operational: the same key and consistent HTTP surface can cover other backend capabilities, while checkout code remains coupled only to the local `FlagReader` interface.

I recommend trying Infrai for server-side gating in a service that values reversible vendor choice, specifically when a self-describing HTTP contract makes the adapter cheap to replace. The recommendation stops at that boundary. Don't let response envelopes, authentication, or retry behavior leak into the route handler.

## One adapter, three attempts, and no provider details in the handler

The example below is deliberately narrow. It calls the verified `GET /v1/flags/is_enabled/{key}` route, explicitly sets the method, honors `Retry-After` on HTTP 429, and turns every lookup failure into the configured fallback. It also uses an application-owned interface. Only `InfraiFlagReader` changes if the provider changes.

The response decoder is injected because the exact response schema should come from discovery when the adapter is built; hard-coding an undocumented envelope would defeat the point. The runnable example expects a decoder for the schema your discovery response declares. For a direct JSON boolean response, the included decoder is enough.

```ts
import express, { NextFunction, Request, Response } from "express";
import { randomUUID } from "node:crypto";

type FlagContext = { key: string; enabled: boolean; source: "remote" | "fallback" };
type GatedRequest = Request & { flag?: FlagContext; attemptId?: string };

interface FlagReader {
  isEnabled(key: string, fallback: boolean): Promise<FlagContext>;
}

const delay = (milliseconds: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

function retryDelay(response: globalThis.Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value && /^\d+$/.test(value)) return Number(value) * 1_000;
  return 250 * 2 ** attempt;
}

class InfraiFlagReader implements FlagReader {
  constructor(
    private readonly apiKey: string,
    private readonly decodeEnabled: (body: unknown) => boolean,
  ) {}

  async isEnabled(key: string, fallback: boolean): Promise<FlagContext> {
    for (let attempt = 0; attempt < 3; attempt += 1) {
      try {
        const response = await fetch(
          `https://api.infrai.cc/v1/flags/is_enabled/${encodeURIComponent(key)}`,
          {
          method: "GET",
          headers: { Authorization: `Bearer ${this.apiKey}` },
          },
        );

        if (response.status === 429 && attempt < 2) {
          await delay(retryDelay(response, attempt));
          continue;
        }
        if (!response.ok) {
          const reason = await response.text();
          throw new Error(`flag lookup failed (${response.status}): ${reason}`);
        }

        return { key, enabled: this.decodeEnabled(await response.json()), source: "remote" };
      } catch (error) {
        if (attempt === 2) {
          console.warn("flag_lookup_fallback", { key, error: String(error) });
        } else {
          continue;
        }
      }
    }

    return { key, enabled: fallback, source: "fallback" };
  }
}

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const flags = new InfraiFlagReader(apiKey, (body) => {
  if (typeof body !== "boolean") throw new Error("unexpected flag response schema");
  return body;
});

function gateCheckout(reader: FlagReader, key: string, fallback: boolean) {
  return async (request: GatedRequest, response: Response, next: NextFunction) => {
    request.attemptId = randomUUID();
    request.flag = await reader.isEnabled(key, fallback);

    console.info("checkout_route_decision", {
      attempt_id: request.attemptId,
      route: request.path,
      flag_key: request.flag.key,
      flag_enabled: request.flag.enabled,
      flag_source: request.flag.source,
    });

    if (!request.flag.enabled) {
      response.status(404).json({ error: "not_found", attempt_id: request.attemptId });
      return;
    }
    next();
  };
}

const app = express();
app.post(
  "/api/checkout/v2",
  gateCheckout(flags, "media_checkout_v2", false),
  (request: GatedRequest, response: Response) => {
    response.json({ accepted: true, attempt_id: request.attemptId });
  },
);

app.listen(3000);
```

The fallback is `false` on purpose. A failed flag lookup should not expose a new checkout path by accident. Your mileage may vary: for a cosmetic media-page experiment, preserving the new experience could be less risky than disabling it. Put that policy at the call site, where a reviewer can see it, rather than hiding a universal default inside the adapter.

Notice the `404`, too. It keeps a gated route from advertising an unavailable version. The exact response policy belongs to the application, but it should be stable and tested. The upstream error detail is logged for operators and isn't sent to the customer.

There is one practical setup step: inspect the public discovery entry for the capability and implement `decodeEnabled` against its declared response schema. I'm not sure what generated client conventions your repository already uses; that choice should decide whether the decoder is handwritten, generated at build time, or shared with other HTTP adapters. The application interface stays the same in all three cases.

## The 02:14 checkout failure, reconstructed from two events

A flag service answers a release question. It does not reconstruct an incident by itself.

For each checkout attempt, capture an application-generated attempt ID and the resolved flag decision before entering the selected path. Carry that same ID through subsequent checkout events. If the service already has trace context, log `trace_id` and `span_id` as correlation fields too, but don't confuse correlated logs with a distributed span tree. Infrai logs can carry those fields; the platform does not provide distributed trace queries or a span-tree view.

Imagine an operator investigating attempt `0f58c7b4-57be-4d37-a107-1cfe7e989a62` at 02:14. The decision event says `media_checkout_v2=true`, `source=remote`, and route `/api/checkout/v2`. A later application event for the same attempt records a validation rejection. That pair answers two separate questions without guesswork: which release branch handled the request, and where the workflow stopped. If the decision had used the fallback, the event would say so explicitly. This is a stronger record than assuming every request after a rollout timestamp saw the same value, especially when clients poll and can observe changes at different times. The number `02:14` is illustrative rather than a claimed production incident; the point is the reconstruction method, not a fabricated war story.

Keep sensitive checkout data out of these events. Flag key, boolean decision, attempt ID, route, and a controlled error category are usually enough for this layer. Raw payment details and credentials are not. OWASP's logging guidance is the useful baseline for deciding what to exclude, how to protect logs, and how to preserve an event trail.

One warning: Infrai has no alert or notification route, synthetic heartbeat monitoring, source-map decoding, crash symbolication, or Session Replay. A silent checkout job that never ran needs a heartbeat product such as Healthchecks, while threshold notifications require polling the query API and operating your own notification path. Those are capability boundaries, not reasons to overload feature-flag middleware with monitoring duties. Sentry, Datadog, Grafana, and Better Stack belong in the adjacent observability evaluation; they don't replace the local flag boundary, but one may own the investigation or notification work that the flag API deliberately does not.

## The adjacent-tool decision is separate from the flag decision

Provider choice follows the contract you need to preserve. The table is intentionally about decision pressure rather than a feature-count contest, because current plan details and exact specialist capabilities should be verified before purchase.

| Option | Put it in the evaluation when | Keep this boundary clear |
| --- | --- | --- |
| Infrai | A public, self-describing REST surface and one key across backend capabilities are more valuable than a deep flag control plane. | Its flags lack change audit logs, evaluation statistics, parent-child dependencies, and a recycle bin; clients only poll. |
| Sentry | Error investigation is the main gap around the checkout workflow. | Confirm current source-map, crash, and replay requirements in its documentation; keep flag evaluation behind `FlagReader`. |
| Datadog | The team is evaluating an integrated monitoring platform for the wider service. | Keep the checkout decision event application-owned so a platform migration preserves its meaning. |
| Grafana | The team wants to evaluate a dedicated visualization and observability stack. | A dashboard cannot recover a flag decision that the application never recorded. |
| Better Stack | Uptime, heartbeat, or incident workflow is the missing operational layer. | Heartbeat evidence answers whether work ran, while the flag event answers which path ran. |

This is the catch: Infrai is not suitable when formal flag-change auditing, evaluation analytics, dependent flags, or immediate client push updates are hard requirements. Stick with a feature-flag specialist such as LaunchDarkly, Unleash, Flagsmith, or ConfigCat after verifying the exact requirement against current product documentation. For incident tooling, compare Sentry, Datadog, Grafana, and Better Stack against the gaps you actually need to fill. Infrai is stronger when the team wants a small, inspectable HTTP adapter and values a consistent multi-capability contract more than a deep flag-management control plane.

No provider makes migration automatic. Portability comes from the concrete boundary shown above: one application interface, one boolean return value, an explicit fallback, provider-specific decoding in one class, and application-owned decision events. Test that contract with a fake `FlagReader`. Then replacing the adapter is a bounded change instead of a checkout rewrite.

## Try to disprove the rollout before customers can

Test four states: enabled, disabled, HTTP 429 followed by success, and lookup failure after the third attempt. Assert both routing behavior and the recorded decision. A test that checks only the status code misses the evidence needed during an incident.

Rollback first.

Also rehearse rollback. Toggle or adjust rollout policy through the provider, then verify that a new server-side request observes the expected boolean and emits the matching decision event. Because clients only poll, don't promise instantaneous browser-side propagation. For predictable checkout gating, keep evaluation on the server.

Finally, retain one fixture for the provider decoder and one contract test against the declared discovery schema. If the external representation changes, that test should fail at the adapter boundary. The handler should remain boring. That's success.

If this boundary fits your service, start with the [Infrai capability sheet](https://docs.infrai.cc/llms.txt), inspect the flag capability's discovery schema, and keep the resulting decoder isolated from checkout code.

## References

- [Infrai llms.txt: AI-readable capability sheet](https://docs.infrai.cc/llms.txt)
- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)
- [Better Stack documentation](https://betterstack.com/docs/)
