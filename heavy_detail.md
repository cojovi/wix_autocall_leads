# Penny Lead-Verification Pipeline — End-to-End Guide

## 0) TL;DR (the flow)

1. **WIX → Ingest Endpoint (Vercel)**
   WIX posts raw form JSON to `/api/webhooks/wix`. We **normalize** it to the exact keys Penny (ElevenLabs) expects and **store** it keyed by E.164 phone and submissionId.
2. **ElevenLabs → Fetch Endpoint (Vercel)**
   When a call starts, ElevenLabs calls `/api/elevenlabs/init`. We **lookup** the lead (by phone/submissionId) and **return** a flat JSON map (first\_name, last\_name, lead\_phone, address\_line1, city, state, zip, notes, request\_type, source\_site, etc.).
   → Penny uses those values immediately in the prompt and Data Collection.
3. **Twilio** is the telephony layer (calls route through ElevenLabs; no special Twilio → our API hop is required for this step).

---

## 1) Repo Layout (GitHub → Vercel)

```
/api
  /webhooks
    wix.ts                 # Ingest endpoint (WIX → us)
  /elevenlabs
    init.ts                # Fetch endpoint (ElevenLabs → us)
 /lib
   normalize.ts            # Maps WIX JSON → flat fields for ElevenLabs
   store.ts                # Storage (Upstash Redis in prod, in-memory in dev)
README.md
INSTRUCTIONS.md
vercel.json
package.json
.env.example              # add this
```

**Why this structure?**

* Vercel automatically turns everything under `/api` into serverless functions.
* We keep mapping and storage concerns decoupled (`/lib`), so you can swap Redis, change normalization logic, or add scrubbing without touching endpoint glue.

---

## 2) Environment & Config

Create `.env` (or set in Vercel Project Settings → Environment Variables):

```
UPSTASH_REDIS_REST_URL=...
UPSTASH_REDIS_REST_TOKEN=...
LEAD_TTL_SEC=86400
WIX_WEBHOOK_SECRET=long-random-string   # optional shared-secret
```

* **Redis**: used in production to persist leads between ingest and fetch. If these aren’t set locally, we fall back to **in-memory** store (only for dev).
* **LEAD\_TTL\_SEC**: how long to keep a lead cached (24h default).
* **WIX\_WEBHOOK\_SECRET**: (optional) a shared token to block random posts.

---

## 3) Normalization (WIX JSON → Penny Variables)

From your sample WIX payload, we normalize to:

* `first_name`, `last_name`, `lead_full_name`
* `lead_phone` (E.164)
* `address_line1`, `city`, `state`, `zip`
* `notes` (sanitized)
* `request_type` (e.g., WIX formName or “Project” field)
* `location`, `source_site` (“wix\_form”)
* metadata: `wix_submission_id`, `wix_contact_id`

**Why flat keys?**
They match your ElevenLabs **Dynamic Variables** and **Data Collection** identifiers, so Penny gets prefilled context automatically.

**Sanitization:**
We replace offensive strings in notes (example included). Expand this policy as needed.

---

## 4) Endpoints (behavior & expectations)

### 4.1 `/api/webhooks/wix` (POST) — Ingest

* **Caller:** WIX webhook (you configure WIX to POST here).

* **Body:** the raw WIX JSON (as in your example).

* **Steps:**

  1. Validate secret if you enabled it (header `X-Webhook-Secret` or `?secret=...`).
  2. Normalize JSON → flat map.
  3. Store under **two keys**:

     * `ph:<E.164 phone>`
     * `sub:<submissionId>`
  4. Return `{ok:true, keys:[...]}`.

* **Common pitfalls:**

  * Phone not in E.164 → fetch will never match; always normalize.
  * Missing `submissionId` → phone lookup is the primary fallback.
  * WIX sometimes changes field IDs; your `normalize.ts` should be resilient (prefer structured `contact.address`, etc.).

### 4.2 `/api/elevenlabs/init` (POST) — Fetch

* **Caller:** ElevenLabs at **call start** (when Penny is about to talk).

* **Body:** Includes Twilio call metadata. We extract the **caller/callee** (varies depending on inbound/outbound).

* **Steps:**

  1. Convert incoming number to E.164.
  2. Try lookup by `ph:<E.164>`; fallback to `sub:<submissionId>` if present (we accept `sid` in query or body).
  3. If found → return the **flat JSON map**. If not → return minimal defaults (don’t 500—calls should still proceed).

* **Common pitfalls:**

  * Wrong field used for phone (log the first request to confirm where the number is).
  * Stale cache (TTL too short) → you’ll get defaults; adjust TTL or re-ingest.
  * Returning nested objects → keep it **flat** so variables bind cleanly.

---

## 5) Security

* **Shared Secret:**
  Use `WIX_WEBHOOK_SECRET` and verify either a custom header (`X-Webhook-Secret`) or a URL param.
  Example check:

  ```ts
  const secret = req.headers['x-webhook-secret'] as string || (req.query.secret as string);
  if (process.env.WIX_WEBHOOK_SECRET && secret !== process.env.WIX_WEBHOOK_SECRET) {
    return res.status(401).json({ ok:false, error:'Unauthorized' });
  }
  ```
* **IP Allowlist / Token for Fetch:**
  In ElevenLabs → **Security**, you can restrict outbound to your domain. Optionally add a bearer token check on `/elevenlabs/init`.
* **PII:**
  You control sanitization in `normalize.ts`. Consider masking emails, profanity, or free-text fields per your policy.

---

## 6) Deploy (GitHub → Vercel)

1. Push repo to GitHub.
2. In Vercel, **Import Project** → link to the repo.
3. Set Environment Variables (see §2).
4. Deploy.
5. Copy live URLs:

   * Ingest: `https://<vercel-app>/api/webhooks/wix`
   * Fetch:  `https://<vercel-app>/api/elevenlabs/init`

---

## 7) Wire the integrations

### WIX → Ingest

* Set the webhook URL to your **ingest** endpoint.
* If WIX supports headers → add `X-Webhook-Secret: <your-secret>`; else use `?secret=...`.

### ElevenLabs → Fetch

* In the agent **Security** tab, enable **Fetch initiation client data from webhook**.
* Set URL to your **fetch** endpoint.
* (Optional) Security allowlist / token.
* **Save**.

### Penny prompt & variables

* Your agent’s System Prompt and **Dynamic Variables** must match your flat keys (`first_name`, `last_name`, `lead_phone`, etc.).
* You already set Evaluation Criteria + Data Collection; those will auto-fill from the returned map.

---

## 8) Testing (quick but thorough)

**Step A – Ingest**

* Submit a WIX form with a known phone.
* Check Vercel logs: you should see `{ok:true, keys:["ph:+1...", "sub:..."]}`.
* Optional: query Redis to see stored JSON.

**Step B – Fetch**

* Place a test call (in/outbound).
* Log the first body ElevenLabs sends to `/elevenlabs/init` so you can confirm the correct phone field.
* Ensure your response is the flat JSON.
* In ElevenLabs **History**, confirm variables are populated and your Data Collection entries show the values.

**If it fails:**

* No match? Confirm stored E.164 matches the number ElevenLabs sends.
* Got defaults? Probably a lookup key mismatch (wrong number format or TTL expiry).

---

## 9) Troubleshooting Playbook

**Symptom:** Penny greets without correct details.

* **Check**: Logs for `/elevenlabs/init`. Is the phone formatted as `+1XXXXXXXXXX`?
* **Fix**: Update `e164()` in both endpoints; ensure `normalize.ts` yields E.164.

**Symptom:** Fetch returns 500, call hangs/behaves oddly.

* **Check**: We intentionally return safe 200 with defaults on error. If you changed that, revert to soft-fail to avoid blocking calls.

**Symptom:** Values show but wrong keys in ElevenLabs.

* **Check**: Your response keys must **exactly** match your agent variables / Data Collection identifiers.

**Symptom:** Notes contain problematic text.

* **Fix**: Expand `sanitizeNotes()` (censor profanity, PII, or replace patterns).

**Symptom:** Data present in ingest logs but not fetched.

* **Check**: TTL not expired; cache key is consistent; ingest and fetch in the same environment (prod vs preview).

**Symptom:** WIX can’t deliver (401).

* **Fix**: Header/param secret mismatch; verify `WIX_WEBHOOK_SECRET`.

---

## 10) Example payloads & cURL

**Ingest (simulate WIX):**

```bash
curl -X POST https://<app>.vercel.app/api/webhooks/wix \
 -H 'Content-Type: application/json' \
 -H 'X-Webhook-Secret: your-secret' \
 -d @sample-wix.json
```

**Fetch (simulate ElevenLabs):**

```bash
curl -X POST https://<app>.vercel.app/api/elevenlabs/init \
 -H 'Content-Type: application/json' \
 -d '{"from":"+12814562323"}'
```

**Expected fetch response:**

```json
{
  "first_name": "Cody",
  "last_name": "Viveiros",
  "lead_full_name": "Cody Viveiros",
  "lead_phone": "+12814562323",
  "address_line1": "199 County Road 4840",
  "city": "Haslet",
  "state": "TX",
  "zip": "76052",
  "notes": "There is a martian on my roof",
  "request_type": "Tile Roofing",
  "location": "Dallas-Fort Worth",
  "source_site": "wix_form",
  "wix_submission_id": "f647af33-0cf8-4d43-8437-c2272fb6135b",
  "wix_contact_id": "08638117-aa3d-4b79-8f72-9f444fae300f"
}
```

---

## 11) Common “gotchas” we’ve pre-handled

* **Multiple address forms:** Prefer structured `contact.address` over free-text `multi_line_address`.
* **Names in multiple fields:** Prefer `field:first_name_*`/`field:last_name_*`; else fallback to `contact.name`.
* **Phone formatting:** Normalize to E.164 in both ingest and fetch.
* **Safety:** Notes sanitized (extend as needed).
* **Non-blocking fetch:** Returns defaults on error so calls don’t fail.

---

## 12) Extensibility

* Add a **status** field on call end (via conversation webhook) to push `VERIFIED/CORRECTED/...` back into your CRM.
* Add **auth** (bearer token) on `/elevenlabs/init`.
* Swap Redis for Supabase/Firestore with a thin adapter.

---
