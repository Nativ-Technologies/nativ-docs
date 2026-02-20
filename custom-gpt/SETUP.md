# Publishing the Nativ Custom GPT

## Prerequisites

- An OpenAI account with GPT creation access (ChatGPT Plus, Team, or Enterprise)
- A Nativ API key for testing (`nativ_xxx...` from dashboard.usenativ.com)

## Understanding the auth model

GPT Actions support three authentication types. This matters because it determines **who pays** for each API call:

| Auth type | How it works | Who pays |
|---|---|---|
| **API Key** | You (the GPT creator) set ONE key. ChatGPT uses it for every user's request. Users never see an auth prompt. | You pay for everyone |
| **OAuth 2.0** | Each user logs in with their own Nativ account. Each call bills to their own credits. | Each user pays for themselves |
| **None** | No auth. Anyone can call the API. | N/A (API would need to be public) |

### Recommended approach

**For testing / internal use:** Use API Key auth with your own key. Simple, fast, no backend changes.

**For public launch:** Implement OAuth 2.0 on the Nativ backend so each user authenticates with their own account and pays with their own credits. Without OAuth, you'd be paying for every user's translations — which gets expensive fast.

### OAuth 2.0 implementation (for public launch) ✅ IMPLEMENTED

The Nativ backend now includes a full OAuth 2.0 Authorization Code flow. Two new endpoints:

1. **Authorization endpoint** (`GET /oauth/authorize`) — Shows a branded login/consent page where users sign in with their Nativ email and password
2. **Token endpoint** (`POST /oauth/token`) — Exchanges the authorization code for access + refresh tokens

ChatGPT handles the redirect flow automatically. When a user first uses the GPT, they're prompted to sign in to their Nativ account. After that, ChatGPT stores their tokens and auto-refreshes them.

#### Backend setup (one-time)

1. **Run the database migration** — Execute `oauth-migration.sql` in the Supabase SQL Editor to create the `oauth_authorization_codes` and `oauth_tokens` tables. If policies already exist, that's fine (the `IF NOT EXISTS` on tables handles re-runs, but policies don't — ignore the "already exists" error).

2. **Generate a client secret** — Run this once and save the output:
   ```bash
   python3 -c "import secrets; print(secrets.token_urlsafe(32))"
   ```

3. **Set environment variables** on Cloud Run (Google Cloud Console > Cloud Run > your service > Edit & Deploy > Variables & Secrets):
   ```
   OAUTH_CLIENT_ID=chatgpt-nativ
   OAUTH_CLIENT_SECRET=<paste the secret you generated>
   ```
   **Critical:** This exact same secret must also be pasted into ChatGPT's GPT Action settings as the Client Secret. If they don't match, the token exchange will fail with `invalid_client`.

4. **Deploy** the updated backend. The `/oauth/authorize` and `/oauth/token` endpoints will be live.

5. **Verify the token endpoint** — After deploying, test that the secret matches:
   ```bash
   curl -s -X POST https://api.usenativ.com/oauth/token \
     -d "grant_type=authorization_code&code=test&redirect_uri=https://chat.openai.com/test&client_id=chatgpt-nativ&client_secret=YOUR_SECRET_HERE"
   ```
   - If you get `{"error":"invalid_grant",...}` → credentials are correct (the code is fake, but auth passed)
   - If you get `{"error":"invalid_client",...}` → the secret doesn't match what's deployed

#### GPT Action configuration (OAuth)

In the GPT editor Actions section, set authentication to:

| Setting | Value |
|---|---|
| **Authentication Type** | OAuth |
| **Client ID** | `chatgpt-nativ` |
| **Client Secret** | The secret you generated above |
| **Authorization URL** | `https://api.usenativ.com/oauth/authorize` |
| **Token URL** | `https://api.usenativ.com/oauth/token` |
| **Scope** | `translate` (or leave blank) |
| **Token Exchange Method** | Default (POST request) |

#### How it works for users

1. User opens the Nativ GPT in ChatGPT
2. On first use, ChatGPT shows a "Sign in to Nativ" button
3. User clicks it → redirected to `api.usenativ.com/oauth/authorize`
4. User sees the Nativ login/consent page, enters email + password
5. After authentication, they're redirected back to ChatGPT automatically
6. All subsequent API calls use their personal access token → billed to their credits
7. Tokens auto-refresh every hour (90-day refresh token lifespan)

---

## Step-by-step setup (API Key auth for testing)

### 1. Open the GPT Editor

Go to **[https://chatgpt.com/gpts/editor](https://chatgpt.com/gpts/editor)**

Click **"Create"** (top tab), then switch to the **"Configure"** tab for manual setup.

### 2. Fill in the basic fields

| Field | Value |
|---|---|
| **Name** | Nativ — AI Localization |
| **Description** | Translate and culturally adapt any text with AI — with translation memory, brand voice, glossary matching, and formality control. Built for marketers, designers, and product teams who need production-quality localization, not just raw translation. Powered by Nativ (usenativ.com). |
| **Instructions** | Paste the full contents of `system-prompt.md` |

### 3. Set conversation starters

Add these four:

```
Translate "The future of luxury, delivered" to French and Japanese
```
```
Translate this tagline to German with formal tone: "Get started today"
```
```
Translate "Add to cart" to Spanish, keep it under 15 characters
```
```
Verify translation quality for "Welcome back" in Japanese
```

### 4. Disable built-in capabilities

Under **Capabilities**, turn OFF:
- Web Browsing
- DALL·E Image Generation
- Code Interpreter

Nativ handles everything through the API action. These add latency and confusion.

### 5. Add the Action

Scroll down to **Actions** and click **"Create new action"**.

#### 5a. Set authentication

Click the **gear icon** next to "Authentication" and configure:

| Setting | Value |
|---|---|
| **Authentication Type** | API Key |
| **Auth Type** | Custom |
| **Custom Header Name** | `X-API-Key` |
| **API Key** | Paste your Nativ API key (e.g. `nativ_abc123...`) |

> **Important:** With API Key auth, this single key is used for ALL users of the GPT. Every translation call bills to YOUR Nativ account. This is fine for testing and internal use, but for public launch you should switch to OAuth 2.0 (see above).

#### 5b. Paste the schema

In the **Schema** text box, paste the full contents of `openapi.yaml`.

After pasting, you should see **4 actions** listed:
- `translateText` (POST /text/culturalize)
- `extractTextFromImage` (POST /text/extract)
- `generateStyledTextImage` (POST /image/culturalize)
- `inspectCulturalSensitivity` (POST /image/inspect)

If you see a red error banner, check that the YAML is valid (no tab characters, proper indentation). Common errors:
- **"schemas subsection is not an object"** — The `components` section must include `schemas: {}` even if empty.
- **YAML parse error** — Make sure there are no tab characters (use spaces only).

#### 5c. Set privacy policy

At the bottom of the Actions section:
- **Privacy policy URL**: `https://usenativ.com/privacy`

### 6. Test the action

Click the **"Test"** button next to the `translateText` action.

In the preview chat, try these tests:

| Test | What to say | What to verify |
|---|---|---|
| **Basic translation** | "Translate 'Hello world' to Spanish" | Returns `translated_text` in JSON |
| **Multiple languages** | "Translate 'Shop now' to French, Japanese, and Arabic" | Makes 3 separate API calls |
| **Formality** | "Translate 'How are you?' to German in formal tone" | Passes `formality: "formal"` |
| **Context** | "Translate 'Add to cart' to French — this is a button in an e-commerce app" | Passes context field |
| **Backtranslation** | "Translate 'The art of living' to Japanese and verify quality" | Returns `backtranslation` and `rationale` |
| **Character limit** | "Translate 'Subscribe now' to German, max 12 characters" | Passes `max_characters: 12` |

**Common issues:**
- **401 error (API Key auth)**: API key is wrong or missing. Re-enter it in the auth settings.
- **401 `invalid_client` (OAuth)**: The `client_secret` doesn't match between ChatGPT and your backend. To debug:
  1. Check your Cloud Run env var: Google Cloud Console > Cloud Run > service > Revisions > env vars > `OAUTH_CLIENT_SECRET`
  2. Check what you entered in ChatGPT: GPT Editor > Actions > gear icon > Client Secret
  3. These must be **identical**. Copy from Cloud Run and paste into ChatGPT (or vice versa).
  4. Verify with curl (see "Verify the token endpoint" above). You should get `invalid_grant` (not `invalid_client`) if the secret matches.
  5. Token Exchange Method: Use "Default (POST request)". The backend supports both POST body and Basic auth header.
  6. If `OAUTH_CLIENT_SECRET` is not set in Cloud Run at all, the backend will reject all OAuth requests.
- **OAuth "nothing happens" on authorize**: Check the backend logs for the POST /oauth/authorize request. Common causes: Supabase auth service is down, or the user's email doesn't have a matching row in the `users` table (they need to have signed up at dashboard.usenativ.com first).
- **402 error**: Your test account has no credits. Add credits at dashboard.usenativ.com.
- **Timeout**: Translation should complete in <10 seconds. If it times out, check your API server.
- **GPT doesn't call the action**: The Instructions or schema descriptions may be unclear. Make sure the schema is pasted correctly.

### 7. Upload a profile image (optional)

Upload a logo or icon. Recommended: the Nativ logo, 512x512px PNG.

### 8. Publish

Click **"Create"** (or **"Update"** if editing) in the top-right corner.

Choose visibility:

| Visibility | When to use |
|---|---|
| **Only me** | Testing only |
| **Anyone with a link** | Share with specific customers for feedback (recommended to start) |
| **Everyone** | Listed in the GPT Store for public discovery |

**Recommended launch sequence:**
1. Start with **"Only me"** — verify everything works
2. Move to **"Anyone with a link"** — share with 5-10 customers, collect feedback
3. Move to **"Everyone"** — publish to the GPT Store once validated

### 9. Share with users

After publishing with "Anyone with a link", copy the share URL and send it to users. The URL looks like:
```
https://chatgpt.com/g/g-XXXXXXXXX-nativ-ai-localization
```

---

## Cost control (API Key auth)

Since API Key auth means you pay for everyone, consider these safeguards:

1. **Dedicated key with credit cap** — Create a separate API key with a limited credit balance. When it runs out, the GPT returns 402 errors instead of draining your main account.
2. **Rate limiting** — Your API already has rate limiting via slowapi. Consider tighter limits for the GPT's API key.
3. **Monitor usage** — Track API calls from the GPT's key in your dashboard to understand cost.
4. **Budget threshold** — Set up alerts when the GPT key's credit usage exceeds a threshold.

For anything beyond testing/demos, implement OAuth 2.0 so users pay with their own accounts.

---

## Notes

- Each translation costs 100 credits (~$0.10) from the account tied to the API key
- The GPT doesn't access TM/glossary lists directly — the server applies them automatically per user
- Users manage their TM, glossaries, style guides, and team settings at dashboard.usenativ.com
- The OpenAPI spec exposes 4 endpoints (text translation, OCR, styled image generation, cultural sensitivity). Image endpoints use multipart/form-data — users can upload images directly in ChatGPT and the GPT will pass them to the API
- The full Nativ API has many more endpoints, but these 4 are the most useful for ChatGPT users

## Updating the GPT

To update instructions or the schema after publishing:
1. Go to [chatgpt.com/gpts/mine](https://chatgpt.com/gpts/mine)
2. Click on the Nativ GPT
3. Click **"Edit GPT"**
4. Make changes and click **"Update"**

Changes take effect immediately for all users.
