# ADR 003: Monolithic Single-VM Deployment

## Status
Accepted

## Context
Hard constraints: 2-week timeline, single VM (4c/8GB/256GB), 2-person team. Must support 3K RPS and 99.5% uptime.

## Decision
Deploy as **monolithic architecture** on a single VM with process supervision (systemd).

## Consequences

### Positive
- **Minimal ops overhead**: No container orchestration, service mesh, or load balancers
- **Fast deployment**: `scp` binary + `systemctl restart` is the deploy pipeline
- **Simple debugging**: All logs in one place, single machine to SSH into
- **Resource efficiency**: No Docker/Kubernetes overhead consuming RAM
- **Team velocity**: Both engineers can focus on features, not infrastructure

### Negative
- **No horizontal scaling**: Cannot add VMs to handle traffic spikes
- **Single point of failure**: VM failure = total outage
- **Maintenance windows**: Deploys require brief downtime (acceptable per 99.5% SLA)
- **Blast radius**: Bug in one component affects all

### Trade-off Analysis
| Alternative | Why Rejected |
|-------------|--------------|
| Docker Compose | Adds complexity without benefit on single VM; resource overhead |
| Kubernetes | Massive overkill; violates 2-week constraint; requires ops expertise |
| Microservices | Network latency between services; operational complexity; team too small |
| Serverless (Lambda/Cloud Functions) | Cold start latency violates 100ms requirement; vendor lock-in |

## Process Architecture

```
Single VM
├── nginx (reverse proxy, static files)
├── api-server (Go binary, port 8080)
├── ws-server (Go binary, port 8081)
├── price-ingestion (Go binary)
├── bet-resolver (Go binary, cron-triggered or daemon)
├── redis-server (port 6379)
└── sqlite.db (single file on SSD)
```

## Deployment Flow

```bash
# Build on CI
go build -o tap-trading-api ./cmd/api
go build -o tap-trading-ws ./cmd/ws

# Deploy via SSH
scp tap-trading-* user@vm:/opt/app/
ssh user@vm "sudo systemctl restart tap-trading-*"
```

## Monitoring

- **Process supervision**: systemd auto-restarts crashed services
- **Health checks**: HTTP `/health` endpoint for each service
- **Log aggregation**: journald + optional forwarding to external log service
- **Metrics**: Prometheus exporter embedded in API server

## Decision Date
2026-02-03

## Author
System Architect
