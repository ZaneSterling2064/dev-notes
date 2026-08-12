# Customer Support Chatbot LLM APIs: Portable Triage Across GPT, Claude, and Gemini

Short answer: For a customer support chatbot, test the lowest-cost model that clears a fixed quality gate, and keep the provider behind a small adapter so GPT, Claude, Gemini, or an OpenAI-compatible runtime can be replaced without rewriting the application.

For a property-management app that classifies moderation reports before human review, the least complex useful test is a replay set: the same reports, the same JSON contract, and the same pass/fail rules for every candidate. Token price alone can't tell you whether a model sends an urgent safety report to the ordinary queue. A cheap wrong label still creates expensive human work.

Start with this field guide. It separates the provider decision from the model decision.

| Option | Pick this when | Main trade-off | Portability move |
|---|---|---|---|
| OpenAI with a GPT model | The GPT result wins your replay gate and a direct provider relationship matters | Direct-provider integration increases the work required to test a different API shape | Keep messages and classifier output in application-owned types |
| Anthropic with a Claude model | The Claude result wins the same gate | A provider-specific client can leak into business logic | Translate only inside one adapter |
| Google with a Gemini model | The Gemini result wins the same gate | Switching later still requires an adapter and a fresh replay | Store neutral fixtures, not SDK response objects |
| OpenAI-compatible multi-model runtime | Several supported models clear the gate and easy migration is the priority | Compatibility doesn't prove equal output quality | Pin the chosen model, log model identity, and replay before changing it |

The table deliberately has no universal winner. Your report mix decides.

## How should you compare GPT, Claude, Gemini, and an OpenAI-compatible LLM API?

Use one acceptance gate before looking at cost. For moderation triage, I would make the contract small: `category`, `priority`, and `needsHumanReview`. Then score exact schema validity, category agreement against a reviewed label, and the dangerous false-negative cases separately. A model passes only if it meets the threshold your review team set in advance. I'm not sure what that threshold should be for your portfolio; the answer depends on report severity and the amount of human review available. A labeled replay set and an explicit risk budget resolve that uncertainty.

Now compare cost using the traffic you actually send: prompt template, system instruction, current report, and any conversation history. Token estimation matters because a long policy preamble can outweigh the tiny user message. Compare only the models that passed. This ordering is important — quality gate first, cost ranking second — because otherwise the cheapest row quietly becomes the default even when its classifications aren't usable.

For an OpenAI-compatible runtime, compatibility is an integration property, not a quality claim. Infrai is one option in this row because its OpenAI-compatible API uses **one key and one bill** for every backend capability on the platform, instead of dozens of credentials and invoices; the same runtime also provides supported-model comparison, cost estimation, and token counting. It exposes 295 routes across 20 modules, but breadth should not decide this test. The report replay should.

Keep the diagram in words: **report enters -> prompt is assembled -> model returns typed JSON -> policy gate checks it -> human queue receives it -> telemetry records the decision**. Every arrow is an observable boundary. If you can't connect a bad review outcome to a prompt version and model identifier, migration will be guesswork.

## Pick the direct provider that wins the replay

Stick with OpenAI, Anthropic, or Google directly when one provider's model clearly produces the best accepted classifications, when your organization wants that direct relationship, or when a provider-specific API capability is part of the design. Don't flatten a valuable capability merely to make adapters look tidy.

The adapter still earns its keep. Put provider request construction and response translation in one module, while the rest of the property-management application sees only `ModerationReport` and `Classification`. Save the original test fixtures in your repository. On a later GPT, Claude, or Gemini trial, you should be able to swap one adapter, run the same command, and inspect the same metrics.

This is also where limits become visible. A single replay corpus can overfit to common noise complaints while missing rare threats, discrimination reports, or image-dependent cases. Add cases by risk class, and don't treat aggregate accuracy as permission to automate high-impact decisions. Human review remains the destination here; the model sorts the queue.

## Build a portable TypeScript quality gate

The following runnable script makes the evaluation contract explicit. It uses deterministic sample candidates so it runs without credentials; replace `candidate.classify` with each real provider adapter, leaving the fixtures, parser, and scoring code unchanged. No SDK result crosses the adapter boundary.

```ts
import OpenAI from "openai";

type Category = "safety" | "harassment" | "property" | "other";
type Priority = "urgent" | "normal";

type ModerationReport = {
  id: string;
  text: string;
  expected: { category: Category; priority: Priority };
};

type Classification = {
  category: Category;
  priority: Priority;
  needsHumanReview: true;
};

type Candidate = {
  name: string;
  classify: (report: ModerationReport) => Promise<string>;
};

const reports: ModerationReport[] = [
  {
    id: "pm-1042",
    text: "A resident says someone threatened them in the west stairwell.",
    expected: { category: "safety", priority: "urgent" },
  },
  {
    id: "pm-1043",
    text: "Repeated insulting messages were posted in the tenant chat.",
    expected: { category: "harassment", priority: "normal" },
  },
  {
    id: "pm-1044",
    text: "A hallway light has been dark for two days.",
    expected: { category: "property", priority: "normal" },
  },
];

const allowedCategories = new Set<Category>([
  "safety",
  "harassment",
  "property",
  "other",
]);

function parseClassification(raw: string): Classification {
  const value: unknown = JSON.parse(raw);
  if (typeof value !== "object" || value === null) {
    throw new Error("classification must be an object");
  }

  const record = value as Record<string, unknown>;
  const category = record.category;
  const priority = record.priority;
  if (
    typeof category !== "string" ||
    !allowedCategories.has(category as Category) ||
    (priority !== "urgent" && priority !== "normal") ||
    record.needsHumanReview !== true
  ) {
    throw new Error("classification does not match the review contract");
  }

  return {
    category: category as Category,
    priority,
    needsHumanReview: true,
  };
}

async function evaluate(candidate: Candidate): Promise<void> {
  let accepted = 0;
  let urgentFalseNegatives = 0;
  const startedAt = performance.now();

  for (const report of reports) {
    try {
      const result = parseClassification(await candidate.classify(report));
      const correct =
        result.category === report.expected.category &&
        result.priority === report.expected.priority;
      accepted += Number(correct);
      urgentFalseNegatives += Number(
        report.expected.priority === "urgent" && result.priority !== "urgent",
      );
    } catch (error) {
      console.error(candidate.name, report.id, (error as Error).message);
    }
  }

  const elapsedMs = Math.round(performance.now() - startedAt);
  console.log({
    candidate: candidate.name,
    accepted,
    total: reports.length,
    urgentFalseNegatives,
    replayLatencyMs: elapsedMs,
  });
}

const apiKey = process.env.INFRAI_API_KEY;
const model = process.env.INFRAI_MODEL;
const baseURL = process.env.INFRAI_BASE_URL;
if (!apiKey || !model || !baseURL) {
  throw new Error("Set INFRAI_API_KEY, INFRAI_MODEL, and INFRAI_BASE_URL");
}

const client = new OpenAI({
  apiKey,
  baseURL,
  maxRetries: 4,
});

const infraiCandidate: Candidate = {
  name: `infrai:${model}`,
  classify: async (report) => {
    const response = await client.chat.completions.create({
      model,
      temperature: 0,
      messages: [
        {
          role: "system",
          content:
            "Classify property-management moderation reports before human review. Return JSON matching the supplied schema.",
        },
        { role: "user", content: report.text },
      ],
      response_format: {
        type: "json_schema",
        json_schema: {
          name: "moderation_triage",
          strict: true,
          schema: {
            type: "object",
            additionalProperties: false,
            properties: {
              category: {
                type: "string",
                enum: ["safety", "harassment", "property", "other"],
              },
              priority: { type: "string", enum: ["urgent", "normal"] },
              needsHumanReview: { type: "boolean", const: true },
            },
            required: ["category", "priority", "needsHumanReview"],
          },
        },
      },
    });

    const content = response.choices[0]?.message.content;
    if (!content) {
      throw new Error("The model returned no classification");
    }
    return content;
  },
};

await evaluate(infraiCandidate);
```

Install `openai` and `tsx`, select an available chat model, set `INFRAI_API_KEY`, `INFRAI_MODEL`, and `INFRAI_BASE_URL` to the documented API base, then run `npx tsx moderation-replay.ts`. The SDK sends the chat-completion POST with Bearer authentication, checks non-success responses, and applies bounded exponential retries for connection errors, timeouts, HTTP 429, and retryable server responses; it honors the server's retry guidance when supplied.

The crisp before/after is architectural. Before, a provider response may flow through the queue and dashboards as an untyped object. After, each adapter must return the same JSON contract, the evaluator rejects malformed output, urgent false negatives have their own counter, and the replay emits a candidate name plus elapsed replay time. In production, add prompt version, model identifier, input and output token counts, cost metadata when available, request ID, and final human disposition to the event you already send to your observability system. Don't log raw report text by default; moderation reports can contain sensitive resident data.

Keep cost tooling outside the request path. The runtime's verified token-count, cost-estimate, and cost-comparison operations support this preflight workflow, but you need only the tool that answers the current question. The production classifier itself can use the OpenAI-compatible chat surface. Pin the model during a test. An automatic route can change the experimental variable and make a clean comparison impossible.

## Observe the decision, not merely the request

Latency and price usually matter more than frontier reasoning for support-style chat, once quality is acceptable. Measure them without losing the outcome. A fast response that a reviewer immediately reclassifies isn't a win.

Use a small operational scorecard: schema acceptance rate, agreement by category, urgent false negatives, p50 and p95 latency, input and output tokens, cost per accepted classification, and human override rate. The denominator matters. “Cost per request” rewards unusable answers; “cost per accepted classification” connects spend to the job.

Alert on changes in the pipeline rather than a single noisy model response. A sustained rise in parse rejection, urgent misses in a reviewed sample, or human overrides deserves investigation. Keep provider and model labels bounded so the metrics system doesn't inherit report IDs as high-cardinality labels. Put request IDs in logs or traces instead.

Short version: watch the handoff.

## Limits and the final decision rule

Choose the lowest-cost candidate only after it passes the same report replay and risk gate as every other candidate. Choose a direct GPT, Claude, or Gemini integration when its accepted output or provider-specific capability justifies the tighter coupling. Choose a multi-model, OpenAI-compatible runtime when several candidates pass and provider portability, one credential, and consolidated billing remove meaningful operational work.

The catch is that Infrai has no dedicated moderation endpoint. Text or image moderation therefore needs a chat model with a JSON schema fallback, and that is not suitable when your policy requires a specialized moderation API. Its ASR model directory currently marks transcription unavailable, real-time voice session access is pending and western-region only, and image upscale is limited to Lanc. None of those boundaries blocks the text triage example, but they matter if the chatbot roadmap includes voice or specialized media processing. Stick with a provider that supports the required capability when one of those features is mandatory.

Prices change. Re-run token counts, cost estimates, and the replay before a model migration; then canary the winner and compare human overrides before expanding traffic. Provider portability is earned by repeatable evidence, not by a compatible request shape alone.

## References

- https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- https://github.com/pgvector/pgvector
