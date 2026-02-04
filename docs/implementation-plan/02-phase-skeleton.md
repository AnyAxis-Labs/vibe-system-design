# Phase 2: Skeleton / Thin Thread (Horizontal)

**Goal:** Deploy minimal versions of all containers to validate connectivity and end-to-end flow.

**Duration:** Days 2-3

---

## 2.1 Minimal API Server

### Task 2.1.1: Skeleton API Server
**Owner:** Backend Engineer  
**Estimated:** 3 hours  
**Source ADR:** ADR-002 (Go Backend)  
**Source C4:** API Server container

**Description:**
Create minimal API server with health endpoint, routing, and middleware scaffolding.

**DoD:**
- [ ] HTTP server on :8080 with graceful shutdown
- [ ] `/healthz` endpoint returning `{"status": "ok"}`
- [ ] Request logging middleware
- [ ] Panic recovery middleware
- [ ] CORS configured for frontend
- [ ] systemd service file working

**Code Skeleton:**
```go
// cmd/api/main.go
package main

import (
    "context"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
    
    "github.com/labstack/echo/v4"
    "github.com/labstack/echo/v4/middleware"
    "github.com/tap-trading/backend/pkg/config"
    "github.com/tap-trading/backend/pkg/logger"
)

func main() {
    cfg := config.Load()
    log := logger.New(cfg.LogLevel)
    
    e := echo.New()
    e.Use(middleware.Recover())
    e.Use(middleware.RequestID())
    e.Use(logger.Middleware(log))
    
    e.GET("/healthz", func(c echo.Context) error {
        return c.JSON(http.StatusOK, map[string]string{
            "status": "ok",
            "component": "api",
        })
    })
    
    // Graceful shutdown
    go func() {
        if err := e.Start(":8080"); err != nil && err != http.ErrServerClosed {
            log.Fatal().Err(err).Msg("server error")
        }
    }()
    
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    
    if err := e.Shutdown(ctx); err != nil {
        log.Fatal().Err(err).Msg("shutdown error")
    }
}
```

---

### Task 2.1.2: Skeleton Bet Endpoint
**Owner:** Backend Engineer  
**Estimated:** 2 hours  
**Source ADR:** ADR-002  
**Source C4:** API Server → Bet Batcher relationship

**Description:**
Create placeholder bet endpoint that validates request format but returns stub response.

**DoD:**
- [ ] `POST /api/bets` endpoint accepting bet request JSON
- [ ] Request validation (required fields)
- [ ] Returns stub 202 Accepted response
- [ ] Error responses following API contract

**Stub Response:**
```json
{
  "status": "accepted",
  "bet_id": "stub_bet_001",
  "message": "Stub implementation - not persisted"
}
```

---

## 2.2 Minimal WebSocket Server

### Task 2.2.1: Skeleton WebSocket Server
**Owner:** Backend Engineer  
**Estimated:** 3 hours  
**Source ADR:** ADR-004 (Redis Pub/Sub)  
**Source C4:** WebSocket Server container

**Description:**
Create minimal WebSocket server with connection handling and basic message echo.

**DoD:**
- [ ] WebSocket server on :8081
- [ ] Connection upgrade handling
- [ ] Connection lifecycle logging
- [ ] Basic message echo for testing
- [ ] Heartbeat/ping-pong handling
- [ ] systemd service file working

**Code Skeleton:**
```go
// cmd/ws/main.go
package main

import (
    "net/http"
    "github.com/gorilla/websocket"
    "github.com/tap-trading/backend/pkg/logger"
)

var upgrader = websocket.Upgrader{
    CheckOrigin: func(r *http.Request) bool { return true },
}

func handleWebSocket(log logger.Logger) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        conn, err := upgrader.Upgrade(w, r, nil)
        if err != nil {
            log.Error().Err(err).Msg("upgrade failed")
            return
        }
        defer conn.Close()
        
        log.Info().Str("remote", r.RemoteAddr).Msg("client connected")
        
        for {
            mt, message, err := conn.ReadMessage()
            if err != nil {
                log.Info().Err(err).Msg("client disconnected")
                break
            }
            // Echo for now
            if err := conn.WriteMessage(mt, message); err != nil {
                log.Error().Err(err).Msg("write failed")
                break
            }
        }
    }
}
```

---

## 2.3 Minimal Price Ingestion

### Task 2.3.1: Skeleton Price Ingestion
**Owner:** Backend Engineer  
**Estimated:** 2 hours  
**Source ADR:** ADR-004 (Redis Pub/Sub)  
**Source C4:** Price Ingestion container

**Description:**
Create minimal price ingestion that generates fake prices and publishes to Redis.

**DoD:**
- [ ] Connects to external price feed (or generates mock data)
- [ ] Publishes to Redis channel `prices:btc:usd`
- [ ] Logs each published price
- [ ] systemd service file working

**Mock Implementation:**
```go
// cmd/price/main.go - stub
func main() {
    redisClient := redis.NewClient(...)
    ticker := time.NewTicker(1 * time.Second)
    
    basePrice := 42000.00
    for range ticker.C {
        // Mock price movement
        change := (rand.Float64() - 0.5) * 100
        price := basePrice + change
        
        redisClient.Publish(ctx, "prices:btc:usd", fmt.Sprintf("%.2f", price))
        redisClient.Set(ctx, "price:btc:usd", price, 1*time.Hour)
        
        // Add to sorted set for history
        redisClient.ZAdd(ctx, "prices:btc:usd:ts", redis.Z{
            Score:  float64(time.Now().Unix()),
            Member: fmt.Sprintf("%.2f", price),
        })
    }
}
```

---

## 2.4 Connectivity Validation

### Task 2.4.1: Integration Smoke Tests
**Owner:** Backend Engineer  
**Estimated:** 3 hours  
**Source ADR:** ADR-003 (Monolithic Deployment)  
**Source C4:** All container relationships

**Description:**
Write and run integration tests to verify all components connect properly.

**DoD:**
- [ ] Test: API server responds via Nginx
- [ ] Test: WebSocket accepts connections via Nginx
- [ ] Test: API can read/write SQLite
- [ ] Test: API can read/write Redis
- [ ] Test: WebSocket can subscribe to Redis pub/sub
- [ ] Test: Price ingestion publishes to Redis
- [ ] All services start via systemd without errors

**Test Commands:**
```bash
# Test Nginx → API
curl http://localhost/api/healthz

# Test Nginx → WebSocket
websocat ws://localhost/ws

# Test Redis connectivity
redis-cli ping

# Test SQLite
sqlite3 /data/test.db "SELECT 1;"

# Test systemd
sudo systemctl start tap-trading-api
curl http://localhost:8080/healthz
```

---

### Task 2.4.2: End-to-End Flow Test
**Owner:** Backend Engineer  
**Estimated:** 2 hours  
**Source ADR:** All  
**Source C4:** Full system

**Description:**
Verify the complete request flow from frontend through all layers.

**DoD:**
- [ ] Frontend can load via Nginx
- [ ] Frontend can place bet (stub)
- [ ] Bet request reaches API server
- [ ] Real-time prices flow to frontend via WebSocket

**Flow Verification:**
```
Frontend (Browser)
    ↓ HTTPS
Nginx (:80/443)
    ↓ HTTP :8080
API Server (responds)
    
Price Ingestion (:mock)
    ↓ Redis Pub/Sub
WebSocket Server (:8081)
    ↓ WebSocket
Frontend (displays price)
```

---

## 2.5 Service Definitions

### Task 2.5.1: Create systemd Services
**Owner:** Backend Engineer  
**Estimated:** 2 hours  
**Source ADR:** ADR-003  
**Source C4:** All Go containers

**Description:**
Create systemd service files for all Go binaries.

**DoD:**
- [ ] `/etc/systemd/system/tap-trading-api.service`
- [ ] `/etc/systemd/system/tap-trading-ws.service`
- [ ] `/etc/systemd/system/tap-trading-price.service`
- [ ] Services auto-restart on failure
- [ ] Services start on boot

**Example Service:**
```ini
# /etc/systemd/system/tap-trading-api.service
[Unit]
Description=BTC Tap Trading API Server
After=network.target redis.service

[Service]
Type=simple
User=app
Group=app
WorkingDirectory=/opt/app
ExecStart=/opt/app/bin/api
Restart=always
RestartSec=5
Environment=LOG_LEVEL=info
Environment=DB_PATH=/data/bets.db
Environment=REDIS_ADDR=localhost:6379

[Install]
WantedBy=multi-user.target
```

---

## Phase 2 Deliverables Checklist

- [ ] API server running on :8080 with /healthz
- [ ] WebSocket server running on :8081
- [ ] Price ingestion running (mock or real)
- [ ] Nginx proxying to both servers
- [ ] SQLite connectivity verified
- [ ] Redis connectivity verified
- [ ] All systemd services defined and tested
- [ ] Integration smoke tests passing
- [ ] End-to-end flow manually verified

---

## Exit Criteria for Phase 2

Phase 2 is complete when:
1. All containers can be started via systemd
2. `curl http://localhost/api/healthz` returns 200
3. WebSocket connection via Nginx succeeds
4. A price update flows from ingestion → Redis → WebSocket
5. API can read/write to both SQLite and Redis
6. No container crashes on startup

---

## Troubleshooting Notes

| Issue | Solution |
|-------|----------|
| SQLite locked | Ensure WAL mode is enabled |
| Redis OOM | Check maxmemory-policy is allkeys-lru |
| Nginx 502 | Verify upstream servers are running |
| WebSocket disconnect | Check upgrade headers in Nginx config |
| Permission denied | Ensure /data is owned by app user |
