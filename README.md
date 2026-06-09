# News Scraper — RSS → Translate → Publish (n8n workflow)

An open-source [n8n](https://n8n.io) workflow that turns any list of RSS/Atom feeds into **bilingual (Arabic ⇄ English) articles** published to your own API or CMS.

For every feed it:

1. Fetches the newest items and **dedupes** against previously published articles
2. Visits each article page to grab the **best image** (RSS → `og:image` → `twitter:image`) and the **full article text**
3. Uses an LLM (OpenAI, structured output) to **fill the missing language** — Arabic feeds get English fields, English feeds get Arabic fields
4. **POSTs** the finished article to your endpoint, and only marks it "seen" on a 2xx response (failed posts retry next run)

It is fault-tolerant by design: a dead feed, an unreachable article page, or a failed LLM call never stops the run.

## Quick start

1. **Import** — in n8n: *Workflows → Import from File* → `news-scraper.json`
2. **Configure** — open the **⚙️ Config** node and set:
   - `apiUrl` — your endpoint
   - `authHeaderName` / `authHeaderValue` — your API auth
3. **Add feeds** — open the **📰 Prepare Sources** node and edit `SOURCES` (your feeds) and `CATEGORIES` (slug → your backend's numeric category IDs). It ships with 5 public example feeds.
4. **Credentials** — open the **Translate** node and attach your OpenAI credential
5. **Run** — click *Execute workflow* to test; for production, enable the **Schedule** trigger node (default: every 6 hours) and activate the workflow

Only the two nodes in step 2–3 ever need editing. Full details in [docs/setup.md](docs/setup.md).

## How it works

```
Run / Schedule
   └─► Config ─► Prepare Sources ─► Per Source (loop) ─► Done
                                       │
                                       ▼
                                   Fetch RSS ─► Build Articles ─► Has Articles?
                                                                    │ skip ──► Skip Source ─► (loop)
                                                                    ▼
                       Fetch Article Page ─► Resolve Image ─► Translate
                                                                    ▼
                                  Build Payload ─► POST Article ─► Mark Seen ─► (loop)
```

Each node carries an explanatory note inside n8n, and sticky notes on the canvas document every section. See [docs/architecture.md](docs/architecture.md) for the full walkthrough.

## Documentation

| Doc | Contents |
|---|---|
| [docs/setup.md](docs/setup.md) | Import, credentials, first run, scheduling |
| [docs/configuration.md](docs/configuration.md) | Every config option, sources/categories, the API payload contract |
| [docs/architecture.md](docs/architecture.md) | Node-by-node walkthrough, dedupe, error handling |
| [CONTRIBUTING.md](CONTRIBUTING.md) | How to contribute |

## Requirements

- n8n ≥ 1.60 (uses the OpenAI node's Responses API + structured output)
- An OpenAI API key (default model: `gpt-4o-mini` — swappable in the Translate node)
- An HTTP endpoint that accepts the article payload ([contract](docs/configuration.md#api-payload-contract))

## Notes & limitations

- Built for the Arabic/English pair; adapting to other language pairs means renaming the `_ar`/`_en` fields and adjusting the Translate prompt
- Dedupe uses n8n workflow static data, which persists only for **active** (scheduled/webhook) executions — manual test runs always start fresh
- Full-text extraction is heuristic (regex over `<article>`/`<p>` tags); it degrades gracefully to the feed summary
- Respect the terms of service and copyright of the feeds you scrape

## License

[MIT](LICENSE)
