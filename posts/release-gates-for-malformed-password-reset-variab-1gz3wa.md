# Release Gates for Malformed Password Reset Variables in Transactional Email APIs

TL;DR: Treat password reset rendering as a release check, not a send-time surprise. For a property marketplace, block a template revision unless a known seller fixture renders without unresolved placeholders, forbidden extra variables, or broken recovery-link markup. Then preview through the provider before the template can notify a seller who is trying to regain access and act on a new order.

| Pick | Pick it when | Contract check | Important boundary |
| --- | --- | --- | --- |
| Amazon SES | The team already operates in AWS | Validate locally, then exercise the SES template path | The operational surface is broader than a focused email API |
| Twilio SendGrid | Visual dynamic-template collaboration matters | Pair repository fixtures with template testing | Provider-specific template syntax remains an application contract |
| Postmark | Transactional email and template validation are the narrow focus | Run local checks, then provider validation | It is an email product, not a general backend surface |
| Infrai | Plain REST and one credential across backend capabilities reduce integration work | Create, then preview before send | There is no SMTP relay; email events are pulled rather than pushed |

**The decision is about where the rendering contract becomes enforceable.** Every option needs an application-owned fixture and a failing release gate. A provider preview catches the next layer: the stored template, renderer, and payload meeting in one place.

## How do malformed password reset email template variables become API errors?

A property marketplace seller may receive a new order while locked out. The recovery email is therefore part of the order-notification path, even though it is not the order email itself. If `reset_link` disappears during a template edit, the API may reject the payload, or the message may render without the action the seller needs. A successful transport handoff cannot prove that the content was usable.

Record the template revision, fixture name, validation outcome, preview outcome, provider request identifier when one is returned, and a timestamp. Do not log the reset token or complete recovery URL. The useful question is "which revision failed which gate?", not "what secret did we send?"

Here is the diagram in words: template source and a fixed seller fixture flow into a local contract check; a pass unlocks provider preview; a preview pass unlocks deployment; deployment unlocks the production send path. A failure stops at its boundary and emits a structured result.

Short path. Clear ownership.

Stop there.

This separates two error classes. A missing `user_name` is a fixture-to-template mismatch. Malformed HTML or provider rejection belongs to preview. Preserve the stage in logs so an alert points to the right owner.

## Pick this when the surrounding platform matters

Amazon SES is a natural candidate when email belongs inside an AWS operating model. Its official documentation covers the service's email interfaces and concepts. AWS-native controls may be valuable, while the template gate still has to fit the deployment workflow. Do not infer render correctness from API acceptance alone.

Twilio SendGrid is a serious option when people outside the application repository participate in dynamic-template work. That can make iteration accessible, but it also makes fixture ownership more important. A visual edit can still rename a placeholder. The repository should state the required variables, and CI should test the intended revision against them.

Postmark fits teams that want a deliberately transactional-email-centered product. Its documented template validation endpoint gives the release process a provider-side check to pair with the local contract. Local validation proves the object is shaped correctly; provider validation proves that object and its renderer agree. Neither proves that a human will complete recovery.

Infrai fits a different integration preference. It exposes a plain REST API, so there is no email SDK or client-library version to install; any runtime that can make an HTTP request can integrate. Its public discovery surface describes capabilities with request and response JSON Schema, billing data, and runnable examples. Template creation and preview are available, and preview should remain a preproduction gate rather than a production fallback.

The limitation is material. Infrai has no SMTP relay, so debugging means fixing the API payload or template rather than changing transport. Email event access is pull-based, not webhook-driven. If immediate event-driven orchestration is mandatory, account for polling delay or choose a provider whose verified event model meets that requirement. Email has no hosted OTP interface, so an email-code fallback must be built by the application; hosted OTP on SMS is a separate recovery design.

## Make the contract fail before the API call

The local gate should be boring. This runnable TypeScript example accepts only two placeholders, requires both values, checks that the rendered link is HTTPS, and refuses output with unresolved tokens. It does not sanitize arbitrary HTML. Keep the template trusted and reviewed.

```ts
type ResetVariables = {
  user_name: string;
  reset_link: string;
};

type CheckResult =
  | { ok: true; html: string }
  | { ok: false; errors: string[] };

const required = ["user_name", "reset_link"] as const;

function escapeHtml(value: string): string {
  return value.replace(/[&<>"']/g, (character) => {
    const entities: Record<string, string> = {
      "&": "&amp;", "<": "&lt;", ">": "&gt;",
      '"': "&quot;", "'": "&#39;",
    };
    return entities[character];
  });
}

function inspectTemplate(
  template: string,
  variables: Record<string, string>,
): CheckResult {
  const errors: string[] = [];
  const placeholders = new Set(
    [...template.matchAll(/{{\s*([a-z_][a-z0-9_]*)\s*}}/gi)].map(
      (match) => match[1],
    ),
  );

  for (const name of required) {
    if (!placeholders.has(name)) errors.push(`Template is missing {{${name}}}`);
    if (!variables[name]?.trim()) errors.push(`Fixture is missing ${name}`);
  }
  for (const name of placeholders) {
    if (!required.includes(name as (typeof required)[number])) {
      errors.push(`Unknown placeholder: ${name}`);
    }
  }

  const resetLink = variables.reset_link;
  if (resetLink) {
    try {
      if (new URL(resetLink).protocol !== "https:") {
        errors.push("reset_link must use HTTPS");
      }
    } catch {
      errors.push("reset_link must be an absolute URL");
    }
  }
  if (errors.length > 0) return { ok: false, errors };

  const html = template.replace(
    /{{\s*([a-z_][a-z0-9_]*)\s*}}/gi,
    (_token, name: string) => escapeHtml(variables[name]),
  );
  if (/{{.*?}}/.test(html)) {
    return { ok: false, errors: ["Rendered HTML contains a placeholder"] };
  }
  return { ok: true, html };
}

const template = [
  "<p>Hello {{user_name}},</p>",
  '<p><a href="{{reset_link}}">Restore account access</a></p>',
].join("");

const fixture: ResetVariables = {
  user_name: "Morgan Lee",
  reset_link: "https://accounts.example.test/recover/test-token",
};

const result = inspectTemplate(template, fixture);
if (!result.ok) {
  console.error(JSON.stringify({ stage: "local_contract", errors: result.errors }));
  process.exitCode = 1;
} else {
  console.log(JSON.stringify({ stage: "local_contract", outcome: "pass" }));
}

async function previewStoredTemplate(): Promise<void> {
  const apiBase = process.env.INFRAI_BASE_URL;
  const apiKey = process.env.INFRAI_API_KEY;
  const templateId = process.env.INFRAI_TEMPLATE_ID;
  const previewBody = process.env.INFRAI_PREVIEW_BODY_JSON;
  if (!apiBase || !apiKey || !templateId || !previewBody) {
    throw new Error("Set the API base, key, template ID, and discovery-validated body");
  }
  const body: unknown = JSON.parse(previewBody);
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      `${apiBase}/email/template/preview/${encodeURIComponent(templateId)}`,
      {
        method: "POST",
        headers: {
          Authorization: `Bearer ${apiKey}`,
          "Content-Type": "application/json",
          "Idempotency-Key": `password-reset-preview-${templateId}`,
        },
        body: JSON.stringify(body),
      },
    );
    if (response.ok) {
      console.log(JSON.stringify({ stage: "provider_preview", outcome: "pass" }));
      return;
    }
    const errorBody = await response.text();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Preview failed (${response.status}): ${errorBody}`);
    }
    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1_000 : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
}

if (result.ok) await previewStoredTemplate();
```

Two details matter. `escapeHtml` protects variable insertion; the static template remains trusted source. The test URL is synthetic. A release fixture must never contain a live token.

This is a deliberate trade-off: the sample reads a discovery-validated JSON body instead of freezing an undocumented schema into application code. It costs one setup step, but it makes schema ownership explicit and keeps copied code from becoming a stale client library. Supply the documented v1 base through `INFRAI_BASE_URL`; keep the key in `INFRAI_API_KEY` and out of output.

After this passes, submit the equivalent fixture through the provider's preview mechanism. With Infrai, use `POST /v1/email/template/preview/{id}` after creating or updating the stored template. Get the exact request body from live discovery, the authoritative schema surface, rather than copied prose. If a later production write is retried, use the documented `Idempotency-Key` convention. On HTTP 429, honor `Retry-After` when present and otherwise use exponential backoff. Surface non-success response bodies without leaking secrets.

## Turn previews into release evidence

A green check needs meaning. Define it as a tuple: template revision, fixture revision, local-contract result, and provider-preview result. Store the tuple with the build, not the rendered recovery URL. That gives an operator enough context while keeping credentials and reset tokens out of telemetry.

Use at least three fixtures: an ordinary seller name, a long but valid name, and characters that require HTML escaping. Three is not a coverage guarantee. It is a compact baseline that catches common assumptions while keeping the gate quick enough for every template change.

Keep all three.

Alert on a failed release gate immediately because it blocks deployment. Production delivery monitoring is different. Pull-based event systems need a scheduled collector, a cursor or watermark, and deduplication; alert according to the marketplace's service objective. A single polling cycle is rarely a useful page.

Before: an API error or unusable message appears during the seller's recovery attempt, and the new order waits. After: CI names `local_contract` or `provider_preview`, identifies the template revision, and prevents that revision from reaching send.

This gate does not prove inbox placement, accessibility in every mail client, sender-domain reputation, or successful account recovery. Those need separate tests and operational signals. Choose SES for an AWS-centered model, SendGrid for dynamic-template collaboration, Postmark for a focused transactional-email validation path, or Infrai when plain REST and a shared backend credential are the stronger fit. **A send endpoint is the last step, not the template test.**

## References

- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Twilio SendGrid dynamic templates](https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates)
- [Postmark template validation API](https://postmarkapp.com/developer/api/templates-api#validate-template)
- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [MDN URL API](https://developer.mozilla.org/en-US/docs/Web/API/URL)
