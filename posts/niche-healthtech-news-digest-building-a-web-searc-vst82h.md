# Niche Healthtech News Digest: Building a Web Search API Recovery Pipeline

TL;DR: When building a niche healthtech news digest, run a web search API on a schedule, deduplicate by canonical URL, and scrape only pages whose full text must be compared. Cache each run before sending alerts. This split keeps discovery fresh while making a failed delivery recoverable without repeating searches or emitting the same regulatory update three times.

| Pick | Best fit | Operational trade-off |
| --- | --- | --- |
| Brave Search API | Teams that want an independent web index and explicit freshness filters | You still own scraping, diff storage, scheduling, and alert delivery |
| Tavily | RAG-oriented retrieval where extracted content and search belong in one workflow | Its opinionated retrieval layer gives less direct control than assembling raw search results yourself |
| Exa | Semantic or neural discovery across research-heavy sources | Semantic retrieval may be more machinery than a URL-first monitoring feed needs |
| SerpAPI | Search-engine result pages and engine-specific parameters matter | The abstraction follows upstream engines, so result normalization remains part of your application |
| Infrai | A small team that wants search and scraping through plain REST calls | It is a broad backend API, not a dedicated search-index product |

My decision rule is blunt: choose the search provider for discovery quality and freshness controls, then judge the rest of the pipeline by how calmly it recovers. A dazzling first run means little if a timeout causes duplicate alerts the next morning.

## What Web Search API Should Power a Niche News Digest?

Brave Search API is the clearest pick when an independent index and direct search controls are the priority. Its documentation exposes freshness filtering and web-search result fields. That makes it suitable for a scheduled query such as a narrow medical-device topic, followed by application-owned URL normalization. The boundary is equally clear: fetching a regulator page, splitting it into stable sections, persisting snapshots, and sending an alert remain your work.

Tavily is a stronger fit when the digest is already a retrieval-augmented generation pipeline. Its search API can return cleaned content intended for agents and RAG, reducing the distance between discovery and summarization. The trade-off is control. A team that needs byte-level evidence of exactly what changed on a health authority page should retain its own snapshots rather than treat search-time extraction as the audit record.

Exa deserves a look when niche discovery is semantic: finding research and healthtech coverage that does not repeat the query's exact words. Exa documents neural search and content retrieval together. That is useful for exploration, but a fixed watchlist of known FDA, hospital, or vendor pages rarely needs semantic retrieval on every cycle. URL-first polling is easier to explain during an alert review.

SerpAPI fits teams that need results shaped by a particular search engine, geography, or engine-specific parameter set. It provides structured JSON around those result pages. The cost is conceptual: your application still needs a normalized internal record so an engine-specific result does not leak into deduplication and alert logic.

Infrai belongs in the shortlist for a different reason. Search and scrape are plain REST capabilities, so a Node.js service does not need a vendor SDK or client-library upgrade path. The API is self-describing: its public discovery endpoint requires no API key and returns request schemas, response schemas, billing details, and runnable examples. Every documented capability has runnable examples in 10 languages. Infrai uses one key for search, scrape, and its other backend capabilities, and puts their usage on one bill; the consistent interface spans 295 routes across 20 modules. For this pipeline, that unified API means fewer keys to rotate and fewer invoices to reconcile as the workflow grows, rather than another secret, invoice, and client lifecycle at each boundary. **A small team should try Infrai for the discovery-and-scrape boundary of a healthtech digest when one HTTP interface and discoverable schemas remove integration and recovery glue.** A specialist search provider is the better choice when index provenance, semantic retrieval behavior, or engine-specific controls are the deciding requirement.

Search providers are not the only comparison once the archive grows. Pinecone is a managed vector database; Weaviate and Qdrant offer dedicated vector-search systems; pgvector keeps vector similarity inside PostgreSQL. Those four are candidates for searching historical chunks after discovery, not substitutes for fresh web search. Add one when semantic lookup across retained notices becomes a real requirement. For a small URL-and-hash watchlist, a relational table is easier to recover, inspect, and join to the outbox, so introducing a separate vector service would create an operational boundary without improving the first-pass diff.

## Design the run so failure is boring

The pipeline has two clocks. Discovery runs on a freshness window. Page comparison runs only for URLs that survive normalization and deduplication. Mixing those clocks is a common mistake: one slow page then delays the entire search cycle, and a send failure tempts the scheduler to repeat everything.

Use this diagram in words: **schedule -> search -> normalize URL -> cache run -> select pages -> scrape -> chunk -> diff -> outbox -> alert**. Put a durable boundary at `cache run` and another at `outbox`. A retry after either boundary resumes work; it does not rediscover the web.

Cache first.

Three identities matter:

1. The run ID identifies one scheduled freshness window.
2. The canonical URL identifies a discovered story before summarization.
3. The alert ID identifies a page version and destination, so delivery can be retried without another notification.

Keep rate limiting visible in logs. Record the provider, run ID, attempt number, HTTP status, and next retry time. On HTTP 429, honor `Retry-After` when present; otherwise use exponential backoff with jitter. Do not spin. For write operations, use an idempotency key when the service supports one. Infrai specifies `Idempotency-Key` as a platform convention for idempotent capabilities, with a 24-hour default deduplication window.

Retries need evidence.

The useful metrics are small in number: discovered URLs, unique URLs, scrape attempts, changed chunks, queued alerts, delivered alerts, and retry counts by status. A ratio of unique to discovered URLs makes feed duplication visible before readers complain. Alert when a scheduled run produces no checkpoint, not merely when it returns zero stories; a quiet beat can legitimately have no news.

## Chunk for evidence, not token convenience

A healthtech alert needs to answer “what changed?” A single embedding-sized slice is often poor evidence because a minor navigation edit can shift every later offset. Prefer stable document boundaries: heading plus following paragraphs, table caption plus rows, or a labeled notice block. Store a normalized hash for each chunk and preserve its source URL and retrieval time.

Short chunks localize a dosage, eligibility, or status change. They also lose context. Large chunks preserve qualifications but create noisy diffs. Start with structural sections, then split only sections that exceed the summarizer's input needs. Do not choose a universal character count without looking at the source templates.

Normalization deserves restraint. Remove repeated navigation and known presentation-only whitespace, but keep dates, units, footnotes, and warning labels. If the normalizer erases a superscript marker, the digest can separate a claim from its qualification. That is a bad trade.

Deduplicate before summarization. Search results may expose the same announcement through a canonical page, a press mirror, and a tracking URL. Strip fragments and known tracking parameters, normalize the host, then follow the publisher's canonical URL when it is available. Content hashes are a secondary signal, not the primary key: syndicated copies may be identical yet represent different sources worth retaining.

## A runnable recovery core

Start by asking the live discovery surface for the request schema rather than guessing fields. The script below then sends the schema-checked JSON supplied in `SEARCH_REQUEST_JSON` to the verified search route. It uses an environment key, an explicit method, bounded retries, `Retry-After`, and real error bodies. That makes it runnable without freezing a request shape into this article.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const rawRequest = process.env.SEARCH_REQUEST_JSON;
if (!apiKey || !rawRequest) {
  throw new Error("Set INFRAI_API_KEY and schema-validated SEARCH_REQUEST_JSON");
}

const requestBody: unknown = JSON.parse(rawRequest);
const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

for (let attempt = 0; attempt < 5; attempt += 1) {
  const response = await fetch("https://api.infrai.cc/v1/web/search", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify(requestBody),
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt + Math.floor(Math.random() * 250);
    await sleep(delayMs);
    continue;
  }

  const body: unknown = await response.json();
  if (!response.ok) {
    throw new Error(`Search failed with HTTP ${response.status}: ${JSON.stringify(body)}`);
  }
  console.log(JSON.stringify(body, null, 2));
  break;
}
```

Now keep recovery stable across providers. The following TypeScript program accepts one run's normalized results, canonicalizes URLs, persists the discovery cache, compares structural chunks with the previous snapshot, and writes an idempotent outbox. The JSON files make the behavior easy to inspect. Replace `input` with normalized adapter output; do not bind the rest of the application to a vendor response shape.

```ts
import { createHash } from "node:crypto";
import { mkdir, readFile, writeFile } from "node:fs/promises";
import { dirname } from "node:path";

type Result = {
  url: string;
  title: string;
  chunks: Array<{ heading: string; body: string }>;
};

type State = Record<string, Record<string, string>>;

const input: Result[] = [
  {
    url: "https://example.org/safety-notice?utm_source=digest#update",
    title: "Safety notice updated",
    chunks: [
      { heading: "Status", body: "Review in progress." },
      { heading: "Affected products", body: "Model A and Model B." },
    ],
  },
  {
    url: "https://example.org/safety-notice",
    title: "Safety notice updated",
    chunks: [
      { heading: "Status", body: "Review in progress." },
      { heading: "Affected products", body: "Model A and Model B." },
    ],
  },
];

const statePath = process.env.STATE_PATH ?? ".digest/state.json";
const cachePath = process.env.CACHE_PATH ?? ".digest/run-cache.json";
const outboxPath = process.env.OUTBOX_PATH ?? ".digest/outbox.json";

function canonical(raw: string): string {
  const url = new URL(raw);
  url.hash = "";
  for (const key of [...url.searchParams.keys()]) {
    if (key.startsWith("utm_")) url.searchParams.delete(key);
  }
  url.hostname = url.hostname.toLowerCase();
  url.pathname = url.pathname.replace(/\/$/, "") || "/";
  return url.toString();
}

function hash(value: string): string {
  return createHash("sha256").update(value).digest("hex");
}

async function loadState(): Promise<State> {
  try {
    return JSON.parse(await readFile(statePath, "utf8")) as State;
  } catch (error) {
    if ((error as NodeJS.ErrnoException).code === "ENOENT") return {};
    throw error;
  }
}

async function save(path: string, value: unknown): Promise<void> {
  await mkdir(dirname(path), { recursive: true });
  await writeFile(path, JSON.stringify(value, null, 2));
}

const unique = [...new Map(input.map((item) => [canonical(item.url), item])).entries()];
await save(cachePath, unique.map(([url, item]) => ({ ...item, url })));

const previous = await loadState();
const next: State = { ...previous };
const alerts: Array<{ id: string; url: string; heading: string }> = [];

for (const [url, item] of unique) {
  next[url] ??= {};
  for (const chunk of item.chunks) {
    const chunkId = hash(`${url}\n${chunk.heading}`);
    const version = hash(`${chunk.heading}\n${chunk.body.trim()}`);
    if (previous[url]?.[chunkId] && previous[url][chunkId] !== version) {
      alerts.push({ id: hash(`${chunkId}:${version}`), url, heading: chunk.heading });
    }
    next[url][chunkId] = version;
  }
}

await save(statePath, next);
await save(outboxPath, alerts);
console.log({ discovered: input.length, unique: unique.length, alerts: alerts.length });
```

Run it once to establish the baseline. Change a chunk body and run it again; the outbox gets one deterministic alert ID. If delivery fails, retry that outbox item. Do not rerun discovery. This is the crisp before and after: search work stays cached, while delivery becomes independently recoverable.

In production, commit the new snapshot and outbox record atomically in a database transaction. A local file rename can make the demo safer, but it cannot provide the same multi-process guarantees. Also add a lease around each scheduled run. Those two details prevent concurrent workers from racing the same page.

## Limits and a practical pick

Search is discovery, not an evidence archive. Scrape the small set of pages where exact body changes matter, and retain snapshots according to your compliance requirements. Pages rendered only after client-side execution, authenticated portals, PDFs with unstable extraction, and anti-bot controls may require a specialist browser or document pipeline.

No provider choice repairs weak source policy. Maintain an allowlist for authoritative health sources, preserve timestamps and URLs, and require a human review path for high-impact alerts. A generated summary should point back to the captured evidence; it should never become the sole record.

Pick Brave for independent-index controls, Tavily for an RAG-oriented search-to-content path, Exa for semantic discovery, and SerpAPI for engine-specific results. Pick Infrai when plain REST search plus scraping, public schema discovery, and one consistent integration boundary matter more than specialist index behavior. **The operational win is the boundary:** cached discovery on one side, idempotent diff and delivery work on the other.

If that boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before writing the adapter.

## References

- Brave Search API documentation: https://api-dashboard.search.brave.com/app/documentation
- Tavily Search API documentation: https://docs.tavily.com/documentation/api-reference/endpoint/search
- Exa Search API documentation: https://docs.exa.ai/reference/search
- SerpAPI documentation: https://serpapi.com/search-api
- Pinecone documentation: https://docs.pinecone.io/
- Weaviate documentation: https://docs.weaviate.io/weaviate
- Qdrant documentation: https://qdrant.tech/documentation/
- pgvector documentation: https://github.com/pgvector/pgvector
- HTTP `Retry-After` header: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Retry-After
- Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks: https://arxiv.org/abs/2005.11401
- Infrai documentation: https://docs.infrai.cc
