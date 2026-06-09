# Setup

## 1. Import the workflow

In n8n: **Workflows → ⋯ → Import from File** and select `news-scraper.json`. You'll see the canvas with sticky notes documenting each section.

## 2. Set your endpoint and auth

Open the **⚙️ Config** node:

```js
const CONFIG = {
  apiUrl: 'https://YOUR-SITE.example.com/api/v1/articles', // your endpoint
  authHeaderName: 'X-Api-Key',                             // your auth header
  authHeaderValue: 'YOUR_API_SECRET',                      // your secret
  ...
};
```

> **Security tip:** anything you type here is stored in the workflow JSON. For production, leave `authHeaderValue` blank, create an n8n **Header Auth credential**, and attach it to the *POST Article* node instead — then the secret never appears in exports.

## 3. Add your feeds and categories

Open **📰 Prepare Sources**. Two things to edit:

```js
const CATEGORIES = { 'technology': 1, 'business': 2, 'news': 3 };
```

The numbers are *your backend's* category IDs. Then list your feeds:

```js
const SOURCES = [
  { name: 'BBC Technology', feedUrl: 'https://feeds.bbci.co.uk/news/technology/rss.xml', lang: 'en', categorySlug: 'technology' },
  ...
];
```

`lang` is the language the feed publishes in (`en` or `ar`); the workflow auto-translates to the other one. `categorySlug` must exist in `CATEGORIES` — the node throws a clear error if it doesn't.

## 4. Attach your OpenAI credential

Open the **Translate** node and select (or create) your OpenAI credential. The default model is `gpt-4o-mini`; pick any model that supports structured JSON output.

## 5. Test run

Click **Execute workflow** (the manual *Run* trigger). Watch the *Per Source* loop process each feed. Check:

- *Skip Source* outputs show skipped feeds and why (`no_rss_items`, `all_already_seen`, `no_usable_url`)
- *POST Article* responses return 2xx from your API
- *Mark Seen* shows `posted: true`

## 6. Go to production

1. Enable the **Schedule** trigger node (default: every 6 hours — adjust as needed)
2. Activate the workflow (toggle top-right)

> Dedupe only persists across **active** runs. Manual test runs don't remember what they posted, so re-running a test may POST duplicates — your API should ideally reject duplicate `source_url`s as a second line of defense.
