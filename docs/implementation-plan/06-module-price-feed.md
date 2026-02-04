# Module 4: Price Feed (Vertical)

**Goal:** Implement real-time price ingestion, circuit breaker, and WebSocket distribution.

**Duration:** Days 7-8  
**Dependencies:** Module 1 (Core)  
**Priority:** HIGH (required for fair betting)

---

## Module Scope

This module implements:
- External price feed ingestion
- Redis pub/sub price distribution
- Circuit breaker for stale prices (ADR-008)
- Price health monitoring
- WebSocket price streaming

**Source ADRs:** ADR-004 (Redis Pub/Sub), ADR-008 (Circuit Breaker)  
**Source C4:** Price Ingestion, Price Health Monitor, WebSocket Server

---

## Task 1: Price Ingestion

### Task 1.1: Implement Price Feed Client
**Owner:** Backend Engineer  
**Estimated:** 4 hours  
**Source ADR:** PRD (Real-time price streaming)  
**Source C4:** Price Ingestion container

**Description:**
Implement WebSocket client for external price feed (Binance/Coinbase).

**DoD:**
- [ ] Connect to Binance/Coinbase WebSocket
- [ ] Handle connection drops with auto-reconnect
- [ ] Parse price messages
- [ ] Publish to Redis pub/sub
- [ ] Store in Redis sorted set for history
- [ ] Connection health logging

**Implementation:**
```go
// cmd/price/ingestion.go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "net/url"
    "time"

    "github.com/gorilla/websocket"
    "github.com/tap-trading/backend/pkg/redis"
)

const (
    BinanceWSURL = "wss://stream.binance.com:9443/ws/btcusdt@trade"
    ReconnectDelay = 5 * time.Second
)

type PriceFeed struct {
    redis    *redis.Client
    health   *HealthMonitor
    shutdown chan struct{}
}

type TradeMessage struct {
    Symbol    string `json:"s"`
    Price     string `json:"p"`
    Time      int64  `json:"T"`
}

func (f *PriceFeed) Start() {
    for {
        select {
        case <-f.shutdown:
            return
        default:
        }
        
        err := f.connectAndStream()
        if err != nil {
            log.Error().Err(err).Msg("price feed error, reconnecting...")
        }
        
        select {
        case <-f.shutdown:
            return
        case <-time.After(ReconnectDelay):
        }
    }
}

func (f *PriceFeed) connectAndStream() error {
    u, _ := url.Parse(BinanceWSURL)
    
    conn, _, err := websocket.DefaultDialer.Dial(u.String(), nil)
    if err != nil {
        return fmt.Errorf("dial: %w", err)
    }
    defer conn.Close()
    
    log.Info().Msg("price feed connected")
    
    for {
        select {
        case <-f.shutdown:
            return nil
        default:
        }
        
        conn.SetReadDeadline(time.Now().Add(30 * time.Second))
        
        _, message, err := conn.ReadMessage()
        if err != nil {
            return fmt.Errorf("read: %w", err)
        }
        
        var trade TradeMessage
        if err := json.Unmarshal(message, &trade); err != nil {
            log.Error().Err(err).RawJSON("message", message).Msg("unmarshal failed")
            continue
        }
        
        // Parse and publish
        price := parsePrice(trade.Price)
        timestamp := time.UnixMilli(trade.Time)
        
        if err := f.publishPrice(price, timestamp); err != nil {
            log.Error().Err(err).Msg("publish failed")
        }
    }
}

func (f *PriceFeed) publishPrice(price float64, timestamp time.Time) error {
    ctx := context.Background()
    priceStr := fmt.Sprintf("%.2f", price)
    
    pipe := f.redis.Pipeline()
    
    // Publish to pub/sub for real-time distribution
    pipe.Publish(ctx, "prices:btc:usd", priceStr)
    
    // Store latest price
    pipe.Set(ctx, "price:btc:usd", priceStr, 1*time.Hour)
    
    // Add to sorted set for history (7-day TTL via Redis config)
    pipe.ZAdd(ctx, "prices:btc:usd:ts", redis.Z{
        Score:  float64(timestamp.Unix()),
        Member: priceStr,
    })
    
    // Trim old entries (keep 7 days worth)
    weekAgo := time.Now().Add(-7 * 24 * time.Hour).Unix()
    pipe.ZRemRangeByScore(ctx, "prices:btc:usd:ts", "0", fmt.Sprintf("%d", weekAgo))
    
    _, err := pipe.Exec(ctx)
    
    // Update health monitor
    f.health.RecordPrice(timestamp)
    
    return err
}
```

---

## Task 2: Circuit Breaker (ADR-008)

### Task 2.1: Implement Price Health Monitor
**Owner:** Backend Engineer  
**Estimated:** 3 hours  
**Source ADR:** ADR-008 (Stale Price Circuit Breaker)  
**Source C4:** Price Health Monitor

**Description:**
Implement health monitoring that tracks price freshness.

**DoD:**
- [ ] Subscribe to price updates
- [ ] Track last update timestamp
- [ ] Evaluate health status (Healthy/Degraded/Critical)
- [ ] Thresholds: 3s = Degraded, 10s = Critical

**Implementation:**
```go
// pkg/circuit/health.go
package circuit

import (
    "context"
    "sync/atomic"
    "time"
    
    "github.com/tap-trading/backend/pkg/redis"
)

const (
    StaleThreshold    = 3 * time.Second
    CriticalThreshold = 10 * time.Second
)

type HealthStatus int32

const (
    Healthy HealthStatus = iota
    Degraded
    Critical
)

type HealthMonitor struct {
    redis       *redis.Client
    lastUpdated atomic.Int64 // Unix nano
    status      atomic.Int32 // HealthStatus
}

func NewHealthMonitor(redis *redis.Client) *HealthMonitor {
    return &HealthMonitor{redis: redis}
}

func (m *HealthMonitor) Start(ctx context.Context) {
    // Subscribe to price updates
    pubsub := m.redis.Subscribe(ctx, "prices:btc:usd")
    
    go func() {
        for msg := range pubsub.Channel() {
            m.RecordPrice(time.Now())
        }
    }()
    
    // Periodic health evaluation
    ticker := time.NewTicker(1 * time.Second)
    go func() {
        defer ticker.Stop()
        for {
            select {
            case <-ctx.Done():
                return
            case <-ticker.C:
                m.evaluate()
            }
        }
    }()
}

func (m *HealthMonitor) RecordPrice(t time.Time) {
    m.lastUpdated.Store(t.UnixNano())
    m.status.Store(int32(Healthy))
}

func (m *HealthMonitor) evaluate() {
    last := time.Unix(0, m.lastUpdated.Load())
    age := time.Since(last)
    
    var status HealthStatus
    switch {
    case age > CriticalThreshold:
        status = Critical
        log.Error().
            Dur("stale_for", age).
            Str("status", "CRITICAL").
            Msg("PRICE_FEED_CRITICAL")
    case age > StaleThreshold:
        status = Degraded
        log.Warn().
            Dur("stale_for", age).
            Str("status", "DEGRADED").
            Msg("PRICE_FEED_DEGRADED")
    default:
        status = Healthy
    }
    
    m.status.Store(int32(status))
}

func (m *HealthMonitor) GetStatus() HealthStatus {
    return HealthStatus(m.status.Load())
}

func (m *HealthMonitor) IsHealthy() bool {
    return m.GetStatus() != Critical
}

func (m *HealthMonitor) GetLastUpdated() time.Time {
    return time.Unix(0, m.lastUpdated.Load())
}
```

---

### Task 2.2: Implement Circuit Breaker Middleware
**Owner:** Backend Engineer  
**Estimated:** 3 hours  
**Source ADR:** ADR-008  
**Source C4:** API Server

**Description:**
Implement middleware that blocks bet placement when price feed is critical.

**DoD:**
- [ ] Middleware checks health status
- [ ] Returns 503 when Critical
- [ ] Includes Retry-After header
- [ ] Applies only to bet placement routes
- [ ] Recovery with 30s hysteresis (Hardening v1.2)

**Implementation:**
```go
// pkg/circuit/middleware.go
package circuit

import (
    "net/http"
    "time"
    
    "github.com/labstack/echo/v4"
)

const RecoveryHysteresis = 30 * time.Second

type CircuitBreaker struct {
    health      *HealthMonitor
    recoveredAt atomic.Int64 // Unix nano
}

func NewCircuitBreaker(health *HealthMonitor) *CircuitBreaker {
    return &CircuitBreaker{health: health}
}

func (cb *CircuitBreaker) Middleware() echo.MiddlewareFunc {
    return func(next echo.HandlerFunc) echo.HandlerFunc {
        return func(c echo.Context) error {
            status := cb.health.GetStatus()
            
            if status == Critical {
                return c.JSON(http.StatusServiceUnavailable, map[string]interface{}{
                    "error":        "PRICE_FEED_STALE",
                    "message":      "Betting temporarily suspended due to stale price data",
                    "retry_after":  10,
                })
            }
            
            // Hysteresis for recovery (Hardening v1.2)
            if status == Healthy {
                lastHealthy := cb.health.GetLastUpdated()
                recoveredAt := time.Unix(0, cb.recoveredAt.Load())
                
                if time.Since(lastHealthy) < RecoveryHysteresis && time.Since(recoveredAt) < RecoveryHysteresis {
                    // Still in recovery window
                    log.Info().Msg("circuit breaker: in recovery hysteresis")
                }
            }
            
            return next(c)
        }
    }
}
```

---

## Task 3: WebSocket Price Streaming

### Task 3.1: Implement Price Subscriber
**Owner:** Backend Engineer  
**Estimated:** 3 hours  
**Source ADR:** ADR-004 (Redis Pub/Sub)  
**Source C4:** WebSocket Server

**Description:**
Subscribe to Redis price channel and broadcast to WebSocket clients.

**DoD:**
- [ ] Subscribe to `prices:btc:usd` channel
- [ ] Broadcast to all connected clients
- [ ] JSON message format
- [ ] Handle client connect/disconnect

**Implementation:**
```go
// cmd/ws/price.go
package main

import (
    "context"
    "encoding/json"
    
    "github.com/tap-trading/backend/pkg/redis"
)

type PriceUpdate struct {
    Symbol    string  `json:"symbol"`
    Price     float64 `json:"price"`
    Timestamp int64   `json:"timestamp"`
}

func (s *WebSocketServer) startPriceSubscriber() {
    ctx := context.Background()
    pubsub := s.redis.Subscribe(ctx, "prices:btc:usd")
    defer pubsub.Close()
    
    for msg := range pubsub.Channel() {
        price, _ := strconv.ParseFloat(msg.Payload, 64)
        
        update := PriceUpdate{
            Symbol:    "BTC/USD",
            Price:     price,
            Timestamp: time.Now().Unix(),
        }
        
        data, _ := json.Marshal(update)
        
        // Broadcast to all clients
        s.broadcast(data)
    }
}

func (s *WebSocketServer) broadcast(data []byte) {
    s.clientsMutex.RLock()
    defer s.clientsMutex.RUnlock()
    
    for client := range s.clients {
        select {
        case client.send <- data:
        default:
            // Client slow, drop message
            close(client.send)
            delete(s.clients, client)
        }
    }
}
```

---

### Task 3.2: System Status Broadcasting
**Owner:** Backend Engineer  
**Estimated:** 2 hours  
**Source ADR:** ADR-008  
**Source C4:** WebSocket Server

**Description:**
Broadcast system status (price health) to clients for UI feedback.

**DoD:**
- [ ] Broadcast `system:status` message
- [ ] Include price health status
- [ ] Frontend disables grid when Critical

**Implementation:**
```go
type SystemStatus struct {
    PriceHealth string `json:"price_health"` // "healthy", "degraded", "critical"
    Timestamp   int64  `json:"timestamp"`
}

func (s *WebSocketServer) broadcastStatus() {
    ticker := time.NewTicker(5 * time.Second)
    defer ticker.Stop()
    
    for range ticker.C {
        status := SystemStatus{
            PriceHealth: s.health.GetStatus().String(),
            Timestamp:   time.Now().Unix(),
        }
        
        data, _ := json.Marshal(map[string]interface{}{
            "type":   "system:status",
            "data":   status,
        })
        
        s.broadcast(data)
    }
}
```

---

## Task 4: Price History Query

### Task 4.1: Implement Price History Service
**Owner:** Backend Engineer  
**Estimated:** 2 hours  
**Source ADR:** PRD (Bet resolution needs price history)  
**Source C4:** Redis (price history)

**Description:**
Service to query price history for bet resolution.

**DoD:**
- [ ] Query prices by time range
- [ ] Check if price hit target range
- [ ] Handle gaps in price data

**Implementation:**
```go
// pkg/price/history.go
package price

import (
    "context"
    "strconv"
    "time"
    
    "github.com/tap-trading/backend/pkg/redis"
)

type HistoryService struct {
    redis *redis.Client
}

func (h *HistoryService) GetPricesInRange(ctx context.Context, from, to time.Time) ([]PricePoint, error) {
    scores, err := h.redis.ZRangeByScoreWithScores(ctx, "prices:btc:usd:ts", &redis.ZRangeBy{
        Min: fmt.Sprintf("%d", from.Unix()),
        Max: fmt.Sprintf("%d", to.Unix()),
    }).Result()
    
    if err != nil {
        return nil, err
    }
    
    points := make([]PricePoint, len(scores))
    for i, z := range scores {
        price, _ := strconv.ParseFloat(z.Member.(string), 64)
        points[i] = PricePoint{
            Price:     price,
            Timestamp: time.Unix(int64(z.Score), 0),
        }
    }
    
    return points, nil
}

func (h *HistoryService) DidPriceHitRange(ctx context.Context, from, to time.Time, minPrice, maxPrice float64) (bool, error) {
    prices, err := h.GetPricesInRange(ctx, from, to)
    if err != nil {
        return false, err
    }
    
    for _, p := range prices {
        if p.Price >= minPrice && p.Price <= maxPrice {
            return true, nil
        }
    }
    
    return false, nil
}
```

---

## Task 5: Testing

### Task 5.1: Circuit Breaker Tests
**Owner:** Backend Engineer  
**Estimated:** 3 hours  
**Source ADR:** ADR-008  
**Source C4:** Circuit Breaker

**DoD:**
- [ ] Healthy → Degraded → Critical transition
- [ ] Critical blocks bet placement
- [ ] Recovery with hysteresis
- [ ] Stale threshold triggers correctly

---

### Task 5.2: Price Feed Resilience Tests
**Owner:** Backend Engineer  
**Estimated:** 2 hours  
**Source ADR:** ADR-004  
**Source C4:** Price Ingestion

**DoD:**
- [ ] Reconnection after disconnect
- [ ] Price buffering during disconnect
- [ ] Graceful degradation with mock data

---

## Module Deliverables

- [ ] External price feed ingestion
- [ ] Redis pub/sub price distribution
- [ ] Price health monitoring
- [ ] Circuit breaker middleware
- [ ] WebSocket price streaming
- [ ] System status broadcasting
- [ ] Price history query service
- [ ] Circuit breaker tests

---

## Exit Criteria

This module is complete when:
1. Real-time prices flow from exchange → Redis → WebSocket → Frontend
2. Circuit breaker blocks bets when price is stale >10s
3. Recovery requires 30s of stable prices (hysteresis)
4. Price history can be queried for bet resolution
5. System status is broadcast to all clients
6. Reconnection happens automatically after disconnect
