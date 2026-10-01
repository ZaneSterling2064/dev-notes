# Embedding Plus Vector Store Combo — 3 One-Key FAQ Bot Contracts

TL;DR: Choose embeddings and vector storage from the same account for an onboarding FAQ bot unless a separate database gives you a capability you can name and test. The deciding constraint is index cost at scale: the embedding dimension, chunking policy, and model version become one operational contract, and changing that contract means rebuilding the index. A unified account leaves one credential to rotate and one place for a dimension mismatch to surface.

This matters for product content. Ten FAQ pages are forgiving; a catalog with repeated descriptions, variant notes, return rules, and merchant onboarding answers is not. Every extra chunk becomes another vector to create, store, update, and search. Start with the contract, not a logo.

## What actually drives the index bill?

The useful before-and-after model is short. Before semantic search, an FAQ answer is text in a content system. After semantic search, it is source text, split into chunks, converted into fixed-length vectors, and written into a collection whose dimension must match the embedding model exactly.

That last constraint is rigid.

If an embedding model emits a different dimension than the collection accepts, ingestion cannot be treated as a harmless partial update. If the team changes models later, it must re-embed the source corpus and write a new index either way. Naming the collection after the embedding generation makes that migration visible rather than surprising: `product-faq-emb-v1` can be built beside `product-faq-emb-v2`, checked, and then promoted by the application.

Index cost therefore begins with a count, not a vendor price page. Estimate source records, chunks per record, update frequency, and the number of complete reindexes you expect during evaluation. A product catalog of 80,000 records with four chunks per record produces 320,000 vectors per index generation. That is an illustrative planning calculation, not a benchmark. Your chunk distribution determines the real number.

Use a small script before opening any console. This one sends a real query while keeping the provider-defined JSON shape outside the program; export the account's API base as `INFRAI_BASE_URL` and the exact request object produced by the public discovery schema as `INFRAI_VECTOR_QUERY_JSON`. That makes the sample runnable without freezing undocumented request fields into application code.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;
const queryJson = process.env.INFRAI_VECTOR_QUERY_JSON;

if (!apiKey || !baseUrl || !queryJson) {
  throw new Error(
    "Set INFRAI_API_KEY, INFRAI_BASE_URL, and INFRAI_VECTOR_QUERY_JSON",
  );
}

const sleep = (milliseconds: number) =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function queryVectorStore(attempt = 0): Promise<unknown> {
  const response = await fetch(`${baseUrl}/v1/vector/query`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    },
    body: queryJson,
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await sleep(delayMs);
    return queryVectorStore(attempt + 1);
  }

  const body = await response.text();
  if (!response.ok) {
    throw new Error(`Vector query failed (${response.status}): ${body}`);
  }

  return JSON.parse(body) as unknown;
}

queryVectorStore().then((result) => console.log(JSON.stringify(result, null, 2)));
```

The body remains data rather than an invented example because the discovery response is the authority for the current request schema. Keep the documented output dimension in the same deployment configuration as the collection name, then reject startup when the configured values disagree. One check can prevent a long ingestion job from failing after work has already begun.

## A one-key design, diagrammed in words

Picture the request path from left to right: product page or FAQ entry -> stable document ID -> chunker -> embedding model -> dimension gate -> vector upsert. At query time: shopper question -> the same embedding generation -> vector query -> retrieved FAQ passages -> answer generator. Logs branch off at the dimension gate and at each write or query; counters track attempted vectors, accepted vectors, rejected vectors, and rebuild progress.

The stable document ID matters because product copy changes. Upserts should replace the intended chunk instead of quietly making duplicates. The model generation belongs in metadata so a query vector from one generation is never compared with stored vectors from another. These are application responsibilities even when one provider supplies both services.

Consider a catalog update that changes a laptop's warranty answer while leaving its title and specifications untouched. A page-level ID is too coarse if three independently retrieved chunks all inherit it; a random ID is worse because the revised warranty can sit beside the stale warranty after an upsert. A stable ID derived from the product record, content field, and chunk position gives the writer a traceable unit to replace. The ingestion log can then say that revision 18 produced four chunks, three retained their content hash, and one required a new embedding. During a model migration, those same source coordinates feed the new collection generation while the live generation remains queryable. This example does not predict a vendor's billing behavior. It exposes the work the application will cause, which is the number procurement and operations both need.

Stale answers are expensive too.

Infrai is one fit for the unified-account shape: its broader backend surface uses one key and one REST contract across 295 routes in 20 modules. For this workflow, the relevant advantage is narrower and more useful: embedding and vector operations can stay behind one credential, while its public discovery surface exposes request schemas and runnable TypeScript examples. That reduces integration boundaries. It does not remove the need to version collections or plan a rebuild.

Keep observability close to the contract. Record the collection generation, configured dimension, source revision, chunk count, and request ID with each ingestion batch. Alert on rejected writes and on a rebuild that stops making progress. Do not log raw shopper questions or retrieved account content by default; product search telemetry should be useful without becoming a second copy of sensitive text.

## Should one API key cover the embedding plus vector store combo?

There is no universal winner. The fair comparison is between an integrated surface and deliberately composed services, with index work held constant.

| Combination | Account and auth boundary | Dimension contract | Best reason to choose it | Cost boundary to inspect |
| --- | --- | --- | --- | --- |
| Infrai embeddings + Infrai vector storage | One account and credential | Checked within one provider workflow | A small team wants one operational surface across backend capabilities | Confirm embedding work, vector writes, storage, and rebuilds |
| OpenAI embeddings + Pinecone | Separate embedding and vector accounts | Application configuration joins model output to index dimension | The team explicitly wants Pinecone's vector database product and accepts another integration | Review both vendors plus full reindex traffic |
| OpenAI embeddings + Qdrant | Separate services; Qdrant may be operated or consumed as a service | Application configuration joins vector size to the collection | The team values Qdrant's deployment choices enough to own the boundary | Include hosting and operations as well as embedding calls |
| Cohere embeddings + Weaviate | Separate services when Cohere supplies embeddings | Application configuration joins model output to collection settings | The team has evaluated this pairing on its own product queries | Include both service boundaries and every model migration |

The table intentionally avoids volatile unit prices. Index economics are more durable when expressed as work: how many chunks are embedded, how many vectors are retained, how often content changes, and how many parallel generations a migration requires. Then apply current vendor pricing to that workload during procurement. This keeps the decision reviewable after a price sheet changes.

Count the work first.

Pinecone, Qdrant, and Weaviate are not interchangeable labels. Their operating models and product surfaces differ, so read their current documentation and test the exact feature that justifies splitting the stack. OpenAI and Cohere likewise expose distinct embedding offerings. A serious evaluation uses the same FAQ corpus and retrieval judgments for every combination; it does not assume that matching dimensions implies matching relevance.

My decision rule is strict: add the second account only when its measured retrieval behavior, required filtering, deployment control, or operational ownership is worth the extra auth path. Without that concrete requirement, the integrated option is easier to reason about. Fewer seams also mean fewer dashboards during an ingestion incident.

## The copyable rollout plan

Begin with a representative slice of product content, including terse titles, long descriptions, variant-specific facts, and policy answers that are easy to confuse. Freeze that evaluation set. Define stable source IDs and a deterministic chunking version before generating vectors.

Next, create a collection whose name carries the embedding generation. Store four pieces of metadata with each chunk: source ID, source revision, chunk version, and embedding generation. Validate the configured dimension before the first batch. Then ingest in bounded batches and make each batch retry-safe through stable vector IDs.

Now test retrieval with actual onboarding questions. Judge whether the returned passages contain the answer, not whether the final prose sounds plausible. Track failures by category: missing source content, poor chunk boundary, weak semantic match, stale revision, or wrong generation. That split tells the team whether to edit content, change chunking, evaluate another embedding model, or tune retrieval.

Finally, build the next generation alongside the current collection. Compare it on the frozen question set. Promote it only after its coverage and operating envelope are acceptable, and retain a clear rollback target until the change is settled.

Four steps. No mystery.

## What if one provider becomes a constraint?

A single account reduces integration work, but concentration is still a trade-off. Preserve your escape path in the data model: keep canonical source text outside the vector index, use stable IDs, record the embedding generation, and make collection selection configurable. Those choices turn migration into a planned reindex instead of a content recovery project.

Do not promise a zero-reindex model swap. The vector dimension may change, and even an equal dimension does not make two embedding spaces compatible. Build a fresh collection, regenerate embeddings from canonical content, and evaluate before switching reads.

The objection behind this question is often vendor lock-in. The sharper question is how expensive a controlled exit would be. If the source corpus, IDs, chunker, and evaluation set remain yours, the unavoidable work is visible: provision the target, embed again, load again, test, and switch. The architecture has done its job when that sequence is boring.

## Is one credential enough for production readiness?

No. One credential solves credential sprawl; it does not solve retrieval quality, privacy, or operations.

Rotate the key through a secret manager and scope its use to the ingestion and query services that need it. Separate staging collections from production. Put a hard dimension check before writes. Use exponential backoff for rate limits, cap retries, and expose failures instead of dropping them. For write retries, stable vector IDs or an idempotency mechanism must prevent duplicate application.

Then watch the system. The minimum useful dashboard shows source records discovered, chunks produced, embeddings attempted, vectors accepted, rejected writes, batch latency, and the active collection generation. A rollout alert should fire on rejection growth or stalled progress, not merely on generic request volume. Crisp signals make the before-and-after migration understandable to whoever is on call.

For an onboarding FAQ bot, the recommendation remains simple: start with embeddings and vector storage under one account, make dimension and generation explicit, and price the full index lifecycle. Split vendors only for a requirement that survives a side-by-side evaluation. The credential count is visible on day one. Reindex discipline is what keeps the design healthy on day 500.

## References

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [OpenAI embeddings documentation](https://platform.openai.com/docs/guides/embeddings)
- [Pinecone index concepts](https://docs.pinecone.io/guides/indexes/understanding-indexes)
- [Qdrant collections](https://qdrant.tech/documentation/concepts/collections/)
- [Weaviate vector configuration](https://docs.weaviate.io/weaviate/config-refs/schema/vector-index)
- [Cohere embeddings documentation](https://docs.cohere.com/docs/embeddings)
