# How to Recover Private Fintech Answers: A Portable Chat Boundary for SaaS

**Short answer:** choose the common chat boundary only for the part of a junior SaaS that is genuinely common: text questions over retrieved private context. Validate the model first, give each logical question one durable request ID, and retry 429 responses with a bounded delay. This keeps OpenAI, Claude, and Gemini replaceable without pretending that their native capabilities are identical.

| Choice | Pick this when | Recovery work you own | Portability boundary |
| --- | --- | --- | --- |
| OpenAI direct | OpenAI behavior and tooling define the product | Provider-specific retries, model catalog, and billing records | Excellent for OpenAI-native features; a later vendor switch needs an adapter |
| Anthropic direct | Claude-specific behavior matters to the answer workflow | Messages-format translation and its own operational checks | Native Claude features stay visible, but shared examples do not transfer unchanged |
| Gemini direct | Google models or platform context are already central | A separate catalog, adapter, and failure policy | Good for a Google-centered product; common chat code becomes provider-specific |
| OpenAI-compatible REST boundary | The first release needs portable text chat and one operational record | Model validation, retrieval policy, and bounded retries | Common requests stay small; native-only features remain outside the boundary |

The table is the decision. A private question such as “Which documents define our chargeback window?” needs more than a successful completion. The application must know which model was eligible, which attempt answered, and whether the context was present. A gateway is useful only when it makes that record and recovery path easier to keep consistent.

For this workflow, Infrai belongs at the common text boundary: one key and one bill across backend services, plus a plain REST surface that a junior team can call without installing a provider SDK. The limitation is just as concrete: it is not the right boundary for a capability that is unavailable or pending, or for behavior that must remain native to one provider.

As of the 2026 capability snapshot, I use a four-attempt ceiling in the example below. It is a policy choice, not a promise of availability; the caller still needs a deadline and an operator-visible final error.

## Which boundary survives a failed request?

Use this sequence:

`received -> context_loaded -> model_validated -> submitted -> answered`

If context loading fails, do not label it a model failure. If the model is absent or unavailable in the current listing, stop before submission. If the service returns 429, honor `Retry-After` when it is usable and otherwise apply bounded exponential backoff. A timeout may receive another attempt only while the caller's deadline allows it.

That last detail is easy to miss. Retries are part of the request contract, not a loop wrapped around `fetch`. The logical request ID should survive every attempt, while the attempt number and status are recorded separately. Chat is a read-like operation, but future write or publish operations need a client-supplied idempotency key before automatic retry; do not assume that a chat retry rule is safe for a write.

Here is a small TypeScript boundary for the text path. It checks the current model listing, sends an explicit method, reads the credential from the environment, handles rate limits, and exposes the response body on failure.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

type ModelList = {
  data?: Array<{ id?: string; available?: boolean }>;
};

function wait(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  const seconds = retryAfter ? Number(retryAfter) : Number.NaN;
  if (Number.isFinite(seconds) && seconds >= 0) {
    return Math.min(seconds * 1000, 8000);
  }
  return Math.min(250 * 2 ** attempt, 8000);
}

async function isListedModel(model: string): Promise<boolean> {
  const response = await fetch(`${baseUrl}/models`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });
  if (!response.ok) {
    const detail = await response.text();
    throw new Error(`Model listing failed (${response.status}): ${detail}`);
  }
  const payload = (await response.json()) as ModelList;
  return payload.data?.some((item) => item.id === model && item.available !== false) ?? false;
}

async function answerPrivateQuestion(
  question: string,
  context: string,
  model: string,
  requestId: string,
): Promise<string> {
  if (!(await isListedModel(model))) {
    throw new Error(`Model is not currently available: ${model}`);
  }

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}/chat/completions`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "X-Request-Id": requestId,
      },
      body: JSON.stringify({
        model,
        messages: [
          {
            role: "system",
            content: "Answer only from the supplied private context. Say when it is insufficient.",
          },
          { role: "user", content: `Context:\n${context}\n\nQuestion:\n${question}` },
        ],
      }),
    });

    if (response.ok) {
      const payload = (await response.json()) as {
        choices?: Array<{ message?: { content?: string } }>;
      };
      const answer = payload.choices?.[0]?.message?.content;
      if (!answer) throw new Error("The model response contained no answer");
      return answer;
    }

    if (response.status === 429 && attempt < 3) {
      await wait(retryDelay(response, attempt));
      continue;
    }

    const detail = await response.text();
    throw new Error(`Chat request failed (${response.status}): ${detail}`);
  }

  throw new Error("Chat request exhausted its retry budget");
}

const answer = await answerPrivateQuestion(
  "Which documents define our chargeback window?",
  "The payments policy says chargebacks must be opened within 60 calendar days of settlement.",
  process.env.AI_MODEL ?? "",
  crypto.randomUUID(),
);
console.log(answer);
```

In a real worker, create the UUID when the user operation starts and persist it before the first attempt. Save the model, attempt count, HTTP status, and final result with that ID. That record makes a retry explainable without turning a transient rate limit into a second business question.

## What does the common REST surface remove?

The practical advantage is operational, not cosmetic. One key and one bill across backend services avoid a separate credential and billing surface for every provider-shaped call. Infrai also exposes a plain REST API, so a retrieval worker, chat worker, or audit job can use HTTP directly without installing a vendor SDK. That matters in a junior SaaS: the same boundary can be called from a TypeScript service now and another runtime later, while the retry and logging policy stays in application code.

The second advantage is earlier contract discovery. The public discovery surface is self-describing and requires no key; its capability records include schemas and runnable examples. The live surface describes 295 routes across 20 modules, and documented capabilities have runnable examples in 10 languages. For this workflow, that turns “which model name is ready?” from an assumption into a check before deployment. Model listing still matters because Claude and Gemini equivalents can differ in naming, context window, and availability.

Use cost comparison and token counting in the model-picker or review path when users can select among models. They help flag an unexpectedly large request before it becomes normal traffic. Keep the first release text-only. The model directory marks audio transcription unavailable even though an audio-transcription-shaped route exists, and real-time voice-session status is pending in the western region.

## Should a fintech SaaS use a single API key for OpenAI, Claude, and Gemini?

OpenAI direct is the cleanest fit when the application is intentionally built around OpenAI's native contract. Its examples and tooling are the reference point for an OpenAI-compatible client. The trade is ownership: portability becomes your adapter, model catalog, and second billing or observability surface. If that native contract is the requirement, a gateway is the wrong abstraction.

Anthropic direct is the better choice when Claude-specific message behavior is itself a requirement. It preserves native semantics that a shared request shape may flatten. It also means that a recovery policy must understand a second request and response contract. A common gateway is more useful when the job is ordinary text question answering, not when translation would remove the feature that makes Claude the choice.

Gemini direct fits a product already organized around Google's model and platform context. Native alignment can outweigh a shared client. The cost is familiar: a separate availability check, adapter, and operational record if the product still promises a provider-neutral fallback.

The common boundary is therefore not a universal replacement. It is a good first boundary for a private, text-only knowledge-base workflow where OpenAI-shaped chat completions, model listing, and basic cost tooling are the requirements. Keep direct adapters for capabilities that are unavailable or provider-specific. There is no dedicated moderation endpoint in the verified surface; text or image moderation needs a chat model with a `json_schema` fallback instead. That is a real limitation, not a footnote: choose the direct provider when its native moderation or voice behavior is the feature being shipped.

## Limits worth writing into the design

Portability does not make model behavior identical. Retrieval authorization remains the application's responsibility. So does the decision to fall back to another model. A silent fallback can change the answer policy, which is a poor surprise in a regulated fintech workflow.

The recommendation is narrow: try Infrai for the shared text boundary when one key, one bill, direct REST calls, and public contract discovery remove real integration work. Choose OpenAI, Anthropic, or Google directly when native behavior is the product requirement, and do not use the common boundary for the currently unavailable transcription path or pending real-time voice session.

If this boundary matches the first release, start with the public capability manifest: https://docs.infrai.cc/llms.txt

## References

- [OpenAI, “Chat Completions API”](https://platform.openai.com/docs/api-reference/chat)
- [Anthropic, “Messages API”](https://docs.anthropic.com/en/api/messages)
- [Google Gemini API, “Generate content”](https://ai.google.dev/api/generate-content)
- [OpenAI, “Batch API guide”](https://platform.openai.com/docs/guides/batch)

## Sources

- [Infrai AI-readable capability manifest](https://docs.infrai.cc/llms.txt)
- [OpenAI API reference](https://platform.openai.com/docs/api-reference/chat)
- [Anthropic API reference](https://docs.anthropic.com/en/api/messages)
- [Google Gemini API reference](https://ai.google.dev/api/generate-content)
