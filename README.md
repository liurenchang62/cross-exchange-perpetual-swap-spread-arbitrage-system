# Cross-Exchange Perpetual Swap Spread Arbitrage System

An event-driven Python system for studying and executing hedged long-short positions in the same USDT-margined perpetual futures across Binance, OKX, Gate, and Bitget.

The project models leader-follower price discovery with rolling-median equilibrium spreads. A candidate is generated only when a leading-venue price move and a same-direction residual deviation occur together. Candidate signals then pass through transaction-cost, contract, liquidity, capital, and operational risk checks before any two-leg order is submitted.

> Research software for educational use. This repository does not provide investment advice or claim that the strategy is profitable.

## Project scope

The repository implements the full path from public market data to guarded execution:

- dynamic selection of liquid perpetual futures shared by all four exchanges;
- normalized WebSocket market data and in-memory online spread estimation;
- leader-move detection and residual-based signal generation;
- order-book-aware evaluation of fees, funding rates, slippage, precision, and minimum notional;
- concurrent two-leg submission with explicit handling of asymmetric fills;
- position monitoring, exit controls, daily-loss limits, failure cooldowns, and capital rebalancing;
- separate demo/testnet and live-trading workflows with live-mode preflight checks.

## Research model

For leader venue `L`, follower venue `F`, and symbol `s`, the instantaneous spread in basis points is:

```text
S_t = 10^4 * (P_t^L - P_t^F) / P_t^F
```

A rolling median provides the equilibrium spread `B_t`. The detector evaluates the residual:

```text
A_t = S_t - B_t
```

This separates temporary dislocations from persistent venue premia. A market event requires:

1. a leader return above the configured threshold within `LEADER_WINDOW_MS`;
2. a residual deviation above `ANOMALY_MIN_PCT`;
3. agreement between the leader move and residual direction;
4. completion of the baseline warm-up period and direction-specific cooldown.

The detector records a market observation rather than an order instruction. Execution feasibility is evaluated separately.

## System architecture

```mermaid
flowchart LR
    A[Binance / OKX / Gate / Bitget] --> B[Normalized WebSocket ticks]
    B --> C[Rolling pair baselines]
    B --> D[Leader-move detector]
    C --> D
    D --> E[Candidate market event]
    E --> F{Cost and risk checks}
    F -->|Reject| G[Signal and audit logs]
    F -->|Accept| H[Concurrent two-leg orders]
    H --> I{Execution result}
    I -->|Both filled| J[Position monitoring]
    I -->|One leg filled| K[Emergency unwind]
    J --> L[Funding / timeout / TP / SL exits]
    L --> M[Concurrent close orders]
```

The signal path runs in one Python process on an `asyncio` event loop. Public WebSocket and REST data are consumed directly; authenticated REST clients handle exchange-specific account and order operations. No database, message broker, or external cache is used on the signal path.

See [Architecture](docs/ARCHITECTURE.md) for component boundaries and execution sequences.

## Execution and risk controls

Before opening a position, the system checks:

- expected fees, funding cost, and order-book slippage;
- contract multipliers, quantity precision, and venue minimums;
- available balances and per-symbol concentration;
- per-exchange order-rate limits;
- consecutive execution failures and cooldown state;
- daily balance halt and post-stop-loss entry suspension.

The two opening legs are submitted concurrently. Cross-exchange execution is not atomic: if only one leg succeeds, the system attempts to reverse the filled leg immediately. Close-leg failures are logged as critical events and require inspection for residual exposure.

Open positions are evaluated from current market prices and actual entry fills. Exit checks include unfavorable funding near settlement, maximum holding time, take-profit, and stop-loss conditions.

## Repository structure

```text
.
├── main.py                  # Process lifecycle and live-mode preflight
├── tracker/                 # Market data, universe, baselines, and signals
├── trader/                  # Cost model, execution, positions, and risk
├── rebalance/               # Cash-distribution checks and transfer planning
├── clients/                 # Venue URLs, symbol mapping, credential templates
├── test_demo/               # Demo/testnet integration checks
├── test_live/               # Live read-only and explicitly confirmed checks
├── tools/                   # Account, cash-flow, and recovery utilities
├── server/                  # Deployment and synchronization helpers
└── docs/                    # Architecture and operating documentation
```

## Quick start

### 1. Create an environment

```bash
python -m venv .venv
source .venv/bin/activate       # Linux/macOS
# .venv\Scripts\Activate.ps1  # Windows PowerShell
pip install -r requirements.txt
```

Python 3.10 or newer is required.

### 2. Configure demo credentials

Copy `clients/api_keys_demo.example.py` to `clients/api_keys_demo.py` and supply demo/testnet credentials. Local credential files are excluded by `.gitignore`. See [Configuration](CONFIG_GUIDE.md).

### 3. Validate before running

```bash
python -m test_demo.run_all --skip-orders
python -m tracker
python main.py
```

The default application mode uses demo/testnet configuration. Live mode performs a read-only preflight and requires an exact `YES` confirmation:

```bash
python main.py --live
```

Read [Operations and validation](docs/OPERATIONS.md) before using live commands. Some utilities can place orders, transfer assets, or request withdrawals.

## Default configuration

| Concern | Checked-in default |
|---|---:|
| Baseline warm-up / window | 60 s / 2,000 ticks |
| Leader window / move threshold | 1,000 ms / 0.3% |
| Minimum anomaly | 0.5% |
| Capital per pair | 1% of minimum futures balance, both legs combined |
| Leverage | 1x |
| Take-profit / stop-loss | 0.20% / 8.0% combined position PnL |
| Maximum holding time | 8 h |
| Daily balance halt | below 95% of day-start balance |

The source of truth is `tracker/config.py` and `trader/config.py`. These values are runtime defaults, not empirical results.

## Validation scope

The included checks are exchange-facing integration checks rather than a hermetic unit-test suite. They cover demo account, position, order, cancellation, and transfer flows; live connectivity, account, order-book, contract, and funding reads; and explicitly confirmed live order or transfer paths.

Results depend on exchange availability, credentials, permissions, clock synchronization, network region, and account state. A successful integration run does not establish strategy profitability.

The system was developed with separate testnet and live-trading workflows and was used for small-scale live validation. Account identifiers, balances, credentials, and transaction records are not published.

## Module guides

- [Market data and signal generation](tracker/README.md)
- [Execution, positions, and risk](trader/README.md)
- [Exchange configuration and credentials](clients/README.md)
- [Capital rebalancing](rebalance/README.md)
- [Validation suites](test_demo/README.md)
- [Operational utilities](tools/README.md)

## Documentation

- [Architecture and design boundaries](docs/ARCHITECTURE.md)
- [Operations and validation](docs/OPERATIONS.md)
- [Configuration and credential handling](CONFIG_GUIDE.md)
- [Command reference](COMMANDS.md)
- [Security policy](SECURITY.md)

## Limitations

- The baseline and thresholds are heuristic online statistics.
- Regime shifts, thin order books, network latency, venue outages, and API changes can invalidate assumptions.
- Temporary one-sided exposure cannot be eliminated because the exchanges do not provide a shared transaction boundary.
- File-based records do not provide transactional persistence or distributed recovery.
- The repository does not include a historical backtester, formal latency benchmark, public profitability report, or production service-level guarantee.

## License

No open-source license has been declared. All rights remain with the repository owner.
