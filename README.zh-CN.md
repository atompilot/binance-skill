[English](README.md) | 中文

# Binance Skill

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

> 面向 AI Agent 的完整 Binance Spot API skill — REST API、WebSocket Streams、认证、限频、错误处理。

## 特性

- **REST API** — 完整的行情和交易端点参考，附权重表
- **WebSocket Streams** — 所有流类型及数据格式（aggTrade、kline、depth、bookTicker、miniTicker、用户数据流）
- **认证** — HMAC-SHA256、RSA、Ed25519 签名，附代码示例
- **限频** — IP 权重、下单频率、封禁升级机制及退避策略
- **错误码** — 常见错误及建议处理方式
- **代码示例** — Python 签名和 WebSocket 示例

## 快速安装

```bash
npx skills add https://github.com/atompilot/binance-skill
```

## 结构

```
skills/binance/
  SKILL.md                       # 主 skill 文件（综合参考）
  references/
    authentication.md            # 签名细节（HMAC/RSA/Ed25519）
    websocket.md                 # WebSocket 流详解
    rate-limits.md               # 限频表和最佳实践
```

## 为什么做这个 Skill？

Binance 官方 [binance-skills-hub](https://github.com/binance/binance-skills-hub) 提供了 spot 交易 skill，但主要覆盖 REST API 端点。本 skill 补充了：

| 功能 | 官方 Spot Skill | 本 Skill |
|------|----------------|----------|
| REST 行情 | ✅ | ✅ |
| REST 交易 | ✅ | ✅ |
| 认证 | ✅ | ✅ + Python 示例 |
| WebSocket Streams | ❌ | ✅ 完整覆盖 |
| 限频 | ❌ | ✅ 权重表 |
| 错误码 | ❌ | ✅ 附处理建议 |
| 用户数据流 | ❌ | ✅ |

待官方 skill 补全这些功能后，可考虑迁移回官方版本。

## 适用场景

- **行情数据采集** — 通过 WebSocket 构建实时 tick/K 线采集器
- **量化交易** — 在合理限频管理下实现交易策略
- **历史数据回填** — 高效分页拉取历史 K 线
- **组合监控** — 通过用户数据流跟踪账户变动

## 许可证

[MIT](LICENSE) © atompilot
