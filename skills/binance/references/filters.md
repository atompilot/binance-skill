# Binance Trading Filters Reference

Filters define the trading rules for each symbol. Orders that violate any filter are rejected. Query filters via `GET /api/v3/exchangeInfo?symbol=BTCUSDT` (weight: 20).

## Symbol Filters

### PRICE_FILTER

Controls price precision and range.

```json
{"filterType": "PRICE_FILTER", "minPrice": "0.01", "maxPrice": "1000000.00", "tickSize": "0.01"}
```

Rules:
- `price` >= `minPrice`
- `price` <= `maxPrice`
- `(price - minPrice) % tickSize == 0`
- Disabled if value is `0`

### LOT_SIZE

Controls quantity (lot) precision and range for LIMIT orders.

```json
{"filterType": "LOT_SIZE", "minQty": "0.00001", "maxQty": "9000.00000", "stepSize": "0.00001"}
```

Rules:
- `quantity` >= `minQty`
- `quantity` <= `maxQty`
- `(quantity - minQty) % stepSize == 0`

### MARKET_LOT_SIZE

Same as LOT_SIZE but for MARKET orders. May have different (usually wider) limits.

```json
{"filterType": "MARKET_LOT_SIZE", "minQty": "0.00001", "maxQty": "100000.00", "stepSize": "0.00001"}
```

### NOTIONAL

Controls the min/max notional value (`price * quantity`).

```json
{
  "filterType": "NOTIONAL",
  "minNotional": "10.00000000",
  "applyMinToMarket": true,
  "maxNotional": "9000000.00000000",
  "applyMaxToMarket": false,
  "avgPriceMins": 5
}
```

Rules:
- For LIMIT: `price * quantity` >= `minNotional`
- For MARKET (if `applyMinToMarket`): uses average price over `avgPriceMins` minutes (0 = last price)
- Most symbols require minimum ~10 USDT notional

### MIN_NOTIONAL (legacy)

Older version of NOTIONAL, only enforces minimum.

```json
{"filterType": "MIN_NOTIONAL", "minNotional": "0.00100000", "applyToMarket": true, "avgPriceMins": 5}
```

### PERCENT_PRICE_BY_SIDE

Limits how far the price can deviate from the average price, separately for BUY and SELL.

```json
{
  "filterType": "PERCENT_PRICE_BY_SIDE",
  "bidMultiplierUp": "5",
  "bidMultiplierDown": "0.2",
  "askMultiplierUp": "5",
  "askMultiplierDown": "0.2",
  "avgPriceMins": 5
}
```

Rules (BUY side):
- `price` <= `avgPrice * bidMultiplierUp`
- `price` >= `avgPrice * bidMultiplierDown`

### ICEBERG_PARTS

Limits the number of iceberg parts.

```json
{"filterType": "ICEBERG_PARTS", "limit": 10}
```

### MAX_NUM_ORDERS

Maximum number of open orders per symbol.

```json
{"filterType": "MAX_NUM_ORDERS", "maxNumOrders": 200}
```

### MAX_NUM_ALGO_ORDERS

Maximum number of open algo orders (STOP_LOSS, TAKE_PROFIT, etc.) per symbol.

```json
{"filterType": "MAX_NUM_ALGO_ORDERS", "maxNumAlgoOrders": 5}
```

### TRAILING_DELTA

Limits the min/max `trailingDelta` for trailing stop orders. Values in BIPS (1 BIP = 0.01%).

```json
{
  "filterType": "TRAILING_DELTA",
  "minTrailingAboveDelta": 10,
  "maxTrailingAboveDelta": 2000,
  "minTrailingBelowDelta": 10,
  "maxTrailingBelowDelta": 2000
}
```

## Practical Guide

### Before Placing an Order

1. Fetch filters: `GET /api/v3/exchangeInfo?symbol=BTCUSDT`
2. Validate price: round to `tickSize`, check within `[minPrice, maxPrice]`
3. Validate quantity: round to `stepSize`, check within `[minQty, maxQty]`
4. Validate notional: `price * quantity >= minNotional`
5. Check open order count: < `maxNumOrders`

### Python Helper

```python
from decimal import Decimal, ROUND_DOWN

def round_step(value, step):
    """Round value down to nearest step size."""
    step = Decimal(str(step))
    return float(Decimal(str(value)).quantize(step, rounding=ROUND_DOWN))

# Example: BTCUSDT with tickSize=0.01, stepSize=0.00001
price = round_step(97500.123, 0.01)      # 97500.12
quantity = round_step(0.015678, 0.00001)  # 0.01567
notional = price * quantity               # must be >= minNotional
```

### Common Rejection Reasons

| Error | Likely Filter | Fix |
|-------|--------------|-----|
| `-2010 Filter failure: LOT_SIZE` | quantity too small/large or wrong step | Round to stepSize |
| `-2010 Filter failure: PRICE_FILTER` | price wrong precision | Round to tickSize |
| `-2010 Filter failure: NOTIONAL` | order value too small | Increase quantity or price |
| `-2010 Filter failure: PERCENT_PRICE` | price too far from market | Use closer price |
| `-2010 Filter failure: MAX_NUM_ORDERS` | too many open orders | Cancel some orders first |
