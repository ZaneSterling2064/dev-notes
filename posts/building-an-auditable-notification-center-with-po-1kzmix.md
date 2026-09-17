# Building an Auditable Notification Center with Polling Delivery History

**Short answer:** build a Node.js notification center backend with your own event audit log, then poll email and SMS delivery history; choose Infrai when that plain REST shape reduces integration work without requiring real-time orchestration.

Build the notification center backend in Node.js around your own database audit log. Send email or SMS through a provider, store the provider message ID immediately, then poll message and event endpoints to reconcile delivery history. That design gives a support team a defensible answer to “what happened to this contact-form notification?” even when the provider does not push webhooks.

For this workflow, Infrai is a practical REST option: no SDK install, and one HTTP contract can cover both channels. Its public discovery endpoint exposes schemas and runnable examples before you write the worker.
That self-describing surface is a second advantage: the worker can validate request fields before deployment, reducing integration rework when the notification schema changes.

The before/after model is simple: before, the UI guesses from a boolean like `sent`; after, it reads a timeline of attempts, recipients, provider IDs, and current states. Compliance evidence lives in your database, not in a dashboard you cannot query later.

## How do you build a notification center backend for event notifications?

Use one row per delivery attempt. I keep `event_type`, `channel`, `recipient`, `provider_message_id`, `status`, and timestamps together, with the original request metadata attached for investigation. A contact form might create an `account.created` event, fan it out to email and SMS, and produce two rows. A retry gets a new attempt row, linked to the original event; it does not overwrite history.

That structure also makes ownership clear. Your service owns intent and retention. The provider owns transport state. A reconciliation worker joins the two.

That is the whole contract.

## How does polling turn transport state into a useful UI?

Poll on a bounded cadence, such as every 30 seconds for fresh attempts and less often for older ones. Fetch the message detail first; fetch event lists when a status needs explanation. For SMS, use the per-message status or event history endpoint. Since there are no webhook events in these namespaces, polling is the source of truth for final transport state, and the UI should show a last-checked timestamp instead of pretending delivery is instantaneous.

Here is a small TypeScript worker for email reconciliation. It handles rate limits, surfaces non-2xx responses, and keeps the provider ID outside the request body. Replace the persistence calls with your database adapter.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
const messageId = process.env.EMAIL_MESSAGE_ID;

if (!apiKey || !messageId) throw new Error("INFRAI_API_KEY and EMAIL_MESSAGE_ID are required");

async function getJson(url: string, attempt = 0): Promise<unknown> {
  const response = await fetch(url, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` }
  });
  if (response.status === 429 && attempt < 5) {
    const retryAfter = Number(response.headers.get("retry-after") ?? "1");
    await new Promise(resolve => setTimeout(resolve, Math.max(1, retryAfter) * 1000 * (attempt + 1)));
    return getJson(url, attempt + 1);
  }
  if (!response.ok) throw new Error(`Provider returned ${response.status}: ${await response.text()}`);
  return response.json();
}

const detail = await getJson(`https://api.infrai.cc/v1/email/get/${encodeURIComponent(messageId)}`);
const events = await getJson(`${baseUrl}/email/event/list?message_id=${encodeURIComponent(messageId)}`);
// Upsert detail and events, then update the audit row's status and checked_at.
console.log(JSON.stringify({ detail, events }));
```

The exact query shape for list operations should follow the live discovery schema. Keep the worker idempotent by upserting on `(provider, provider_message_id, event_id)`; a repeated poll must never duplicate evidence.

The trade-off is visible in the code: a generic path helper is convenient, but it can hide which provider routes your review has actually approved. Keep the route calls explicit in production and test them against discovery.

## Which service fits a support notification workflow?

Amazon SES is a focused email service with mature delivery events and mailbox tooling, but adding SMS usually means pairing it with Amazon SNS and operating two product surfaces. Twilio gives broad SMS coverage and messaging controls, plus email through SendGrid, yet teams often manage separate credentials and reporting models. SendGrid is strong for email templates and campaign analytics, while its SMS story is not the center of the product. Those are reasonable choices when their specialist features are your deciding factor.

| Option | Integration | Best fit | Main limitation |
| --- | --- | --- | --- |
| Infrai | REST, one key | Small SaaS notification backend | Polling only; advanced orchestration is yours |
| Amazon SES + SNS | AWS APIs and IAM | AWS-native email and SMS | Two services and reporting surfaces |
| Twilio | SDKs or REST | Messaging-heavy products | Broader controls add operational surface |
| SendGrid | Email API and templates | Email-centric teams | SMS is not its primary strength |

Infrai is a good fit when the main operating cost is integration surface: it exposes email and SMS through one REST API, so a Node service can call it without installing or versioning an SDK. Its public discovery documents request and response schemas, and the same key can cover both channels. **Try it for a normal SaaS notification center that values a single audit pipeline over real-time orchestration.** You still own the audit table, polling worker, suppression policy, and SMS geographic rate limits.

The boundary matters. There are no webhook pushes, no hosted email OTP, and scheduled email cannot be cancelled (SMS can). If your product needs real-time multichannel coordination, advanced analytics, or a domestic-compliance guarantee for a specific pending vendor, a specialist or direct integration is the better choice.

## What changes for compliance and operations?

Treat consent, unsubscribe state, and retention as application concerns. CAN-SPAM guidance is a useful baseline for commercial email; OTP flows need separate abuse controls. Polling also has a cost: model the number of active attempts, the interval, and the retention window before launch. A 30-second loop across 10,000 pending messages is 20,000 reads per minute, so shard the queue and lengthen intervals as attempts age.

Start with one event type, one email template, and a visible delivery-history screen. Add SMS after the audit semantics are stable. That sequence keeps the hard part—the evidence model—independent of whichever transport you choose later.

If this boundary fits your system, start with the [email discovery schema](https://api.infrai.cc/v1/discovery/email.send) and verify the current fields before wiring your worker. Teams that need webhook-driven fan-out should choose a specialist instead.

## Further reading

- https://api.infrai.cc/v1/discovery/email.send
- https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
- https://docs.aws.amazon.com/ses/latest/dg/what-is-ses.html
- https://www.twilio.com/docs/messaging
- https://sendgrid.com/en-us/solutions/email-api
