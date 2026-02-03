# Product Requirements Document (PRD)

## 1. Functional Requirements

The product is a web-based tap trading application that streams the real-time price of BTC and allows users to place directional bets on future price movements using a grid-based interface.

The core interaction model is a 2D grid where:

* The x-axis represents future timestamps divided into discrete columns.
* The y-axis represents BTC price ranges divided into discrete rows.
* Each grid cell corresponds to a bounded price range at a specific future time window.

Users can place a bet by tapping on a single grid cell, subject to the following rules:

* A user may place at most one bet per grid cell.
* Users select a bet size using predefined amount presets.
* Cells that are in the past, in the current column, or within one column ahead of the current price column are not placeable.
* No secondary confirmation step is required; a valid tap immediately places the bet.

Bet resolution logic:

* A cell becomes eligible for resolution only after the market price has passed its associated time column.
* A bet is considered a win if, during or after the column is passed, the BTC price enters the cell’s defined price range (price >= lower bound AND price <= upper bound).
* If the bet wins, the user receives: bet size × reward rate of that cell.
* If the bet loses, the user loses 100% of the bet size.

Reward rates:

* Each cell has an associated reward rate.
* Reward rates are computed and provided by an internal R&D-owned algorithm.
* Reward rates may vary across cells in both the time and price dimensions.

Account and custody model:

* The system is fully custodial.
* Users register and log in via Web3 wallet signature (no password-based authentication).
* User balances, bets, and payouts are managed off-chain within the application backend.

Market data:

* The application streams real-time BTC price data.
* Price updates must propagate to the UI fast enough to maintain a coherent and responsive grid state.

## 2. Non-Functional Requirements

Availability:

* Target SLA: 99.5% uptime.
* Scheduled maintenance is acceptable during low-trading hours.

Performance and latency:

* End-to-end perceived latency per user interaction must be ≤ 100 ms.
* Tap interactions must be recognized instantly with no blocking confirmation dialogs.
* Price updates should feel real-time and visually continuous.

Scalability:

* The system must sustain approximately 3,000 requests per second at peak load.
* Graceful degradation is acceptable beyond this target, but correctness must be preserved.

Reliability and correctness:

* Bet placement must be atomic and idempotent.
* A bet must never be placed twice for the same user-cell pair.
* Bet resolution must be deterministic and reproducible from market price history.

Security:

* Web3 signature-based authentication must prevent replay attacks.
* Custodial balances must be protected against double-spend and race conditions.
* Server-side validation must enforce all betting constraints regardless of client behavior.

Observability:

* The system must expose basic metrics for request rate, latency, error rate, and bet resolution outcomes.
* Logging must be sufficient to reconstruct disputes around bet placement and resolution.

Data retention:

* Bet history must be retained indefinitely (forever).
* Market price data retained for the last 7 days only.

## 3. Constraints

Time-to-market:

* The initial production-ready version must be delivered within 2 weeks.

Infrastructure:

* Deployment is constrained to a single virtual machine with:

  * 4-core CPU (2.5 GHz)
  * 8 GB RAM
  * 256 GB SSD

Team capacity:

* Development resources are limited to:

  * 1 senior backend engineer
  * 1 senior frontend engineer

Technology scope implications:

* The architecture must be simple, operationally lightweight, and fast to implement.
* Complex distributed systems, multi-region deployments, or heavy data pipelines are out of scope for the initial release.

## 4. Explicit Non-Goals

The following are explicitly out of scope for this product phase:

* Ultra-low-latency engineering targeting ≤ 10 ms per user interaction.
* Scaling to extreme throughput levels such as 10,000+ RPS.
* Supporting assets other than BTC.
* On-chain settlement, non-custodial wallets, or smart contract–based bet resolution.
* Advanced trading features such as hedging, cash-out before resolution, or secondary markets for bets.
* Mobile native applications (iOS/Android) in the initial phase.
