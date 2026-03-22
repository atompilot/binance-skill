# Binance WebSocket API Reference

The WebSocket API allows you to send requests and receive responses over a persistent WebSocket connection — including placing orders, querying account info, and managing subscriptions. It is **different from WebSocket Streams** (which only push market data).

## WS Streams vs WS API

| Feature | WS Streams | WS API |
|---------|-----------|--------|
| URL | `wss://stream.binance.com:9443` | `wss://ws-api.binance.com:443/ws-api/v3` |
| Direction | Server → Client (push only) | Bidirectional (request/response) |
| Use case | Market data feeds | Trading, queries, account ops |
| Auth | None (market data) / listenKey (user data) | API Key + Signature or Session auth |
| Latency | N/A | Lower than REST (no HTTP overhead) |

## Connection

### Endpoints

| Environment | URL |
|-------------|-----|
| Production | `wss://ws-api.binance.com:443/ws-api/v3` |
| Production (alt) | `wss://ws-api.binance.com:9443/ws-api/v3` |
| Testnet | `wss://ws-api.testnet.binance.vision/ws-api/v3` |

### Connection Rules

- Connection valid for **24 hours** (auto-disconnect)
- Server sends **ping every 20s** — must reply pong within 60s
- Server sends `serverShutdown` event before planned disconnection
- Request processing timeout: **10 seconds**

### Optional URL Parameters

| Param | Values | Description |
|-------|--------|-------------|
| `timeUnit` | `MILLISECOND` (default), `MICROSECOND` | Timestamp precision in responses |
| `responseFormat` | `json` (default), `sbe` | Response encoding format |

Example: `wss://ws-api.binance.com:443/ws-api/v3?timeUnit=MICROSECOND`

## Request / Response Format

### Request

```json
{
  "id": "my-request-1",
  "method": "order.place",
  "params": {
    "symbol": "BTCUSDT",
    "side": "BUY",
    "type": "LIMIT",
    "timeInForce": "GTC",
    "quantity": "0.001",
    "price": "95000",
    "apiKey": "your_api_key",
    "signature": "...",
    "timestamp": 1700000000000
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `id` | string or int | Unique request ID (correlates with response) |
| `method` | string | API method name (e.g. `order.place`, `account.status`) |
| `params` | object | Method-specific parameters |

### Response (Success)

```json
{
  "id": "my-request-1",
  "status": 200,
  "result": { ... },
  "rateLimits": [
    {"rateLimitType": "REQUEST_WEIGHT", "interval": "MINUTE", "intervalNum": 1, "limit": 6000, "count": 71},
    {"rateLimitType": "ORDERS", "interval": "SECOND", "intervalNum": 10, "limit": 50, "count": 1}
  ]
}
```

### Response (Error)

```json
{
  "id": "my-request-1",
  "status": 400,
  "error": {"code": -1102, "msg": "Mandatory parameter 'side' was not sent."},
  "rateLimits": [...]
}
```

### Suppress Rate Limit Info

Add `"returnRateLimits": false` to `params` to omit the `rateLimits` field from responses — reduces payload size in high-frequency scenarios.

```json
{"id": 1, "method": "time", "params": {"returnRateLimits": false}}
```

## Session Authentication

Instead of signing every request, you can authenticate once per session with `session.logon`. After that, `apiKey` and `signature` can be omitted from subsequent requests (you still need `timestamp` for SIGNED requests).

### Login

```json
{
  "id": "login",
  "method": "session.logon",
  "params": {
    "apiKey": "your_api_key",
    "signature": "...",
    "timestamp": 1700000000000
  }
}
```

Response:
```json
{
  "id": "login",
  "status": 200,
  "result": {"apiKey": "your_api_key", "authorizedSince": 1700000000000, "connectedSince": 1699999990000, "returnRateLimits": false, "serverTime": 1700000000050}
}
```

### After Login — Simplified Requests

```json
{
  "id": "order-1",
  "method": "order.place",
  "params": {
    "symbol": "BTCUSDT",
    "side": "BUY",
    "type": "MARKET",
    "quantity": "0.001",
    "timestamp": 1700000001000
  }
}
```

### Logout

```json
{"id": "logout", "method": "session.logout"}
```

### Session Status

```json
{"id": "status", "method": "session.status"}
```

## Common Methods

### Market Data (No Auth)

| Method | REST Equivalent | Description |
|--------|----------------|-------------|
| `ping` | `GET /api/v3/ping` | Test connectivity |
| `time` | `GET /api/v3/time` | Server time |
| `exchangeInfo` | `GET /api/v3/exchangeInfo` | Trading rules |
| `depth` | `GET /api/v3/depth` | Order book |
| `trades.recent` | `GET /api/v3/trades` | Recent trades |
| `trades.aggregate` | `GET /api/v3/aggTrades` | Aggregate trades |
| `klines` | `GET /api/v3/klines` | Kline/candlestick |
| `ticker.24hr` | `GET /api/v3/ticker/24hr` | 24hr stats |
| `ticker.price` | `GET /api/v3/ticker/price` | Price ticker |
| `ticker.book` | `GET /api/v3/ticker/bookTicker` | Book ticker |

### Trading (TRADE auth)

| Method | REST Equivalent | Description |
|--------|----------------|-------------|
| `order.place` | `POST /api/v3/order` | Place new order |
| `order.test` | `POST /api/v3/order/test` | Test order (no execution) |
| `order.status` | `GET /api/v3/order` | Query order |
| `order.cancel` | `DELETE /api/v3/order` | Cancel order |
| `order.cancelReplace` | `POST /api/v3/order/cancelReplace` | Atomic cancel + replace |
| `openOrders.status` | `GET /api/v3/openOrders` | Open orders |
| `openOrders.cancelAll` | `DELETE /api/v3/openOrders` | Cancel all |

### Account (USER_DATA auth)

| Method | REST Equivalent | Description |
|--------|----------------|-------------|
| `account.status` | `GET /api/v3/account` | Account info |
| `myTrades` | `GET /api/v3/myTrades` | Trade history |
| `myAllocations` | `GET /api/v3/myAllocations` | Allocation history |
| `account.commission` | `GET /api/v3/account/commission` | Commission rates |

### User Data Stream

| Method | REST Equivalent | Description |
|--------|----------------|-------------|
| `userDataStream.start` | `POST /api/v3/userDataStream` | Create listenKey |
| `userDataStream.ping` | `PUT /api/v3/userDataStream` | Keepalive |
| `userDataStream.stop` | `DELETE /api/v3/userDataStream` | Close |

## Security Types

| Type | Description | Examples |
|------|-------------|---------|
| `NONE` | Public market data | `ping`, `time`, `depth`, `klines` |
| `TRADE` | Place/cancel orders | `order.place`, `order.cancel` |
| `USER_DATA` | Account info, history | `account.status`, `myTrades` |
| `USER_STREAM` | listenKey management | `userDataStream.start` |

## Complete Python Example

```python
import hashlib, hmac, json, time
from websocket import create_connection

API_KEY = "your_api_key"
SECRET_KEY = "your_secret_key"

def sign(params):
    params["timestamp"] = int(time.time() * 1000)
    query = "&".join(f"{k}={v}" for k, v in sorted(params.items()))
    params["signature"] = hmac.new(
        SECRET_KEY.encode(), query.encode(), hashlib.sha256
    ).hexdigest()
    return params

ws = create_connection("wss://ws-api.binance.com:443/ws-api/v3")

# 1. Session login (sign once)
ws.send(json.dumps({
    "id": "login",
    "method": "session.logon",
    "params": sign({"apiKey": API_KEY})
}))
print(json.loads(ws.recv()))

# 2. Place order (no apiKey/signature needed after login)
ws.send(json.dumps({
    "id": "order-1",
    "method": "order.place",
    "params": {
        "symbol": "BTCUSDT",
        "side": "BUY",
        "type": "MARKET",
        "quantity": "0.001",
        "timestamp": int(time.time() * 1000)
    }
}))
print(json.loads(ws.recv()))

# 3. Check account
ws.send(json.dumps({
    "id": "account",
    "method": "account.status",
    "params": {"timestamp": int(time.time() * 1000)}
}))
print(json.loads(ws.recv()))

ws.close()
```

## When to Use WS API vs REST

| Scenario | Use |
|----------|-----|
| One-off queries | REST (simpler) |
| High-frequency trading | WS API (lower latency, session auth) |
| Market data collection | WS Streams (push-based, no weight cost) |
| Order + data in same connection | WS API + subscribe to streams |
| Simple scripts | REST (no connection management) |
