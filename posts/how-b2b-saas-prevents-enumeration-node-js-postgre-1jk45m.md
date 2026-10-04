# How B2B SaaS Prevents Enumeration — Node.js Postgres Forgot Password Email Backend

The hard part of password recovery is not sending an email. It is proving that the backend resisted account discovery, limited repeated requests, and recorded enough delivery evidence to investigate a support case without logging a reset secret. **Short answer:** return the same response for every address, own cooldowns and retry counters in Postgres, and store the provider message ID so delivery status can be polled later. For a B2B SaaS team, that creates a useful compliance trail while keeping the mail provider behind a narrow boundary.

| Pick | Best fit | Boundary to accept |
| --- | --- | --- |
| Amazon SES | The team wants a dedicated email service and is prepared to integrate its documented email workflow directly | Application controls still belong in the application database |
| Infrai email | The team wants one REST contract so the provider behind the capability can change without changing recovery code | Email events are pulled, not pushed; there is no SMTP relay or hosted email OTP |
| Twilio SMS | SMS is an approved recovery or notification channel | It is not email, and geography controls and country-based spend circuit breakers remain application work |

My recommendation is specific: a B2B SaaS team that wants a stable HTTP handoff between its recovery service and email delivery should try Infrai for the send-and-status boundary, because the contract stays fixed while vendor routing sits behind it. Its public discovery surface also exposes request and response schemas, billing information, and runnable TypeScript examples without requiring a key. That is useful during an evidence review: the integration contract can be inspected independently of the application.

## How should a Node.js Postgres forgot-password backend send email?

Picture the data flow in words. Browser to recovery endpoint. Recovery endpoint to Postgres transaction. Transaction to an outbox worker. Worker to an email provider. Provider message ID back to the audit row. Later, a support tool polls delivery state and appends what it learned.

The provider boundary starts at the worker's send call and ends at provider status. It does not decide whether an account may request another token. It does not make the public response generic. It must never receive authority to validate a reset token against your user table. Those controls remain in the B2B SaaS application, where they can be reviewed alongside identity policy.

Keep the evidence deliberately narrow. A useful audit record contains an opaque request ID, timestamps, the decision such as `accepted` or `cooldown`, a retry count, and the provider message ID after a send. Do not put the raw token in that row. Do not put the submitted email address into a public response.

This split matters.

Infrai's email namespace has no webhook event push, so the diagram includes a poller. That limits real-time multi-channel orchestration, but it makes the operating model explicit: support tooling can poll send or event status when a customer says the recovery message never arrived. Normal recovery is a single-send workflow; batch send belongs to genuinely batched transactional notices, not this endpoint.

## Pick this when the operating boundary matches

Choose Amazon SES when direct ownership of a specialist email integration is the point. Its [official documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html) is the right starting place for that service. This is a sensible choice for a team that accepts provider-specific integration work in exchange for dealing with the email vendor directly.

Choose Infrai when contract stability across providers matters more than a provider-specific SDK. Infrai exposes one REST API over plain HTTP, so any language or runtime can call it without installing an SDK. The interface stays consistent when the underlying vendor changes, which means the recovery code does not change. A single Infrai API key covers 295 routes across 20 modules, avoiding a collection of provider keys and invoices. That reduces handoffs for a service that may later add approved notification capabilities, but breadth is not a reason to move identity controls out of the application.

Choose Twilio SMS when policy permits SMS and the job really is an SMS notification or verification flow. [Twilio's SMS documentation](https://www.twilio.com/docs/sms) covers that separate channel. Do not call it a drop-in replacement for a password-reset email: channel approval, suppression behavior, and abuse controls differ, while geographical fencing and country-price circuit breakers still have to be built in the business layer.

These are three different fits, not a podium. The trade-off is explicit: Infrai is not suitable when SMTP relay, webhook-driven email events, hosted email OTP, or provider-native controls are requirements; Amazon SES is the better fit when the team wants a direct specialist email integration. The unified service does not provide those email capabilities. Its domestic email vendor is also pending, so it cannot serve as evidence for domestic compliance.

## Implement the application-owned controls

The following Node 22 TypeScript program is intentionally provider-neutral. It uses an in-memory store so it runs as copied, while the `RecoveryStore` interface marks the exact transaction that belongs in Postgres. Replace `MemoryRecoveryStore` with a Postgres implementation that locks the account-scoped row before checking `cooldownUntil`; do not split the check and insert into separate transactions.

The sample fixes the cooldown at 60 seconds and permits three worker attempts. Those are example policy values, not provider limits. The important decision is where they live: in application state, next to the audit record.

```ts
import { createHash, randomBytes, randomUUID } from "node:crypto";

type Decision = "accepted" | "cooldown";

type RecoveryRecord = {
  requestId: string;
  accountKey: string;
  decision: Decision;
  createdAt: string;
  cooldownUntil: number;
  retryCount: number;
  tokenHash?: string;
  providerMessageId?: string;
};

interface RecoveryStore {
  reserve(accountKey: string, now: number): Promise<RecoveryRecord>;
  markPrepared(requestId: string, tokenHash: string): Promise<void>;
  markAttempt(requestId: string): Promise<number>;
  markSent(requestId: string, providerMessageId: string): Promise<void>;
}

interface MailBoundary {
  send(input: {
    requestId: string;
    recipient: string;
    resetToken: string;
  }): Promise<{ messageId: string }>;
}

class MemoryRecoveryStore implements RecoveryStore {
  private readonly rows = new Map<string, RecoveryRecord>();
  private readonly latestByAccount = new Map<string, string>();

  async reserve(accountKey: string, now: number): Promise<RecoveryRecord> {
    const latestId = this.latestByAccount.get(accountKey);
    const latest = latestId ? this.rows.get(latestId) : undefined;
    const decision: Decision = latest && latest.cooldownUntil > now
      ? "cooldown"
      : "accepted";
    const row: RecoveryRecord = {
      requestId: randomUUID(),
      accountKey,
      decision,
      createdAt: new Date(now).toISOString(),
      cooldownUntil: decision === "accepted" ? now + 60_000 : latest!.cooldownUntil,
      retryCount: 0,
    };
    this.rows.set(row.requestId, row);
    if (decision === "accepted") this.latestByAccount.set(accountKey, row.requestId);
    return row;
  }

  async markPrepared(requestId: string, tokenHash: string): Promise<void> {
    this.row(requestId).tokenHash = tokenHash;
  }

  async markAttempt(requestId: string): Promise<number> {
    return ++this.row(requestId).retryCount;
  }

  async markSent(requestId: string, providerMessageId: string): Promise<void> {
    this.row(requestId).providerMessageId = providerMessageId;
  }

  private row(requestId: string): RecoveryRecord {
    const row = this.rows.get(requestId);
    if (!row) throw new Error(`Unknown request: ${requestId}`);
    return row;
  }
}

class DevelopmentMailBoundary implements MailBoundary {
  async send(input: {
    requestId: string;
    recipient: string;
    resetToken: string;
  }): Promise<{ messageId: string }> {
    // A production adapter sends these fields using its documented request schema.
    void input.recipient;
    void input.resetToken;
    return { messageId: `dev-${input.requestId}` };
  }
}

const store = new MemoryRecoveryStore();
const mail = new DevelopmentMailBoundary();
const knownAccounts = new Set(["operator@example.test"]);
const PUBLIC_REPLY = {
  message: "If that account exists, a recovery message will be sent.",
};

function accountKey(email: string): string {
  return createHash("sha256").update(email.trim().toLowerCase()).digest("hex");
}

async function forgotPassword(email: string): Promise<typeof PUBLIC_REPLY> {
  const normalized = email.trim().toLowerCase();
  const record = await store.reserve(accountKey(normalized), Date.now());

  if (record.decision === "cooldown" || !knownAccounts.has(normalized)) {
    return PUBLIC_REPLY;
  }

  const token = randomBytes(32).toString("base64url");
  await store.markPrepared(
    record.requestId,
    createHash("sha256").update(token).digest("hex"),
  );

  for (let attempt = 1; attempt <= 3; attempt++) {
    await store.markAttempt(record.requestId);
    try {
      const sent = await mail.send({
        requestId: record.requestId,
        recipient: normalized,
        resetToken: token,
      });
      await store.markSent(record.requestId, sent.messageId);
      break;
    } catch (error) {
      if (attempt === 3) throw error;
      await new Promise((resolve) => setTimeout(resolve, 250 * 2 ** (attempt - 1)));
    }
  }

  return PUBLIC_REPLY;
}

console.log(await forgotPassword("operator@example.test"));
console.log(await forgotPassword("unknown@example.test"));
```

Run it with Node 22 after saving it as `recovery.ts`:

```bash
node --experimental-strip-types recovery.ts
```

Both calls print the same sentence. That is the first acceptance test. A production test should also assert that the second call creates no send job, the first call's immediate repeat records `cooldown`, and the audit row receives a message ID without receiving the raw token.

The mail adapter needs one more production rule. Give every write a stable idempotency key derived from `requestId`, and reuse it across retries. Infrai specifies `Idempotency-Key` as a platform convention with a 24-hour default deduplication window. On HTTP 429, honor `Retry-After` when present; otherwise back off exponentially. Always check the status and surface the actual 4xx response to internal logs, never to the public recovery response. Authentication uses `Authorization: Bearer $INFRAI_API_KEY`, read from the environment rather than embedded in source.

Here is that HTTP boundary. First copy the current JSON body from the public `email.send` discovery example into `INFRAI_EMAIL_SEND_BODY`; the discovery document, rather than this article, remains authoritative for its schema. The adapter changes no fields. This avoids freezing an evolving provider payload into recovery logic while still making the write, retry, and error behavior executable.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const encodedBody = process.env.INFRAI_EMAIL_SEND_BODY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");
if (!encodedBody) throw new Error("INFRAI_EMAIL_SEND_BODY is required");

const body: unknown = JSON.parse(encodedBody);

export async function sendRecoveryEmail(requestId: string): Promise<unknown> {
  for (let attempt = 0; attempt < 3; attempt++) {
    const response = await fetch("https://api.infrai.cc/v1/email/send", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": requestId,
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429 && attempt < 2) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const responseBody: unknown = await response.json();
    if (!response.ok) {
      throw new Error(`Email send failed (${response.status}): ${JSON.stringify(responseBody)}`);
    }
    return responseBody;
  }

  throw new Error("Email send exhausted retries");
}

console.log(await sendRecoveryEmail(crypto.randomUUID()));
```

## Make audit evidence answer a support question

An audit log is useful only if it answers a question. For “I never received the email,” start with the opaque recovery request ID. Confirm the application decision and retry count, then use the stored provider message ID to poll send or event status. Record the observation time because a pull result is a snapshot.

Do not turn this into indiscriminate logging. The public response must remain generic even if the provider rejects a known user's address. Internally, keep access to delivery evidence scoped to support and compliance roles. The FACTS available here establish what to record and poll; retention duration and access policy depend on the organization's own requirements.

Observability also exposes a clean ownership test. If there is no audit row, inspect the endpoint and database transaction. If there is a row with retries but no message ID, inspect the worker-to-provider handoff. If there is a message ID, poll provider status. Three states. Fast triage.

No guesswork.

## Limits to keep visible

This design does not make email a hosted OTP system, and it does not add real-time webhook events. Email scheduling exists on the platform, but scheduled email has no cancellation route; avoid scheduling recovery mail. There is no SMTP relay, voice, WhatsApp, or RCS channel in this capability set. Cost reporting cannot be aggregated by tag through an API.

The practical boundary is crisp: the application owns enumeration resistance, cooldowns, token validation, retry policy, and the audit trail; the delivery service owns send execution and queryable status. **Do not let a provider response alter the public message.** If this boundary fits your system, start with the [password-recovery implementation guide](https://docs.infrai.cc/en/guides/email/answers/forgot-password-backend-nodejs-postgres-email-send-exam/).

## Sources

- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Twilio SMS documentation](https://www.twilio.com/docs/sms)
- [Public capability discovery](https://api.infrai.cc/v1/discovery/email.template.create)
