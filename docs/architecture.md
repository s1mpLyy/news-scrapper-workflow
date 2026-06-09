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
                                          Needs Translation? ──(no)──┐
                                                   │ (yes)           │
                                                   ▼                 │
                                               Translate ────────────┤
                                                                     ▼
                                                               Build Payload
                                                                     ▼
                                                                POST Article
                                                                     ▼
                                                                 Mark Seen ──► back to Per Source
```

## Node-by-node

**Run / Schedule** — manual trigger for testing; schedule trigger (disabled by default, every 6 h) for production.

**Config** *(edit me)* — returns a single item with all settings, including `languages` (default `['en']`). Downstream nodes read it via `$('Config').first().json`, so nothing is hardcoded elsewhere. Validates that `languages` is a non-empty array.

**Prepare Sources** *(edit me)* — emits one item per feed with the numeric `category_id` resolved from the slug map. Throws a descriptive error on unknown slugs.

**Per Source** — `splitInBatches` loop, one feed at a time. Isolation is the point: one broken feed can't take down the run. Output 1 fires when all sources are done; output 2 hands the next source onward.

**Fetch RSS** — n8n RSS node with `customFields: enclosure, media:content, media:thumbnail` so feed images are available. `alwaysOutputData` + continue-on-error: a dead feed produces an empty result, not a crash.

**Build Articles** — normalizes raw items:

- takes the newest `maxItemsPerFeed` items
- derives a stable 16-char ID (`sha256(url)`) and skips IDs already in the dedupe store
- strips HTML and generates **one field set per configured language** (`title_<lang>`, `short_description_<lang>`, `description_<lang>`): the feed's own language is pre-filled, the rest stay empty for the LLM
- keeps the original text in `source_title` / `source_short_description` / `source_description` so translation works even when the feed's language isn't a configured output language
- if nothing usable remains, emits one `{ _skip: true, _reason }` marker item

**Has Articles?** — routes `_skip` markers to *Skip Source* (which records the reason and re-enters the loop); real articles continue to enrichment.

**Fetch Article Page** — plain HTTP GET of the article URL (text response, 8 s timeout, `neverError`). Supplies the HTML for image and full-text extraction.

**Resolve Image** — image priority: RSS image → `og:image` → `twitter:image`. Full text: prefers `<article>` scope, collects `<p>` blocks ≥ 40 chars, falls back to a full tag-strip; the result replaces the source description **only if longer**, capped at `maxDescriptionChars`.

**Needs Translation?** — checks whether any configured language still has an empty field. If not (the default English-only setup with English feeds), the LLM is bypassed entirely — articles go straight to *Build Payload* and **no OpenAI credential is required**.

**Translate** — OpenAI node. The prompt is built dynamically from `Config.languages`: it sends the per-language field object plus the original source text, and instructs the model to fill only empty fields (suffix = target language) and return filled ones unchanged. Because the keys are dynamic, output is prompt-enforced JSON (no static schema) parsed defensively downstream. Continue-on-error: if the call fails, the pipeline still proceeds.

**Build Payload** — merges the LLM output back in. A depth-limited recursive search (`findPayload`) locates the language-field object anywhere in the response, including inside JSON strings — and it works identically when translation was skipped (the article itself carries the fields). A per-language fallback chain (own field → same field in another language → source text → title) guarantees every field is non-empty. Returns `{ id, payload }` where `payload` is exactly what gets POSTed.

**POST Article** — URL and auth header come from Config via expressions; the body is `Build Payload`'s `payload` object, so language fields adapt automatically. `fullResponse` + `neverError` so the status code reaches the next node.

**Mark Seen** — writes the article ID into the dedupe store **only on 2xx**, then loops back. Failed posts are retried on subsequent runs.

## Deduplication

`$getWorkflowStaticData('global').seenIds` maps `sha256(url).slice(0,16)` → timestamp. Properties:

- persists between executions, but **only for active (scheduled/webhook) runs** — manual executions read/write a throwaway copy
- grows unbounded; for very large source lists consider periodically pruning old entries or moving dedupe to your API (e.g., unique constraint on `source_url`)

## Error-handling philosophy

Every external interaction (RSS fetch, page fetch, LLM, POST) uses `alwaysOutputData` and/or `onError: continueRegularOutput`. The loop always completes; failures degrade individual articles (summary instead of full text, fallback text instead of a missing translation, retry next run) rather than aborting the run.
