---
layout: post
title: "Building a Production RAG API with Vector Search"
date: 2026-04-14
categories: ai
---

Most RAG tutorials stop at "retrieve documents, pass to LLM, return answer." That's about 20% of what a production deployment actually requires. The remaining 80% is concurrency control, adaptive retrieval, structured citations, and graceful degradation when your upstream services throttle you.

I built this for an internal documentation assistant — engineers asking questions about proprietary system docs that couldn't be indexed by public LLMs. The corpus was ~2,000 pages of technical documentation across PDFs, markdown, and HTML. Traffic was modest (a few hundred queries per day) but bursty — entire teams would hit it during incidents, which is exactly when you can't afford it to fall over.

The stack: FastAPI, a columnar database with vector search capabilities, an embedding model behind an API gateway, and an LLM for generation. Everything async, everything behind rate limits.

## Adaptive retrieval depth

Not every query needs the same number of documents. "What port does the connector use?" needs 3 chunks. "Explain all the differences between mode A and mode B" might need 10.

I started with a fixed k=5 and quickly saw two failure modes: simple lookups returned irrelevant padding documents that confused the LLM, and complex questions missed critical context because 5 chunks weren't enough.

```python
def determine_k(query: str) -> int:
    length = len(query)
    if length < 30:
        k = 3
    elif length <= 100:
        k = 5
    else:
        k = 7
    if COMPLEXITY_PATTERN.search(query):
        k += 2
    return max(2, min(k, 10))
```

The complexity pattern matches terms like "compare", "list all", "comprehensive", "every". It's a crude heuristic — query length is a weak proxy for complexity. But it eliminated the worst cases of over-retrieval and under-retrieval without adding an LLM call to classify the query (which would double latency for every request).

## Multi-query retrieval with early exit

Single-query retrieval has a blind spot: if the user's phrasing doesn't match how the source documents express the concept, you miss relevant chunks. Multi-query generates rephrased variants and merges results.

The problem: multi-query is expensive. An extra LLM call for rephrasing plus N additional vector searches. For most queries the original phrasing works fine — you don't want to pay that cost unconditionally.

```python
original_results = await self._search_async(query, k)

if high_confidence(original_results, threshold=0.7):
    rewrite_task.cancel()
    docs = [doc for doc, score in original_results if score >= min_similarity]
else:
    variants = await rewrite_task
    docs = merge_results(result_sets, min_similarity)
```

The query rewrite and original retrieval run concurrently from the start. If the original results score above threshold on at least 3 documents, we cancel the rewrite and skip additional searches. In practice, ~70% of requests take the fast path. The remaining 30% — usually jargon-heavy or vaguely phrased queries — benefit measurably from the rephrased variants.

The early-exit pattern means multi-query adds zero latency to the majority path while still catching the long tail of poorly-phrased queries.

## Concurrency control

The vector database and the LLM gateway both had hard connection limits. During an incident, 15 engineers would simultaneously ask questions about the same system, and without backpressure the service would exhaust connection pools and cascade into 429s from the LLM gateway.

```python
self._hana_semaphore = asyncio.Semaphore(4)   # max concurrent DB queries
self._aicore_semaphore = asyncio.Semaphore(3)  # max concurrent LLM calls
```

```python
async def _search_async(self, query: str, k: int):
    async with self._hana_semaphore, self._aicore_semaphore:
        return await with_retry(self._search, query, k)
```

Deliberately conservative. Under load, requests queue at the semaphore rather than hammering upstream. The alternative — letting all requests through and handling 429s reactively — creates worse tail latency because retries compound with each other.

The retry layer underneath handles rate limits with exponential backoff, respecting `Retry-After` headers:

```python
async def with_retry(fn, *args, max_retries=3):
    for attempt in range(max_retries + 1):
        try:
            return await asyncio.to_thread(fn, *args)
        except RateLimitError as exc:
            if attempt == max_retries:
                raise
            retry_after = float(
                exc.response.headers.get("Retry-After", 0)
            ) if exc.response else 0
            delay = max(retry_after, 0.5 * (2 ** attempt))
            await asyncio.sleep(delay)
```

The `asyncio.to_thread` is important — the SDK clients are synchronous, so blocking calls go to the thread pool to keep the event loop responsive for other requests.

## Structured citations

A RAG answer without citations is just a hallucination with extra steps. Early user feedback was clear: "I don't trust this answer unless I can verify it." So the system injects source headers that the LLM can reference:

```python
def format_docs(docs: list[Document]) -> str:
    parts = []
    for doc in docs:
        meta = doc.metadata
        name = meta.get("document_name", "Unknown")
        page = meta.get("page")
        chapter = meta.get("chapter")

        if page is not None:
            ref = f"Chapter: {chapter}, Page {page}" if chapter else f"Page {page}"
            parts.append(f"[Source: {name}, {ref}]\n{doc.page_content}")
        else:
            parts.append(doc.page_content)
    return "\n\n".join(parts)
```

Documents without pagination metadata (parsed markdown files) are included without a source header. The prompt explicitly instructs the LLM not to cite them — this prevents fabricated page numbers. The trade-off: some answers lack citations even when the information is correct. We accepted this over the alternative of the LLM inventing "Page 47" for a markdown file.

## Streaming with error boundaries

The response streams token-by-token. This creates an error handling problem: once you've started streaming, you can't return a JSON error response. The HTTP status is already 200.

```python
async def retrieve_stream(self, query: str) -> AsyncGenerator[str, None]:
    try:
        docs = await self._retrieve(query)
    except DatabaseError:
        yield "Sorry, an error occurred while searching."
        return

    try:
        async for chunk in self.document_chain.astream({...}):
            yield chunk
    except Exception:
        yield "\n\nSorry, an error occurred while generating the answer."
```

Retrieval errors fail fast — if the database is unreachable, the user gets an immediate message rather than waiting for a timeout. Generation errors degrade gracefully: partial answers are already on the wire, so we append an error notice. Users preferred seeing a partial answer with an error notice over getting nothing.

## Timeouts as graceful degradation

Each sub-operation has an independent timeout tuned to its importance:

- **Query rewrite:** 500ms. If the LLM takes longer to rephrase, proceed with the original phrasing. The rewrite is an optimization, not a requirement.
- **Variant retrieval:** 1 second per variant. If one rephrased query's retrieval is slow, merge results from whichever variants completed.

```python
async def _search_async_with_timeout(self, query: str, k: int):
    return await asyncio.wait_for(
        self._search_async(query, k), timeout=self.retrieval_timeout
    )
```

Under normal load, everything completes within budget. Under burst load (incident-driven traffic), the enhancements gracefully shed while the core path — original query retrieval + generation — always completes. Users get a slightly less optimized answer rather than a timeout error.

## What I learned

The production gap in RAG isn't retrieval quality — it's operational resilience. The entire service is under 300 lines. Most of that is plumbing: semaphores, retries, timeouts, error boundaries. The actual RAG logic is maybe 40 lines.

Things I'd do differently next time: add a response cache keyed on query embedding similarity (many incident-driven questions are near-duplicates), instrument retrieval quality metrics (track how often multi-query actually changes the result set), and add a fallback path that returns raw document snippets without LLM generation when the gateway is fully saturated. An imperfect answer fast beats a perfect answer never.
