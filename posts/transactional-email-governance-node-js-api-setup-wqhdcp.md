# Transactional Email Governance: Node.js API Setup for Auditable Property Receipts

A transactional email setup in Node.js should preserve the evidence that cannot be reconstructed later. For a property payment receipt sent through an API, that means the settled financial inputs, exact rendered message, custom-domain authentication observations, and transport events. Do not treat a successful response or an open pixel as the receipt record.

| Pick this evidence strategy when... | Store at send time | Accept the trade-off |
| --- | --- | --- |
| The exact customer-facing receipt may be disputed | Immutable rendered body plus settlement and template revision | More sensitive data needs access and retention controls |
| The organization can reproduce rendering deterministically | Inputs, renderer revision, and integrity digest | Reproduction depends on retaining executable rules and dependencies |
| Mail operations and finance have different retention needs | A financial record linked to a separately retained delivery record | Investigations require an authorized join |

**Short answer:** use the first strategy for the initial launch. It gives a reviewer direct evidence of what was generated after payment `SET-7319` settled, without depending on the current template or a future rebuild. Split financial and mail retention later if policy requires it, but keep a stable receipt identifier across both records.

## How should a Node.js API setup preserve transactional email evidence?

Start with the future question, not the send call. A resident disputes property order `PO-4182`. Finance can show that settlement `SET-7319` occurred, but the current receipt template has changed. Can the team still show what the system generated at that moment?

This is a governance problem before it is a transport problem. The settlement system establishes the payment facts. The renderer turns those facts into customer-facing content. The mail transport accepts a submission and may later report an outcome. Each system can make only its own claim.

DKIM has a similarly narrow boundary. RFC 6376 defines a mechanism by which a signing domain takes responsibility for a message through a cryptographic signature over selected headers and the body. It does not certify that a payment amount is correct. SPF, defined by RFC 7208, lets a receiver evaluate whether the connecting host is authorized for the relevant identity. It does not preserve the receipt.

A clean review therefore begins with one sentence: “Show receipt `RCP-4182`, as rendered from settlement `SET-7319`, and then show its mail history.” If the system can answer only by querying the current template and scattered logs, the evidence is fragile.

Very fragile.

And avoid a subtler trap: a welcome-style message may share branding or layout with the receipt, but the receipt artifact still needs its own revision and settlement link. Reconstructing it from a general welcome email template later can produce a polished message that was never actually sent.

## Pick an artifact policy before an email API

The strongest launch option is an immutable rendered artifact. Store the subject, HTML or text body, recipient, sender identity, settlement reference, template revision, creation time, and a digest. Restrict access because the artifact contains personal and financial information. Retention duration belongs to the organization's approved policy; email standards do not prescribe a property manager's legal schedule.

Deterministic reconstruction is a serious option too. It stores less duplicated content, but “same inputs” are not enough. The exact renderer, locale data, formatting behavior, and template revision must remain available. A dependency upgrade can change output while the database row stays identical. Pick this model only when the team can test reproduction and retain everything it depends on.

The third option separates custody. Finance retains settlement and receipt facts. Messaging operations retain submission and receiver events. A stable, non-secret receipt ID connects them through an authorized audit view. This limits broad access to combined data, though it makes investigations operationally more involved.

None of these choices proves delivery or reading. They decide what content evidence survives.

Content first.

## Implement the frozen-artifact path in TypeScript

Diagram in words: settled payment -> frozen receipt artifact -> queued intent -> transport acceptance -> receiver event. The first arrow establishes content. The other arrows establish custody and outcomes. Keep them separate in storage and in dashboards.

The following example goes deep on the boundary before submission. It uses a generic transport interface, so domain records do not inherit a commercial SDK's request shape.

```ts
type SettledPayment = Readonly<{
  settlementId: string;
  orderId: string;
  amountMinor: number;
  currency: string;
  settledAt: string;
  receiptEmail: string;
}>;

type FrozenReceipt = Readonly<{
  receiptId: string;
  settlementId: string;
  templateRevision: "property-receipt-v5";
  from: string;
  to: string;
  subject: string;
  html: string;
  createdAt: string;
}>;

interface ReceiptStore {
  createOnce(receipt: FrozenReceipt): Promise<FrozenReceipt>;
  recordSubmission(input: {
    receiptId: string;
    attemptId: string;
    transportMessageId: string;
    acceptedAt: string;
  }): Promise<void>;
}

interface MailTransport {
  submit(input: {
    correlationKey: string;
    from: string;
    to: string;
    subject: string;
    html: string;
  }): Promise<{ messageId: string; acceptedAt: string }>;
}

function freezeReceipt(payment: SettledPayment, now: string): FrozenReceipt {
  const amount = new Intl.NumberFormat("en-US", {
    style: "currency",
    currency: payment.currency,
  }).format(payment.amountMinor / 100);

  return {
    receiptId: `receipt:${payment.settlementId}`,
    settlementId: payment.settlementId,
    templateRevision: "property-receipt-v5",
    from: "receipts@notices.example-property.test",
    to: payment.receiptEmail,
    subject: `Receipt for order ${payment.orderId}`,
    html: [
      "<h1>Payment receipt</h1>",
      `<p>Order ${payment.orderId}</p>`,
      `<p>Amount ${amount}</p>`,
      `<p>Settled ${payment.settledAt}</p>`,
    ].join(""),
    createdAt: now,
  };
}

async function submitReceipt(
  store: ReceiptStore,
  transport: MailTransport,
  payment: SettledPayment,
  attemptId: string,
  now: string,
): Promise<void> {
  const receipt = await store.createOnce(freezeReceipt(payment, now));
  const accepted = await transport.submit({
    correlationKey: receipt.receiptId,
    from: receipt.from,
    to: receipt.to,
    subject: receipt.subject,
    html: receipt.html,
  });

  await store.recordSubmission({
    receiptId: receipt.receiptId,
    attemptId,
    transportMessageId: accepted.messageId,
    acceptedAt: accepted.acceptedAt,
  });
}
```

`createOnce` is the important operation. A retry for `SET-7319` must retrieve the existing artifact rather than render a newer receipt. `attemptId` changes for each local try; `receiptId` does not. The transport message identifier belongs to the submission record because transport acceptance and financial settlement are different events.

There is an unavoidable uncertain window after external acceptance and before the local evidence write. Model it. Preserve the attempt, reconcile it against later transport records, and do not label acceptance as delivery. A durable queue can retain the send intent across a worker restart, but it cannot remove that boundary.

Unknown means unknown.

## Test evidence decay, not only successful sending

A first-message test should use a mailbox outside the sending environment. Inspect the received authentication results and confirm the expected DKIM signing domain and selector. Check SPF as a receiver would; RFC 7208 section 4.6.4 limits terms that cause DNS queries to 10 during an evaluation, so nested includes deserve explicit validation. For DKIM key rotation, publish the new public key before changing the signer, then retain the selector observed for each relevant message.

Now change time. Deploy `property-receipt-v6` and prove that `RCP-4182` still returns the v5 artifact. Retry the same settlement twice and require one frozen receipt with distinct attempts. Stop the worker after transport acceptance but before the evidence write; the resulting state must remain “uncertain,” not “failed” and not “delivered.” Replay a delayed or bounced event and require idempotent ingestion while preserving both its occurrence time and ingestion time.

These tests suggest the operational signals. Alert on oldest queued-receipt age because a stalled queue can be quiet. Track submission failure count, acceptance latency, delayed outcomes, bounced outcomes, and reconciliation backlog. Put receipt IDs in restricted structured logs, but keep email addresses and rendered bodies out of metric labels. Metrics should be low-cardinality trends; the authorized audit view carries detail.

The before-and-after is crisp. Before: one `sent` boolean and a mutable template. After: one immutable content artifact, many explicit attempts, and separately recorded outcomes.

## Limits of the evidence

Transport acceptance does not prove inbox delivery. A delivery event does not prove that a resident read or understood a receipt. Apple's Mail Privacy Protection can download remote content in the background and prevent senders from seeing Mail activity, which makes apparent opens unsuitable as compliance evidence of human viewing.

The launch boundary is therefore modest: freeze what was generated, authenticate the custom sending domain with SPF and DKIM, retain correlation across retries, test the uncertain acceptance window, and expose the result through access-controlled evidence views. The architecture can support defensible claims about content and system events. It cannot prove comprehension.

## References

- RFC 6376, DomainKeys Identified Mail (DKIM): https://datatracker.ietf.org/doc/html/rfc6376
- RFC 7208, Sender Policy Framework (SPF): https://datatracker.ietf.org/doc/html/rfc7208
- Apple, Use Mail Privacy Protection on iPhone: https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
