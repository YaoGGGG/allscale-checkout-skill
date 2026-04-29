You are helping a developer integrate Allscale Checkout into their app or website. Follow these steps in order. Be conversational — go step by step, confirm each step works before moving on.

---

## CRITICAL SAFETY RULES

You MUST follow these rules at all times:

1. **NEVER write API Secret values into any source code file, config file, or any file that is not `.env`.**
2. **NEVER log, print, or echo the API Secret in any code you generate.**
3. **ALWAYS ensure `.env` is in `.gitignore` BEFORE writing credentials to `.env`.** Check first, add it if missing.
4. **NEVER commit `.env` files.** If you see one staged in git, warn the developer immediately.
5. **All API signing MUST happen server-side.** Never generate code that signs requests in frontend/browser code. The API Secret must never be in any client-side bundle.
6. **ALWAYS add rate limiting** to any endpoint you create that calls the Allscale API.
7. **ALWAYS validate amounts server-side** — never trust client-sent values without bounds checking.

---

## Step 0: Allscale Account and Credentials

Before writing any code, ask the developer:

> Do you have an Allscale account with Commerce enabled? You'll need:
>
> 1. An **Allscale account** — sign up at [allscale.io](https://allscale.io)
> 2. **Allscale Commerce** enabled on your account
> 3. A **store** created in the Allscale Commerce dashboard, with your **receiving wallet address** configured
> 4. Your **API Key** and **API Secret** (generated in the dashboard — the secret is shown only once)
>
> If you don't have these yet, go to [allscale.io](https://allscale.io) to get set up, or contact the Allscale BD team for help.

**Do NOT proceed until they confirm they have their API Key and API Secret.**

Once they confirm, ask them to provide:
- Their **API Key** (starts with `st_`)
- Their **API Secret** (starts with `st_`)

Then immediately proceed to set up their `.env` file safely (Step 1).

---

## Step 1: Secure Credential Setup

**Before writing the `.env` file, you MUST:**

1. Check if `.gitignore` exists. If not, create it.
2. Check if `.env` is listed in `.gitignore`. If not, add it.
3. Only THEN create or update the `.env` file with their credentials.

Write a `.env` file:

```
ALLSCALE_API_KEY=<their api key>
ALLSCALE_API_SECRET=<their api secret>
ALLSCALE_BASE_URL=https://openapi-sandbox.allscale.io
ALLSCALE_CURRENCY=USD
```

Also create a `.env.example` file (safe to commit) with placeholder values:

```
ALLSCALE_API_KEY=your_api_key_here
ALLSCALE_API_SECRET=your_api_secret_here
ALLSCALE_BASE_URL=https://openapi-sandbox.allscale.io
ALLSCALE_CURRENCY=USD
```

Tell them:
- Start with the **sandbox** URL (`openapi-sandbox.allscale.io`) for testing
- Switch to **production** (`openapi.allscale.io`) when ready for real payments
- Available currency codes: `USD`, `EUR`, `GBP`, `CAD`, `AUD`, `JPY`, `CNY`, `SGD`, `HKD`

---

## Step 2: Understand Their Project

Now ask:

1. **What are you building?** (donation page, e-commerce checkout, tipping feature, subscription flow, etc.)
2. **What's your tech stack?** (framework, language, platform — e.g., Next.js, Flask, Rails, vanilla JS, mobile, etc.)
3. **Where will you deploy?** (Vercel, Netlify, AWS, self-hosted, etc.)

Adapt all code in the following steps to their specific stack.

---

## Step 3: Implement API Authentication (HMAC-SHA256 Request Signing)

Every Allscale API request must be signed. This is the most critical part.

### Required headers on every request:

| Header | Description |
|---|---|
| `X-API-Key` | Their API key |
| `X-Timestamp` | Unix timestamp in seconds |
| `X-Nonce` | Random unique string (UUID recommended) |
| `X-Signature` | `v1=<signature>` |

### Signing algorithm:

1. Build a **canonical string** by joining these with newline (`\n`):
   ```
   METHOD          (e.g., "POST", "GET" — uppercase)
   PATH            (e.g., "/v1/checkout_intents/")
   QUERY_STRING    (e.g., "" for no query, or "key=value" without the ?)
   TIMESTAMP       (same value as X-Timestamp header)
   NONCE           (same value as X-Nonce header)
   BODY_SHA256     (SHA-256 hex digest of the raw request body; for GET with no body, hash the empty string "")
   ```

2. Compute signature:
   ```
   signature = Base64( HMAC-SHA256( api_secret, canonical_string ) )
   ```

3. Set header:
   ```
   X-Signature: v1=<signature>
   ```

### Reference implementations:

**Node.js:**
```javascript
import crypto from "crypto";

function signRequest(method, path, query, body, apiSecret) {
  const timestamp = Math.floor(Date.now() / 1000).toString();
  const nonce = crypto.randomUUID();
  const bodyHash = crypto.createHash("sha256").update(body).digest("hex");

  const canonical = [method, path, query, timestamp, nonce, bodyHash].join("\n");
  const signature = crypto.createHmac("sha256", apiSecret).update(canonical).digest("base64");

  return {
    "X-Timestamp": timestamp,
    "X-Nonce": nonce,
    "X-Signature": `v1=${signature}`,
  };
}
```

**Python:**
```python
import hmac, hashlib, base64, time, uuid

def sign_request(method, path, query, body, api_secret):
    timestamp = str(int(time.time()))
    nonce = str(uuid.uuid4())
    body_hash = hashlib.sha256(body.encode()).hexdigest()

    canonical = "\n".join([method, path, query, timestamp, nonce, body_hash])
    signature = base64.b64encode(
        hmac.new(api_secret.encode(), canonical.encode(), hashlib.sha256).digest()
    ).decode()

    return {
        "X-Timestamp": timestamp,
        "X-Nonce": nonce,
        "X-Signature": f"v1={signature}",
    }
```

**Important gotchas:**
- `BODY_SHA256` must be computed from the **raw body string**, not a parsed/re-serialized object
- For GET requests (no body), hash the empty string `""`
- Timestamp must be within ±5 minutes of server time or request is rejected
- Each nonce can only be used once — always generate a fresh UUID

Write the signing utility in whatever language/framework they're using. Read the API secret from the environment variable `ALLSCALE_API_SECRET` — NEVER hardcode it.

---

## Step 4: Test Connectivity

Before building the checkout flow, verify auth works. Write a small test script or route that calls all three test endpoints:

1. `GET /v1/test/ping` — should return `{"code":0,"payload":{"pong":"ok"}}`
2. `GET /v1/test/fail` — should return an error response (this is expected, it tests error handling)
3. `POST /v1/test/post` — send a JSON body, the server echoes it back in `payload`

Run the tests and confirm all three pass before proceeding. If they get error code `20002` (bad signature), help them debug — see the debugging section at the bottom.

---

## Step 5: Create Checkout Intent

This is the core payment flow. The server creates a checkout intent, gets back a hosted checkout URL, and opens that URL for the user to pay.

### Endpoint: `POST /v1/checkout_intents/`

**Important:** The trailing slash is required.

### Request body:

```json
{
  "currency": 1,
  "amount_cents": 500,
  "order_id": "order_123",
  "order_description": "Monthly subscription",
  "user_id": "user_456",
  "user_name": "Tom",
  "extra": {
    "source": "web"
  }
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `currency` | int | YES | Fiat currency as integer enum (see table below) |
| `amount_cents` | int | YES | Amount in cents ($5.00 = 500) |
| `order_id` | string or null | no | Your internal order ID |
| `order_description` | string or null | no | Description shown to payer |
| `user_id` | string or null | no | Your internal user ID |
| `user_name` | string or null | no | Payer display name |
| `extra` | object or null | no | Arbitrary metadata |

**CRITICAL: `currency` must be an integer, NOT a string. Do NOT send `"USD"` — send `1`.**

### Currency enum (common values):

| Value | Code |
|---|---|
| 1 | USD |
| 9 | AUD |
| 27 | CAD |
| 31 | CNY |
| 44 | EUR |
| 48 | GBP |
| 57 | HKD |
| 72 | JPY |
| 126 | SGD |

### Successful response:

```json
{
  "code": 0,
  "payload": {
    "checkout_url": "https://checkout.allscale.io/abc123",
    "allscale_checkout_intent_id": "65b2f3d0d2d9c0a1b2c3d4e5",
    "amount_coins": "5.0000",
    "stable_coin_type": 1,
    "rate": "1.0000"
  },
  "error": null,
  "request_id": "req_xxxxx"
}
```

- `checkout_url` — redirect or open this URL for the user to pay
- `allscale_checkout_intent_id` — save this to poll status later
- Settlement is currently **USDT only** (`stable_coin_type: 1`)

### What to do with the response:

- **Web app:** Open `checkout_url` in a new tab (`window.open`) or redirect (`window.location.href`)
- **Mobile app:** Open `checkout_url` in an in-app browser or system browser
- **API/backend:** Return the `checkout_url` to your frontend

### Security requirements for the checkout endpoint:

When writing the server-side route that calls this API, you MUST:
1. **Rate limit** the endpoint (e.g., 5 requests per minute per IP)
2. **Validate the amount** server-side (positive number, within reasonable bounds)
3. **Read credentials from environment variables only**

---

## Step 6: Choose Your Confirmation Strategy

After the checkout intent is created and the user opens `checkout_url`, you need to know when payment confirms so you can deliver the goods (unlock content, send the digital file, generate the AI reading, etc.). There are two ways:

| Strategy | When to use | Pros | Cons |
|---|---|---|---|
| **Webhook (recommended)** | Any backend that can expose a public HTTPS endpoint — chat bots, mobile-app backends, web apps with a server, SaaS | Real-time, ~1s latency, zero wasted requests, scales to thousands of concurrent intents, no client polling cost | Needs a publicly reachable URL; needs idempotency / nonce store |
| **Polling (fallback)** | Static sites, single-page apps with no backend, prototypes, environments behind NAT with no public URL | Simple to implement, works anywhere with outbound HTTPS | Wastes requests, slower (5s polling interval), client-side timer ties up the user's browser |

**Default to webhooks.** Tell the developer: "If your app has a backend that can expose a public HTTPS endpoint, use webhooks (Step 7). Polling (Step 8) is the simpler fallback for static sites or quick prototypes."

If the developer is unsure, ask:
1. **"Do you have a backend, or is this a static page / SPA only?"** → Backend = Step 7. Static = Step 8.
2. **"Can you expose a public HTTPS URL? (Vercel/Railway/Fly/Render all do this. Localhost needs `ngrok` or `cloudflared` during development.)"** → Yes = Step 7. No = Step 8.

---

## Step 7: Webhook (Recommended)

Configure a webhook URL in your Allscale dashboard. Allscale sends a `POST` to that URL the moment payment is confirmed on-chain.

### Webhook headers Allscale sends:

| Header | Description |
|---|---|
| `X-API-Key` | Your API key (so you can match the right secret if you have multiple) |
| `X-Webhook-Id` | Unique webhook ID — also present in the body, must match |
| `X-Webhook-Timestamp` | Unix timestamp (seconds) |
| `X-Webhook-Nonce` | Unique nonce — store and reject replays |
| `X-Webhook-Signature` | `v1=<signature>` |

### Webhook signature canonical string

The canonical-string format for webhooks is **different** from regular API requests — note the literal prefix line `allscale:webhook:v1` at the top:

```
allscale:webhook:v1
METHOD
PATH
QUERY_STRING
WEBHOOK_ID
TIMESTAMP
NONCE
BODY_SHA256
```

Then: `expected = Base64( HMAC-SHA256( api_secret, canonical ) )`

Compare with **timing-safe equality** against the `v1=...` value in `X-Webhook-Signature`.

### Webhook payload fields:

| Field | Type | Description |
|---|---|---|
| `all_scale_transaction_id` | string | Allscale transaction ID |
| `all_scale_checkout_intent_id` | string | Checkout intent ID — match this against your saved intent IDs |
| `webhook_id` | string | Must match `X-Webhook-Id` header |
| `amount_cents` | int | Fiat amount in cents |
| `currency` | int | Currency enum |
| `currency_symbol` | string | e.g., `"USD"` |
| `amount_coins` | string | Stablecoin amount (decimal string) |
| `coin_symbol` | string | e.g., `"USDT"` |
| `chain_id` | int | EIP-155 chain ID |
| `tx_hash` | string | On-chain transaction hash |
| `tx_from` | string | Sender wallet address |
| `order_id` | string or null | Your order ID (you set this in the checkout intent) |
| `user_id` | string or null | Your user ID (you set this in the checkout intent) |

### Reference implementation — Node.js (Express)

```javascript
import express from "express";
import crypto from "crypto";

const app = express();

// IMPORTANT: capture the raw body for signature verification.
// JSON re-serialization will produce a different SHA-256.
app.post(
  "/webhooks/allscale",
  express.raw({ type: "application/json" }),
  async (req, res) => {
    const apiSecret = process.env.ALLSCALE_API_SECRET;
    const rawBody = req.body.toString("utf8"); // raw bytes as string

    const apiKey       = req.header("X-API-Key");
    const webhookId    = req.header("X-Webhook-Id");
    const timestamp    = req.header("X-Webhook-Timestamp");
    const nonce        = req.header("X-Webhook-Nonce");
    const signature    = req.header("X-Webhook-Signature");

    if (!apiKey || !webhookId || !timestamp || !nonce || !signature) {
      return res.status(400).send("missing headers");
    }

    // 1. Reject stale requests (±5 minute window)
    const now = Math.floor(Date.now() / 1000);
    if (Math.abs(now - parseInt(timestamp, 10)) > 300) {
      return res.status(400).send("stale timestamp");
    }

    // 2. Reject replayed nonces (use Redis in production with TTL > 5 min)
    if (await nonceStore.has(nonce)) {
      return res.status(400).send("nonce replay");
    }

    // 3. Compute expected signature
    const bodyHash = crypto.createHash("sha256").update(rawBody).digest("hex");
    const canonical = [
      "allscale:webhook:v1",
      "POST",
      "/webhooks/allscale",
      "",                       // query string
      webhookId,
      timestamp,
      nonce,
      bodyHash,
    ].join("\n");

    const expected = "v1=" + crypto
      .createHmac("sha256", apiSecret)
      .update(canonical)
      .digest("base64");

    // 4. Timing-safe compare
    const sigBuf = Buffer.from(signature);
    const expBuf = Buffer.from(expected);
    if (sigBuf.length !== expBuf.length || !crypto.timingSafeEqual(sigBuf, expBuf)) {
      return res.status(401).send("bad signature");
    }

    // 5. Mark nonce used (TTL = 10 min covers the ±5 min window)
    await nonceStore.set(nonce, true, 600);

    // 6. Now safe to parse + handle the event
    const payload = JSON.parse(rawBody);
    if (payload.webhook_id !== webhookId) {
      return res.status(400).send("webhook_id mismatch");
    }

    await handlePayment(payload); // your business logic — fulfill the order

    return res.status(200).send("ok");
  }
);
```

### Reference implementation — Python (Flask)

```python
import hmac, hashlib, base64, os, time, json
from flask import Flask, request, abort

app = Flask(__name__)
API_SECRET = os.environ["ALLSCALE_API_SECRET"]

@app.post("/webhooks/allscale")
def allscale_webhook():
    raw_body = request.get_data()  # raw bytes — DO NOT re-serialize
    api_key   = request.headers.get("X-API-Key")
    webhook_id = request.headers.get("X-Webhook-Id")
    timestamp  = request.headers.get("X-Webhook-Timestamp")
    nonce      = request.headers.get("X-Webhook-Nonce")
    signature  = request.headers.get("X-Webhook-Signature")

    if not all([api_key, webhook_id, timestamp, nonce, signature]):
        abort(400, "missing headers")

    # 1. Stale-timestamp check (±5 min)
    if abs(int(time.time()) - int(timestamp)) > 300:
        abort(400, "stale timestamp")

    # 2. Nonce replay protection (use Redis in production)
    if nonce_store.has(nonce):
        abort(400, "nonce replay")

    # 3. Compute expected signature
    body_hash = hashlib.sha256(raw_body).hexdigest()
    canonical = "\n".join([
        "allscale:webhook:v1",
        "POST",
        "/webhooks/allscale",
        "",                # query string
        webhook_id,
        timestamp,
        nonce,
        body_hash,
    ])
    expected = "v1=" + base64.b64encode(
        hmac.new(API_SECRET.encode(), canonical.encode(), hashlib.sha256).digest()
    ).decode()

    # 4. Timing-safe compare
    if not hmac.compare_digest(expected, signature):
        abort(401, "bad signature")

    # 5. Mark nonce used
    nonce_store.set(nonce, True, ttl_seconds=600)

    # 6. Parse + handle
    payload = json.loads(raw_body.decode())
    if payload["webhook_id"] != webhook_id:
        abort(400, "webhook_id mismatch")

    handle_payment(payload)  # your business logic
    return ("ok", 200)
```

### Verification checklist (always do all six)

1. ✅ Timestamp within ±5 minutes of server time
2. ✅ Nonce not seen before — store in Redis/Memcached/in-memory with TTL ≥ 10 min
3. ✅ Body SHA-256 computed from **raw bytes** before any JSON parse / re-serialization
4. ✅ Canonical string built with the literal `allscale:webhook:v1` prefix and `\n` separators
5. ✅ Compared with **timing-safe** equality (never `===` / `==`)
6. ✅ Respond with `200 OK` only after verification passes

### Local development with webhooks

To test webhooks before deploying, expose your local server with a tunnel:

```bash
# ngrok
ngrok http 3000
# → use the https URL as your webhook URL in the Allscale dashboard

# or Cloudflare Tunnel
cloudflared tunnel --url http://localhost:3000
```

Webhooks **must** be HTTPS — Allscale will not call HTTP endpoints. Both ngrok and cloudflared give you HTTPS for free.

### Idempotency reminder

Allscale may retry a webhook on transient failures. Make your handler idempotent:
- Use the `all_scale_checkout_intent_id` as the dedupe key
- If the intent is already marked paid in your DB, return `200 OK` without re-running fulfillment

---

## Step 8: Polling (Fallback)

Use polling only if webhooks aren't an option (no backend, can't expose a public URL).

### Endpoint: `GET /v1/checkout_intents/{intent_id}/status`

Sign this request the same way as Step 3 (regular API auth, not the webhook prefix).

No request body. Returns:

```json
{
  "code": 0,
  "payload": 20,
  "error": null,
  "request_id": "req_xxxxx"
}
```

`payload` is the status integer:

| Value | Name | Meaning | Terminal? |
|---|---|---|---|
| -4 | CANCELED | User canceled | Yes |
| -3 | UNDERPAID | Paid less than required | Yes |
| -2 | REJECTED | Failed KYT checks | Yes |
| -1 | FAILED | Processing error | Yes |
| 1 | CREATED | Intent created, not yet viewed | No |
| 2 | VIEWED | User opened checkout page | No |
| 3 | TEMP_WALLET_RECEIVED | Deposit wallet assigned | No |
| 10 | ON_CHAIN | Transaction detected, awaiting confirmation | No |
| 20 | CONFIRMED | Payment confirmed on-chain | Yes |

**Recommended polling strategy:**
- Poll every 5 seconds
- Stop when status is terminal (negative values or `20`)
- Timeout after 10 minutes
- Show user-friendly messages for each state transition

### Full intent details (optional):

`GET /v1/checkout_intents/{intent_id}` returns the complete object including `tx_hash`, `tx_from`, `actual_paid_amount`, etc. Useful for showing a payment receipt after confirmation.

---

## Debugging Signature Errors (Error Code 20002)

If they get `20002` (bad signature), check these in order:

1. Is the body hash computed from the **exact raw body string** (not re-serialized)?
2. Is the canonical string joined with `\n` (not `\r\n` or other separators)?
3. Is the path exactly right? `POST /v1/checkout_intents/` needs the **trailing slash**.
4. Is the query string an empty string `""` (not `undefined`, `null`, or missing)?
5. Is the timestamp fresh (within ±5 minutes of current UTC time)?
6. Is the API secret correct with no extra whitespace?
7. Is the HMAC using SHA-256 and the output encoded as Base64 (not hex)?

---

## Error Codes Reference

| Code | Meaning |
|---|---|
| 10001 | Validation error |
| 20001 | Missing authentication headers |
| 20002 | Invalid signature |
| 30001 | Forbidden / IP not allowed |
| 40001 | Rate limit exceeded |
| 50001 | Checkout intent not found |
| 90000 | Internal server error |

---

## Working Examples

| Example | Stack | Confirmation flow | Best for |
|---|---|---|---|
| [Buy Me a Bagel](https://github.com/allscale-io/buy_me_a_bagel) | Vanilla JS + Vercel serverless | **Polling** | Static donation/tipping pages, no backend |

Buy Me a Bagel files to point at:
- `api/checkout.js` — checkout intent creation with HMAC signing and rate limiting
- `api/status.js` — status polling with signing
- `app.js` — frontend checkout flow with status polling UI

A webhook-first reference example is in active development. Until it ships, use the webhook code samples in **Step 7** of this skill as the canonical reference.

API documentation: https://github.com/allscale-io/AllScale_Third-Party_API_Doc
