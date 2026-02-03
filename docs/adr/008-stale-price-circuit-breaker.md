# ADR 008: Stale Price Circuit Breaker

## Status
Accepted (Hardening Response)

## Context
Hardening Audit #001 asked: "What happens if the price feed stops?" Users could bet on stale prices, or resolutions could use incorrect data.

## Decision
Implement **automatic circuit breaker** that suspends betting when price feed is stale.

## Solution Design

### Health Check Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    Price Health Monitor                       │
│                                                               │
│  ┌─────────────────┐     ┌─────────────────┐                 │
│  │ Price Ingestion │────▶│  Redis (pub/sub)│                 │
│  │   Service       │     │  + Last Updated │                 │
│  └─────────────────┘     │  Timestamp      │                 │
│                          └────────┬────────┘                 │
│                                   │                           │
│                                   ▼                           │
│  ┌─────────────────┐     ┌─────────────────┐                 │
│  │   API Server    │◄────│  Circuit Breaker│                 │
│  │  (Bet Handler)  │     │   (Middleware)  │                 │
│  └─────────────────┘     └─────────────────┘                 │
│          ▲                                                    │
│          │ REJECT if unhealthy                                 │
│          ▼                                                    │
│  ┌─────────────────┐                                          │
│  │   Health Status │  [HEALTHY / DEGRADED / CRITICAL]         │
│  │   /healthz      │                                          │
│  └─────────────────┘                                          │
└──────────────────────────────────────────────────────────────┘
```

### Implementation

```go
const (
    PriceIntervalExpected = 1 * time.Second  // Expect 1 tick/sec
    StaleThreshold        = 3 * time.Second  // Degraded after 3s
    CriticalThreshold     = 10 * time.Second // Stop betting after 10s
)

type PriceHealthMonitor struct {
    redis       *redis.Client
    lastUpdated atomic.Int64  // Unix nano timestamp
    status      atomic.Value  // HealthStatus
}

type HealthStatus int

const (
    Healthy HealthStatus = iota
    Degraded            // Price delayed but recent
    Critical            // Price stale, betting suspended
)

func (m *PriceHealthMonitor) Start() {
    // Subscribe to price updates
    pubsub := m.redis.Subscribe(context.Background(), "prices:btc:usd")
    
    go func() {
        for msg := range pubsub.Channel() {
            m.lastUpdated.Store(time.Now().UnixNano())
            m.status.Store(Healthy)
        }
    }()
    
    // Background health checker
    ticker := time.NewTicker(1 * time.Second)
    go func() {
        for range ticker.C {
            m.evaluateHealth()
        }
    }()
}

func (m *PriceHealthMonitor) evaluateHealth() {
    last := time.Unix(0, m.lastUpdated.Load())
    age := time.Since(last)
    
    switch {
    case age > CriticalThreshold:
        m.status.Store(Critical)
        log.Error().Dur("stale_for", age).Msg("PRICE_FEED_CRITICAL")
        
    case age > StaleThreshold:
        m.status.Store(Degraded)
        log.Warn().Dur("stale_for", age).Msg("PRICE_FEED_DEGRADED")
        
    default:
        m.status.Store(Healthy)
    }
}

func (m *PriceHealthMonitor) IsHealthy() bool {
    return m.status.Load().(HealthStatus) != Critical
}

func (m *PriceHealthMonitor) GetStatus() HealthStatus {
    return m.status.Load().(HealthStatus)
}
```

### API Integration

```go
func BetMiddleware(health *PriceHealthMonitor) echo.MiddlewareFunc {
    return func(next echo.HandlerFunc) echo.HandlerFunc {
        return func(c echo.Context) error {
            if !health.IsHealthy() {
                return c.JSON(503, map[string]string{
                    "error": "PRICE_FEED_STALE",
                    "message": "Betting temporarily suspended due to stale price data",
                    "retry_after": "10",
                })
            }
            return next(c)
        }
    }
}

// Apply to bet placement routes only
api.POST("/bets", placeBetHandler, BetMiddleware(healthMonitor))
```

### Frontend Behavior

```typescript
// WebSocket status handler
ws.on('system:status', (status: SystemStatus) => {
    if (status.priceHealth === 'CRITICAL') {
        grid.disableInteractions();
        showBanner('Betting suspended - price feed issue detected');
    } else if (status.priceHealth === 'DEGRADED') {
        showWarning('Price feed delayed - trades at your own risk');
    } else {
        grid.enableInteractions();
        hideBanner();
    }
});
```

### Resolution Safety

Bet resolution also validates price freshness:

```go
func (r *BetResolver) Resolve(ctx context.Context, bet Bet) error {
    // Never resolve with potentially stale data
    if !r.priceMonitor.IsHealthy() {
        return ErrResolutionPaused
    }
    
    // Verify price history covers the resolution window
    prices := r.getPriceHistory(bet.TargetTimeStart, bet.TargetTimeEnd)
    if len(prices) == 0 {
        return ErrInsufficientPriceData
    }
    
    // Proceed with resolution...
}
```

### Alerting

| Condition | Severity | Action |
|-----------|----------|--------|
| DEGRADED > 30s | Warning | PagerDuty low-priority |
| CRITICAL > 10s | Critical | PagerDuty high-priority, auto-suspend betting |
| Recovery | Info | Resume betting, log incident |

## Health Endpoint

```json
GET /healthz
{
  "status": "healthy",
  "checks": {
    "price_feed": {
      "status": "healthy",
      "last_update": "2026-02-03T06:50:10Z",
      "stale_for_ms": 120
    },
    "database": {
      "status": "healthy",
      "connections": 15
    },
    "redis": {
      "status": "healthy",
      "latency_ms": 0.5
    }
  }
}
```

## Trade-off Analysis

| Alternative | Why Rejected |
|-------------|--------------|
| No circuit breaker | Users bet on stale prices = fairness violation |
| Only log warnings | Not proactive; ops may miss alert |
| Automatic failover to backup feed | Adds complexity; 2-week constraint |
| Full exchange integration | Out of scope; we consume feeds |

## NFR Compliance

| NFR | Compliance |
|-----|------------|
| Correctness | ✅ Never resolve with stale data |
| Fairness | ✅ Users can't exploit price delays |
| Observability | ✅ Health endpoint + metrics |

## Decision Date
2026-02-03

## Author
System Architect (Hardening Response)
