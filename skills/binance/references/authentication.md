# Binance Authentication

All trading and account endpoints require signed requests.

## Base URLs

| Environment | URL |
|-------------|-----|
| Mainnet | `https://api.binance.com` |
| Testnet | `https://testnet.binance.vision` |
| Demo | `https://demo-api.binance.com` |

## Required Headers

| Header | Value |
|--------|-------|
| `X-MBX-APIKEY` | Your API key |

## Signing Process

### Step 1: Build Query String

Include all parameters plus `timestamp` (current Unix time in milliseconds):

```
symbol=BTCUSDT&side=BUY&type=MARKET&quantity=0.001&timestamp=1234567890123
```

Optional: Add `recvWindow` (default 5000ms, max 60000ms) for timestamp tolerance.

### Step 2: Percent-Encode Parameters (RFC 3986)

Before signing, percent-encode all parameter names and values using UTF-8.
Unreserved characters that must NOT be encoded: `A-Z a-z 0-9 - _ . ~`

The exact encoded query string must be used for both signing and the HTTP request.

### Step 3: Generate Signature

#### HMAC SHA256 (most common)

```bash
echo -n "symbol=BTCUSDT&side=BUY&type=MARKET&quantity=0.001&timestamp=1234567890123" | \
  openssl dgst -sha256 -hmac "your_secret_key"
```

#### Python

```python
import hashlib, hmac

query = "symbol=BTCUSDT&side=BUY&type=MARKET&quantity=0.001&timestamp=1234567890123"
signature = hmac.new(
    "your_secret_key".encode(), query.encode(), hashlib.sha256
).hexdigest()
```

#### RSA

```bash
echo -n "<query_string>" | openssl dgst -sha256 -sign private_key.pem | base64
```

#### Ed25519

```bash
echo -n "<query_string>" | openssl pkeyutl -sign -inkey private_key.pem | base64
```

### Step 4: Append Signature

Add `signature` parameter to the query string:

```
symbol=BTCUSDT&side=BUY&type=MARKET&quantity=0.001&timestamp=1234567890123&signature=abc123...
```

### Complete Example

```bash
#!/bin/bash
API_KEY="your_api_key"
SECRET_KEY="your_secret_key"
BASE_URL="https://api.binance.com"

TIMESTAMP=$(date +%s000)
QUERY="symbol=BTCUSDT&side=BUY&type=MARKET&quantity=0.001&timestamp=${TIMESTAMP}"
SIGNATURE=$(echo -n "$QUERY" | openssl dgst -sha256 -hmac "$SECRET_KEY" | cut -d' ' -f2)

curl -X POST "${BASE_URL}/api/v3/order?${QUERY}&signature=${SIGNATURE}" \
  -H "X-MBX-APIKEY: ${API_KEY}"
```

## Timestamp Sync

If you get error `-1021 Timestamp outside recvWindow`:

1. Check server time: `GET /api/v3/time`
2. Calculate offset: `server_time - local_time`
3. Apply offset to all subsequent requests
4. Or increase `recvWindow` (max 60000ms)

## Security Best Practices

- Never share your secret key
- Use IP whitelist in Binance API settings
- Enable only required permissions (e.g. spot trading, no withdrawals)
- Use testnet for development
- Testnet credentials are separate from mainnet
- Never log or display full API keys/secrets
