# Configuration reference

All settings live in two Code nodes: **⚙️ Config** and **📰 Prepare Sources**. No other node needs editing.

## Config node

| Key | Default | Meaning |
|---|---|---|
| `apiUrl` | `https://YOUR-SITE.example.com/api/v1/articles` | Endpoint that receives finished articles (POST) |
| `authHeaderName` | `X-Api-Key` | Name of the auth header sent with each POST |
| `authHeaderValue` | `YOUR_API_SECRET` | Value of the auth header (prefer an n8n credential in production) |
| `maxItemsPerFeed` | `5` | Newest items taken from each feed per run |
| `shortDescriptionWords` | `30` | Words kept when generating the short description |
| `shortDescriptionMaxChars` | `500` | Hard cap on short descriptions before POSTing |
| `maxDescriptionChars` | `5000` | Cap on extracted full text (also bounds LLM cost) |
| `placeholderImage` | placehold.co URL | Used when no image is found anywhere |

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
| `lang` | `'en'` or `'ar'` | Language the feed publishes in; the other is auto-translated |
| `categorySlug` | `'technology'` | Must be a key of `CATEGORIES` |

## API payload contract

*POST Article* sends this JSON body to `apiUrl` with header `{authHeaderName}: {authHeaderValue}`:

```json
{
  "category_id": 1,
  "source_url": "https://example.com/some-article",
  "published_at": "2026-06-10T12:00:00.000Z",
  "title_ar": "...",
  "title_en": "...",
  "short_description_ar": "...",
  "short_description_en": "...",
  "description_ar": "...",
  "description_en": "...",
  "image": "https://...",
  "status": 2
}
```

Guarantees:

- Every text field is **non-empty** (fallback chain: translation → same-language summary → title → other language)
- `short_description_*` ≤ `shortDescriptionMaxChars`
- `image` is always a URL (placeholder if nothing was found)
- `status` is fixed at `2` — change it in the *Build Payload* node if your API uses different status codes

A **2xx** response marks the article as published (it won't be sent again). Any other response leaves it unmarked, so it's retried on the next run.

## Common customizations

- **Different language pair** — rename the `_ar`/`_en` fields in *Build Articles*, *Translate* (prompt + JSON schema), *Build Payload*, and *POST Article*, and update the `lang` values in your sources
- **Different LLM / no translation** — swap the model in *Translate*, or delete the node and connect *Resolve Image* → *Build Payload* directly (fallbacks keep all fields filled in one language)
- **More/fewer items per run** — `maxItemsPerFeed` in Config
- **Different schedule** — edit the *Schedule* trigger node
