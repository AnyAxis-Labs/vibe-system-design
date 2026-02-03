# ADR 004: Redis for Cache, Sessions, and Pub/Sub

## Status
Accepted

## Context
The system needs:
1. Session storage for Web3-authenticated users
2. Rate limiting on bet placement API
3. Real-time price distribution to multiple WebSocket server instances (if ever scaled)
4. 7-day price history retention for bet resolution
5. Hot data caching to reduce SQLite load

## Decision
Use **Redis** for caching, session storage, rate limiting, and price pub/sub.

## Consequences

### Positive
- **Pub/Sub**: Clean decoupling between price ingestion and WebSocket broadcasting
- **Speed**: Sub-millisecond operations for session validation and rate limiting
- **Data structures**: Built-in sorted sets perfect for time-series price history
- **TTL support**: Automatic expiration of 7-day price data
- **Lightweight**: Fits in 8GB alongside SQLite and Go services

### Negative
- **Additional process**: One more thing to monitor and restart
- **Memory limit**: Price data must be pruned (TTL configured to 7 days)
- **Durability**: Redis AOF/RDB adds disk I/O; acceptable for ephemeral data

## Data Stored in Redis

| Key Pattern | Type | TTL | Purpose |
|-------------|------|-----|---------|
| `session:{wallet}` | String | 24h | Web3 auth session |
| `rate_limit:{wallet}` | String | 1m | Bet placement rate limiting |
| `price:btc:usd` | String | 1h | Latest price cache |
| `prices:btc:usd:ts` | Sorted Set | 7d | Time-series price history for resolution |
| `cache:grid:{params}` | String | 5m | Computed reward rates grid |
| `ws:clients` | Set | - | Connected client tracking (optional) |

## Pub/Sub Flow

```
Price Ingestion Service
    ↓ (PUBLISH prices:btc:usd $price)
Redis Pub/Sub
    ↓
WebSocket Server (SUBSCRIBE)
    ↓ (Broadcast)
Connected Clients
```

## Trade-off Analysis

| Alternative | Why Rejected |
|-------------|--------------|
| In-memory only | Lost on restart; no shared state between services |
| SQLite for sessions | Write amplification; slower than Redis for high-frequency reads |
| Kafka | Overkill for single VM; massive resource consumption |
| NATS | Good alternative, but Redis already needed for caching; minimize moving parts |

## Redis Configuration

```conf
# redis.conf for 8GB VM
maxmemory 512mb
maxmemory-policy allkeys-lru
appendonly yes
appendfsync everysec
```

## Decision Date
2026-02-03

## Author
System Architect
