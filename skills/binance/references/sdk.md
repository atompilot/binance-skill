# Binance Official SDKs & Resources

## Official Connectors

Binance maintains official API connectors in multiple languages:

| Language | Repository | Package |
|----------|-----------|---------|
| Python | [binance/binance-connector-python](https://github.com/binance/binance-connector-python) | `pip install binance-connector` |
| Node.js | [binance/binance-connector-node](https://github.com/binance/binance-connector-node) | `npm install @binance/connector` |
| Java | [binance/binance-connector-java](https://github.com/binance/binance-connector-java) | Maven |
| Go | [binance/binance-connector-go](https://github.com/binance/binance-connector-go) | `go get` |
| Rust | [binance/binance-spot-connector-rust](https://github.com/binance/binance-spot-connector-rust) | Cargo |
| PHP | [binance/binance-connector-php](https://github.com/binance/binance-connector-php) | Composer |
| Ruby | [binance/binance-connector-ruby](https://github.com/binance/binance-connector-ruby) | Gem |
| DotNET | [binance/binance-connector-dotnet](https://github.com/binance/binance-connector-dotnet) | NuGet |
| TypeScript | [binance/binance-connector-typescript](https://github.com/binance/binance-connector-typescript) | `npm install @binance/connector-typescript` |

## Development & Testing Resources

| Resource | URL | Purpose |
|----------|-----|---------|
| Spot Testnet | https://testnet.binance.vision | Practice trading (only `/api/*`) |
| Postman Collection | [binance/binance-api-postman](https://github.com/binance/binance-api-postman) | Quick API testing |
| OpenAPI Spec | [binance/binance-api-swagger](https://github.com/binance/binance-api-swagger) | Auto-generate clients |
| API Announcements | Telegram: @binance_api_announcements | Breaking changes, downtime |

## Testnet Limitations

- Only supports `/api/*` endpoints (spot trading, market data)
- Does **NOT** support `/sapi/*` endpoints (wallet, sub-account, savings, etc.)
- Testnet credentials are completely separate from mainnet
- Create testnet API keys at https://testnet.binance.vision

## Python Connector Quick Example

```python
from binance.spot import Spot

# Public endpoints (no auth)
client = Spot()
print(client.klines("BTCUSDT", "1m", limit=5))
print(client.depth("BTCUSDT", limit=10))
print(client.agg_trades("BTCUSDT", limit=10))

# Authenticated endpoints
client = Spot(api_key="your_key", api_secret="your_secret")
print(client.account())
order = client.new_order(symbol="BTCUSDT", side="BUY", type="MARKET", quantity=0.001)

# Testnet
client = Spot(base_url="https://testnet.binance.vision", api_key="test_key", api_secret="test_secret")

# WebSocket
from binance.websocket.spot.websocket_stream import SpotWebsocketStreamClient

def on_message(_, msg):
    print(msg)

ws = SpotWebsocketStreamClient(on_message=on_message)
ws.agg_trade(symbol="btcusdt")
```

## Community Libraries (Popular)

| Library | Language | Notes |
|---------|----------|-------|
| [python-binance](https://github.com/sammchardy/python-binance) | Python | Most starred, async support |
| [node-binance-api](https://github.com/jaggedsoft/node-binance-api) | Node.js | Widely used |
| [binance-rs](https://github.com/wisespace-io/binance-rs) | Rust | Community maintained |
