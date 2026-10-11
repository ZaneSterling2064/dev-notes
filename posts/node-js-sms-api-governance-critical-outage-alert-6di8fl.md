# Node.js SMS API Governance — Critical Outage Alert Delivery Without Stale Resends

Short answer: choose the least complex SMS service that lets your Node.js backend submit a password-reset message, poll its delivery state, and cancel stale work. Keep the short expiry, retry budget, and escalation policy in your application. For this job, a controllable message lifetime matters more than a long feature list.

| Option | Pick this when | Boundary to design around |
| --- | --- | --- |
| Twilio Programmable Messaging | You want a messaging-focused API with a documented message status model | Your app still owns token expiry and retry policy |
| Vonage SMS API | Its SMS workflow and regional operating model match your existing stack | Normalize delivery receipts before they reach product logic |
| Amazon SNS | SMS belongs inside an AWS notification estate already governed through IAM | Maintain a reset-attempt record and expiry state machine |
| Infrai | A self-describing REST contract and pull-based status checks fit your worker model | No webhook pushes; polling controls detection time |

The table is the decision in miniature. Draw the system in words: a support agent triggers a reset, the application creates one expiring attempt, an SMS adapter submits it, a worker polls delivery state, and an expiry guard shuts the door. Every arrow carries an attempt ID. The token and message body stay out of logs.

## How should a Node.js app choose an SMS API for critical outage alerts?

Start with expiry, not the vendor logo. A reset link that has timed out is no longer useful, even if the carrier eventually delivers it. Define an application deadline, persist it beside the attempt, and require every poll, resend, and fallback decision to check that deadline first.

Short means short.

Twilio is a serious choice when a team wants a dedicated messaging product and can map its documented Message resource states into a small internal vocabulary. Vonage deserves the same evaluation when its SMS API, delivery-receipt behavior, and your existing vendor relationship fit the deployment. In both cases, provider states should stop at the adapter boundary; customer-support logic should see stable states such as `pending`, `delivered`, `failed`, `expired`, and `suppressed`.

Amazon SNS makes more sense when AWS operations are already the control plane. IAM and the surrounding notification model can reduce organizational friction, but SNS does not remove the need for a password-reset ledger. The product still needs to know which token, attempt, and deadline belong together.

Infrai fits a different operating preference. Its public discovery surface requires no key and returns a capability's request JSON Schema, response schema, billing information, and runnable examples. That makes adapter work a matter of reading the live contract rather than learning another SDK. Every documented capability has runnable examples in 10 languages. The same key spans 295 routes across 20 modules, reducing credential and invoice sprawl if the support backend later uses other capabilities.

Those are concrete workflow advantages, but they do not erase the pull model. SMS state and events are polled, not pushed by webhook. Choose a provider with callbacks when immediate event-driven escalation is mandatory. A polling service is reasonable when bounded observation delay is acceptable and you already operate scheduled workers.

The limitation is explicit: Infrai is not suitable when the escalation design requires webhook pushes. In that case, choose Twilio or Vonage after verifying that the documented callback behavior meets the application's timing and regional requirements.

The same test applies to critical outage alerts for a restaurant waitlist app: if staff must see delivery changes immediately, prefer callbacks; if the application can poll within its escalation window, a pull API can cover the core mechanics. The password-reset policy remains distinct because its expiry protects an account rather than restaurant operations.

## Why must expiry outrank delivery state?

A useful state machine has two clocks. The provider clock answers, "What happened to the SMS?" The application clock answers, "Can this reset still help the user?" The second clock wins.

Expiry wins.

Picture the transitions. `created` moves to `submitted`; polling may move the attempt to `delivered` or `failed`; an approved resend creates another transport attempt under the same reset flow; reaching the deadline moves unfinished work to `expired`; resolving or replacing the reset moves queued work to `suppressed`. An accepted request is not a delivered message. A delivered message is not a completed password reset.

That distinction shapes observability. Record the reset-flow ID, transport-attempt ID, destination country, provider, provider message ID, deadline, normalized state, attempt count, and next-poll time. Do not record the reset token, phone number, or message body. Emit a structured event only for submission, state transition, expiry, suppression, or completion.

Watch two funnels. The transport funnel runs from accepted to delivered. The product funnel runs from reset requested to reset completed. A healthy first funnel beside a collapsing second one points away from carrier delivery and toward the link, form, or expiry policy. Also alert on poll lag. If `now - nextPollAt` keeps growing, your delivery view is stale even when the provider is fine.

## Build one narrow Node.js status adapter

The adapter below polls one verified status route. It returns `unknown` because the response fields belong to the live discovery contract and should not be guessed here. Set `INFRAI_API_KEY`, `INFRAI_BASE_URL`, and `SMS_ID`; Node.js 18 or later supplies `fetch`. Set the base URL to the service's documented v1 API origin.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;
const smsId = process.env.SMS_ID;

if (!apiKey || !baseUrl || !smsId) {
  throw new Error("Set INFRAI_API_KEY, INFRAI_BASE_URL, and SMS_ID");
}

const sleep = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

function backoffMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) {
    return Number(retryAfter) * 1_000;
  }
  return Math.min(30_000, 1_000 * 2 ** attempt);
}

async function readStatus(id: string): Promise<unknown> {
  const url = `${baseUrl.replace(/\/$/, "")}/sms/status/${encodeURIComponent(id)}`;

  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 4) {
      await sleep(backoffMs(response, attempt));
      continue;
    }

    const raw = await response.text();
    if (!response.ok) {
      throw new Error(`Status lookup failed (${response.status}): ${raw}`);
    }
    return raw.length > 0 ? JSON.parse(raw) : null;
  }

  throw new Error("Status lookup exhausted its retry budget");
}

readStatus(smsId).then((status) => {
  process.stdout.write(`${JSON.stringify(status)}\n`);
});
```

Five lookup attempts are a ceiling. They are not five message sends.

Count them separately.

The write side needs a separate rule: retry an ambiguous transport failure with the same idempotency key, but treat a product resend as a new, policy-approved attempt. Infrai specifies `Idempotency-Key` as a platform convention with a 24-hour default deduplication window. Before any resend, re-read the reset record and refuse the operation if the flow is complete, replaced, suppressed, or expired. This separation prevents a network retry from becoming two valid reset messages.

Read the live `sms.send` discovery contract before implementing its serializer. Generate the request path from the discovery path field, not description prose.

## What guardrails belong before fallback channels?

US and EU destinations should pass through explicit business rules. Maintain country allowlists, applicable sender and template requirements, and per-country volume or cost circuit breakers in your own service. Geographic anti-abuse fencing and country-priced circuit breakers are not supplied for this workflow. Reject disallowed destinations before making a provider call, then alert on unusual request volume by country.

SMS cancellation is useful for stale work after a reset is completed or replaced. Treat cancel as suppression, not proof that a carrier can recall a message already delivered. This platform also has email sending, but no managed email OTP interface; an email fallback therefore needs your own token flow. Scheduled email has no cancellation operation.

Channel breadth has limits. There is no SMTP relay, voice, WhatsApp, or RCS channel here. If support policy requires one of those channels, integrate a provider that explicitly offers it and keep the same application expiry check in front of the adapter. The domestic Chinese email vendor is pending, so it cannot support a domestic-compliance decision.

Keep finance telemetry local too. There is no cost-report aggregation by tag, and SMS templates do not have a list operation. Persist the identifiers and dimensions that support and finance will need rather than assuming they can be reconstructed later.

## Limits and the final pick

Polling trades immediacy for a simple, auditable worker. Faster polling updates knowledge sooner, but it does not strengthen the carrier's delivery guarantee. Set the interval from the shortest useful reaction time, cap concurrent checks, honor rate limits, and measure queue lag.

Pick Twilio or Vonage when their messaging-specific status and callback model best matches the escalation requirement. Pick Amazon SNS when AWS governance is decisive. Consider Infrai when public schema discovery, runnable TypeScript examples, and one credential across a broad backend surface remove meaningful integration work, and when pull-based status observation is acceptable.

Then enforce the same rule everywhere: no retry, resend, escalation, or fallback may outlive the reset. The provider transports the message. Your Node.js state machine protects the account.

## Further reading

## References

- [Twilio: Message resource](https://www.twilio.com/docs/messaging/api/message-resource)
- [Vonage: SMS API overview](https://developer.vonage.com/en/messaging/sms/overview)
- [Amazon SNS: Sending SMS messages](https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html)
- [OWASP: Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
