# Binance User Data Stream Reference

Real-time push of your private account data: balance changes, order updates, trade executions.

## listenKey Management

A `listenKey` is required to connect. It acts as a temporary auth token.

| Action | Method | Endpoint | Weight | Auth |
|--------|--------|----------|--------|------|
| Create | POST | `/api/v3/userDataStream` | 2 | API Key header |
| Keepalive | PUT | `/api/v3/userDataStream?listenKey=...` | 2 | API Key header |
| Close | DELETE | `/api/v3/userDataStream?listenKey=...` | 2 | API Key header |

### Create

```bash
curl -X POST "https://api.binance.com/api/v3/userDataStream" \
  -H "X-MBX-APIKEY: your_api_key"
```

Response: `{"listenKey": "pqia91ma19a5s61cv6a81va65sd..."}`

### Keepalive

Must be sent **every 30 minutes**. listenKey expires after 60 minutes without keepalive.

```bash
curl -X PUT "https://api.binance.com/api/v3/userDataStream?listenKey=pqia91ma..." \
  -H "X-MBX-APIKEY: your_api_key"
```

### Connect

```
wss://stream.binance.com:9443/ws/pqia91ma19a5s61cv6a81va65sd...
```

## Events

| Event | Type | Trigger |
|-------|------|---------|
| Account Update | `outboundAccountPosition` | Any balance change |
| Balance Update | `balanceUpdate` | Deposit, withdrawal, transfer |
| Order Update | `executionReport` | Order placed, filled, canceled, expired, rejected |
| List Status | `listStatus` | OCO/OTO order list updates |
| Stream Terminated | `eventStreamTerminated` | listenKey expired, logout, or unsubscribe |
| External Lock | `externalLockUpdate` | Balance locked/unlocked by external system (margin, etc.) |

## Account Update — `outboundAccountPosition`

Pushed whenever account balances change (trade, deposit, withdrawal, etc.).

```json
{
  "e": "outboundAccountPosition",
  "E": 1564034571105,
  "u": 1564034571073,
  "B": [
    {"a": "ETH", "f": "10000.000000", "l": "0.000000"},
    {"a": "BTC", "f": "1.000000", "l": "0.500000"}
  ]
}
```

| Field | Description |
|-------|-------------|
| `E` | Event time (ms) |
| `u` | Last account update time (ms) |
| `B` | Balances array |
| `B[].a` | Asset |
| `B[].f` | Free balance |
| `B[].l` | Locked balance |

## Balance Update — `balanceUpdate`

Pushed for deposits, withdrawals, or inter-account transfers.

```json
{
  "e": "balanceUpdate",
  "E": 1573200697110,
  "a": "BTC",
  "d": "100.00000000",
  "T": 1573200697068
}
```

| Field | Description |
|-------|-------------|
| `a` | Asset |
| `d` | Balance delta (positive = deposit, negative = withdrawal) |
| `T` | Clear time (ms) |

## Order Update — `executionReport`

The primary event for tracking order lifecycle. Pushed on every order state change.

### Full Payload

```json
{
  "e": "executionReport",
  "E": 1499405658658,
  "s": "ETHBTC",
  "c": "mUvoqJxFIILMdfAW5iGSOW",
  "S": "BUY",
  "o": "LIMIT",
  "f": "GTC",
  "q": "1.00000000",
  "p": "0.10264410",
  "P": "0.00000000",
  "F": "0.00000000",
  "g": -1,
  "C": "",
  "x": "NEW",
  "X": "NEW",
  "r": "NONE",
  "i": 4293153,
  "l": "0.00000000",
  "z": "0.00000000",
  "L": "0.00000000",
  "n": "0",
  "N": null,
  "T": 1499405658657,
  "t": -1,
  "v": 3,
  "I": 8641984,
  "w": true,
  "m": false,
  "M": false,
  "O": 1499405658657,
  "Z": "0.00000000",
  "Y": "0.00000000",
  "Q": "0.00000000",
  "W": 1499405658657,
  "V": "NONE"
}
```

### Field Reference

| Field | Description |
|-------|-------------|
| `e` | Event type (`executionReport`) |
| `E` | Event time (ms) |
| `s` | Symbol |
| `c` | Client order ID |
| `S` | Side: `BUY` / `SELL` |
| `o` | Order type: `LIMIT`, `MARKET`, `STOP_LOSS`, etc. |
| `f` | Time in force: `GTC`, `IOC`, `FOK` |
| `q` | Order quantity |
| `p` | Order price |
| `P` | Stop price |
| `F` | Iceberg quantity |
| `g` | Order list ID (-1 if not part of a list) |
| `C` | Original client order ID (for cancel events) |
| `x` | **Execution type** (see enum below) |
| `X` | **Order status** (see enum below) |
| `r` | Order reject reason (see enum below) |
| `i` | Order ID |
| `l` | Last executed quantity (this fill) |
| `z` | Cumulative filled quantity (all fills) |
| `L` | Last executed price (this fill) |
| `n` | Commission amount (this fill) |
| `N` | Commission asset (e.g. `BNB`) |
| `T` | Transaction time (ms) |
| `t` | Trade ID (-1 if not a trade) |
| `v` | Prevented match ID (only on STP expiry) |
| `I` | Execution ID |
| `w` | Is the order on the book? |
| `m` | Is this trade the maker side? |
| `M` | Ignore |
| `O` | Order creation time (ms) |
| `Z` | Cumulative quote asset transacted quantity |
| `Y` | Last quote asset transacted quantity (`lastPrice * lastQty`) |
| `Q` | Quote order quantity (for `quoteOrderQty` orders) |
| `W` | Working time (ms, only if order is on book) |
| `V` | Self-trade prevention mode |

### Conditional Fields

These appear only under specific conditions:

| Field | Description | When |
|-------|-------------|------|
| `d` | Trailing delta | Trailing stop orders |
| `D` | Trailing time (ms) | Trailing stop orders |
| `j` | Strategy ID | If provided at order placement |
| `J` | Strategy type | If provided at order placement |
| `eR` | Expiry reason | When order expires |

### Calculating Average Price

```
average_price = Z / z
```

Where `Z` = cumulative quote qty, `z` = cumulative filled qty.

## Enums

### Execution Type (`x`)

| Value | Description |
|-------|-------------|
| `NEW` | Order accepted, placed on book |
| `CANCELED` | Order canceled (by user or engine) |
| `REPLACED` | Order replaced (cancel-replace) |
| `REJECTED` | Order rejected |
| `TRADE` | Order partially or fully filled |
| `EXPIRED` | Order expired (TTL, FOK/IOC unfilled, etc.) |

### Order Status (`X`)

| Value | Description |
|-------|-------------|
| `NEW` | Active, on the book |
| `PARTIALLY_FILLED` | Partially filled, still active |
| `FILLED` | Fully filled, completed |
| `CANCELED` | Canceled by user |
| `PENDING_CANCEL` | Cancel in progress (rarely seen) |
| `REJECTED` | Rejected (failed filters, insufficient balance) |
| `EXPIRED` | Expired (timeInForce, trailing stop, etc.) |
| `EXPIRED_IN_MATCH` | Expired during matching (e.g. STP) |

### Reject Reason (`r`)

| Value | Description |
|-------|-------------|
| `NONE` | Not rejected |
| `UNKNOWN_INSTRUMENT` | Unknown symbol |
| `MARKET_NOT_ALLOWED` | Market orders not allowed |
| `PRICE_QTY_EXCEED_HARD_LIMITS` | Price or quantity exceeds limits |
| `UNKNOWN_ORDER` | Order not found (for cancel) |
| `DUPLICATE_ORDER` | Duplicate client order ID |
| `UNKNOWN_ACCOUNT` | Account not found |
| `INSUFFICIENT_BALANCE` | Not enough balance |
| `ACCOUNT_NOT_ACTIVE` | Account disabled |
| `ACCOUNT_CANNOT_SETTLE` | Account cannot settle |

### Expiry Reason (`eR`)

| Value | Description |
|-------|-------------|
| `INSUFFICIENT_LIQUIDITY` | IOC/FOK could not fill enough |
| `EXCHANGE_CANCEL` | Canceled by exchange |
| `CONTINGENT_REJECT` | Contingent order rejected (OCO leg) |
| `TRIGGER_PRICE_IMMEDIATELY` | Stop order would trigger immediately |
| `PRICE_RELATIONSHIP` | OCO price relationship invalid |

## List Status — `listStatus`

Pushed for OCO/OTO order list updates.

```json
{
  "e": "listStatus",
  "E": 1564035303637,
  "s": "ETHBTC",
  "g": 2,
  "c": "OCO",
  "l": "EXEC_STARTED",
  "L": "EXECUTING",
  "r": "NONE",
  "C": "F4QN4G8DlFATFlIUQ0cjdD",
  "T": 1564035303625,
  "O": [
    {"s": "ETHBTC", "i": 17, "c": "AJYsMjErWJesZvqlJCTUgL"},
    {"s": "ETHBTC", "i": 18, "c": "bfYPSQdLoqAJeNrOaN9KQD"}
  ]
}
```

| Field | Description |
|-------|-------------|
| `g` | Order list ID |
| `c` | Contingency type (e.g. `OCO`) |
| `l` | List status type |
| `L` | List order status |
| `r` | List reject reason |
| `C` | List client order ID |
| `O` | Array of orders in the list |

## Stream Terminated — `eventStreamTerminated`

Pushed when the stream is about to close.

```json
{
  "e": "eventStreamTerminated",
  "E": 1564035303637
}
```

Reasons: listenKey expired, explicit logout, or unsubscribe.

## Python Example

```python
import asyncio, json, aiohttp, websockets

API_KEY = "your_api_key"
BASE = "https://api.binance.com"

async def get_listen_key():
    async with aiohttp.ClientSession() as s:
        resp = await s.post(f"{BASE}/api/v3/userDataStream",
                            headers={"X-MBX-APIKEY": API_KEY})
        return (await resp.json())["listenKey"]

async def keepalive(listen_key):
    """Send keepalive every 30 min."""
    async with aiohttp.ClientSession() as s:
        while True:
            await asyncio.sleep(30 * 60)
            await s.put(f"{BASE}/api/v3/userDataStream?listenKey={listen_key}",
                        headers={"X-MBX-APIKEY": API_KEY})

async def stream(listen_key):
    url = f"wss://stream.binance.com:9443/ws/{listen_key}"
    async with websockets.connect(url) as ws:
        async for msg in ws:
            event = json.loads(msg)
            t = event["e"]
            if t == "executionReport":
                print(f"Order {event['i']}: {event['x']} → {event['X']}, "
                      f"filled={event['z']}/{event['q']}, price={event['L']}")
            elif t == "outboundAccountPosition":
                for b in event["B"]:
                    print(f"  {b['a']}: free={b['f']}, locked={b['l']}")
            elif t == "balanceUpdate":
                print(f"Balance: {event['a']} delta={event['d']}")
            elif t == "eventStreamTerminated":
                print("Stream terminated, reconnecting...")
                break

async def main():
    lk = await get_listen_key()
    asyncio.create_task(keepalive(lk))
    await stream(lk)

asyncio.run(main())
```
