# Setup

## 1. Import the workflow

In n8n: **Workflows → ⋯ → Import from File** and select `news-scraper.json`. You'll see the canvas with sticky notes documenting each section.

## 2. Set your endpoint, auth, and languages

Open the **⚙️ Config** node:

```js
const CONFIG = {
  apiUrl: 'https://YOUR-SITE.example.com/api/v1/articles', // your endpoint
  authHeaderName: 'X-Api-Key',                             // your auth header
  authHeaderValue: 'YOUR_API_SECRET',                      // your secret
  languages: ['en'],                                       // output languages
  ...
};
```

`languages` controls which language versions every article gets. The default `['en']` means English-only — and if your feeds are English too, the workflow never calls the LLM, so **no OpenAI key is needed**. Add codes for multilingual publishing, e.g. `['en', 'es']`.

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

`lang` is the language the feed publishes in — any code is fine, it doesn't have to be one of `CONFIG.languages` (missing languages are auto-translated). `categorySlug` must exist in `CATEGORIES` — the node throws a clear error if it doesn't.

## 4. Attach your OpenAI credential (multilingual setups only)

Skip this step if you run the default English-only config with English feeds.

Otherwise open the **Translate** node and select (or create) your OpenAI credential. The default model is `gpt-4o-mini`; any chat model that reliably follows JSON-output instructions works.

## 5. Test run

Click **Execute workflow** (the manual *Run* trigger). Watch the *Per Source* loop process each feed. Check:

- *Skip Source* outputs show ski