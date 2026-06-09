# AGENTS.md — guide for AI agents working on this repo

This file orients AI coding agents (Claude Code, Cursor, Codex, etc.). Human-facing docs live in [README.md](README.md) and [docs/](docs/).

## What this repo is

A single **n8n workflow** ([news-scraper.json](news-scraper.json)) plus its documentation. There is no application code, no package manager, no build step, and no test suite. The workflow fetches RSS/Atom feeds, dedupes articles, enriches them with an image and full text from the article page, optionally translates them via an LLM, and POSTs them to a user-configured API.

**The "codebase" is one JSON file.** All logic lives in `jsCode` strings (plain JavaScript) and node parameters inside `news-scraper.json`. Everything else in the repo is documentation about that file.

## Repo map

| Path | What it is |
|---|---|
| `news-scraper.json` | The entire workflow — 22 nodes (5 sticky notes + 17 functional) |
| `README.md` | Overview, quick start, flow diagram, requirements |
| `docs/setup.md` | Import, configuration, credentials, first run, production |
| `docs/configuration.md` | Every config option + the **API payload contract** |
| `docs/architecture.md` | Node-by-node walkthrough, dedupe, error-handling philosophy |
| `CONTRIBUTING.md` | Contribution rules — **binding on agents too** |

## Workflow pipeline (node names as they appear in the JSON)

```
Run / Schedule → Config → Prepare Sources → Per Source (loop)
  loop body: Fetch RSS → Build Articles → Has Articles?
    skip → Skip Source → back to Per Source
    articles → Fetch Article Page → Resolve Image → Needs Translation?
      yes → Translate → Build Payload
      no  → Build Payload
    Build Payload → POST Article → Mark Seen → back to Per Source
  loop done → Done
```

- **Config** / **Prepare Sources** (Code nodes) — the only two nodes users edit; all settings originate here.
- **Per Source** (`splitInBatches`) — one feed at a time, so one broken feed never kills the run.
- **Build Articles** — newest N items, dedupe via `$getWorkflowStaticData('global').seenIds` (key = `sha256(url).slice(0,16)`), generates one field set per configured language (`title_<lang>`, `short_description_<lang>`, `description_<lang>`), emits a `{_skip, _reason}` marker when a feed yields nothing.
- **Resolve Image** — image priority RSS → `og:image` → `twitter:image`; upgrades description to extracted page text only when longer.
- **Needs Translation?** — LLM is bypassed entirely when all configured language fields are already filled (default `['en']` + English feeds ⇒ no OpenAI key needed).
- **Translate** (OpenAI, `gpt-4o-mini`) — fills only empty language fields; output is prompt-enforced JSON parsed defensively downstream.
- **Build Payload** — recursive `findPayload` search locates the LLM output anywhere in the response; per-language fallback chain guarantees every text field is non-empty.
- **Mark Seen** — records the article in the dedupe store **only on a 2xx** POST response; failures retry next run.

## Invariants — do not break these

1. **Configuration lives only in `Config` and `Prepare Sources`.** Downstream nodes read it via `$('Config').first().json` expressions. Never hardcode URLs, secrets, limits, or language codes in any other node.
2. **Language fields are dynamic**, driven by `Config.languages`. No hardcoded `_en`/`_es` suffixes anywhere outside Config defaults. Any new text field must follow the `<field>_<lang>` pattern and be generated per configured language.
3. **Fault tolerance over hard failure.** Every external interaction (RSS fetch, page fetch, LLM, POST) uses `alwaysOutputData` and/or `onError: "continueRegularOutput"`. One bad feed/article/LLM call must never abort the run. Keep this property when adding nodes.
4. **The API payload contract is backward-compatible** (see [docs/configuration.md](docs/configuration.md#api-payload-contract)). Every text field non-empty, `image` always a URL, `status: 2` (set in Build Payload). Breaking changes must be clearly documented.
5. **Code nodes are plain JavaScript, Node built-ins only** (e.g. `crypto`). No external npm packages — they aren't available inside n8n Code nodes.
6. **Sanitized exports only.** The committed JSON must contain no credential IDs/blocks, no real API URLs or secrets, no `pinData`, no `instanceId`/`versionId`/workflow `id`.
7. **Documentation conventions inside the workflow:** every functional node has a `notes` field; each canvas section has a sticky note; Code-node JavaScript is commented.

## Editing news-scraper.json safely

This is n8n's export format — structural rules an agent must respect:

- **Node logic is JavaScript embedded in JSON strings** (`jsCode` parameters). Mind JSON escaping (`\n`, `\"`, `\\`) when editing. Prefer surgical string edits over regenerating whole nodes.
- **Node display names are load-bearing.** Cross-node expressions reference nodes by name: `$('Config')`, `$('Per Source')`, `$('Build Articles')`, `$('Resolve Image')`, `$('Build Payload')`. The `connections` object is also keyed by name. **Renaming a node requires updating every expression and connection that references it** — grep the whole file for the old name first.
- **Keep `connections` in sync** when adding/removing nodes. Multi-output nodes: `Per Source` (output 0 = done, output 1 = next source), `Has Articles?` and `Needs Translation?` (output 0 = true branch, output 1 = false branch).
- **Don't bump `typeVersion`** on existing nodes unless you know the new schema; versions are pinned to what n8n ≥ 1.60 supports.
- **Sticky notes are documentation** — update them when behavior they describe changes.

## Verification (no test suite exists)

After any change to `news-scraper.json`:

1. **JSON validity** — e.g. `Get-Content -Raw news-scraper.json | ConvertFrom-Json` (PowerShell) or `node -e "JSON.parse(require('fs').readFileSync('news-scraper.json','utf8'))"`.
2. **Reference check** — every `$('Node Name')` in expressions/jsCode matches an existing node `name`; every node in `connections` exists.
3. **JS syntax in edited jsCode** — extract the string and run it through `node --check` or careful review; n8n only surfaces syntax errors at runtime.
4. Full end-to-end testing requires importing into an n8n instance (≥ 1.60) and executing against a test endpoint — ideally once with default `['en']` (LLM bypassed) and once with two languages. If you can't run n8n, say so explicitly rather than claiming the change is tested.

## Docs must stay in sync

| If you change… | Update… |
|---|---|
| Config keys / defaults | `docs/configuration.md` (table), `README.md`, Config node comments |
| Payload shape / `status` / field names | `docs/configuration.md` § API payload contract, `README.md` |
| Node behavior, flow, or error handling | `docs/architecture.md`, the node's `notes`, relevant sticky note |
| Flow structure (nodes added/removed) | ASCII diagrams in **both** `README.md` and `docs/architecture.md` |
| Setup steps / credentials | `docs/setup.md` |

## Gotchas

- **Dedupe is ephemeral in manual runs.** `$getWorkflowStaticData` persists only for active (scheduled/webhook) executions; manual test runs always start fresh and may re-POST articles.
- **The dedupe store grows unbounded** — known limitation, documented in `docs/architecture.md`.
- **Translate output has no fixed schema** (keys depend on `Config.languages`), so Build Payload's `findPayload` walks the entire LLM response, parsing JSON strings, depth-limited to 8. Don't replace this with a static-path lookup.
- **Full-text extraction is heuristic** (regex over `<article>`/`<p>`), intentionally dependency-free. It degrades to the feed summary; don't "fix" short output by adding an HTML-parsing library (see invariant 5).
- **`status: 2` is intentionally hardcoded** in Build Payload as the example "published" status — it's the documented place users customize it.

## Git conventions

- Branches: `feat/...` or `fix/...`; main branch is `main`.
- Never commit real credentials, API URLs, or n8n instance metadata (see invariant 6 and `.gitignore`).
- License: MIT — keep headers/attribution consistent with that.
