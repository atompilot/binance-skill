---
name: binance
description: Binance Spot API integration for cryptocurrency market data and trading. Covers REST API (klines, trades, depth, ticker, orders, account), WebSocket Streams (aggTrade, kline, depth, bookTicker, miniTicker, user data), trading filters, rate limits, authentication (HMAC-SHA256/RSA/Ed25519), and error handling. Use when building trading bots, market data collectors, quantitative strategies, or any application integrating with Binance Spot.
metadata:
  version: 1.2.0
  author: atompilot
license: MIT
---

# Binance Spot Skill

Complete Binance Spot API reference for AI agents.

## When to use this skill

Use this skill when the user asks about or needs to build:
- Binance market data fetching (klines, trades, depth, ticker)
- WebSocket stream subscriptions (aggTrade, kline, depth, bookTicker, miniTicker)
- Real-time price monitoring or data collection
- Order placement and management (limit, market, OCO, SOR)
- Account information and trade history
- Rate limit management and error handling

## References

| Topic | File | When to read |
|-------|------|-------------|
| WebSocket Streams | [`references/websocket.md`](./references/websocket.md) | Real-time market data feeds, stream payloads |
| WebSocket API | [`references/websocket-api.md`](./references/websocket-api.md) | Low-latency trading via WS, session auth, request/response format |
| User Data Stream | [`references/user-data-stream.md`](./references/user-data-stream.md) | Account/order/balance updates, executionReport fields, enums |
| Authentication | [`references/authentication.md`](./references/authentication.md) | Signing requests, API key setup |
| Rate Limits | [`references/rate-limits.md`](./references/rate-limits.md) | Weight budgets, backoff strategy, ban handling |
| Trading Filters | [`references/filters.md`](./references/filters.md) | Order validation rules (LOT_SIZE, PRICE_FILTER, etc.) |
| Official SDKs | [`references/sdk.md`](./references/sdk.md) | Using official connectors (Python, Node.js, etc.) |
| Demo Mode | [`references/demo-mode.md`](./references/demo-mode.md) | Simulated trading with near-real market data |

## API Configuration

| API | Base URL | Auth | Purpose |
|-----|----------|------|---------|
| REST (Mainnet) | `https://api.binance.com` | API Key + Signature for trade endpoints | Market data, orders, account |
| REST (Backup) | `https://api1~4.binance.com`, `https://api-gcp.binance.com` | Same as mainnet | Higher perf, lower stability |
| REST (Market only) | `https://data-api.binance.vision` | None | Public market data, no trade endpoints |
| REST (Testnet) | `https://testnet.binance.vision` | Separate testnet credentials | Development/testing (only `/api/*`, not `/sapi/*`) |
| WebSocket Streams | `wss://stream.binance.com:9443` | None for market data | Real-time market data |
| WebSocket API | `wss://ws-api.binance.com:443/ws-api/v3` | API Key + Signature | Request/response over WS |
| User Data Stream | `wss://stream.binance.com:9443/ws/<listenKey>` | listenKey from REST | Account/order updates |

## General Rules

**Request parameters:**
- GET → query string only
- POST/PUT/DELETE → query string or body (`Content-Type: application/x-www-form-urlencoded`)
- If same param in both, query string wins. Parameters can be in any order.

**Data ordering (startTime / endTime):**
- Neither → most recent items up to limit
- `startTime` only → oldest items from that point
- `endTime` only → most recent items before that point
- Both → from `startTime`, not exceeding `endTime`

**Timestamps:** milliseconds by default. Add header `X-MBX-TIME-UNIT: MICROSECOND` for microsecond precision.

**Request timeout:** 10 seconds. A -1007 TIMEOUT does **not** mean the order failed — check User Data Stream or query order status.

**WAF warning:** Avoid SQL keywords (`SELECT`, `DROP`, etc.) in parameter values — may be blocked by Binance's firewall.

## REST API — Market Data (No Auth)

| Endpoint | Method | Weight | Description |
|----------|--------|--------|-------------|
| `/api/v3/ping` | GET | 1 | Test connectivity |
| `/api/v3/time` | GET | 1 | Server time |
| `/api/v3/exchangeInfo` | GET | 20 | Trading rules, filters, symbol list |
| `/api/v3/depth` | GET | 5-50 | Order book (limit: 5/10/20/50/100/500/1000/5000) |
| `/api/v3/trades` | GET | 25 | Recent trades (limit max 1000) |
| `/api/v3/aggTrades` | GET | 2 | Compressed/aggregate trades |
| `/api/v3/klines` | GET | 2 | Kline/candlestick data |
| `/api/v3/avgPrice` | GET | 2 | Current average price |
| `/api/v3/ticker/24hr` | GET | 2-80 | 24hr price change statistics |
| `/api/v3/ticker/price` | GET | 2-4 | Symbol price ticker |
| `/api/v3/ticker/bookTicker` | GET | 2-4 | Best price/qty on the order book |

### Klines — Key Parameters

| Param | Required | Description |
|-------|----------|-------------|
| symbol | Yes | e.g. `BTCUSDT` |
| interval | Yes | `1s` `1m` `3m` `5m` `15m` `30m` `1h` `2h` `4h` `6h` `8h` `12h` `1d` `3d` `1w` `1M` |
| startTime / endTime | No | Timestamp in ms |
| limit | No | Default 500, max 1000 |

Response: array of `[openTime, open, high, low, close, volume, closeTime, quoteVolume, trades, takerBuyBase, takerBuyQuote, ignore]`

Pagination: set `startTime` = previous batch's last `closeTime + 1ms`.

## REST API — Trading (Auth Required)

| Endpoint | Method | Weight | Description |
|----------|--------|--------|-------------|
| `/api/v3/order` | POST | 1 | New order |
| `/api/v3/order` | GET | 4 | Query order |
| `/api/v3/order` | DELETE | 1 | Cancel order |
| `/api/v3/order/test` | POST | 1 | Test new order (no execution) |
| `/api/v3/openOrders` | GET | 6-40 | Current open orders |
| `/api/v3/allOrders` | GET | 20 | All orders (incl. historical) |
| `/api/v3/account` | GET | 20 | Account information |
| `/api/v3/myTrades` | GET | 20 | Account trade list |

### Order Types

| Type | Required Fields |
|------|----------------|
| LIMIT | timeInForce, quantity, price |
| MARKET | quantity OR quoteOrderQty |
| STOP_LOSS | quantity, stopPrice OR trailingDelta |
| STOP_LOSS_LIMIT | timeInForce, quantity, price, stopPrice OR trailingDelta |
| TAKE_PROFIT | quantity, stopPrice OR trailingDelta |
| TAKE_PROFIT_LIMIT | timeInForce, quantity, price, stopPrice OR trailingDelta |
| LIMIT_MAKER | quantity, price |

**Enums**: side = `BUY`/`SELL`, timeInForce = `GTC`/`IOC`/`FOK`

**Important**: Orders must pass symbol filters (LOT_SIZE, PRICE_FILTER, NOTIONAL, etc.) — see [`references/filters.md`](./references/filters.md).

## WebSocket Streams — Quick Reference

Full details and payload formats: [`references/websocket.md`](./references/websocket.md)

| Stream | Name Format | Speed | Description |
|--------|-------------|-------|-------------|
| Aggregate Trade | `<symbol>@aggTrade` | Real-time | Compressed trades |
| Trade | `<symbol>@trade` | Real-time | Individual trades |
| Kline | `<symbol>@kline_<interval>` | 1s→1000ms, others→2000ms | Candlestick data |
| Mini Ticker | `<symbol>@miniTicker` | 1000ms | 24hr rolling mini stats |
| Rolling Window | `<symbol>@ticker_<1h/4h/1d>` | 1000ms | Custom window stats |
| Book Ticker | `<symbol>@bookTicker` | Real-time | Best bid/ask |
| Average Price | `<symbol>@avgPrice` | 1000ms | 5-min average price |
| Partial Depth | `<symbol>@depth<levels>@<speed>` | 1000/100ms | Top 5/10/20 levels |
| Diff. Depth | `<symbol>@depth@<speed>` | 1000/100ms | Incremental updates |

**Connection rules**: symbols lowercase, max 1024 streams/conn, 24h lifetime, pong within 60s of ping.

### Dynamic Subscribe/Unsubscribe

```json
{"method": "SUBSCRIBE", "params": ["btcusdt@aggTrade", "btcusdt@depth"], "id": 1}
{"method": "UNSUBSCRIBE", "params": ["btcusdt@depth"], "id": 2}
```

## WebSocket API — Quick Reference

Full details: [`references/websocket-api.md`](./references/websocket-api.md)

The WS API (`wss://ws-api.binance.com:443/ws-api/v3`) allows **bidirectional request/response** over WebSocket — lower latency than REST, supports session auth.

- **Request**: `{"id": "req-1", "method": "order.place", "params": {...}}`
- **Response**: `{"id": "req-1", "status": 200, "result": {...}, "rateLimits": [...]}`
- **Session auth**: `session.logon` once → skip `apiKey`/`signature` on subsequent requests
- **Microsecond timestamps**: append `?timeUnit=MICROSECOND` to connection URL
- **Suppress rate limits**: add `"returnRateLimits": false` in params

Key methods: `order.place`, `order.cancel`, `order.status`, `account.status`, `klines`, `depth`

## Rate Limits — Quick Reference

Full details and backoff strategies: [`references/rate-limits.md`](./references/rate-limits.md)

| Type | Limit | Window |
|------|-------|--------|
| REQUEST_WEIGHT | 6000 | 1 minute per IP |
| ORDERS | 50 | 10 seconds per account |
| ORDERS | 160,000 | 24 hours per account |
| WS requests | 5 | 1 second per connection |
| WS connections | 300 | 5 minutes per IP |

Monitor via `X-MBX-USED-WEIGHT-1M` header. 429 = back off, 418 = IP banned (2 min → 3 days).

## Error Codes — Quick Reference

| Code | Description | Action |
|------|-------------|--------|
| -1003 | Too many requests / IP banned | Back off, check `Retry-After` |
| -1015 | Too many orders | Reduce order rate |
| -1021 | Timestamp outside recvWindow | Sync clock, increase recvWindow |
| -1022 | Invalid signature | Check secretKey, encoding |
| -1102 | Mandatory parameter missing | Check required params |
| -2010 | New order rejected | Check balance and filters |
| -2013 | Order does not exist | Verify orderId |

## Quick Examples

### Signed REST Request (Python)

```python
import hashlib, hmac, time, requests

API_KEY, SECRET_KEY = "your_key", "your_secret"

def signed_request(method, endpoint, params=None):
    params = params or {}
    params["timestamp"] = int(time.time() * 1000)
    query = "&".join(f"{k}={v}" for k, v in params.items())
    sig = hmac.new(SECRET_KEY.encode(), query.encode(), hashlib.sha256).hexdigest()
    return requests.request(method, f"https://api.binance.com{endpoint}?{query}&signature={sig}",
                            headers={"X-MBX-APIKEY": API_KEY})
```

### WebSocket Stream (Python)

```python
import asyncio, json, websockets

async def stream():
    async with websockets.connect("wss://stream.binance.com:9443/ws/btcusdt@aggTrade") as ws:
        async for msg in ws:
            data = json.loads(msg)
            print(f"Price: {data['p']}, Qty: {data['q']}, Time: {data['T']}")

asyncio.run(stream())
```
