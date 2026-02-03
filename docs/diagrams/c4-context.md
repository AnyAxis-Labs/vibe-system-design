# C4 Context Diagram (L1) — BTC Tap Trading System

## System Context

```mermaid
C4Context
    title System Context for BTC Tap Trading Application

    Person(trader, "Trader", "Web3 wallet user placing directional bets on BTC price movements")
    Person(admin, "Admin/Operator", "System monitor resolving disputes and monitoring health")
    
    System(tapTrading, "BTC Tap Trading System", "Allows users to place real-time directional bets on BTC price movements via a grid-based interface")
    
    System_Ext(priceFeed, "Crypto Price Feed", "External WebSocket API (Binance/Coinbase) providing real-time BTC/USD price updates")
    
    Rel(trader, tapTrading, "Places bets via Web UI", "HTTPS/WebSocket")
    Rel(trader, tapTrading, "Authenticates with", "Web3 Wallet Signature")
    Rel(admin, tapTrading, "Monitors metrics and health", "HTTPS")
    Rel(tapTrading, priceFeed, "Streams real-time BTC prices from", "WebSocket")
    
    UpdateElementStyle(trader, $fontColor="#fff", $bgColor="#08427b")
    UpdateElementStyle(admin, $fontColor="#fff", $bgColor="#08427b")
    UpdateElementStyle(tapTrading, $bgColor="#1168bd", $fontColor="#fff")
    UpdateElementStyle(priceFeed, $bgColor="#999999", $fontColor="#fff")
```

### Diagram Description

| Element | Type | Description |
|---------|------|-------------|
| **Trader** | Person | End-user with Web3 wallet placing directional bets on BTC price movements via grid interface |
| **Admin/Operator** | Person | Internal team member monitoring system health, resolving disputes |
| **BTC Tap Trading System** | System | The complete betting application including API, WebSocket, background workers, and data stores |
| **Crypto Price Feed** | External System | Third-party WebSocket API providing real-time BTC/USD price updates |

### Relationships

| Source | Target | Description | Protocol |
|--------|--------|-------------|----------|
| Trader | BTC Tap Trading System | Places bets via Web UI | HTTPS / WebSocket |
| Trader | BTC Tap Trading System | Authenticates with Web3 wallet | Web3 Signature |
| Admin | BTC Tap Trading System | Monitors metrics and system health | HTTPS |
| BTC Tap Trading System | Crypto Price Feed | Subscribes to real-time price stream | WebSocket |
