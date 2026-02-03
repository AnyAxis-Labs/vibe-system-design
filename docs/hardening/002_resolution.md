# Hardening Resolution: 002_followup_critique.md

**Date:** 2026-02-03  
**Status:** ALL RISKS ADDRESSED  
**Architecture Version:** v1.2 (Operationally Hardened)

---

## Summary

All 5 operational risks from Round 002 have been addressed:

| # | Risk | Severity | Solution | ADR/Config |
|---|------|----------|----------|------------|
| 1 | Optimistic UI Trust Gap | High | "Pending" UX until WS confirm | UX Spec |
| 2 | Redis Nonce Memory Bloat | Medium | TTL 24h → 90s | Config |
| 3 | Archival Silent Death | Medium | Dead man's switch metric | Ops |
| 4 | **Double-Spend Race** | **CRITICAL** | **Conditional SQL deduction** | **ADR-009** |
| 5 | Circuit Breaker Flapping | Low | 30s hysteresis on recovery | Config |

---

## Detailed Resolution

### 1. Data Loss vs. User Trust (Optimistic UI)

**Problem:** User sees "Success" but server crashes before batch flush → phantom bet.

**Solution:** Strict UX state machine

```
User Tap
    ↓
Grid Cell: "PENDING" (pulsing animation)
    ↓
WebSocket receives "bet:confirmed"
    ↓
Grid Cell: "CONFIRMED" (solid color)
    ↓
WebSocket disconnects during pending
    ↓
Grid Cell: "UNKNOWN" (error state, prompt refresh)
```

**Implementation:** Client MUST NOT persist "confirmed" until WebSocket event received.

---

### 2. Redis Memory Policy (Nonce TTL)

**Problem:** 24h nonce TTL × 3K RPS = ~260M keys = ~13GB RAM.

**Solution:** Reduce TTL to match timestamp window + buffer.

```
Timestamp window: 30 seconds
Safety buffer:    60 seconds
────────────────────────────────
Nonce TTL:        90 seconds
```

**Impact:** ~270K keys max = ~13MB RAM (1000× reduction).

**Config:**
```bash
SET nonce:{uuid} {user_id} EX 90 NX
```

---

### 3. Archival Job Silent Death

**Problem:** Weekly archiver fails silently → disk fills → outage.

**Solution:** Dead man's switch metric.

**Implementation:**
```go
// In Weekly Archiver
func (a *Archiver) Run() error {
    // ... archival logic ...
    
    // Success: update timestamp
    a.metrics.GaugeSet("last_successful_archive_timestamp", time.Now().Unix())
    return nil
}
```

**Alert:**
```yaml
# Prometheus AlertManager
- alert: ArchiverStale
  expr: time() - last_successful_archive_timestamp > 8 * 24 * 3600
  severity: critical
  summary: Weekly archiver hasn't run in 8+ days
```

---

### 4. Double-Spend Race Condition (CRITICAL)

**Problem:** Redis balance check + SQLite batch deduction = race window.

**Solution:** Database as single source of truth.

```sql
-- Conditional deduction (atomic)
UPDATE users 
SET balance = balance - ?
WHERE id = ? 
AND balance >= ?;  -- <-- This prevents overdraft

-- Check RowsAffected
-- 0 = insufficient funds (reject)
-- 1 = success (insert bet)
```

**Flow:**
```
1. Redis check: Fast early rejection (cache, not authoritative)
2. Submit to batcher
3. Batcher transaction:
   a. UPDATE users (conditional)
   b. IF success → INSERT bet
   c. IF fail → INSERT bet with status='insufficient_funds'
4. Commit
5. Best-effort Redis cache update
```

**Recovery:** On startup, rebuild Redis cache from SQLite:
```go
SELECT id, balance FROM users;
// Pipeline SET to Redis
```

**See:** `docs/adr/009-conditional-balance-deduction.md`

---

### 5. Flapping Circuit Breaker

**Problem:** Unstable feed (9s, 11s, 9s) causes rapid Healthy/Critical toggling.

**Solution:** Hysteresis on recovery.

```go
const (
    DegradedThreshold  = 3 * time.Second
    CriticalThreshold  = 10 * time.Second
    RecoveryStability  = 30 * time.Second  // NEW
)

type CircuitBreaker struct {
    status          Status
    lastHealthyTime time.Time
}

func (cb *CircuitBreaker) Evaluate(age time.Duration) {
    switch {
    case age > CriticalThreshold:
        cb.status = CRITICAL
        cb.lastHealthyTime = time.Time{}  // Reset
        
    case age < DegradedThreshold && cb.status == CRITICAL:
        // Only recover after 30s of stable updates
        if cb.lastHealthyTime.IsZero() {
            cb.lastHealthyTime = time.Now()
        } else if time.Since(cb.lastHealthyTime) > RecoveryStability {
            cb.status = HEALTHY
        }
        
    case age > DegradedThreshold:
        cb.status = DEGRADED
        cb.lastHealthyTime = time.Time{}  // Reset
    }
}
```

**Behavior:**
| Transition | Delay |
|------------|-------|
| HEALTHY → CRITICAL | Immediate (at 10s) |
| CRITICAL → HEALTHY | 30s of stable updates |

---

## Verdict

**Reviewer:** System Architecture Guardian  
**Verdict:** ✅ **ACCEPTED FOR IMPLEMENTATION**

All operational risks addressed. Architecture v1.2 is production-ready for the 2-week timeline.

### Final Checklist

- [ ] ADR-009 implemented (conditional balance deduction)
- [ ] Nonce TTL configured to 90 seconds
- [ ] Dead man's switch metric + alerting configured
- [ ] Circuit breaker hysteresis (30s) implemented
- [ ] UX "Pending" state spec delivered to frontend team
- [ ] Redis cache rebuild on startup implemented
