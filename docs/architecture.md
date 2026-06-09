# Architecture

## Flow

```
Run (manual) ──┐
Schedule ──────┴─► Config ─► Prepare Sources ─► Per Source ──(done)──► Done
                                                   │ (next source)
                                                   ▼
                                               Fetch RSS
                                                   ▼
                                             Build Articles
                                                   ▼
                                             Has Articles? ──(skip)──► Skip Source ──► back to Per Source
                                                   │ (articles)
                                                   ▼
                                           Fetch Article Page
                                                   ▼
                                             Resolve Image
                                                   ▼
                                               Translate
                                                   ▼
                                             Build Payload
                                                   ▼
                                              POST Article
                                                   ▼
                                               Mark Seen ──► back to Per Source
```

## Node-by-node

**Run / Schedule** — manual trigger for testing; schedule trigger (disabled by default, every 6 h) for production.

**Config** *(edit me)* — returns a single item with all settings. Downstream nodes read it via `$('Config').first().json`, so nothing is hardcoded elsewhere.

**Prepare Sources** *(edit me)* — emits one item per feed with the numeric `category_id` resolved from the slug map. Throws a descriptive error on unknown slugs.

**Per Source** — `splitInBatches` loop, one feed at a time. Isolation is the point: one broken feed can't take down the run. Output 1 fires when all sources are done; output 2 hands the next source onward.

**Fetch RSS** — n8n RSS node with `customFields: enclosure, media:content, media:thumbnail` so feed images are available. `alwaysOutputData` + continue-on-error: a dead feed produces an empty result, not a crash.

**Build Articles** — normalizes raw items:

- takes the newest `maxItemsPerFeed` items
- derives a stable 16-char ID (`sha256(url)`) and skips IDs already in the dedupe store
- strips HTML, builds title / short description / description in the **source language only**, leaving the other language empty for the LLM
- if nothing usable remains, emits one `{ _skip: true, _reason }` marker item

**Has Articles?** — routes `_skip` markers to *Skip Source* (which records the reason and re-enters the loop); real articles continue to enrichment.

**Fetch Article Page** — plain HTTP GET of the article URL (text response, 8 s timeout, `neverError`). Supplies the HTML for image and full-text extraction.

**Resolve Image** — image priority: RSS image → `og:image` → `twitter:image`. Full text: prefers `<article>` scope, collects `<p>` blocks ≥ 40 chars, falls back to a full tag-strip; the result replaces the feed summary **only if longer**, capped at `maxDescriptionChars`.

**Translate** — OpenAI node with a strict JSON schema (six string fields). The prompt fills only empty fields and passes filled ones through unchanged, which keeps cost down and prevents rewrite drift. Continue-on-error: if the call fails, the pipeline still proceeds.

**Build Payload** — merges the LLM output back in. Because the OpenAI node nests its result, a depth-limited recursive search (`findPayload`) locates the six-field object anywhere in the response, including inside JSON strings. A fallback chain then guarantees every field is non-empty.

**POST Article** — URL and auth header come from Config via expressions. `fullResponse` + `neverError` so the status code reaches the next node.

**Mark Seen** — writes the article ID into the dedupe store **only on 2xx**, then loops back. Failed posts are retried on subsequent runs.

## Deduplication

`$getWorkflowStaticData('global').seenIds` maps `sha256(url).slice(0,16)` → timestamp. Properties:

- persists between executions, but **only for active (scheduled/webhook) runs** — manual executions read/write a throwaway copy
- grows unbounded; for very large source lists consider periodically pruning old entries or moving dedupe to your API (e.g., unique constraint on `source_url`)

## Error-handling philosophy

Every external interaction (RSS fetch, page fetch, LLM, POST) uses `alwaysOutputData` and/or `onError: continueRegularOutput`. The loop always completes; failures degrade individual articles (summary instead of full text, single language via fallbacks, retry next run) rather than aborting the run.
