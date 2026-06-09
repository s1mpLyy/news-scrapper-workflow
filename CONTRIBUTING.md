# Contributing

Thanks for your interest in improving this workflow! Contributions of all kinds are welcome — bug reports, docs fixes, new features.

## Reporting issues

Open a GitHub issue with:

- Your n8n version
- Your `languages` setting and the feed language(s) involved
- What you expected vs. what happened
- The relevant node's error output (redact your API URLs and secrets!)

## Submitting changes

1. Fork the repo and create a branch (`feat/...` or `fix/...`)
2. Make your change in n8n, then export the workflow (*Download*) and replace `news-scraper.json`
3. **Sanitize before committing** — exported JSON must contain:
   - no credential IDs (remove the `credentials` block or leave only placeholder names)
   - no real API URLs, secrets, or tokens
   - no `pinData`, no `instanceId` / `versionId` / workflow `id`
4. Keep the documentation conventions:
   - sticky notes for section-level explanations
   - a `notes` field on every functional node
   - comments inside Code-node JavaScript
   - configuration only in the **Config** and **Prepare Sources** nodes — never hardcode values in downstream nodes
   - language fields must stay dynamic (driven by `Config.languages`) — no hardcoded language suffixes outside Config defaults
5. Update `README.md` / `docs/` if behavior or configuration changed
6. Test: import your exported JSON into a clean n8n instance and run it end-to-end against a test endpoint — ideally once with the default `['en']` (LLM bypassed) and once with two languages — before opening the PR

## Style

- Code nodes: plain JavaScript, no external npm packages (only Node built-ins like `crypto`)
- Prefer contin