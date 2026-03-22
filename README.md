English | [中文](README.zh-CN.md)

# Binance Skill

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

> Complete Binance Spot API skill for AI agents — REST API, WebSocket Streams, authentication, rate limits, and error handling.

## Features

- **REST API** — Full market data and trading endpoint reference with weights
- **WebSocket Streams** — All stream types with payload formats (aggTrade, kline, depth, bookTicker, miniTicker, user data)
- **Authentication** — HMAC-SHA256, RSA, Ed25519 signing with code examples
- **Rate Limits** — IP weights, order limits, ban escalation, and backoff strategies
- **Error Codes** — Common errors with recommended actions
- **Code Examples** — Python examples for signing and WebSocket streaming

## Quick Start

```bash
npx skills add https://github.com/atompilot/binance-skill
```

## Structure

```
skills/binance/
  SKILL.md                       # Main skill file (comprehensive reference)
  references/
    authentication.md            # Signing details (HMAC/RSA/Ed25519)
    websocket.md                 # WebSocket streams deep dive
    rate-limits.md               # Rate limit tables and best practices
```

## Why This Skill?

The official [binance-skills-hub](https://github.com/binance/binance-skills-hub) provides a solid spot trading skill, but focuses on REST API endpoints. This skill adds:

| Feature | Official Spot Skill | This Skill |
|---------|-------------------|------------|
| REST Market Data | ✅ | ✅ |
| REST Trading | ✅ | ✅ |
| Authentication | ✅ | ✅ + Python examples |
| WebSocket Streams | ❌ | ✅ Full coverage |
| Rate Limits | ❌ | ✅ Weight tables |
| Error Codes | ❌ | ✅ With actions |
| User Data Stream | ❌ | ✅ |

When the official skill adds these features, consider migrating back.

## Use Cases

- **Market data collection** — Build real-time tick/kline collectors via WebSocket
- **Quantitative trading** — Implement strategies with proper rate limit management
- **Data backfill** — Paginate historical klines efficiently
- **Portfolio monitoring** — Track account balances via User Data Stream

## License

[MIT](LICENSE) © atompilot
