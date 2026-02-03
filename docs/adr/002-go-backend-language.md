# ADR 002: Go as Backend Language

## Status
Accepted

## Context
We have 1 senior backend engineer and 2 weeks to deliver a production system handling 3K RPS with strict correctness requirements around concurrent bet placement.

## Decision
Use **Go** for all backend services (API, WebSocket, Workers).

## Consequences

### Positive
- **Concurrency primitives**: Goroutines + channels ideal for WebSocket handling and background workers
- **Type safety**: Compile-time checks reduce runtime errors in financial logic
- **Static binary**: Single binary deployment simplifies ops on single VM
- **Performance**: Efficient memory usage fits 8GB constraint; handles 3K RPS easily
- **SQLite ecosystem**: Excellent `modernc.org/sqlite` and `mattn/go-sqlite3` drivers
- **Team fit**: Senior backend engineer likely already productive in Go

### Negative
- **Verbosity**: More boilerplate than Python/Node.js for simple CRUD
- **Frontend gap**: No code sharing with frontend (unlike Node.js), but team is separate anyway

### Trade-off Analysis
| Alternative | Why Rejected |
|-------------|--------------|
| Node.js | Event-loop can block on CPU-heavy bet resolution; type safety requires TypeScript overhead |
| Python (FastAPI) | GIL limits true concurrency; async/await complexity for WebSocket broadcasting |
| Rust | Steeper learning curve, slower development velocity; 2-week timeline risk |

## Decision Date
2026-02-03

## Author
System Architect
