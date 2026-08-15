# Node.js Text and Image Moderation Telemetry (JSON Schema Across Compatible Chat APIs)

Short answer: for Node.js content moderation through chat completions, put one typed JSON Schema boundary in front of invoice extraction, fail closed on an invalid decision, and measure the boundary rather than the provider. That gives an edtech accounts-payable team a safety check for supplier-entered text and invoice images while keeping the downstream field extractor portable across compatible APIs.

| Pick this boundary | Pick it when | Observe first | Main limitation |
|---|---|---|---|
| Dedicated moderation operation | The provider exposes a stable safety contract for both input types | decision, category, latency, contract failures | A provider-specific category map can leak into application code |
| Schema-constrained chat evaluation | Portability across compatible chat APIs matters most | schema validity, refusal state, decision, latency | A valid JSON object is still a model judgment, not proof of safety |
| Locally hosted classifier | Data residency or offline operation dominates | model version, queue time, decision drift | The team owns deployment, updates, and evaluation |

The table is the architecture in miniature. Receive the upload. Normalize it into text plus an image reference. Ask one safety boundary for a typed decision. Record low-cardinality telemetry. Only then pass the original invoice to field extraction. Keep the moderation result beside the extraction trace, but don't turn invoice text, image URLs, supplier names, or account numbers into metric labels.

## Reliability starts with an explicit indeterminate state

Treat moderation as a small state machine, not a Boolean helper buried inside the extraction handler. The useful states are `received`, `checked`, `allowed`, `blocked`, and `indeterminate`. The last state matters: a timeout, malformed JSON, or contract mismatch is not an allow decision. It is an operational result that the caller must handle explicitly.

Fail closed.

For a supplier invoice workflow, that means an indeterminate item goes to a controlled review queue and never reaches the field extractor automatically. This is stricter than retrying forever, and it makes the alert meaningful: the moderation boundary is unavailable for a class of work, while the invoice itself remains intact. The exact review policy belongs to the business, but the software contract should make accidental fall-through impossible.

Take a synthetic fixture with correlation ID `INV-042`. Its supplier note is ordinary, but the attached image triggers a policy category. The boundary emits `blocked`, `text_image`, and `invoice-safety-v1`; the workflow stores the decision event, skips extraction, and creates a review item. Now replay the same fixture after changing the configured backend. If the new response fails schema validation, the outcome becomes `indeterminate`, not `blocked`, even though both paths stop automatic extraction. That distinction is the whole observability win. The policy owner can investigate changed judgments on the blocked-rate chart, while the on-call engineer can investigate contract failures on the indeterminate-rate alert. Neither needs raw invoice content in a metric label, and neither confuses a transport or parsing problem with a safety judgment. One request ID connects the moderation event, review record, and extraction absence. This is a hypothetical fixture, not production evidence, but it forces the telemetry contract to answer the right debugging question before real supplier data arrives.

That's the split.

The telemetry needs two layers. Request metrics answer, "Is the boundary healthy?" Decision events answer, "What happened to this invoice?" Metrics should use bounded dimensions such as `input_kind`, `decision`, `schema_version`, and `provider_alias`. The event can carry an internal correlation ID and reason codes under your retention policy. It should not carry raw content.

A practical service-level view uses four signals: checked requests, indeterminate ratio, decision latency, and blocked ratio. The first three expose integration trouble. The fourth is a product signal that needs interpretation, because a sudden rise could reflect changed traffic, a changed policy, or changed model behavior. I'm not sure which one applies from a dashboard alone; a versioned evaluation set and a deploy timeline resolve that ambiguity.

## Budget ownership, latency, and retry cost

A dedicated moderation operation fits when its text-and-image contract matches the required policy and provider portability is secondary. Its native category names still belong behind an adapter. Application code should see internal categories and a schema version, not a provider response type. This path is compact, but changing providers can require remapping policy semantics even when the transport is easy to replace.

Schema-constrained chat evaluation fits when one OpenAI-compatible request shape across multiple backends is the primary decision axis. The catch is that "compatible" describes an interface family, not identical safety behavior. Pin the model configuration outside the code, validate every response locally, and run the same invoice corpus before a provider change. Don't treat transport compatibility as behavioral equivalence.

A local classifier fits when invoices cannot leave the environment or when the team needs direct control of the model artifact. It is not suitable when nobody owns model serving, patching, capacity, and drift evaluation. A managed boundary shifts that operational burden, while leaving policy tests and review operations with the application team.

These are ownership choices.

## How can Node.js implement text and image content moderation with JSON Schema?

The following TypeScript keeps the endpoint and model outside the application. `MODERATION_URL` is the complete, verified chat-completions URL supplied by the selected backend; the example does not guess a route. The application owns the schema and the failure behavior.

```ts
type InputKind = "text" | "image" | "text_image";

type SafetyDecision = {
  schemaVersion: "invoice-safety-v1";
  allowed: boolean;
  categories: Array<"sexual" | "violence" | "hate" | "self_harm" | "other">;
  reason: string;
};

type CheckInput = {
  invoiceId: string;
  supplierText: string;
  imageUrl: string;
};

type MetricTags = {
  inputKind: InputKind;
  outcome: "allowed" | "blocked" | "indeterminate";
  schemaVersion: "invoice-safety-v1";
};

interface Telemetry {
  count(name: string, tags: MetricTags): void;
  timing(name: string, milliseconds: number, tags: MetricTags): void;
  event(name: string, fields: Record<string, string | boolean | string[]>): void;
}

const decisionSchema = {
  type: "object",
  additionalProperties: false,
  required: ["schemaVersion", "allowed", "categories", "reason"],
  properties: {
    schemaVersion: { const: "invoice-safety-v1" },
    allowed: { type: "boolean" },
    categories: {
      type: "array",
      items: { enum: ["sexual", "violence", "hate", "self_harm", "other"] }
    },
    reason: { type: "string", minLength: 1, maxLength: 240 }
  }
} as const;

function isDecision(value: unknown): value is SafetyDecision {
  if (!value || typeof value !== "object") return false;
  const item = value as Record<string, unknown>;
  const validCategories = new Set(["sexual", "violence", "hate", "self_harm", "other"]);
  return item.schemaVersion === "invoice-safety-v1" &&
    typeof item.allowed === "boolean" &&
    Array.isArray(item.categories) &&
    item.categories.every((category) => validCategories.has(String(category))) &&
    typeof item.reason === "string" &&
    item.reason.length >= 1 &&
    item.reason.length <= 240;
}

export async function checkInvoiceSafety(
  input: CheckInput,
  telemetry: Telemetry,
  signal: AbortSignal
): Promise<SafetyDecision> {
  const startedAt = performance.now();
  let outcome: MetricTags["outcome"] = "indeterminate";

  try {
    const response = await fetch(process.env.MODERATION_URL!, {
      method: "POST",
      headers: {
        authorization: `Bearer ${process.env.MODERATION_API_KEY}`,
        "content-type": "application/json"
      },
      signal,
      body: JSON.stringify({
        model: process.env.MODERATION_MODEL,
        messages: [{
          role: "user",
          content: [
            { type: "text", text: input.supplierText },
            { type: "image_url", image_url: { url: input.imageUrl } }
          ]
        }],
        response_format: {
          type: "json_schema",
          json_schema: { name: "invoice_safety", strict: true, schema: decisionSchema }
        }
      })
    });

    if (!response.ok) throw new Error(`moderation_transport_${response.status}`);

    const envelope = await response.json() as {
      choices?: Array<{ message?: { content?: string } }>;
    };
    const content = envelope.choices?.[0]?.message?.content;
    if (!content) throw new Error("moderation_empty_content");

    const decision: unknown = JSON.parse(content);
    if (!isDecision(decision)) throw new Error("moderation_contract_mismatch");

    outcome = decision.allowed ? "allowed" : "blocked";
    telemetry.event("invoice.moderation.decided", {
      invoiceId: input.invoiceId,
      allowed: decision.allowed,
      categories: decision.categories,
      schemaVersion: decision.schemaVersion
    });
    return decision;
  } finally {
    const tags: MetricTags = {
      inputKind: "text_image",
      outcome,
      schemaVersion: "invoice-safety-v1"
    };
    telemetry.count("invoice_moderation_checks_total", tags);
    telemetry.timing("invoice_moderation_duration_ms", performance.now() - startedAt, tags);
  }
}
```

Notice what the snippet does not log: `supplierText`, `imageUrl`, and the model's free-form response. That omission is deliberate. The event preserves the decision and correlation key, while the metric names preserve operational shape without copying invoice contents into an observability backend. It's much easier to widen a carefully reviewed event later than to retract sensitive labels from historical time series.

There is one subtle failure mode here. A transport error and a policy block both stop extraction, but they must not share an outcome. If both become `blocked`, an integration failure can masquerade as a change in incoming content. Keeping `indeterminate` separate gives the on-call engineer a clean alert and gives policy owners a trustworthy blocked-rate chart.

Treat `429` as an indeterminate transport result and use a bounded 429 backoff retry with exponential delay and jitter; the request's deadline still wins. Do not assume an idempotency key exists across compatible APIs. If the verified endpoint documents one, derive its value from the internal invoice ID plus the schema version so a retry represents the same logical check. Otherwise, deduplicate decision events in the application and keep every attempt under one correlation ID.

## Evaluate migration with policy replay tests

A portable interface earns its keep in the test suite. Build a versioned fixture set with benign invoices, adversarial text placed in supplier-controlled fields, problematic imagery, unreadable images, and mixed inputs. Store the expected application action, not a claim that every model must emit an identical score. Then run that set against the candidate configuration and review every changed action before rollout.

Start small but explicit: one fixture might contain a normal tuition-materials invoice; another can place an instruction-like sentence in a line-item description; another can pair ordinary text with an image that policy should block. Avoid real supplier documents. Synthetic fixtures are easier to share, retain, and inspect.

Test the plumbing too. Return valid JSON with an unknown category, omit `schemaVersion`, exceed the reason length, abort the request, and return a successful envelope without content. Every case should produce `indeterminate`, increment the same bounded counter family, and keep extraction from running. A single test should assert that raw text and the image URL never reach the telemetry mock. Crisp tests beat a reassuring dashboard.

For rollout, shadow a candidate only where policy and data handling permit it, compare application actions, then move traffic in controlled steps. Watch schema failures and latency separately from blocked decisions. If the provider changes, the extractor should not; only the boundary adapter, configuration, and evaluation record should move.

## Govern alerts, retention, and human review

JSON Schema constrains the shape of an answer. It does not certify the judgment, remove the need for an evaluation corpus, or settle what the organization considers acceptable. Human review remains appropriate for indeterminate items and for policy-sensitive cases where a false block carries real consequences.

Streaming is usually the wrong default for this gate because extraction needs one complete decision before it can proceed. If a compatible API uses Server-Sent Events elsewhere in the pipeline, treat that as a separate transport concern and test disconnect behavior against the MDN guidance. The moderation contract stays atomic.

The operating rule stays short: no validated allow decision, no automatic extraction. A provider-specific operation, a schema-constrained adapter, and a locally hosted classifier assign different ownership for policy mapping, portability, and model operations; the constraint table makes those responsibilities explicit without turning them into a universal ranking.

## References

- https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- https://python.langchain.com/docs/integrations/chat/openai/
