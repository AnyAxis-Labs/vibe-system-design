# ADR 001: SQLite as Primary Database

## Status
Accepted

## Context
We need a database solution that:
- Supports 3,000 RPS peak load
- Provides atomic transactions for bet placement (prevent double-spend)
- Fits in 8GB RAM on a single VM
- Can be deployed and operational within 2 weeks by 1 backend engineer
- Retains bet history indefinitely (forever retention requirement)

## Decision
Use **SQLite** in WAL (Write-Ahead Logging) mode as the primary database.

## Consequences

### Positive
- **Zero operational complexity**: No separate process to manage, backup, or monitor
- **Single-file portability**: Easy backups, migrations, and local development
- **ACID compliance**: Native support for atomic transactions critical for betting correctness
- **Sufficient performance**: SQLite with WAL can handle 10,000+ writes/sec on SSD; our 3K RPS is read-heavy
- **Tiny resource footprint**: Fits comfortably in 8GB RAM constraint
- **Team velocity**: 1 senior backend engineer can be productive immediately

### Negative
- **No horizontal scaling**: Cannot shard across multiple VMs later without migration
- **Single-writer bottleneck**: All writes serialized through one connection pool
- **Manual backup strategy**: Need to implement automated backups (sqlite3 .backup or file-level)
- **No replication**: No hot standby; 99.5% SLA relies on VM reliability + quick restore from backup

### Trade-off Analysis
| Alternative | Why Rejected |
|-------------|--------------|
| PostgreSQL | Requires separate process, more config, connection pooling complexity; overkill for single VM |
| MySQL | Same as PostgreSQL; adds unnecessary operational burden |
| MongoDB | No ACID transactions across documents; harder to guarantee betting correctness |

## Mitigation Strategies

1. **WAL Mode**: Enables concurrent reads during writes, improves concurrency
2. **Connection Pooling**: Use `sql.DB` with appropriate `SetMaxOpenConns()` (suggest 10-20)
3. **Read Replicas (Future)**: If needed, can add read-only replicas via SQLite's backup API
4. **Backup Strategy**: Hourly incremental backups using `sqlite3 .backup`, daily full backups to S3
5. **Monitoring**: Alert on disk usage (256GB constraint), query slow log

## Compliance with NFRs

| NFR | Compliance |
|-----|------------|
| 3K RPS | ✅ SQLite handles 10K+ writes/sec on SSD; our workload is read-heavy |
| ≤100ms latency | ✅ In-process database eliminates network round-trip |
| Atomic bet placement | ✅ ACID transactions with `BEGIN IMMEDIATE` |
| 99.5% uptime | ⚠️ Acceptable risk given constraint; mitigated by backups |
| Forever retention | ✅ Single-file growth managed; archive old bets to cold storage if needed |

## Decision Date
2026-02-03

## Author
System Architect
