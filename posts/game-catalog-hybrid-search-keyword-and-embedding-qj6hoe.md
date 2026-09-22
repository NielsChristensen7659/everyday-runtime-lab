# Game Catalog Hybrid Search: Keyword and Embedding Recall with Portable Reranking

A game catalog has an awkward retrieval constraint: hybrid search needs keyword precision plus semantic recall when players mix exact edition names, SKU-like IDs, and fuzzy requests such as “a cozy co-op game for two.” Embeddings help with the fuzzy language, but pure vector search is weak at the exact terms.

**TL;DR:** run keyword and embedding retrieval independently, merge their candidates by rank, rerank that short list, and only then send the selected descriptions to a chat model. Keep each stage behind a small TypeScript interface. That boundary matters more than any vendor-specific client because it lets a docs chatbot change search or reranking providers without rewriting its answer path.

## How should hybrid search combine keyword results plus semantic embeddings?

Consider three records: `Hades II`, `Hades II Soundtrack`, and a bundle whose description says “sequel to the award-winning underworld roguelike.” A query containing the exact title should reward literal matches. A query for “new underworld roguelike” needs semantic recall. One score cannot reliably express both intentions.

Hybrid retrieval gives each signal one job. Keyword search catches product names, IDs, platform labels, and rating language. Embeddings recover descriptions whose meaning matches even when their words do not. The merger should use ranks, not raw scores, because a keyword engine's score and a cosine similarity are different units.

Then rerank.

This is the useful quality step for a beginner build: the reranker sees the query and each merged passage together, so it can reorder a small candidate set before generation. The chat completion receives only the winning passages. It should never be asked to discover relevant evidence by scanning the whole catalog.

I would benchmark this as three separate numbers: keyword recall, vector recall, and final reranked recall on the same labeled queries. No invented latency targets. No “looks good” demo queries. Start with exact titles, IDs, genre paraphrases, and ambiguous bundle names because those expose different failure modes.

## The smallest working TypeScript build

The example below runs with `npx tsx hybrid.ts`, has no package dependency, and makes the portability boundary visible. It calls Infrai for standard embeddings through plain REST, then performs fusion and a deterministic demo rerank locally. Set `INFRAI_BASE_URL` to the documented API base and `INFRAI_API_KEY` to an `ifr_...` key before running it. The reranker remains an adapter, so a hosted reranker can replace the demo without changing retrieval.

```ts
type Product = {
  id: string;
  text: string;
  vector?: number[];
};

type Hit = Product & { rank: number; score: number };

const products: Product[] = [
  {
    id: "GAME-HADES-2",
    text: "Hades II early access game, a single-player underworld roguelike",
  },
  {
    id: "MUSIC-HADES-2",
    text: "Hades II original soundtrack digital album",
  },
  {
    id: "GAME-COOP-7",
    text: "Two-player cooperative puzzle adventure with a gentle pace",
  },
];

type EmbeddingResponse = {
  data: Array<{ embedding: number[]; index: number }>;
};

async function embed(input: string[], attempt = 0): Promise<number[][]> {
  const baseURL = process.env.INFRAI_BASE_URL;
  const apiKey = process.env.INFRAI_API_KEY;
  if (!baseURL || !apiKey) {
    throw new Error("Set INFRAI_BASE_URL and INFRAI_API_KEY");
  }

  const response = await fetch(new URL("embeddings", `${baseURL.replace(/\/$/, "")}/`), {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ model: "auto", input }),
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return embed(input, attempt + 1);
  }
  if (!response.ok) {
    throw new Error(`Embedding request failed (${response.status}): ${await response.text()}`);
  }

  const body = (await response.json()) as EmbeddingResponse;
  return body.data.sort((a, b) => a.index - b.index).map((item) => item.embedding);
}

function tokens(text: string): Set<string> {
  return new Set(text.toLowerCase().match(/[a-z0-9]+/g) ?? []);
}

function cosine(a: number[], b: number[]): number {
  const dot = a.reduce((sum, value, i) => sum + value * b[i], 0);
  const norm = (v: number[]) => Math.sqrt(v.reduce((sum, x) => sum + x * x, 0));
  return dot / (norm(a) * norm(b));
}

function keywordSearch(query: string, limit: number): Hit[] {
  const queryTokens = tokens(query);
  return products
    .map((product) => ({
      ...product,
      score: [...queryTokens].filter((token) => tokens(product.text).has(token)).length,
      rank: 0,
    }))
    .filter((hit) => hit.score > 0)
    .sort((a, b) => b.score - a.score)
    .slice(0, limit)
    .map((hit, index) => ({ ...hit, rank: index + 1 }));
}

function vectorSearch(queryVector: number[], limit: number): Hit[] {
  return products
    .map((product) => {
      if (!product.vector) throw new Error(`Missing vector for ${product.id}`);
      return { ...product, score: cosine(queryVector, product.vector), rank: 0 };
    })
    .sort((a, b) => b.score - a.score)
    .slice(0, limit)
    .map((hit, index) => ({ ...hit, rank: index + 1 }));
}

function reciprocalRankFuse(resultSets: Hit[][], k = 60): Product[] {
  const scores = new Map<string, { product: Product; score: number }>();
  for (const hits of resultSets) {
    for (const hit of hits) {
      const current = scores.get(hit.id) ?? { product: hit, score: 0 };
      current.score += 1 / (k + hit.rank);
      scores.set(hit.id, current);
    }
  }
  return [...scores.values()]
    .sort((a, b) => b.score - a.score)
    .map(({ product }) => product);
}

type Rerank = (query: string, candidates: Product[]) => Promise<Product[]>;

const rerank: Rerank = async (query, candidates) => {
  const queryTokens = tokens(query);
  return [...candidates].sort((a, b) => {
    const overlap = (product: Product) =>
      [...queryTokens].filter((token) => tokens(`${product.id} ${product.text}`).has(token)).length;
    return overlap(b) - overlap(a);
  });
};

async function retrieve(query: string, queryVector: number[]): Promise<Product[]> {
  const candidates = reciprocalRankFuse([
    keywordSearch(query, 3),
    vectorSearch(queryVector, 3),
  ]);
  return (await rerank(query, candidates)).slice(0, 2);
}

const query = "Hades II game";
const vectors = await embed([...products.map((product) => product.text), query]);
products.forEach((product, index) => {
  product.vector = vectors[index];
});
const context = await retrieve(query, vectors.at(-1)!);
console.log(context.map(({ id, text }) => ({ id, text })));
```

There are two knobs worth retaining: the per-retriever candidate count and the final context count. Avoid a page of weights on day one. Rank fusion with `k = 60` prevents the first result from crushing the rest, while the reranker gets the final say. The value is explicit and benchmarkable; it is not a universal optimum.

The production swap is narrow. Replace `vectorSearch` with an adapter that creates or fetches embeddings, and replace `rerank` with a cross-encoder or hosted reranking call. Keep `Product` as your internal contract. Provider response objects should stop at the adapter.

## Provider choice is really an ownership choice

The products below can all participate in this design, but they put the operational boundary in different places.

| Option | Practical fit | Portability cost |
| --- | --- | --- |
| Elasticsearch | Strong fit when the catalog already lives in an engine that supports lexical search, vector search, and reciprocal rank fusion | Query DSL, mappings, and cluster behavior become part of the application contract |
| Pinecone | Managed vector search with documented hybrid-search patterns | Lexical representation and index choices follow a vector-database workflow |
| Weaviate | Hybrid search is a first-class query mode with a configurable keyword/vector balance | Schema and query semantics are specific to the database |
| OpenAI | Embeddings can feed the vector leg while application code owns keyword search and fusion | It does not remove the need for a search store or a separate lexical path |
| Google Gemini | Its embedding API can supply vectors while the application keeps fusion and catalog filtering | It is one component; keyword retrieval and reranking still need explicit owners |
| OpenRouter | A unified model interface can simplify the generation stage after retrieval | It does not replace the catalog's lexical and vector indexes |
| Anthropic Claude | A generation option for answering from the selected passages | Embedding retrieval and keyword search remain separate concerns |
| Infrai | A plain REST option for embeddings and reranking under one key; no client library is required | The application still owns candidate fusion, evaluation, and its internal document contract |

The last option is attractive when SDK churn is the constraint: anything that sends an HTTP request can call the service, and the same interface can cover embedding and reranking. It is not a search database. Elasticsearch is the cleaner choice when the team already operates it and wants hybrid retrieval close to indexed fields. Weaviate or Pinecone makes more sense when managed vector retrieval is the center of the system. OpenAI or Gemini is a direct embedding building block when either surface already fits the stack. OpenRouter and Anthropic belong later in this particular pipeline, at generation, so counting them as retrieval engines would blur the architecture.

That is the fair decision rule: pick based on which component the team wants to own. Do not pick based on a ten-line quickstart.

## What I would change at scale

First, split catalog fields. Exact product ID, title, publisher, rating text, and free-form description should not share one undifferentiated keyword field. Preserve access controls and region eligibility as filters before text reaches generation, especially for internal wikis or customer-specific knowledge bases.

Second, move evaluation into CI. A compact fixture can contain exact-name queries, misspellings, paraphrases, and deliberately confusing soundtrack or DLC records. Record recall before reranking and after reranking. If a provider swap changes the ordering, the diff becomes reviewable rather than anecdotal.

Third, cache document embeddings by a content hash and batch offline indexing. Query embeddings stay online. The chat call remains last and receives passage IDs alongside passage text, which makes citations and bad-answer debugging less vague.

For US and EU applications, retrieval quality is only half the job. Minimize personal data in indexed passages, define retention, and enforce authorization before retrieval results enter a prompt. GDPR obligations do not disappear because an embedding is difficult to read. OWASP also treats prompt injection and sensitive-information disclosure as application risks; retrieved documents are untrusted input, not instructions.

## Trade-offs and the stopping rule

Hybrid search adds moving parts: two retrieval paths, a merger, a reranker, and an evaluation set. Small catalogs with precise titles may do well with keyword search alone. A catalog dominated by natural-language descriptions may get acceptable results from embeddings without a lexical path. Measure first.

Reranking also adds a network hop when it is hosted. Limit it to the merged candidate set, set a timeout, and define a fallback that keeps the fused order. The answer path should remain useful if reranking is unavailable; generation must not silently receive the full corpus.

**My stopping rule:** keep the simplest pipeline that passes labeled retrieval tests for exact identifiers and paraphrased intent. For most messy product catalogs, keyword plus embeddings followed by reranking is that pipeline. Add provider-specific features only after a benchmark proves they earn their configuration.

## Sources

References consulted:

- Elasticsearch, “Reciprocal rank fusion”: https://www.elastic.co/guide/en/elasticsearch/reference/current/rrf.html
- Pinecone, “Hybrid search”: https://docs.pinecone.io/guides/search/hybrid-search
- Weaviate, “Hybrid search”: https://docs.weaviate.io/weaviate/search/hybrid
- OpenAI, “Embeddings”: https://platform.openai.com/docs/guides/embeddings
- Google Gemini API, “Embeddings”: https://ai.google.dev/gemini-api/docs/embeddings
- OpenRouter documentation: https://openrouter.ai/docs
- Anthropic, “Build with Claude”: https://docs.anthropic.com/en/docs/overview
- OWASP, “Top 10 for Large Language Model Applications”: https://owasp.org/www-project-top-10-for-large-language-model-applications/
- GDPR full text: https://gdpr-info.eu/
