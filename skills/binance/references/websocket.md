# Binance WebSocket Streams Reference

## Connection

### Endpoints

| Type | URL |
|------|-----|
| Single stream | `wss://stream.binance.com:9443/ws/<streamName>` |
| Combined streams | `wss://stream.binance.com:9443/stream?streams=<s1>/<s2>` |
| Market data only | `wss://data-stream.binance.vision` (no user streams) |
| Testnet | `wss://testnet.binance.vision/ws/<streamName>` |

### Connection Rules

- Symbols must be **lowercase** (e.g. `btcusdt`, not `BTCUSDT`)
- Max **1024 streams** per connection
- Connection valid for **24 hours** (auto-disconnect after)
- Server sends **ping every 20s** — must reply pong (with ping's payload) within 60s
- Max **5 incoming messages/second** per connection (includes ping/pong frames + JSON commands)
- Max **300 connection attempts** per 5 min per IP
- Repeated violations result in **IP bans**
- Optional: append `?timeUnit=MICROSECOND` for microsecond timestamps

### Combined Stream Format

When using `/stream?streams=`, each event is wrapped:

```json
{
  "stream": "btcusdt@aggTrade",
  "data": { /* raw payload */ }
}
```

## Dynamic Subscription

Send JSON messages to manage subscriptions on an existing connection:

### Subscribe

```json
{"method": "SUBSCRIBE", "params": ["btcusdt@aggTrade", "btcusdt@depth@100ms"], "id": 1}
```

Response: `{"result": null, "id": 1}`

### Unsubscribe

```json
{"method": "UNSUBSCRIBE", "params": ["btcusdt@depth@100ms"], "id": 2}
```

### List Subscriptions

```json
{"method": "LIST_SUBSCRIPTIONS", "id": 3}
```

Response: `{"result": ["btcusdt@aggTrade"], "id": 3}`

### Set/Get Property

```json
{"method": "SET_PROPERTY", "params": ["combined", true], "id": 5}
{"method": "GET_PROPERTY", "params": ["combined"], "id": 6}
```

Note: `combined` defaults to `false` for `/ws/` endpoints, `true` for `/stream/` endpoints.

### WS Error Codes

| Code | Message |
|------|---------|
| 0 | Unknown property |
| 1 | Invalid value type (expected Boolean) |
| 2 | Invalid request (bad ID, unknown method, too many params, missing `method`) |
| 3 | Invalid JSON |

## Stream Catalog

### Market Data Streams

| Stream | Name Format | Update Speed | Description |
|--------|-------------|-------------|-------------|
| Aggregate Trade | `<symbol>@aggTrade` | Real-time | Compressed trades |
| Trade | `<symbol>@trade` | Real-time | Individual trades |
| Kline (UTC) | `<symbol>@kline_<interval>` | 1000ms (1s) / 2000ms (others) | Candlestick data |
| Kline (UTC+8) | `<symbol>@kline_<interval>@+08:00` | 1000ms (1s) / 2000ms (others) | Candlestick in UTC+8 |
| Mini Ticker | `<symbol>@miniTicker` | 1000ms | 24hr rolling mini stats |
| All Mini Tickers | `!miniTicker@arr` | 1000ms | All symbols mini ticker |
| Ticker (24hr) | `<symbol>@ticker` | 1000ms | 24hr rolling full stats |
| All Tickers | `!ticker@arr` | 1000ms | All symbols ticker |
| Rolling Window | `<symbol>@ticker_<window>` | 1000ms | 1h/4h/1d window stats |
| All Rolling Window | `!ticker_<window>@arr` | 1000ms | All symbols rolling |
| Book Ticker | `<symbol>@bookTicker` | Real-time | Best bid/ask |
| Average Price | `<symbol>@avgPrice` | 1000ms | 5-min average price |
| Reference Price | `<symbol>@referencePrice` | 1000ms | Reference price |
| Partial Depth | `<symbol>@depth<levels>` | 1000ms | Top N levels (5/10/20) |
| Partial Depth (fast) | `<symbol>@depth<levels>@100ms` | 100ms | Top N levels, faster |
| Diff. Depth | `<symbol>@depth` | 1000ms | Incremental depth updates |
| Diff. Depth (fast) | `<symbol>@depth@100ms` | 100ms | Incremental, faster |

### Kline Intervals

`1s` `1m` `3m` `5m` `15m` `30m` `1h` `2h` `4h` `6h` `8h` `12h` `1d` `3d` `1w` `1M`

## Payload Examples

### aggTrade

```json
{
  "e": "aggTrade",
  "E": 1672515782136,
  "s": "BTCUSDT",
  "a": 12345,
  "p": "97500.10",
  "q": "0.015",
  "f": 100,
  "l": 105,
  "T": 1672515782136,
  "m": true,
  "M": true
}
```

| Field | Description |
|-------|-------------|
| `e` | Event type |
| `E` | Event time (ms) |
| `s` | Symbol |
| `a` | Aggregate trade ID |
| `p` | Price |
| `q` | Quantity |
| `f` / `l` | First / Last trade ID |
| `T` | Trade time (ms) — use for latency calculation |
| `m` | Is buyer the market maker? |

### kline

```json
{
  "e": "kline",
  "E": 1672515782136,
  "s": "BTCUSDT",
  "k": {
    "t": 1672515780000,
    "T": 1672515839999,
    "s": "BTCUSDT",
    "i": "1m",
    "f": 100,
    "L": 200,
    "o": "97500.00",
    "c": "97510.50",
    "h": "97520.00",
    "l": "97490.00",
    "v": "1000",
    "n": 100,
    "x": false,
    "q": "1000000",
    "V": "500",
    "Q": "500000",
    "B": "0"
  }
}
```

| Field | Description |
|-------|-------------|
| `k.t` / `k.T` | Kline start / close time |
| `k.i` | Interval |
| `k.o/c/h/l` | Open / Close / High / Low |
| `k.v` | Volume (base) |
| `k.q` | Volume (quote) |
| `k.n` | Number of trades |
| `k.x` | Is this kline closed? (final update) |

### bookTicker

```json
{
  "u": 400900217,
  "s": "BTCUSDT",
  "b": "97500.00",
  "B": "1.500",
  "a": "97500.50",
  "A": "2.300"
}
```

### depthUpdate

```json
{
  "e": "depthUpdate",
  "E": 1672515782136,
  "s": "BTCUSDT",
  "U": 157,
  "u": 160,
  "b": [["97500.00", "1.500"]],
  "a": [["97500.50", "0.000"]]
}
```

Quantity `"0.000"` = remove that price level.

## Maintaining a Local Order Book

### Initial Setup

1. Connect to `<symbol>@depth` or `<symbol>@depth@100ms` stream
2. **Buffer** all received events
3. Fetch snapshot: `GET /api/v3/depth?symbol=BTCUSDT&limit=5000` (max 5000 levels/side)
4. If snapshot's `lastUpdateId` < first buffered event's `U`, snapshot is stale — re-fetch
5. Discard buffered events where `u <= lastUpdateId`
6. First remaining event must satisfy: `U <= lastUpdateId+1` AND `u >= lastUpdateId+1`
7. Set local book = snapshot, then apply buffered events in order

### Applying Updates

For each incoming event:

1. **Skip** if event `u` < local book's update ID (already applied)
2. **Restart from scratch** if event `U` > local book's update ID + 1 (missed events)
3. For each price level in `b` (bids) and `a` (asks):
   - Quantity = 0 → **remove** that price level
   - Price not in book → **insert** new level
   - Otherwise → **update** quantity
4. Set local update ID = event's `u`

**Normal sequence**: each event's `U` = previous event's `u` + 1.

**Note**: Snapshot limited to 5000 levels per side. Levels beyond this are unknown unless they change.

## User Data Stream

### Create listenKey

```bash
curl -X POST "https://api.binance.com/api/v3/userDataStream" \
  -H "X-MBX-APIKEY: your_api_key"
```

Response: `{"listenKey": "pqia91ma19a5s61cv6a81va65sd..."}`

### Keepalive (every 30 min)

```bash
curl -X PUT "https://api.binance.com/api/v3/userDataStream?listenKey=pqia91ma..." \
  -H "X-MBX-APIKEY: your_api_key"
```

### Connect

```
wss://stream.binance.com:9443/ws/pqia91ma19a5s61cv6a81va65sd...
```

### Events

- `outboundAccountPosition` — balance changes
- `balanceUpdate` — deposits/withdrawals/transfers
- `executionReport` — order status updates
- `listStatus` — OCO order updates

## Reconnection Best Practices

1. Use exponential backoff: 1s → 2s → 4s → 8s → max 60s
2. Re-subscribe to all streams after reconnect
3. For order book: re-fetch snapshot after reconnect
4. For user data: check listenKey validity, renew if expired
5. Monitor ping/pong — if no ping for 30s, reconnect proactively
