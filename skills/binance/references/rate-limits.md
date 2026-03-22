# Binance Rate Limits Reference

## Rate Limit Types

| Type | Scope | Limit | Window |
|------|-------|-------|--------|
| REQUEST_WEIGHT | Per IP | 6000 | 1 minute |
| ORDERS | Per account | 50 | 10 seconds |
| ORDERS | Per account | 160,000 | 24 hours |
| RAW_REQUESTS | Per IP | 61,000 | 5 minutes |

## Monitoring Usage

### REST API Response Headers

| Header | Description |
|--------|-------------|
| `X-MBX-USED-WEIGHT-1M` | Current IP weight usage in 1-minute window |
| `X-MBX-ORDER-COUNT-10S` | Orders placed in 10-second window |
| `X-MBX-ORDER-COUNT-1D` | Orders placed in 24-hour window |

### WebSocket API Rate Limits in Response

```json
{
  "rateLimits": [
    {"rateLimitType": "REQUEST_WEIGHT", "interval": "MINUTE", "intervalNum": 1, "limit": 6000, "count": 70},
    {"rateLimitType": "ORDERS", "interval": "SECOND", "intervalNum": 10, "limit": 50, "count": 5},
    {"rateLimitType": "ORDERS", "interval": "DAY", "intervalNum": 1, "limit": 160000, "count": 100}
  ]
}
```

## Common Endpoint Weights

### Market Data (No Auth)

| Endpoint | Weight | Notes |
|----------|--------|-------|
| `GET /api/v3/ping` | 1 | |
| `GET /api/v3/time` | 1 | |
| `GET /api/v3/exchangeInfo` | 20 | |
| `GET /api/v3/klines` | 2 | limit max 1000 |
| `GET /api/v3/aggTrades` | 2 | limit max 1000 |
| `GET /api/v3/trades` | 25 | |
| `GET /api/v3/historicalTrades` | 25 | |
| `GET /api/v3/avgPrice` | 2 | |
| `GET /api/v3/depth` limit ≤ 100 | 5 | |
| `GET /api/v3/depth` limit 500 | 10 | |
| `GET /api/v3/depth` limit 1000 | 20 | |
| `GET /api/v3/depth` limit 5000 | 50 | |
| `GET /api/v3/ticker/24hr` (1 symbol) | 2 | |
| `GET /api/v3/ticker/24hr` (all) | 80 | |
| `GET /api/v3/ticker/price` (1 symbol) | 2 | |
| `GET /api/v3/ticker/price` (all) | 4 | |

### Trading (Auth Required)

| Endpoint | Weight | Notes |
|----------|--------|-------|
| `POST /api/v3/order` | 1 | +1 order count |
| `POST /api/v3/order/test` | 1 | No order count |
| `GET /api/v3/order` | 4 | |
| `DELETE /api/v3/order` | 1 | |
| `GET /api/v3/openOrders` (1 symbol) | 6 | |
| `GET /api/v3/openOrders` (all) | 40 | |
| `GET /api/v3/allOrders` | 20 | |
| `GET /api/v3/account` | 20 | |
| `GET /api/v3/myTrades` | 20 | |
| `GET /api/v3/rateLimit/order` | 40 | |

## Error Handling

### 429 — Too Many Requests

```json
{
  "code": -1003,
  "msg": "Too much request weight used; current limit is 6000 request weight per 1 MINUTE."
}
```

**Action**: Implement exponential backoff. Check `Retry-After` header.

### 418 — IP Banned

```json
{
  "code": -1003,
  "msg": "Way too much request weight used; IP banned until 1659146400000.",
  "data": {"serverTime": 1659142907531, "retryAfter": 1659146400000}
}
```

**Action**: Wait until `retryAfter` timestamp. Ban duration escalates:
- 1st offense: 2 minutes
- Repeat: up to 3 days

### -1015 — Too Many Orders

```json
{"code": -1015, "msg": "Too many new orders; current limit is 50 orders per 10 SECOND."}
```

**Action**: Reduce order submission rate. Consider batching.

## WebSocket Rate Limits

| Limit | Value |
|-------|-------|
| Requests per second per connection | 5 (includes ping/pong frames) |
| Max streams per connection | 1024 |
| Connection attempts | 300 per 5 min per IP |
| Connection lifetime | 24 hours |

## Best Practices

### For Market Data Collection

1. **Prefer WebSocket over REST polling** — no weight cost for WS events
2. **Batch klines requests** — each call costs 2 weight, with limit=1000 you get 1000 candles
3. **Calculate weight budget**: 6000/min ÷ 2 weight = 3000 klines calls/min max
4. **Monitor `X-MBX-USED-WEIGHT-1M`** — stop if approaching 5000

### For Trading

1. **Track order count** — 50/10s is easy to hit with HFT
2. **Use `POST /api/v3/order/test`** for validation without counting
3. **Batch cancels** — use `DELETE /api/v3/openOrders` to cancel all at once (weight 1 vs N)
4. **Use WebSocket API** for lower latency order submission

### Backoff Strategy

```python
import time

def backoff_request(func, max_retries=5):
    for attempt in range(max_retries):
        resp = func()
        if resp.status_code == 429:
            retry_after = int(resp.headers.get("Retry-After", 2 ** attempt))
            time.sleep(retry_after)
            continue
        if resp.status_code == 418:
            retry_after = int(resp.headers.get("Retry-After", 300))
            time.sleep(retry_after)
            continue
        return resp
    raise Exception("Max retries exceeded")
```
