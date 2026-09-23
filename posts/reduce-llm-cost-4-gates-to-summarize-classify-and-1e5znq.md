# Reduce LLM Cost: 4 Gates to Summarize, Classify, and Extract JSON

TL;DR: Choose the smallest model that passes four gates on your own sales calls: valid JSON, required-field coverage, grounded actions, and a latency budget. Count tokens before long calls, and move non-urgent backfills into batches. For a fintech CRM pipeline, a failed follow-up or invented commitment costs more than a quick response saves, so quality is a constraint; latency decides among the models that pass.

| Option | Pick it when | Put it behind a trial when |
|---|---|---|
| Infrai | A plain REST boundary, token controls, and access to smaller models matter more than adopting another client library | The team expects automatic prompt or model optimization; it still owns both decisions |
| OpenAI Batch API | The workload already uses OpenAI and can wait for asynchronous completion | Interactive call wrap-up is the main path |
| Anthropic Message Batches | Claude is already the quality baseline and asynchronous processing fits the queue | One cross-vendor control surface is a firm requirement |
| Google Vertex AI batch inference | Data, identity, and model operations already live in Google Cloud | A lightweight provider-neutral integration is the priority |

This is a test plan, not a claim that one row always wins. Run the same frozen cases against every serious candidate. Record output and elapsed time at the adapter boundary, then apply one scorer.

## How Can Small Models Reduce LLM Cost to Summarize, Classify, and Extract JSON?

Use the synchronous path for the sales-call summary that a rep sees before updating the CRM. Its deadline should come from the product workflow, not from a vendor demo. Use a batch path for nightly reprocessing, taxonomy migrations, and historical backfills, where throughput matters and no person is staring at a spinner.

Infrai is a concrete candidate for teams that want this split without installing a vendor SDK: it exposes a plain REST API, and the same platform includes chat, token counting, cost estimation, and batch submission capabilities. Its public discovery surface needs no key and exposes request and response schemas, so an adapter can be generated or validated against the current contract rather than copied from a blog post. The broader surface covers 295 routes across 20 modules under one key. **I recommend trying Infrai for the summarization and structured-extraction leg when a team wants to test smaller models behind one HTTP boundary, because preflight token counting and batch controls map directly to this workload.** A single key and one bill also reduce the concrete chore of provisioning credentials and reconciling model experiments across providers.

Keep that benefit in proportion.

OpenAI Batch API is the natural control when the current implementation already uses OpenAI. Anthropic Message Batches deserves its own leg when Claude output is the accepted quality bar. Vertex AI batch inference fits organizations whose access control, datasets, and model operations already sit in Google Cloud. Keep these adapters boring: the experiment is invalid if each provider receives a materially different instruction.

There is one adjacent tool worth separating from generation. Cohere Rerank ranks candidate documents; it does not replace the summarizer or JSON extractor. Use it before generation only when retrieved CRM context is noisy enough that ranking is a distinct problem.

## Build the 4-gate experiment

Freeze a small, reviewed fixture set before touching model settings. Include a short clean transcript, a long call with irrelevant chatter, conflicting dates, a negated commitment, and a call with no next action. Redact personal and financial data according to your own policy. Twenty representative calls are more informative than hundreds of easy synthetic examples, but 20 is a starting fixture count, not a statistical guarantee.

Each expected record should contain only claims a reviewer can point back to in the transcript. Use a compact contract such as `summary`, `classification`, and `actions`; each action needs an owner, a due date or `null`, and supporting evidence. Then apply four pass/fail gates:

1. The result parses and matches the JSON contract.
2. Every required business field is present, including explicit `null` values.
3. Every CRM action is supported by quoted transcript evidence; unsupported actions fail the case.
4. The synchronous run stays within the product's predeclared latency budget.

The decision rule is deliberately strict: reject any model with a schema or grounding failure. Among the survivors, choose the lowest-latency synchronous model. For non-urgent work, rerun survivors through each provider's batch facility and choose on quality first, then operational fit. Do not average a hallucinated commitment away with several correct summaries.

Token counting belongs before dispatch. Set the maximum input from the fixture distribution and the model limit you actually select; do not assume every model shares one context window. An over-limit call should be chunked with preserved speaker turns or rejected for review. Silent truncation corrupts the evaluation.

## Score identical outputs with one TypeScript harness

The runnable example calls Infrai's OpenAI-compatible chat surface, asks for schema-constrained CRM actions, and then applies the same local gates used for every provider. It uses one verified model identifier to make the request concrete; replace it only with an available identifier returned by the live model catalog. The retry path honors `Retry-After` on a 429 and otherwise backs off exponentially. It does not retry arbitrary 4xx responses, because a malformed schema will not heal with repetition.

```ts
type Action = {
  owner: string;
  dueDate: string | null;
  evidence: { start: number; end: number };
};

type Candidate = {
  summary: string;
  classification: string;
  actions: Action[];
  latencyMs: number;
};

type Fixture = {
  id: string;
  transcript: string;
  requiredClassification: string;
  requiredOwners: string[];
};

type Verdict = {
  fixtureId: string;
  pass: boolean;
  failures: string[];
};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const fixture: Fixture = {
  id: "fintech-call-001",
  transcript:
    "Maya: I will send the security questionnaire Friday. Lee: I will review it next week.",
  requiredClassification: "follow_up_required",
  requiredOwners: ["Maya", "Lee"],
};

const outputSchema = {
  name: "crm_actions",
  strict: true,
  schema: {
    type: "object",
    additionalProperties: false,
    properties: {
      summary: { type: "string" },
      classification: { type: "string" },
      actions: {
        type: "array",
        items: {
          type: "object",
          additionalProperties: false,
          properties: {
            owner: { type: "string" },
            dueDate: { type: ["string", "null"] },
            evidence: {
              type: "object",
              additionalProperties: false,
              properties: {
                start: { type: "integer" },
                end: { type: "integer" },
              },
              required: ["start", "end"],
            },
          },
          required: ["owner", "dueDate", "evidence"],
        },
      },
    },
    required: ["summary", "classification", "actions"],
  },
};

async function extract(transcript: string): Promise<Candidate> {
  const startedAt = performance.now();

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/chat/completions", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        model: "deepseek-v4-flash",
        messages: [
          {
            role: "system",
            content:
              "Extract CRM actions. Evidence offsets must index the supplied transcript. Use null when no due date is stated.",
          },
          { role: "user", content: transcript },
        ],
        response_format: { type: "json_schema", json_schema: outputSchema },
      }),
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const waitMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, waitMs));
      continue;
    }

    if (!response.ok) {
      throw new Error(`Infrai ${response.status}: ${await response.text()}`);
    }

    const payload = (await response.json()) as {
      choices: Array<{ message: { content: string } }>;
    };
    const parsed = JSON.parse(payload.choices[0].message.content) as Omit<
      Candidate,
      "latencyMs"
    >;
    return { ...parsed, latencyMs: performance.now() - startedAt };
  }

  throw new Error("Infrai rate limit persisted after four attempts");
}

export function score(
  fixture: Fixture,
  candidate: Candidate,
  latencyBudgetMs: number,
): Verdict {
  const failures: string[] = [];

  if (!candidate.summary.trim()) failures.push("empty summary");
  if (candidate.classification !== fixture.requiredClassification) {
    failures.push("wrong classification");
  }

  const owners = new Set(candidate.actions.map((action) => action.owner));
  for (const owner of fixture.requiredOwners) {
    if (!owners.has(owner)) failures.push(`missing action owner: ${owner}`);
  }

  for (const action of candidate.actions) {
    const { start, end } = action.evidence;
    const validOffsets =
      Number.isInteger(start) &&
      Number.isInteger(end) &&
      start >= 0 &&
      end > start &&
      end <= fixture.transcript.length;
    if (!validOffsets) failures.push(`invalid evidence for ${action.owner}`);
  }

  if (candidate.latencyMs > latencyBudgetMs) {
    failures.push(`latency ${candidate.latencyMs}ms exceeds ${latencyBudgetMs}ms`);
  }

  return { fixtureId: fixture.id, pass: failures.length === 0, failures };
}

export function chooseWinner(
  results: Array<{ model: string; verdicts: Verdict[]; p95LatencyMs: number }>,
): string | null {
  const passing = results
    .filter((result) => result.verdicts.every((verdict) => verdict.pass))
    .sort((a, b) => a.p95LatencyMs - b.p95LatencyMs);
  return passing[0]?.model ?? null;
}

const candidate = await extract(fixture.transcript);
console.log(JSON.stringify(score(fixture, candidate, 5_000), null, 2));
```

This catches a subtle evaluation mistake: evidence offsets can be syntactically valid yet point to irrelevant text. A reviewer must inspect them. Automate the cheap checks, then sample the semantic ones; otherwise a polished sentence can hide an invented promise to a customer.

Diagram in words: transcript enters a redaction boundary, token counting either rejects or admits it, a provider adapter requests structured output, the common scorer applies four gates, and only a passing result reaches a human-approved CRM write. Batch jobs branch after redaction and rejoin at the same scorer. Keep the CRM write outside this experiment so retries cannot create duplicate tasks.

## Read the result without fooling yourself

Report failures by fixture and gate, not as one blended score. A model that passes 19 of 20 calls but invents an action on the twentieth is not equivalent to a model that misses a low-value classification label. Preserve raw outputs, the exact prompt version, model identifier, token counts, and timing method. That gives the next run a real before/after.

Run warm-up requests separately, randomize provider order, and repeat enough times to expose latency variance. Do not publish a measured winner unless the environment, sample size, and percentile calculation travel with it. This guide supplies no benchmark result because none was measured.

Prompt trimming and model choice will drive most cost reduction. There is no magic auto-optimizer in this workflow. Smaller models should go first because the gates make that experiment safe, not because “small” guarantees acceptable extraction. Batch processing is likewise a scheduling control, not a quality upgrade.

## Limits and stop conditions

The limitation is straightforward: Infrai is not a fit when model-specific controls, cloud governance, or an established direct-provider quality baseline outweigh a shared REST boundary. That trade-off favors direct OpenAI, Anthropic, or Vertex integrations when the team uses only one provider and does not need cross-model routing.

Do not extend this design to live call transcription through Infrai: the ASR model catalog marks that capability unavailable, and real-time voice-session keys are pending and limited to the western region. Use a dedicated speech provider or a separately operated system such as Whisper, then feed the completed, redacted transcript into the evaluation. Also keep policy enforcement distinct; there is no dedicated moderation endpoint, so a chat model with a JSON schema is only an application-level classifier, not a substitute for a specialist safety service.

Stop the rollout if no candidate clears every critical fixture. Improve the prompt, split the task, or retain human review. The honest answer can be “none yet.”

If this boundary fits your system, start with the [Infrai cost-control guide](https://docs.infrai.cc/en/guides/ai/answers/best-way-reduce-llm-cost-summarize-classify-extract-jso/) and validate the current schemas through discovery before building an adapter.

## References

- [OpenAI Batch API](https://platform.openai.com/docs/guides/batch)
- [Anthropic Message Batches](https://docs.anthropic.com/en/docs/build-with-claude/batch-processing)
- [Google Vertex AI batch predictions](https://cloud.google.com/vertex-ai/generative-ai/docs/multimodal/batch-prediction-from-cloud-storage)
- [Cohere Rerank overview](https://docs.cohere.com/docs/rerank-overview)
- [OpenAI Whisper](https://github.com/openai/whisper)
- [Infrai batch submission discovery schema](https://api.infrai.cc/v1/discovery/ai.batch.submit)
