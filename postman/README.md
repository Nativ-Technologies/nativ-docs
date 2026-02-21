# Nativ API – Postman Collection

Ready-to-use Postman collection covering the full Nativ Localization API.

## Quick start

1. **Import** – Open Postman → Import → drag in `Nativ_API.postman_collection.json`
2. **Set your API key** – Edit the collection variables and replace `nativ_your_api_key_here` with your real key (get one at [dashboard.usenativ.com](https://dashboard.usenativ.com) → Settings → API Keys)
3. **Send a request** – Open *Translation → Translate text* and hit Send

## What's included

| Folder | Requests | Description |
|---|---|---|
| Translation | 2 | Translate text with TM, brand voice, formality, backtranslation |
| OCR | 1 | Extract text from images |
| Image | 2 | Culturalize images + cultural sensitivity check |
| Translation Memory | 10 | Fuzzy search, list, add, update, delete, bulk ops, stats |
| Style Guides | 4 | List, create, update, delete |
| Brand Voice | 2 | Get brand voice prompt + combined prompt |
| Languages | 4 | Get languages, update formality & custom style |
| Feedback | 2 | Submit + list translation feedback |
| API Keys | 2 | List + revoke |

**29 requests** across 9 folders.

## Authentication

Every request uses the `X-API-Key` header. The collection inherits this from the collection-level auth, so you only set it once in the variables.

## Variables

| Variable | Default | Description |
|---|---|---|
| `baseUrl` | `https://api.usenativ.com` | API base URL |
| `NATIV_API_KEY` | `nativ_your_api_key_here` | Your API key |

## Interactive docs

Full API reference with try-it-out: [api.usenativ.com/redoc](https://api.usenativ.com/redoc)
