# Phase 4: System-Level Hardening (Horizontal)

**Goal:** Validate emergent system behavior, failure modes, and operational readiness.

**Duration:** Days 9-10  
**Dependencies:** All Phase 3 modules complete  
**Priority:** CRITICAL (production readiness)

---

## Phase Scope

This phase validates:
- End-to-end workflows across modules
- Load testing at 3K RPS
- Failure mode testing (circuit breaker, reconnection, etc.)
- Operational readiness (monitoring, alerting, runbooks)

**Source ADRs:** All (system-wide validation)  
**Source C4:** Full system integration

---

## Task 1: Integration Testing

### Task 1.1: End-to-End Workflow Tests
**Owner:** Backend Engineer  
**Estimated:** 4 hours  
**Source ADR:** All  
**Source C4:** Full system

**Description:**
Test complete user journeys through the system.

**DoD:**
- [ ] User registration → deposit → bet → resolution → withdrawal flow
- [ ] WebSocket price updates reach frontend
- [ ] Bet placement returns 202, confirms via WebSocket
- [ ] Circuit breaker blocks bets during stale price
- [ ] Recovery allows bets after 30s stable prices

**Test Scenarios:**
```go
// Test: Complete Bet Flow
func TestCompleteBetFlow(t *testing.T) {
    // 1. User authenticates
    // 2. Price feed streams to WebSocket
    // 3. User places bet
    // 4. Bet confirmed via WebSocket
    // 5. Wait for resolution time
    // 6. Bet resolved, balance updated
}

// Test: Circuit Breaker Flow
func TestCircuitBreakerFlow(t *testing.T) {
    // 1. Normal operation - bets accepted
    // 2. Stop price feed
    // 3. After 10s, bets rejected with 503
    // 4. Resume price feed
    // 5. After 30s hysteresis, bets accepted again
}
```

---

### Task 1.2: Failure Mode Testing
**Owner:** Backend Engineer  
**Estimated:** 4 hours  
**Source ADR:** ADR-008 (Circuit Breaker)  
**Source C4:** All containers

**Description:**
Test system behavior under various failure conditions.

**DoD:**
- [ ] Redis failure: API degrades gracefully
- [ ] SQLite BUSY: Batcher retries correctly
- [ ] Price feed disconnect: Auto-reconnects
- [ ] Price feed stale: Circuit breaker activates
- [ ] Network partition: Timeouts handled correctly

**Failure Scenarios:**

| Scenario | Expected Behavior |
|----------|-------------------|
| Redis down | API returns 503, logs error |
| SQLite locked | Batcher retries with backoff |
| Price feed 500 | Reconnects with exponential backoff |
| Stale prices >10s | Circuit breaker blocks bets |
| WebSocket disconnect | Client reconnects with jitter |

---

## Task 2: Load Testing

### Task 2.1: 3K RPS Load Test
**Owner:** Backend Engineer  
**Estimated:** 4 hours  
**Source ADR:** ADR-005 (Write Batching)  
**Source C4:** Full system

**Description:**
Verify system sustains 3,000 requests per second.

**DoD:**
- [ ] 3,000 concurrent requests/second for 10 minutes
- [ ] p99 latency < 100ms
- [ ] p95 latency < 50ms
- [ ] Zero 5xx errors
- [ ] No SQLITE_BUSY errors
- [ ] Memory usage stable (no leaks)
- [ ] CPU usage < 80%

**Load Test Configuration:**
```yaml
# k6 or similar load test
declare
stages:
  - duration: 2m
    target: 1000  # Ramp up
  - duration: 5m
    target: 3000  # Sustained load
  - duration: 2m
    target: 1000  # Ramp down
  - duration: 1m
    target: 0

thresholds:
  http_req_duration{p(95)}: ['p(95)<50']
  http_req_duration{p(99)}: ['p(99)<100']
  http_req_failed: ['rate<0.001']  # <0.1% errors
```

**Monitoring During Test:**
```bash
# Watch key metrics
watch -n 1 'echo "=== SQLite ===" && sqlite3 /data/bets.db "PRAGMA wal_checkpoint;" && \
echo "=== Redis Memory ===" && redis-cli INFO memory | grep used_memory_human && \
echo "=== Disk ===" && df -h /data && \
echo "=== Load ===" && uptime'
```

---

### Task 2.2: Batcher Throughput Test
**Owner:** Backend Engineer  
**Estimated:** 2 hours  
**Source ADR:** ADR-005  
**Source C4:** Bet Batcher

**Description:**
Verify batcher handles 3K RPS without queue buildup.

**DoD:**
- [ ] Channel utilization < 50% at 3K RPS
- [ ] Batch flush rate ~30/second
- [ ] No bets timeout waiting for result
- [ ] Channel backpressure triggers when full

---

## Task 3: Stress Testing

### Task 3.1: Spike Test
**Owner:** Backend Engineer  
**Estimated:** 2 hours  
**Source ADR:** All  
**Source C4:** Full system

**Description:**
Test system response to sudden traffic spikes.

**DoD:**
- [ ] 0 to 5K RPS in 10 seconds
- [ ] No crashes
- [ ] Graceful degradation (503 with retry-after)
- [ ] Recovery to normal latency after spike

---

### Task 3.2: Soak Test
**Owner:** Backend Engineer  
**Estimated:** 4 hours (automated)  
**Source ADR:** ADR-006 (Archival)  
**Source C4:** Full system

**Description:**
Run sustained load to catch resource leaks.

**DoD:**
- [ ] 1K RPS sustained for 4 hours
- [ ] Memory usage stable (no growth trend)
- [ ] File descriptors stable
- [ ] Goroutine count stable
- [ ] Disk space growth matches projection

---

## Task 4: Operational Readiness

### Task 4.1: Health Endpoint Verification
**Owner:** Backend Engineer  
**Estimated:** 2 hours  
**Source ADR:** All  
**Source C4:** All containers

**Description:**
Verify all health endpoints report correct status.

**DoD:**
- [ ] `GET /healthz` returns 200 when healthy
- [ ] Returns 503 when dependency unhealthy
- [ ] Includes sub-system status (DB, Redis, price feed)
- [ ] Circuit breaker state included

**Expected Response:**
```json
GET /healthz
{
  "status": "healthy",
  "timestamp": "2026-02-03T12:00:00Z",
  "checks": {
    "database": {
      "status": "healthy",
      "connections": 15,
      "wal_size_mb": 12
    },
    "redis": {
      "status": "healthy",
      "latency_ms": 0.5,
      "memory_used_mb": 45
    },
    "price_feed": {
      "status": "healthy",
      "last_update": "2026-02-03T11:59:59Z",
      "stale_for_ms": 120
    },
    "circuit_breaker": {
      "status": "closed"
    }
  }
}
```

---

### Task 4.2: Metrics and Alerting
**Owner:** Backend Engineer  
**Estimated:** 3 hours  
**Source ADR:** All  
**Source C4:** All containers

**Description:**
Set up Prometheus metrics and alerting rules.

**DoD:**
- [ ] Request rate, latency, error rate metrics
- [ ] Bet placement counter (by outcome)
- [ ] Resolution counter (by win/loss)
- [ ] Circuit breaker state gauge
- [ ] Price feed staleness gauge
- [ ] Archive timestamp gauge
- [ ] Alert rules documented

**Key Metrics:**
```go
// pkg/metrics/metrics.go
var (
    RequestsTotal = prometheus.NewCounterVec(prometheus.CounterOpts{
        Name: "http_requests_total",
        Help: "Total HTTP requests",
    }, []string{"method", "endpoint", "status"})
    
    RequestDuration = prometheus.NewHistogramVec(prometheus.HistogramOpts{
        Name:    "http_request_duration_seconds",
        Help:    "HTTP request duration",
        Buckets: prometheus.DefBuckets,
    }, []string{"method", "endpoint"})
    
    BetsPlacedTotal = prometheus.NewCounterVec(prometheus.CounterOpts{
        Name: "bets_placed_total",
        Help: "Total bets placed",
    }, []string{"status"}) // confirmed, rejected, timeout
    
    BetsResolvedTotal = prometheus.NewCounterVec(prometheus.CounterOpts{
        Name: "bets_resolved_total",
        Help: "Total bets resolved",
    }, []string{"outcome"}) // won, lost
    
    PriceFeedStaleness = prometheus.NewGauge(prometheus.GaugeOpts{
        Name: "price_feed_staleness_seconds",
        Help: "Seconds since last price update",
    })
    
    CircuitBreakerState = prometheus.NewGauge(prometheus.GaugeOpts{
        Name: "circuit_breaker_state",
        Help: "Circuit breaker state (0=closed, 1=open)",
    })
    
    LastArchiveTimestamp = prometheus.NewGauge(prometheus.GaugeOpts{
        Name: "last_successful_archive_timestamp",
        Help: "Unix timestamp of last archive",
    })
)
```

**Alert Rules:**
```yaml
groups:
  - name: tap-trading
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.01
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          
      - alert: HighLatency
        expr: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "p99 latency > 100ms"
          
      - alert: PriceFeedStale
        expr: price_feed_staleness_seconds > 10
        for: 10s
        labels:
          severity: critical
        annotations:
          summary: "Price feed is stale"
          
      - alert: CircuitBreakerOpen
        expr: circuit_breaker_state == 1
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Circuit breaker is open"
          
      - alert: ArchiveStale
        expr: time() - last_successful_archive_timestamp > 8 * 24 * 3600
        for: 1h
        labels:
          severity: critical
        annotations:
          summary: "Weekly archive has not run in 8+ days"
          
      - alert: DiskSpaceLow
        expr: (node_filesystem_avail_bytes{mountpoint="/data"} / node_filesystem_size_bytes{mountpoint="/data"}) < 0.15
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disk space < 15%"
```

---

### Task 4.3: Runbook Documentation
**Owner:** Backend Engineer  
**Estimated:** 2 hours  
**Source ADR:** All  
**Source C4:** Operations

**Description:**
Create operational runbooks for common scenarios.

**DoD:**
- [ ] Restart procedure for each service
- [ ] Database backup/restore procedure
- [ ] Circuit breaker manual override
- [ ] Price feed failover procedure
- [ ] Disk space emergency procedures

**Example Runbook:**
```markdown
# Runbook: Circuit Breaker Manual Override

## Scenario
Circuit breaker is stuck open due to price feed issues.

## Procedure
1. Check price feed status: `curl /healthz | jq .checks.price_feed`
2. Verify external feed is healthy
3. Manual override: `redis-cli PUBLISH circuit:override "close"`
4. Monitor for 5 minutes
5. Remove override: `redis-cli PUBLISH circuit:override "auto"`

## Rollback
Set back to auto mode if issues persist.
```

---

## Task 5: Security Validation

### Task 5.1: Replay Attack Test
**Owner:** Backend Engineer  
**Estimated:** 2 hours  
**Source ADR:** ADR-007 (Replay Protection)  
**Source C4:** API Server

**Description:**
Verify replay protection is effective.

**DoD:**
- [ ] Replay same request with same nonce → rejected
- [ ] Replay with old timestamp → rejected
- [ ] Replay with future timestamp → rejected
- [ ] Cross-chain replay → rejected

---

### Task 5.2: Double-Spend Prevention Test
**Owner:** Backend Engineer  
**Estimated:** 2 hours  
**Source ADR:** ADR-009 (Conditional Deduction)  
**Source C4:** Bet Batcher

**Description:**
Verify double-spend is impossible under concurrent load.

**DoD:**
- [ ] 100 concurrent bets from same user
- [ ] Total spent never exceeds balance
- [ ] No negative balances in database
- [ ] Ledger balances match user balances

---

## Phase 4 Deliverables

- [ ] End-to-end workflow tests passing
- [ ] Failure mode tests passing
- [ ] 3K RPS load test passing (p99 < 100ms)
- [ ] Spike test passing
- [ ] Soak test passing (4 hours, no leaks)
- [ ] Health endpoint verified
- [ ] Prometheus metrics exported
- [ ] Alert rules configured
- [ ] Runbooks created
- [ ] Replay attack test passing
- [ ] Double-spend prevention verified

---

## Exit Criteria for Phase 4

Phase 4 (and the entire implementation) is complete when:

1. **Functional:** All end-to-end workflows complete successfully
2. **Performance:** 3,000 RPS sustained with p99 < 100ms
3. **Reliability:** All failure modes handled gracefully
4. **Security:** Replay attacks blocked, double-spend impossible
5. **Observability:** Metrics, logs, and alerts operational
6. **Operations:** Runbooks exist for common scenarios
7. **Data Integrity:** Daily integrity check passes
8. **Archival:** Weekly archival functional with dead man's switch

---

## Final Checklist

- [ ] SQLite WAL mode enabled
- [ ] Redis AOF + RDB persistence configured
- [ ] Nginx rate limiting active
- [ ] S3 bucket with STANDARD_IA lifecycle
- [ ] Circuit breaker thresholds configured
- [ ] Nonce TTL set to 90 seconds
- [ ] Circuit breaker hysteresis set to 30s
- [ ] Pending UX state implemented
- [ ] Redis balance cache rebuild on startup
- [ ] Transaction ledger table created
- [ ] Daily integrity check cron installed
- [ ] Archiver includes ledger table
- [ ] Dead man's switch metric alerting
