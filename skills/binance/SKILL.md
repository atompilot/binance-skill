---
name: binance
description: Binance Spot API integration for cryptocurrency market data and trading. Covers REST API (klines, trades, depth, ticker, orders, account), WebSocket Streams (aggTrade, kline, depth, bookTicker, miniTicker, user data), rate limits, authentication (HMAC-SHA256/RSA/Ed25519), and error handling. Use when building trading bots, market data collectors, quantitative strategies, or any application integrating with Binance Spot.
metadata:
  version: 1.0.0
  author: atompilot
license: MIT
---

# Binance Spot Skill

Complete Binance Spot API reference for AI agents. Covers REST API, WebSocket Streams, authentication, rate limits, and error handling.

## When to use this skill

Use this skill when the user asks about or needs to build:
- Binance market data fetching (klines, trades, depth, ticker)
- WebSocket stream subscriptions (aggTrade, kline, depth, bookTicker, miniTicker)
- Real-time price monitoring or data collection
- Order placement and management (limit, market, OCO, SOR)
- Account information and trade history
- HMAC-SHA256 / RSA / Ed25519 request signing
- Rate limit management and error handling
- User Data Stream (listenKey, account updates, order updates)

## API Configuration

| API | Base URL | Auth | Purpose |
|-----|----------|------|---------|
| REST (Mainnet) | `https://api.binance.com` | API Key + Signature for trade endpoints | Market data, orders, account |
| REST (Testnet) | `https://testnet.binance.vision` | Separate testnet credentials | Development/testing |
| WebSocket Streams | `wss://stream.binance.com:9443` | None for market data | Real-time market data |
| WebSocket API | `wss://ws-api.binance.com:443/ws-api/v3` | API Key + Signature | Request/response over WS |
| User Data Stream | `wss://stream.binance.com:9443/ws/<listenKey>` | listenKey from REST | Account/order updates |

## REST API — Market Data (No Auth Required)

| Endpoint | Method | Weight | Description |
|----------|--------|--------|-------------|
| `/api/v3/ping` | GET | 1 | Test connectivity |
| `/api/v3/time` | GET | 1 | Server time |
| `/api/v3/exchangeInfo` | GET | 20 | Exchange info, trading rules, symbol list |
| `/api/v3/depth` | GET | 5-50 | Order book (limit: 5/10/20/50/100/500/1000/5000) |
| `/api/v3/trades` | GET | 25 | Recent trades (limit max 1000) |
| `/api/v3/historicalTrades` | GET | 25 | Old trades (needs API Key header) |
| `/api/v3/aggTrades` | GET | 2 | Compressed/aggregate trades |
| `/api/v3/klines` | GET | 2 | Kline/candlestick data |
| `/api/v3/uiKlines` | GET | 2 | Modified klines optimized for UI |
| `/api/v3/avgPrice` | GET | 2 | Current average price |
| `/api/v3/ticker/24hr` | GET | 2-80 | 24hr price change statistics |
| `/api/v3/ticker/price` | GET | 2-4 | Symbol price ticker |
| `/api/v3/ticker/bookTicker` | GET | 2-4 | Best price/qty on the order book |
| `/api/v3/ticker` | GET | 4-80 | Rolling window price change stats |

### Klines Parameters

| Param | Required | Description |
|-------|----------|-------------|
| symbol | Yes | e.g. `BTCUSDT` |
| interval | Yes | `1s` `1m` `3m` `5m` `15m` `30m` `1h` `2h` `4h` `6h` `8h` `12h` `1d` `3d` `1w` `1M` |
| startTime | No | Timestamp in ms |
| endTime | No | Timestamp in ms |
| timeZone | No | Default: 0 (UTC) |
| limit | No | Default 500, max 1000 |

### Klines Response Format

```json
[
  [
    1499040000000,      // Kline open time
    "0.01634000",       // Open price
    "0.80000000",       // High price
    "0.01575800",       // Low price
    "0.01577100",       // Close price
    "148976.11427815",  // Volume
    1499644799999,      // Kline close time
    "2434.19055334",    // Quote asset volume
    308,                // Number of trades
    "1756.87402397",    // Taker buy base asset volume
    "28.46694368",      // Taker buy quote asset volume
    "0"                 // Unused field, ignore
  ]
]
```

### Klines Pagination

Next page: set `startTime` = previous batch's last `close_time + 1ms`.

## REST API — Trading (Auth Required)

| Endpoint | Method | Weight | Description |
|----------|--------|--------|-------------|
| `/api/v3/order` | POST | 1 | New order |
| `/api/v3/order` | GET | 4 | Query order |
| `/api/v3/order` | DELETE | 1 | Cancel order |
| `/api/v3/order/test` | POST | 1 | Test new order (no execution) |
| `/api/v3/openOrders` | GET | 6-40 | Current open orders |
| `/api/v3/openOrders` | DELETE | 1 | Cancel all open orders on a symbol |
| `/api/v3/allOrders` | GET | 20 | All orders (incl. historical) |
| `/api/v3/account` | GET | 20 | Account information |
| `/api/v3/myTrades` | GET | 20 | Account trade list |
| `/api/v3/rateLimit/order` | GET | 40 | Query unfilled order count |

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

### Enums

- **side**: `BUY` | `SELL`
- **timeInForce**: `GTC` (Good Til Canceled) | `IOC` (Immediate Or Cancel) | `FOK` (Fill Or Kill)
- **newOrderRespType**: `ACK` | `RESULT` | `FULL`

## WebSocket Streams

### Connection Info

| Property | Value |
|----------|-------|
| Base URL | `wss://stream.binance.com:9443` |
| Single stream | `/ws/<streamName>` |
| Combined streams | `/stream?streams=<stream1>/<stream2>` |
| Connection lifetime | 24 hours (auto-disconnect) |
| Server ping | Every 20 seconds |
| Pong required | Within 60 seconds of ping |
| Max streams per connection | 1024 |
| Request rate limit | 5 requests/second/connection |
| Connection limit | 300 attempts per 5 min per IP |
| Symbols | Must be **lowercase** (e.g. `btcusdt`) |

### Combined Stream Wrapper

When using `/stream?streams=`, events are wrapped:
```json
{"stream": "btcusdt@aggTrade", "data": { ... raw payload ... }}
```

### Stream Subscription (Dynamic)

```json
{"method": "SUBSCRIBE", "params": ["btcusdt@aggTrade", "btcusdt@depth"], "id": 1}
```
```json
{"method": "UNSUBSCRIBE", "params": ["btcusdt@depth"], "id": 2}
```
```json
{"method": "LIST_SUBSCRIPTIONS", "id": 3}
```

### Available Streams

#### Aggregate Trade Stream — `<symbol>@aggTrade`
Real-time. Compressed trades (trades with same price, direction, and time aggregated).
```json
{
  "e": "aggTrade",       // Event type
  "E": 1672515782136,    // Event time (ms)
  "s": "BTCUSDT",        // Symbol
  "a": 12345,            // Aggregate trade ID
  "p": "97500.10",       // Price
  "q": "0.015",          // Quantity
  "f": 100,              // First trade ID
  "l": 105,              // Last trade ID
  "T": 1672515782136,    // Trade time (ms)
  "m": true,             // Is buyer the market maker?
  "M": true              // Ignore
}
```

#### Trade Stream — `<symbol>@trade`
Real-time. Individual raw trades.
```json
{
  "e": "trade",          // Event type
  "E": 1672515782136,    // Event time
  "s": "BTCUSDT",        // Symbol
  "t": 12345,            // Trade ID
  "p": "97500.10",       // Price
  "q": "0.015",          // Quantity
  "T": 1672515782136,    // Trade time
  "m": true,             // Is buyer the market maker?
  "M": true              // Ignore
}
```

#### Kline/Candlestick Stream — `<symbol>@kline_<interval>`
Updates every 2 seconds. Interval: `1s` `1m` `3m` `5m` `15m` `30m` `1h` `2h` `4h` `6h` `8h` `12h` `1d` `3d` `1w` `1M`
```json
{
  "e": "kline",
  "E": 1672515782136,
  "s": "BTCUSDT",
  "k": {
    "t": 1672515780000,  // Kline start time
    "T": 1672515839999,  // Kline close time
    "s": "BTCUSDT",
    "i": "1m",           // Interval
    "f": 100,            // First trade ID
    "L": 200,            // Last trade ID
    "o": "97500.00",     // Open price
    "c": "97510.50",     // Close price
    "h": "97520.00",     // High price
    "l": "97490.00",     // Low price
    "v": "1000",         // Base asset volume
    "n": 100,            // Number of trades
    "x": false,          // Is this kline closed?
    "q": "1000000",      // Quote asset volume
    "V": "500",          // Taker buy base asset volume
    "Q": "500000",       // Taker buy quote asset volume
    "B": "0"             // Ignore
  }
}
```

#### Individual Symbol Mini Ticker — `<symbol>@miniTicker`
Updates every 1000ms. 24hr rolling window.
```json
{
  "e": "24hrMiniTicker",
  "E": 1672515782136,
  "s": "BTCUSDT",
  "c": "97510.50",       // Close price
  "o": "96800.00",       // Open price
  "h": "98000.00",       // High price
  "l": "96500.00",       // Low price
  "v": "15000",          // Total traded base asset volume
  "q": "1460000000"      // Total traded quote asset volume
}
```

#### Individual Symbol Book Ticker — `<symbol>@bookTicker`
Real-time. Best bid/ask price and quantity.
```json
{
  "u": 400900217,        // Order book updateId
  "s": "BTCUSDT",
  "b": "97500.00",       // Best bid price
  "B": "1.500",          // Best bid qty
  "a": "97500.50",       // Best ask price
  "A": "2.300"           // Best ask qty
}
```

#### Partial Book Depth — `<symbol>@depth<levels>` or `<symbol>@depth<levels>@<speed>`
Levels: 5, 10, 20. Speed: 1000ms (default) or 100ms.
```json
{
  "lastUpdateId": 160,
  "bids": [["97500.00", "1.500"], ["97499.50", "3.200"]],
  "asks": [["97500.50", "2.300"], ["97501.00", "1.100"]]
}
```

#### Diff. Depth Stream — `<symbol>@depth` or `<symbol>@depth@100ms`
Updates: 1000ms (default) or 100ms. For maintaining a local order book.
```json
{
  "e": "depthUpdate",
  "E": 1672515782136,
  "s": "BTCUSDT",
  "U": 157,              // First update ID in event
  "u": 160,              // Final update ID in event
  "b": [["97500.00", "1.500"]],  // Bids [price, qty]
  "a": [["97500.50", "0.000"]]   // Asks (qty=0 means remove)
}
```

### User Data Stream

Requires a `listenKey` obtained via REST API.

| Endpoint | Method | Weight | Description |
|----------|--------|--------|-------------|
| `/api/v3/userDataStream` | POST | 2 | Create listenKey |
| `/api/v3/userDataStream` | PUT | 2 | Keepalive (every 30 min) |
| `/api/v3/userDataStream` | DELETE | 2 | Close listenKey |

Connect: `wss://stream.binance.com:9443/ws/<listenKey>`

#### Account Update
```json
{
  "e": "outboundAccountPosition",
  "E": 1564034571105,
  "u": 1564034571073,
  "B": [{"a": "BTC", "f": "10.000000", "l": "0.000000"}]
}
```

#### Balance Update (deposit/withdrawal/transfer)
```json
{
  "e": "balanceUpdate",
  "E": 1573200697110,
  "a": "BTC",
  "d": "100.00000000",
  "T": 1573200697068
}
```

#### Order Update
```json
{
  "e": "executionReport",
  "E": 1499405658658,
  "s": "BTCUSDT",
  "S": "BUY",
  "o": "LIMIT",
  "q": "1.00000000",
  "p": "97500.00",
  "X": "NEW",
  "i": 4293153,
  "l": "0.00000000",
  "z": "0.00000000",
  "T": 1499405658657
}
```

## Rate Limits

### IP Limits (REQUEST_WEIGHT)

| Limit | Value |
|-------|-------|
| Request weight | 6000 per minute per IP |
| Response header | `X-MBX-USED-WEIGHT-1M` |
| 429 response | Back off immediately |
| 418 response | IP banned (2 min → 3 days for repeat offenders) |
| `Retry-After` header | Seconds to wait |

### Order Limits (ORDERS)

| Limit | Value |
|-------|-------|
| Per 10 seconds | 50 orders |
| Per 24 hours | 160,000 orders |

### WebSocket Limits

| Limit | Value |
|-------|-------|
| Requests per second per connection | 5 |
| Max streams per connection | 1024 |
| Connection attempts | 300 per 5 min per IP |
| Connection lifetime | 24 hours |

### Common Endpoint Weights

| Endpoint | Weight |
|----------|--------|
| `/api/v3/klines` | 2 |
| `/api/v3/aggTrades` | 2 |
| `/api/v3/depth` (limit ≤ 100) | 5 |
| `/api/v3/depth` (limit 500) | 10 |
| `/api/v3/depth` (limit 1000) | 20 |
| `/api/v3/depth` (limit 5000) | 50 |
| `/api/v3/trades` | 25 |
| `/api/v3/exchangeInfo` | 20 |
| `/api/v3/account` | 20 |
| `/api/v3/order` POST | 1 |

## Error Codes

| Code | HTTP | Description | Action |
|------|------|-------------|--------|
| -1000 | 500 | Unknown error | Retry with backoff |
| -1001 | 500 | Disconnected | Retry |
| -1002 | 401 | Unauthorized | Check API key |
| -1003 | 429/418 | Too many requests / IP banned | Back off, check `Retry-After` |
| -1006 | 500 | Unexpected response | Retry |
| -1007 | 500 | Timeout | Retry |
| -1008 | 503 | Server busy | Retry with backoff |
| -1015 | 429 | Too many orders | Reduce order rate |
| -1021 | 400 | Timestamp outside recvWindow | Sync clock, increase recvWindow |
| -1022 | 400 | Invalid signature | Check secretKey, encoding |
| -1100 | 400 | Illegal characters | Validate parameters |
| -1102 | 400 | Mandatory parameter missing | Check required params |
| -2010 | 400 | New order rejected | Check balance/filters |
| -2011 | 400 | Cancel order rejected | Check orderId exists |
| -2013 | 400 | Order does not exist | Verify orderId |

## Authentication

See [`references/authentication.md`](./references/authentication.md) for HMAC-SHA256, RSA, and Ed25519 signing details.

### Quick Signing Example (Python)

```python
import hashlib, hmac, time, requests

API_KEY = "your_api_key"
SECRET_KEY = "your_secret_key"
BASE_URL = "https://api.binance.com"

def signed_request(method, endpoint, params=None):
    params = params or {}
    params["timestamp"] = int(time.time() * 1000)
    query = "&".join(f"{k}={v}" for k, v in params.items())
    signature = hmac.new(
        SECRET_KEY.encode(), query.encode(), hashlib.sha256
    ).hexdigest()
    query += f"&signature={signature}"
    headers = {"X-MBX-APIKEY": API_KEY}
    return requests.request(method, f"{BASE_URL}{endpoint}?{query}", headers=headers)

# Example: get account info
resp = signed_request("GET", "/api/v3/account")
```

### Quick WebSocket Example (Python)

```python
import asyncio, json, websockets

async def stream_agg_trades():
    url = "wss://stream.binance.com:9443/ws/btcusdt@aggTrade"
    async with websockets.connect(url) as ws:
        while True:
            msg = json.loads(await ws.recv())
            print(f"Price: {msg['p']}, Qty: {msg['q']}, Time: {msg['T']}")

asyncio.run(stream_agg_trades())
```

## Differences from Official binance-skills-hub

This skill extends `binance/binance-skills-hub/spot` with:
- **WebSocket Streams**: Full coverage of all market data streams with payload formats
- **Rate Limits**: Detailed weight tables, IP/order limits, ban escalation
- **Error Codes**: Common errors with recommended actions
- **User Data Stream**: listenKey management, account/order update payloads
- **Code Examples**: Python examples for signing and WebSocket

When the official skill adds these features, consider migrating back.
