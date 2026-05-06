---
layout: post
title: "Building a Production RAG API with Vector Search"
date: 2026-04-14
categories: ai
---

Most RAG tutorials stop at "retrieve documents, pass to LLM, return answer." That's about 20% of what a production deployment actually requires. The remaining 80% is concurrency control, adaptive retrieval, structured citations, and graceful degradation when your upstream services throttle you.

This post covers the design of a RAG API I built — a FastAPI service backed by a columnar database with vector search, an embedding model, and an LLM accessed through an API gateway. The focus is on the engineering decisions that don't show up in starter templates.

## Architecture at a glance

The request flow is simple:

```
POST /ask → FastAPI → vector similarity search → LLM generation → streamed response
```

The service initializes all heavy resources (database connection, embedding model proxy, LLM client) once at startup via a lifespan handler. Each request shares these resources through concurrency-controlled paths rather than creating new connections per request.

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    rag_service.init()
    await rag_service.init_semaphores()
    yield
    rag_service.close()
```

Semaphores are created separately because they're bound to the running event loop — you can't instantiate them during synchronous initialization.

## Adaptive retrieval depth

Not every query needs the same number of documents. A short factual question like "what port does the connector use?" needs 3 documents at most. A comparative question like "explain all the differences between mode A and mode B" might need 7-10.

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

The complexity pattern matches terms like "compare", "list all", "comprehensive", "every". It's a simple heuristic, but it prevents wasting tokens on over-retrieval for simple lookups while ensuring complex questions get enough context.

## Multi-query retrieval with early exit

Single-query retrieval has a blind spot: if the user's phrasing doesn't match how the source documents express the concept, you'll miss relevant chunks. Multi-query retrieval generates rephrased variants and merges results.

The key insight is that multi-query is expensive (extra LLM call + extra vector searches), so we only do it when the original results are weak:

```python
original_results = await self._search_async(query, k)

if high_confidence(original_results, threshold=0.7):
    rewrite_task.cancel()
    docs = [doc for doc, score in original_results if score >= min_similarity]
else:
    variants = await rewrite_task
    # retrieve variants in parallel, merge and deduplicate
    docs = merge_results(result_sets, min_similarity)
```

The query rewrite and original retrieval run concurrently. If the original results score above threshold on at least 3 documents, we cancel the rewrite task and skip the additional searches entirely. This means multi-query adds zero latency for the majority of requests that already have strong matches.

## Concurrency control with semaphores

The vector database and the LLM gateway both have connection limits. Without backpressure, concurrent requests will exhaust connection pools and trigger cascading 429s.

Two semaphores gate access:

```python
self._hana_semaphore = asyncio.Semaphore(4)   # max concurrent DB queries
self._aicore_semaphore = asyncio.Semaphore(3)  # max concurrent LLM calls
```

Each search operation acquires both semaphores (because it calls the embedding API to vectorize the query, then queries the database):

```python
async def _search_async(self, query: str, k: int):
    async with self._hana_semaphore, self._aicore_semaphore:
        return await with_retry(self._search, query, k)
```

This is deliberately conservative. Under load, requests queue at the semaphore rather than hammering the upstream service and getting rate-limited. The retry layer underneath handles 429s with exponential backoff, respecting the `Retry-After` header when the gateway provides one.

## Retry with exponential backoff

The retry wrapper is intentionally minimal — it only catches rate limit errors, not general failures:

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

The `asyncio.to_thread` call is important — the underlying SDK clients are synchronous, so we push blocking calls to the thread pool to avoid stalling the event loop.

## Structured citations

A RAG answer without citations is just a hallucination with extra steps. The prompt enforces a strict citation format, and the document formatter injects source headers that the LLM can reference:

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

Documents without pagination metadata (like parsed markdown) are included without a source header — the prompt explicitly instructs the LLM not to cite them. This prevents fabricated page numbers for sources that don't have them.

## Streaming and error boundaries

The response streams token-by-token via `StreamingResponse`. This creates an error handling challenge: once you've started streaming, you can't return a JSON error response. Errors during generation are yielded as plain text at the end of the stream:

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

The retrieval phase fails fast — if the database is unreachable, the user gets an immediate error rather than waiting for a timeout. The generation phase degrades more gracefully: partial answers are already on the wire, so we append an error notice.

## Timeouts as feature flags

Each sub-operation has an independent timeout:

- **Query rewrite:** 500ms. If the LLM takes longer to rephrase the query, we proceed with the original phrasing only. The rewrite is an optimization, not a requirement.
- **Variant retrieval:** 1 second per variant. If one rephrased query's retrieval is slow, we merge results from whichever variants completed.

```python
async def _search_async_with_timeout(self, query: str, k: int):
    return await asyncio.wait_for(
        self._search_async(query, k), timeout=self.retrieval_timeout
    )
```

This means the system degrades gracefully under load: you still get an answer from the original query even if all the fancy multi-query machinery times out.

## Operational takeaway

The production gap in RAG isn't the retrieval algorithm — it's everything around it. Concurrency limits prevent thundering herds. Adaptive k avoids wasting tokens. Multi-query with early exit adds relevance without penalizing latency on easy questions. Strict timeouts ensure the system always responds, even if some enhancements get dropped.

The entire service stays under 300 lines of application code. Most of that is plumbing — semaphores, retries, timeouts, error boundaries. The actual RAG logic is maybe 40 lines. That ratio is about right for production systems.
