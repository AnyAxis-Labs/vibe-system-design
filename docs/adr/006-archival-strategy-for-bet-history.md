# ADR 006: Archival Strategy for Bet History

## Status
Accepted (Hardening Response)

## Context
Hardening Audit #001 calculated that "Forever" retention on 256GB SSD will exhaust disk in ~25 days:
- 600 bets/sec × 86400 × 200 bytes = ~10 GB/day
- 256GB ÷ 10GB/day = 25.6 days

## Decision
Implement **time-based database partitioning** with automated archival to cold storage.

## Solution Design

### Partitioning Strategy

```
┌──────────────────────────────────────────────────────────────┐
│                    SQLite Partition Layout                    │
│                                                               │
│  ┌─────────────────┐    ┌─────────────────┐                  │
│  │  bets_2025_w05  │    │  bets_2025_w06  │  (Cold - S3)     │
│  │  ~2GB (zstd)    │    │  ~2GB (zstd)    │                  │
│  └─────────────────┘    └─────────────────┘                  │
│                                                               │
│  ┌─────────────────┐    ┌─────────────────┐                  │
│  │  bets_2025_w07  │    │  bets_current   │  (Hot - Local)   │
│  │  (read-only)    │    │  (read-write)   │                  │
│  └─────────────────┘    └─────────────────┘                  │
│         ▲                      ▲                              │
│         │                      │                              │
│    Historical queries      Live betting                     │
│    (ATTACH DATABASE)       (INSERT)                          │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Weekly Rotation

| Phase | Action | Schedule |
|-------|--------|----------|
| **Active** | Current week bets go to `bets_yyyy_ww.db` | Continuous |
| **Freeze** | Close write, run `VACUUM INTO` | Sunday 00:00 UTC |
| **Compress** | `zstd bets_yyyy_ww.db` | Sunday 00:30 UTC |
| **Upload** | `aws s3 cp` to cold storage | Sunday 01:00 UTC |
| **Attach** | `ATTACH DATABASE` for reads | Sunday 01:30 UTC |
| **Purge** | Delete local compressed file after 4 weeks | 28 days later |

### Local Hot Storage Budget

| Retention | Size | Notes |
|-----------|------|-------|
| Current week (read-write) | ~10 GB | Active betting |
| Previous 4 weeks (read-only) | ~40 GB | Recent history queries |
| **Total Hot** | **~50 GB** | Leaves 200GB headroom |

### Query Routing

```go
type PartitionManager struct {
    currentDB *sql.DB
    attached  map[string]*sql.DB  // week -> read-only connection
}

func (pm *PartitionManager) QueryBets(ctx context.Context, userID string, from, to time.Time) ([]Bet, error) {
    // Determine which partitions to query
    weeks := getWeeksBetween(from, to)
    
    var results []Bet
    for _, week := range weeks {
        if week == currentWeek {
            // Query active DB
            results = append(results, pm.query(pm.currentDB, userID, from, to))
        } else {
            // Query attached historical DB
            if db, ok := pm.attached[week]; ok {
                results = append(results, pm.query(db, userID, from, to))
            } else {
                // Return "data archived" error or trigger async restore
                return nil, ErrArchived
            }
        }
    }
    return results, nil
}
```

### Automated Archival Script

```bash
#!/bin/bash
# /opt/app/scripts/archive-weekly.sh

WEEK=$(date -d 'last week' +%Y_w%U)
DB_FILE="/data/bets_${WEEK}.db"
ARCHIVE_FILE="/tmp/bets_${WEEK}.db.zst"

# 1. Freeze: VACUUM INTO (creates optimized copy)
sqlite3 /data/bets_current.db "VACUUM INTO '${DB_FILE}'"

# 2. Compress (zstd for speed + ratio)
zstd -19 -T4 -o "${ARCHIVE_FILE}" "${DB_FILE}"

# 3. Upload to S3
aws s3 cp "${ARCHIVE_FILE}" s3://tap-trading-archives/bets/ \
    --storage-class STANDARD_IA

# 4. Verify upload
aws s3 ls s3://tap-trading-archives/bets/bets_${WEEK}.db.zst

# 5. Update metadata
sqlite3 /data/metadata.db "INSERT INTO archives (week, s3_path, size) VALUES ('${WEEK}', 's3://...', $(stat -c%s ${ARCHIVE_FILE}))"

# 6. Attach new partition for local reads (optional, based on retention policy)
# 7. Purge old local files (>4 weeks)
find /data/bets_*.db -mtime +28 -delete
```

### Auto-Vacuum Configuration

```sql
-- Enable incremental auto-vacuum on each partition
PRAGMA auto_vacuum = INCREMENTAL;
PRAGMA journal_mode = WAL;

-- After deletions (if any), reclaim space without full VACUUM
PRAGMA incremental_vacuum(1000);
```

## Trade-off Analysis

| Alternative | Why Rejected |
|-------------|--------------|
| Single DB forever | Disk death in 25 days |
| DELETE + VACUUM | Locks database for hours; violates uptime |
| TimescaleDB | Extra process, complexity, not "boring" tech |
| Partition by day | 365 files/year = too many ATTACH operations |
| Partition by month | 4-5 weeks of hot data still ~50GB, acceptable |

## Operational Runbook

### Restoring Archived Data (User Dispute)

```bash
# 1. Identify week from timestamp
WEEK="2025_w05"

# 2. Download from S3
aws s3 cp s3://tap-trading-archives/bets/bets_${WEEK}.db.zst /tmp/

# 3. Decompress
unzstd /tmp/bets_${WEEK}.db.zst -o /data/restore/

# 4. Attach and query
sqlite3 /data/bets_current.db "ATTACH '/data/restore/bets_${WEEK}.db' AS h; SELECT * FROM h.bets WHERE user_id = '...';"
```

### Disk Alert Thresholds

| Threshold | Action |
|-----------|--------|
| 70% (180GB) | Warning: Accelerate archival |
| 85% (218GB) | Critical: Stop new bets, emergency archive |
| 95% (243GB) | Emergency: Purge oldest local partitions |

## NFR Compliance

| NFR | Compliance |
|-----|------------|
| Bet history forever | ✅ Archived to S3 with STANDARD_IA |
| 99.5% uptime | ✅ No full VACUUM; incremental only |
| Query performance | ✅ Current week fast; historical via ATTACH |
| 256GB constraint | ✅ ~50GB hot, ~200GB headroom |

## Decision Date
2026-02-03

## Author
System Architect (Hardening Response)
