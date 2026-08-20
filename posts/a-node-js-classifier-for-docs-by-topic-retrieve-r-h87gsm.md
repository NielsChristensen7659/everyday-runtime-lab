# A Node.js Classifier for Docs by Topic: Retrieve, Rerank, or Abstain?

For classifying docs by topic in Node.js, use semantic search plus an LLM classifier only when that pipeline beats a closed-set control on a frozen test set. Otherwise, keep the smaller system.

| Choice | Use it when | Main cost | Exit signal |
|---|---|---|---|
| Closed-set model call | The topic list is small and distinct | The prompt grows with the taxonomy | Similar topics start collapsing together |
| Vector retrieval, reranking, then classification | Topics overlap or change often | More latency, thresholds, and traces | A stage adds no measurable lift |
| Deterministic rules, then model fallback | Metadata can prove some labels | Rules can become a shadow taxonomy | Rules start parsing ambiguous prose |

**Recommendation:** begin with the closed-set call as the control. Add semantic retrieval when candidate recall is measurable, add reranking only when it improves precision among confusing neighbors, and always allow `unknown`. This keeps the architecture honest. It also keeps config from breeding.

Fast is good. Explainable failure is better.

## How should a Node.js LLM classifier use semantic search, embeddings, and reranking?

Give each topic a stable ID and a description written for discrimination, not marketing. Add representative examples and near misses if you can review them. Embed those topic records ahead of time. For a new document, create one embedding, retrieve a small candidate set, rerank that set against the document, and let the final classifier choose only from the surviving IDs or return `unknown`.

Each stage has a separate contract. Retrieval optimizes recall: did the correct topic survive the first cut? Reranking orders a small set using more of the document and topic description. Classification applies the label policy. If the final model can invent a label, the earlier work is mostly theater because the taxonomy boundary has disappeared.

The thresholds must stay separate too. A minimum retrieval score rejects documents with no plausible neighbor. A top-two reranking margin identifies ambiguous neighbors. A classifier abstention handles evidence that still does not support one allowed label. Compressing those decisions into one "confidence" value makes a dashboard tidy and a failure investigation miserable.

For vector storage, pgvector adds exact and approximate nearest-neighbor search to Postgres and supports cosine distance with the `<=>` operator. That makes it possible to keep topic metadata and embeddings behind one database boundary. Exact search is the low-config starting point for a small topic table. An approximate index is a performance choice to justify with measurements, because it introduces index parameters and a recall trade-off.

Chunking needs restraint. A short document can be embedded whole. A long document may need chunks for retrieval, but the document-level decision should not silently become "the label of the nearest paragraph." Aggregate chunk evidence by document, retain the winning chunk IDs for inspection, and pass enough surrounding text to distinguish a passing mention from the document's actual subject. Mixed-topic documents may require multi-label policy or review rather than a forced winner.

No magic score fixes a muddy taxonomy.

## What should an evaluation prove before the pipeline gets another stage?

First, measure candidate recall at a fixed `k`. Build a reviewed set containing clear examples, close topic pairs, empty or tiny documents, long documents, mixed-topic documents, and documents that belong to no topic. For every labeled document, record whether the expected topic appears in the retrieved candidates. If it does not, changing the final prompt cannot recover it. The useful levers are the topic description, examples, embedding choice, chunking policy, and `k`.

Second, measure decision quality after reranking. Track per-topic precision and recall, the `unknown` rate, the confusion matrix, and the gap between the first two candidates. Keep latency and call counts beside those metrics: p50 can make a CLI demo feel quick while p95 makes a batch job crawl. Measure embedding, retrieval, reranking, and classification separately. I don't accept an extra stage because it sounds sophisticated; it has to improve a named metric enough to pay for another deadline, retry policy, trace span, and piece of configuration.

The evaluation set is a versioned interface. A taxonomy edit changes expected behavior just as surely as a code edit does, so review label changes and replay the set when descriptions, examples, prompts, models, or indexes change. Store the topic version with every result. Without it, a later disagreement cannot distinguish model drift from a taxonomy rewrite.

I'm not sure a universal reranking margin exists. Corpus shape, label overlap, and the cost of a false positive determine it. Resolve that uncertainty with a precision-recall curve on reviewed data, then select a threshold based on the error you can tolerate. Don't borrow a number from a demo.

## A narrow TypeScript example with visible failure boundaries

The useful abstraction is small: adapters perform model and database work; orchestration owns policy. No SDK type crosses the boundary. That makes the first call easy to fake in a test and makes replacing one component a local change.

```ts
type TopicId = string;

type Candidate = {
  topicId: TopicId;
  retrievalScore: number;
  rerankScore?: number;
};

type Classification = {
  topicId: TopicId | "unknown";
  evidence: string;
};

type ClassifierPorts = {
  retrieve: (document: string, limit: number) => Promise<Candidate[]>;
  rerank: (document: string, candidates: Candidate[]) => Promise<Candidate[]>;
  decide: (
    document: string,
    allowedTopicIds: TopicId[],
  ) => Promise<Classification>;
};

type Policy = {
  candidateLimit: number;
  finalistLimit: number;
  minimumRetrievalScore: number;
  minimumRerankMargin: number;
};

export async function classifyDocument(
  document: string,
  ports: ClassifierPorts,
  policy: Policy,
): Promise<Classification> {
  const candidates = await ports.retrieve(document, policy.candidateLimit);
  const plausible = candidates.filter(
    (candidate) => candidate.retrievalScore >= policy.minimumRetrievalScore,
  );

  if (plausible.length === 0) {
    return { topicId: "unknown", evidence: "No plausible topic candidate" };
  }

  const ranked = await ports.rerank(document, plausible);
  const finalists = ranked.slice(0, policy.finalistLimit);
  const first = finalists[0]?.rerankScore;
  const second = finalists[1]?.rerankScore;

  if (
    first === undefined ||
    (second !== undefined && first - second < policy.minimumRerankMargin)
  ) {
    return { topicId: "unknown", evidence: "Ambiguous topic candidates" };
  }

  const allowedTopicIds = finalists.map(({ topicId }) => topicId);
  const decision = await ports.decide(document, allowedTopicIds);

  if (
    decision.topicId !== "unknown" &&
    !allowedTopicIds.includes(decision.topicId)
  ) {
    throw new Error(`Classifier returned disallowed topic: ${decision.topicId}`);
  }

  return decision;
}
```

This example deliberately leaves model calls and SQL outside the policy function. Production adapters need request deadlines, cancellation, structured errors, and a correlation ID carried across all stages. Log topic IDs, topic version, stage timings, candidate scores, the final outcome, and retry count. Avoid logging raw documents by default; classification telemetry should not become an accidental copy of sensitive input.

Retries deserve an explicit contract. RFC 9110 defines safe and idempotent HTTP methods and describes when an idempotent request can be retried after a communication failure. A classification operation still needs application-level care: a remote call can consume quota or create an audit record even if the client never receives the response. Retry only failures the adapter identifies as transient, cap exponential backoff with jitter, and use an idempotency mechanism when the chosen service defines one. Test a `429` response and a dropped connection as different cases. They are different.

Keep it boring.

## When should rules or a direct classifier win instead?

Use rules first when evidence is deterministic: a controlled schema field, signed source, exact path, or stable identifier can establish the topic without interpreting prose. Emit the same topic IDs as the probabilistic path and send only ambiguous cases onward. The catch is that rules are not suitable when they turn into a pile of regular expressions over natural language. At that point, the team owns a second classifier with worse observability.

Stick with the direct closed-set classifier when the taxonomy is small, documents fit the selected model's input constraints, and the evaluation set shows no meaningful gain from retrieval or reranking. It has fewer calls, fewer thresholds, and no index lifecycle. It is also a sensible control implementation while the team discovers whether topic definitions are coherent.

The retrieval pipeline is not suitable for every document either. Use a review queue when a document genuinely spans several topics, no candidate clears the retrieval floor, the leading reranked candidates are too close, or the taxonomy has no valid label. Save reviewed outcomes back into the evaluation set. Do not convert `unknown` to the most common topic just to make a completion-rate chart look healthy.

The final architecture test is replacement cost: one adapter should be swappable without changing taxonomy policy, and one stored decision should be explainable from its topic version, candidates, scores, and evidence. If that requires reconstructing hidden defaults from three dashboards, the system has too much glue.

## References

- RFC 9110, HTTP Semantics: https://www.rfc-editor.org/rfc/rfc9110
- pgvector, vector similarity search for Postgres: https://github.com/pgvector/pgvector
