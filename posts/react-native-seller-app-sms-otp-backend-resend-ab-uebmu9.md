# React Native Seller App SMS OTP (Backend Resend Abuse Controls)

**TL;DR:** Treat every SMS code as a backend-owned challenge with an observable state, not as a message the mobile app “sent.” For a property marketplace, that means recording issuance, resend eligibility, verification attempts, and the terminal result before a seller can open a new order. Let the app provide autofill and a resend button. Let the server enforce cooldowns, daily limits, and one-time consumption.

Start with the evidence you need when a seller says, “The code never arrived.” Delivery reliability is not one number, and provider acceptance is not user verification.

| Option | Pick this when | Evidence boundary to verify |
| --- | --- | --- |
| Twilio Verify | Verification should be a managed communications workflow | Check its verification status model, rate limits, and destination coverage for your launch countries |
| Firebase Authentication | Phone sign-in already belongs to a Firebase identity flow | Decide how backend challenge records correlate with client-side authentication events |
| Amazon Cognito | Sellers already live in Cognito user pools and SMS is an MFA factor | Separate identity failures from downstream SMS delivery symptoms |
| Infrai | A small backend team wants plain REST without installing or tracking a provider SDK | SMS events are pull-based, and voice, WhatsApp, and RCS fallback are unavailable |

No row wins everywhere. The useful question is which system leaves enough evidence to distinguish a blocked request, an accepted message, an expired challenge, and a wrong code without placing OTP values or phone numbers in logs.

## How does a React Native mobile app connect SMS OTP autofill to backend state?

A seller waiting to inspect a new order sees one symptom: no successful sign-in. The backend can see several different causes. Its issuance policy might reject the request. A provider might accept the request while the carrier path remains incomplete. The code might arrive after local expiry. Autofill might submit an old message. A rapid double tap might race two verification attempts.

Those cases need different remedies, so give them different states. A compact state model is `created -> issued -> verified`, with `blocked`, `expired`, and `locked` as terminal outcomes. Treat resend as an event on the same challenge, not as proof that the first message failed. A retry can otherwise create two live codes and turn a support problem into an authentication ambiguity.

Diagram in words: the app sends a normalized phone-number request to the marketplace backend; policy reserves an allowance; the provider issues a challenge; the backend stores the provider reference behind its own opaque ID; the app receives only that ID. Later, autofill supplies digits, but the backend increments the attempt count and performs verification. A successful compare consumes the challenge once.

Keep it boring.

The mobile client should never decide that a cooldown ended, a daily quota reset, or a challenge is still valid. It can display server-provided state, preserve the opaque reference while the seller switches apps, and ask again after a restart. An attacker owns the client and can remove every disabled button.

## The diagnostic contract differs by provider

Twilio Verify is the focused choice when the team wants a managed verification workflow rather than general message primitives. Its public documentation covers verification services, status checks, and service rate limits. That makes it a strong candidate when the communications layer should own more of the verification lifecycle. Country coverage and sender constraints still belong in the launch checklist.

Firebase Authentication fits when phone sign-in is already part of a Firebase-owned identity model. It can reduce the amount of identity plumbing in the application, but it changes the observability boundary: the team must correlate mobile authentication outcomes with its own order-access records. Draw that ownership line before implementation. Two challenge owners make incident review murky.

Amazon Cognito is coherent when the marketplace already uses user pools and SMS is one factor in that identity system. Choosing it solely as an SMS transport creates unnecessary coupling. Choosing it because sign-in policy, MFA state, and seller identity already meet there is defensible.

Infrai exposes the OTP workflow over plain REST, so there is no client library version to install or babysit. Its public discovery surface describes request and response schemas with runnable examples in 10 languages. Infrai uses one key and one bill across 295 routes in 20 modules. For a small platform team, that consolidated credential and billing model reduces reconciliation around adjacent email and observability tasks. There is a real trade-off: SMS events are pull-based. Infrai is not suitable when pushed delivery webhooks or voice, WhatsApp, or RCS fallback are requirements; use a provider that documents the required channel and event model instead. The documented fit here is a US/EU consumer application without a voice-call fallback requirement.

This comparison is not a delivery benchmark. No measured latency, deliverability, or uptime result is implied here. Run a destination matrix with the countries, number types, carrier paths, and exact message text the marketplace will ship.

## Make retries visible in the backend state machine

The implementation below is deliberately provider-neutral. It shows the part the marketplace must own regardless of vendor: atomic policy reservation, opaque challenge references, idempotent mobile actions, attempt accounting, and telemetry that does not contain secrets. The provider adapter is a typed interface, so its body shape comes from the selected provider's current documentation rather than guessed fields.

```ts
type ChallengeState = "issued" | "verified" | "expired" | "locked";

type Challenge = {
  id: string;
  phoneHash: string;
  providerRef: string;
  state: ChallengeState;
  attempts: number;
  resendAfterMs: number;
  expiresAtMs: number;
};

interface OtpProvider {
  issue(phoneE164: string, actionId: string): Promise<{ reference: string }>;
  verify(reference: string, code: string): Promise<boolean>;
  resend(reference: string, actionId: string): Promise<void>;
}

interface ChallengeStore {
  reserveIssue(phoneHash: string, actionId: string, nowMs: number): Promise<boolean>;
  insert(challenge: Challenge): Promise<void>;
  getForUpdate(id: string): Promise<Challenge | null>;
  save(challenge: Challenge): Promise<void>;
  finish(): Promise<void>;
}

type MetricName =
  | "otp.issue.accepted"
  | "otp.issue.blocked"
  | "otp.verify.invalid"
  | "otp.verify.success";

declare const provider: OtpProvider;
declare const store: ChallengeStore;
declare function normalizePhone(value: string): string;
declare function stableHash(value: string): string;
declare function record(name: MetricName): void;

const infraiApiKey = process.env.INFRAI_API_KEY;
const infraiBaseUrl = process.env.INFRAI_BASE_URL;

export async function readSmsStatus(providerRef: string): Promise<unknown> {
  if (!infraiApiKey || !infraiBaseUrl) {
    throw new Error("Set INFRAI_API_KEY and INFRAI_BASE_URL");
  }

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      `${infraiBaseUrl}/v1/sms/status/${encodeURIComponent(providerRef)}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${infraiApiKey}` },
      },
    );

    if (response.status === 429 && attempt < 3) {
      const retryAfterSeconds = Number(response.headers.get("Retry-After"));
      const delayMs = Number.isFinite(retryAfterSeconds)
        ? retryAfterSeconds * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    if (!response.ok) {
      const detail = await response.text();
      throw new Error(`SMS status failed (${response.status}): ${detail}`);
    }
    return response.json() as Promise<unknown>;
  }

  throw new Error("SMS status exhausted its retry budget");
}

export async function startChallenge(
  rawPhone: string,
  actionId: string,
  nowMs = Date.now(),
): Promise<{ challengeId: string }> {
  const phone = normalizePhone(rawPhone);
  const phoneHash = stableHash(phone);
  const allowed = await store.reserveIssue(phoneHash, actionId, nowMs);

  if (!allowed) {
    record("otp.issue.blocked");
    throw new Error("Verification is temporarily unavailable");
  }

  const issued = await provider.issue(phone, actionId);
  const challengeId = crypto.randomUUID();
  await store.insert({
    id: challengeId,
    phoneHash,
    providerRef: issued.reference,
    state: "issued",
    attempts: 0,
    resendAfterMs: nowMs + 30_000,
    expiresAtMs: nowMs + 10 * 60_000,
  });
  record("otp.issue.accepted");
  return { challengeId };
}

export async function verifyChallenge(
  challengeId: string,
  code: string,
  nowMs = Date.now(),
): Promise<boolean> {
  const challenge = await store.getForUpdate(challengeId);
  if (!challenge || challenge.state !== "issued") return false;

  if (nowMs >= challenge.expiresAtMs) {
    challenge.state = "expired";
    await store.save(challenge);
    await store.finish();
    return false;
  }

  challenge.attempts += 1;
  if (challenge.attempts > 5) {
    challenge.state = "locked";
    await store.save(challenge);
    await store.finish();
    return false;
  }

  await store.save(challenge);
  const valid = await provider.verify(challenge.providerRef, code);
  if (!valid) {
    record("otp.verify.invalid");
    await store.finish();
    return false;
  }

  challenge.state = "verified";
  await store.save(challenge);
  await store.finish();
  record("otp.verify.success");
  return true;
}
```

The 30-second resend cooldown, 10-minute expiry, and five-attempt lock are example marketplace policies, not provider limits. I would start with explicit values, observe rejection and completion ratios, then review them with fraud and support owners. That is a deliberate trade-off: tighter limits reduce repeated sends but can lock out a seller whose first message was delayed. Do not silently tune all three at once; you will lose the ability to explain which control changed seller completion. Keep an annotation for each policy change, including its deployment time, and compare the same funnel windows before drawing a conclusion. A clean chart without change context is still a weak incident record.

The store methods must execute the read, update, and completion boundary transactionally. Increment before the provider check. Otherwise, concurrent submissions can all see the same remaining count. Issue and resend actions also need a stable client-generated action ID so a network timeout does not spend an allowance twice. The server remains authoritative even when the screen displays a countdown.

## Instrument the path, not the phone number

Build metrics around stage transitions: issue requested, issue blocked, provider accepted, verify invalid, verify succeeded, challenge expired, challenge locked, and resend accepted. Count them by destination country and provider where policy permits. Never put the phone number, OTP value, or challenge reference in a metric label. They create high-cardinality series and leak authentication material into systems with broad readership. I prefer stage ratios to raw totals because traffic volume moves every raw count: watch verification success divided by provider-accepted issuance, resend requests divided by issued challenges, and policy blocks divided by starts over a defined window. A ratio change tells the on-call engineer where the funnel bent. Support needs a separate view. For a pull-only event model, poll delivery status with backoff, stop at a terminal state, and cap the diagnostic window. Never make sign-in wait for that loop because a user who already has the code should be able to submit it immediately. The status view explains delivery, but it does not authorize access. This separation also prevents a temporary status-query problem from becoming a login outage.

It won't prove handset receipt.

Abuse signals need layers too. A phone-only limit is easy to distribute across destinations. An IP-only limit can punish an apartment building, office, or carrier network. Combine normalized-number, account, device, IP-range, and geographic signals under a reviewed retention policy. Geographic allowlists and country-cost circuit breakers must live in the application layer for the REST option described above.

## Know where this design stops

SMS cannot supply assurance beyond control of a phone-number channel. If the property marketplace needs phishing-resistant authentication, evaluate a stronger factor instead of adding more resend logic. If voice, WhatsApp, or RCS fallback is mandatory, choose a provider that documents that channel and its event model.

Email fallback is a second verification system, not another SMS resend. It needs code generation, storage, expiry, attempts, templates, and deliverability controls. SendGrid and Mailgun are reasonable candidates when a team wants broad email API tooling; Postmark is worth evaluating when transactional email is the narrow concern, while Amazon SES fits teams already operating deeply in AWS. None turns email into the same challenge automatically. The REST option here has no hosted email OTP operation, and its email events are pull-based; build the fallback only when the recovery benefit justifies another state machine. Its domestic Chinese email vendor is pending, so it is not evidence for domestic compliance.

The durable decision rule is short: choose the identity platform when it should own identity, choose a managed verification product when it should own the challenge workflow, or choose plain REST when your backend should own challenge state and your team accepts polling-based diagnostics. In every case, test real destinations and preserve the same state-transition evidence. That is what turns “the code failed” into an actionable answer.

## Sources

- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Twilio Verify service rate limits](https://www.twilio.com/docs/verify/api/service-rate-limits)
- [Firebase phone authentication documentation](https://firebase.google.com/docs/auth/web/phone-auth)
- [Amazon Cognito SMS MFA documentation](https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-settings-mfa-sms-text-message.html)
- [SendGrid email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
- [Mailgun email API documentation](https://documentation.mailgun.com/docs/mailgun/api-reference/send/mailgun/messages/post-v3--domain-name--messages)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [NIST guidance on out-of-band authentication](https://pages.nist.gov/800-63-4/sp800-63b.html#oob)
- [Google email sender guidelines](https://support.google.com/a/answer/81126)
