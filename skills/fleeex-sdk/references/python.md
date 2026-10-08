# fleeex from Python (and any other language)

There is no fleeex SDK outside JS/TS. The wire is OpenAI's, so use the official `openai`
client, and plain HTTP for the billing helpers.

## Client

```python
import os
from openai import OpenAI

FLEEEX_BASE_URL = os.environ.get("FLEEEX_BASE_URL", "https://api.fleeex.dev")


def fleeex_for(user_id: str) -> OpenAI:
    """OpenAI client billing the given end user's fleeex wallet. Server-side only."""
    return OpenAI(
        api_key=os.environ["FLEEEX_API_KEY"],
        base_url=f"{FLEEEX_BASE_URL}/v1",
        default_headers={"x-fleeex-user": user_id},
    )


completion = fleeex_for(session_user_id).chat.completions.create(
    model="nova-lite",
    messages=[{"role": "user", "content": "Hello!"}],
)
```

`AsyncOpenAI` works the same way.

## 402

The `openai` package keeps only the `error` field in `e.body`; read the full response for
the top-up link:

```python
import openai

HTTP_PAYMENT_REQUIRED = 402

try:
    ...
except openai.APIStatusError as e:
    if e.status_code != HTTP_PAYMENT_REQUIRED:
        raise
    body = e.response.json()
    topup_url = body.get("topupUrl")          # may be None
    code = (body.get("error") or {}).get("code")  # PAYMENT_REQUIRED, APP_SPEND_CAP_EXCEEDED, WALLET_SUSPENDED
```

## Billing helpers over HTTP

Same headers as the proxy: `Authorization: Bearer <FLEEEX_API_KEY>`, `x-fleeex-user: <userId>`.

| Need | Request | Response |
|---|---|---|
| Connection status + onboarding link (never `402`) | `POST /connect`, JSON body `{}` or `{"redirectUri": "..."}` | `{ connected, funded, connectUrl }` |
| Balance (`402` if no wallet connected to the app, no link: use `/connect`) | `GET /balance` | `{ balanceMicros, currency }` |
| Month-to-date usage | `GET /usage/summary` | `{ spentMicros, totalTokens, requestCount, currency, periodStart, periodEnd }` |
| Model catalog (no `x-fleeex-user` needed) | `GET /v1/models` | OpenAI list shape |

Paths are relative to `https://api.fleeex.dev` (no `/v1`, except the catalog).
`redirectUri` must exactly match a URI registered on the app. Amounts are EUR micros.

## Webhook verification

```python
import hashlib
import hmac
import time

TOLERANCE_SECONDS = 300


def verify_fleeex_webhook(raw_body: bytes, header: str, secret: str) -> bool:
    parts = dict(el.strip().split("=", 1) for el in header.split(",") if "=" in el)
    t, v1 = parts.get("t"), parts.get("v1")
    if not t or not v1 or not t.isdigit():
        return False

    expected = hmac.new(secret.encode(), f"{t}.".encode() + raw_body, hashlib.sha256).hexdigest()
    if not hmac.compare_digest(expected.encode(), v1.encode()):
        return False

    return abs(int(time.time()) - int(t)) <= TOLERANCE_SECONDS
```

`secret` is the full `whsec_...` value. Pass the **raw** request bytes (FastAPI
`await request.body()`, Flask `request.get_data()`, Django `request.body`), never
re-serialized JSON. Header: `x-fleeex-signature`.
