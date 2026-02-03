# Phase 4: Hardening (Day 10)

**Goal:** Load testing, monitoring setup, documentation, production readiness check.

---

## 10.1 Load Testing
**Owner:** Backend (Lead), Frontend (Support)  
**Estimated:** 3 hours  
**Source:** NFR: 3,000 RPS

**Tasks:**
Create load test with `k6` or `wrk`:

```javascript
// load-test.js (k6)
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '1m', target: 100 },   // Ramp up
    { duration: '3m', target: 3000 },  // Peak load
    { duration: '1m', target: 0 },     // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<100'],   // 95% under 100ms
    http_req_failed: ['rate<0.01'],     // <1% errors
  },
};

export default function () {
  // Simulate bet placement
  const res = http.post('http://localhost:8080/api/bets', {
    cell_id: 'cell-1-2',
    amount: 1000,
    nonce: generateUUID(),
    timestamp: Date.now(),
    signature: generateSignature(),
  });
  
  check(res, {
    'status is 202': (r) => r.status === 202,
    'response time < 100ms': (r) => r.timings.duration < 100,
  });
  
  sleep(0.01); // 100 RPS per VU
}
```

**Test Scenarios:**
1. Steady state: 3,000 RPS for 5 minutes
2. Spike test: 0 → 3,000 RPS in 10 seconds
3. Soak test: 1,000 RPS for 30 minutes

**DoD:**
- [ ] System handles 3,000 RPS
- [ ] p95 latency < 100ms
- [ ] Error rate < 1%
- [ ] No SQLite contention errors

---

## 10.2 Monitoring & Alerting Setup
**Owner:** Backend  
**Estimated:** 2 hours  
**Source:** Observability Requirements

**Tasks:**
Configure Prometheus + Grafana (or cloudwatch/datadog):

**Metrics to Collect:**
```go
// Application metrics
requestDuration := prometheus.NewHistogramVec(...)
requestCount := prometheus.NewCounterVec(...)
batchQueueDepth := prometheus.NewGauge(...)
circuitBreakerState := prometheus.NewGaugeVec(...)
priceFeedAge := prometheus.NewGauge(...)

// System metrics (node_exporter)
cpuUsage, memoryUsage, diskUsage, diskIO

// Business metrics
betsPlacedTotal, betsResolvedTotal, 
winsTotal, lossesTotal, payoutTotal
```

**Alerts:**
```yaml
# AlertManager rules
- alert: HighLatency
  expr: http_request_duration_seconds{quantile="0.95"} > 0.1
  for: 5m
  
- alert: CircuitBreakerOpen
  expr: circuit_breaker_state == 2  # CRITICAL
  for: 1m
  
- alert: StalePriceFeed
  expr: price_feed_age_seconds > 10
  for: 30s
  
- alert: ArchiverStale
  expr: time() - last_successful_archive_timestamp > 691200  # 8 days
  for: 1h
  
- alert: DiskFull
  expr: disk_usage_percent > 85
  for: 5m
```

**DoD:**
- [ ] Metrics endpoint `/metrics` returns Prometheus format
- [ ] Grafana dashboards created
- [ ] Alerts configured and tested

---

## 10.3 Backup & Recovery Testing
**Owner:** Backend  
**Estimated:** 2 hours  
**Source:** SLA 99.5%

**Tasks:**
1. **Automated Backups:**
   ```bash
   # Hourly incremental backup
   0 * * * * sqlite3 /data/sqlite/tap_trading.db ".backup /data/backups/hourly/tap_trading_$(date +\%Y\%m\%d_\%H).db"
   
   # Sync to S3
   aws s3 sync /data/backups/ s3://tap-trading-backups/
   ```

2. **Recovery Test:**
   ```bash
   # Simulate failure
   sudo systemctl stop tap-trading-*
   rm /data/sqlite/tap_trading.db
   
   # Restore from backup
   cp /data/backups/hourly/tap_trading_20250203_12.db /data/sqlite/tap_trading.db
   
   # Restart
   sudo systemctl start tap-trading-*
   ```

3. **Document RTO/RPO:**
   - RTO (Recovery Time Objective): 15 minutes
   - RPO (Recovery Point Objective): 1 hour (last backup)

**DoD:**
- [ ] Hourly backups configured
- [ ] Backup uploaded to S3
- [ ] Recovery test successful
- [ ] RTO/RPO documented

---

## 10.4 Security Hardening
**Owner:** Backend  
**Estimated:** 2 hours  
**Source:** Security Requirements

**Tasks:**
1. **CORS Configuration:**
   ```go
   // Only allow frontend origin
   e.Use(middleware.CORSWithConfig(middleware.CORSConfig{
       AllowOrigins: []string{"https://taptrading.example.com"},
       AllowMethods: []string{http.MethodGet, http.MethodPost},
   }))
   ```

2. **Rate Limiting:**
   ```go
   // Per-wallet rate limit
   limiter := rate.NewLimiter(rate.Limit(10), 20) // 10/s burst 20
   ```

3. **Input Validation:**
   - Cell ID format validation
   - Amount range checks (min/max bet)
   - Timestamp sanity checks

4. **Secrets Management:**
   - Move secrets to environment variables
   - No hardcoded credentials
   - Use AWS IAM roles (not access keys)

**DoD:**
- [ ] CORS restricted to production domain
- [ ] Rate limiting enforced
- [ ] All inputs validated
- [ ] No secrets in code

---

## 10.5 Documentation
**Owner:** Both  
**Estimated:** 3 hours  
**Source:** General

**Tasks:**
Create documentation:

1. **README.md** (root)
   - Project overview
   - Quick start guide
   - Architecture diagram
   - Environment variables

2. **API.md**
   - OpenAPI spec
   - Example requests/responses
   - Error codes

3. **OPERATIONS.md**
   - Deployment procedures
   - Monitoring dashboards
   - Runbooks:
     - "Price feed down"
     - "Database corruption"
     - "High latency"
     - "Disk full"

4. **FRONTEND.md**
   - Component structure
   - State management
   - WebSocket protocol

**DoD:**
- [ ] README complete
- [ ] API documented
- [ ] Runbooks written
- [ ] Frontend documented

---

## 10.6 Production Readiness Checklist
**Owner:** Both  
**Estimated:** 2 hours  
**Source:** General

**Final Review:**

| Check | Status |
|-------|--------|
| All ADRs implemented | ☐ |
| Load test passed (3K RPS) | ☐ |
| Circuit breaker tested | ☐ |
| Archiver tested | ☐ |
| Recovery tested | ☐ |
| Monitoring in place | ☐ |
| Alerts configured | ☐ |
| Security review passed | ☐ |
| Documentation complete | ☐ |
| Code reviewed | ☐ |
| Tests passing (>80% coverage) | ☐ |

**Sign-off:**
- [ ] Backend Engineer: _______________
- [ ] Frontend Engineer: ______________

---

## Phase 4 Exit Criteria

- [ ] Load test confirms 3K RPS capability
- [ ] Monitoring and alerting operational
- [ ] Backup/recovery tested
- [ ] Security hardening complete
- [ ] Documentation comprehensive
- [ ] Production readiness checklist complete

---

## Post-Launch (Week 2+)

| Task | Frequency | Owner |
|------|-----------|-------|
| Review metrics dashboard | Daily | Both |
| Archive verification | Weekly | Backend |
| Integrity check review | Daily | Backend |
| Security patch review | Weekly | Backend |
| Performance optimization | As needed | Both |

---

## Traceability Matrix (Phase 4)

| Task | ADR/NFR Reference | Output |
|------|-------------------|--------|
| 10.1 Load Test | NFR: 3K RPS | Test Report |
| 10.2 Monitoring | Observability | Dashboards |
| 10.3 Backups | SLA 99.5% | Runbook |
| 10.4 Security | Security Controls | Hardening |
| 10.5 Documentation | - | Docs |
| 10.6 Readiness | All | Sign-off |
