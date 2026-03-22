# Binance Demo Mode Reference

Demo Mode is a simulation environment with virtual funds and near-real market data. Useful for testing trading strategies before going live.

## Demo vs Testnet vs Production

| Feature | Production | Demo Mode | Testnet |
|---------|-----------|-----------|---------|
| URL | `api.binance.com` | `demo-api.binance.com` | `testnet.binance.vision` |
| Market data | Real | Near-real (similar but not identical) | Independent (fake) |
| Balance | Real funds | Virtual, user-controlled reset | Virtual, monthly auto-reset |
| Feature parity | — | Identical to production | May include unreleased features |
| Rate limits | Production limits | Same as production | Generally aligned |
| API credentials | Production keys | Separate (from `demo.binance.com`) | Separate (from testnet site) |
| Endpoint support | All | `/api/*` | `/api/*` only (no `/sapi/*`) |

**Important**: Realistic market data ≠ real market data. Strategies that work in Demo Mode may not work in production.

## Endpoints

| Service | URL |
|---------|-----|
| REST API | `https://demo-api.binance.com/api` |
| WebSocket API | `wss://demo-ws-api.binance.com/ws-api/v3` |
| Market Streams | `wss://demo-stream.binance.com/ws` or `/stream` |
| SBE Streams | `wss://demo-stream-sbe.binance.com/ws` or `/stream` |

## Getting Started

1. Go to https://demo.binance.com
2. Log in with your Binance account
3. Generate Demo API credentials (separate from production/testnet)
4. Use the demo endpoints above instead of production URLs

## Code Example

Just swap the base URL:

```python
from binance.spot import Spot

# Production
# client = Spot(base_url="https://api.binance.com", api_key=..., api_secret=...)

# Demo Mode
client = Spot(
    base_url="https://demo-api.binance.com",
    api_key="your_demo_api_key",
    api_secret="your_demo_secret"
)

# Same API calls as production
print(client.account())
order = client.new_order(symbol="BTCUSDT", side="BUY", type="MARKET", quantity=0.001)
```

## Notes

- During maintenance, orders cannot be placed or canceled (check CHANGELOG for notices)
- Virtual balance can be reset via the Demo UI at any time
- All trading rules (filters, rate limits) are identical to production
