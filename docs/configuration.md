# Configuration reference

All settings live in two Code nodes: **⚙️ Config** and **📰 Prepare Sources**. No other node needs editing.

## Config node

| Key | Default | Meaning |
|---|---|---|
| `apiUrl` | `https://YOUR-SITE.example.com/api/v1/articles` | Endpoint that receives finished articles (POST) |
| `authHeaderName` | `X-Api-Key` | Name of the auth header sent with each POST |
| `authHeaderValue` | `YOUR_API_SECRET` | Value of the auth header (prefer an n8n credential in production) |
| `languages` | `['en']` | **Output languages** — see below |
| `maxItemsPerFeed` | `5` | Newest items taken from each feed per run |
| `shortDescriptionWords` | `30` | Words kept when generating the short description |
| `shortDescriptionMaxChars` | `500` | Hard cap on short descriptions before POSTing |
| `maxDescriptionChars` | `5000` | Cap on extracted full text (also bounds LLM cost) |
| `placeholderImage` | placehold.co URL | Used when no image is found anywhere |

### `languages`

A non-empty array of language codes. Every published article gets one field set **per language**:

| `languages` | Fields POSTed per article |
|---|---|
| `['en']` (default) | `title_en`, `short_description_en`, `description_en` |
| `['en', 'es']` | the three `_en` fields **plus** `title_es`, `short_description_es`, `description_es` |
| `['en', 'ar', 'fr']` | three field sets, one per code |

Behavior:

- Fields matching the feed's own language are taken directly from the feed/article page
- Any other configured language is filled by the **Translate** node (LLM)
- The LLM only runs when something is actually missing — default English-only + English feeds = **zero LLM calls, no OpenAI key needed**
- A feed's `lang` does not have to appear in `languages`: a Spanish feed with `languages: ['en']` is translated to English
- Your API must accept the matching field names

## Prepare Sources node

### CATEGORIES

Maps a slug (your choice of name) to the **numeric category ID your backend expects**:

```js
const CATEGORIES = { 'technology': 1, 'business': 2, 'news': 3 };
```

### SOURCES

One object per feed:

| Field | Example | Notes |
|---|---|---|
| `name` | `'BBC Technology'` | Label used in logs/skip reports |
| `feedUrl` | `'https://feeds.bbci.co.uk/news/technology/rss.xml'` | RSS or Atom |
| `lang` | `'en'` | Language the feed publishes in (any ISO code) |
| `categorySlug` | `'technology'` | Must be a key of `CATEGORIES` |

## API payload contract

*POST Article* sends this JSON body to `apiUrl` with header `{authHeaderName}: {authHeaderValue}`. With the default `languages: ['en']`:

```json
{
  "category_id": 1,
  "source_url": "https://example.com/some-article",
  "published_at": "2026-06-10T12:00:00.000Z",
  "image": "https://...",
  "status": 2,
  "title_en": "...",
  "short_description_en": "...",
  "description_en": "..."
}
```

With more languages, the three `title_<lang>` / `short_description_<lang>` / `description_<lang>` fields repeat per configured code.

Guarantees:

- Every text field is 