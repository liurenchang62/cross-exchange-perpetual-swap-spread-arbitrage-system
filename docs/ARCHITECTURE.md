# Architecture

[Project overview](../README.md) · [Operations](OPERATIONS.md) · [Configuration](../CONFIG_GUIDE.md)

This document describes internal component ownership, execution sequencing, and design boundaries. The project-level workflow is summarized in the [README](../README.md); it is not repeated here.

## Component architecture

The system runs as one Python process on an `asyncio` event loop. It connects directly to Binance, OKX, Gate, and Bitget through public WebSocket/REST endpoints and authenticated REST endpoints. No database, broker, or external cache sits on the signal path.

The diagram below shows code-level dependencies rather than the chronological trading workflow. It uses a fixed SVG canvas so labels and component boundaries remain visible at GitHub's responsive widths.

![Component architecture with venue, signal, execution, and persistence layers](component-architecture.svg)

**Legend:** ⬜ Processing or state · 🟨 Decision or control · 🟥 Exception recovery

The single event loop keeps state coordination direct. Synchronous work on the tick path must remain short; network operations are scheduled as asynchronous tasks.

## Component boundaries

| Component | Responsibility | Out of scope |
|---|---|---|
| `clients/` | Endpoints, exchange lists, symbols, credential loading | Trading policy |
| `tracker/symbol_selector.py` | Shared liquid instrument universe | Position sizing |
| `tracker/ws_feed.py` | Connections, parsing, normalization, reconnects | Signal interpretation |
| `tracker/baseline.py` | Rolling spread state and warm-up | Order decisions |
| `tracker/signal_detector.py` | Leader history, anomaly rules, cooldowns | Transaction costs |
| `tracker/tracker.py` | Market-data orchestration and callbacks | Authenticated execution |
| `trader/cost_model.py` | Fees, funding, slippage, quantity feasibility | Order transport |
| `trader/exchange_client.py` | Venue-specific authenticated REST operations | Cross-venue policy |
| `trader/risk.py` | Halts, limits, cooldowns, balance state | Signal generation |
| `trader/position_manager.py` | Position registry and trade records | Market-data collection |
| `trader/trader.py` | Entry, exit, and failure recovery | Raw feed parsing |
| `rebalance/` | Cash distribution, route planning, transfers | Alpha signal |

## Market-data and signal path

1. `SymbolSelector` ranks Binance USDT perpetuals by reported 24-hour volume, intersects the result with the other active venues, and applies the minimum-volume filter.
2. `WSFeed` converts venue payloads into `Tick(exchange, symbol, bid, ask, mid, ts_ns)`.
3. `Tracker` stores the latest observations, updates pair baselines, and evaluates signals after warm-up.
4. `BaselineTracker` maintains bounded rolling spread histories.
5. `SignalDetector` checks leader returns, baseline residuals, direction agreement, and cooldown state.
6. A `MarketEvent` is passed to the trader callback; it is not itself an instruction to trade.

Direct in-process callbacks avoid a queue on the normal signal path. This is a latency-oriented design choice, not a measured latency guarantee.

## Entry sequence

```mermaid
%%{init: {"theme":"base","themeVariables":{"fontFamily":"-apple-system, BlinkMacSystemFont, Segoe UI, sans-serif","actorBkg":"#ffffff","actorBorder":"#57606a","actorTextColor":"#24292f","actorLineColor":"#8c959f","signalColor":"#57606a","signalTextColor":"#24292f","labelBoxBkgColor":"#f6f8fa","labelBoxBorderColor":"#d0d7de","labelTextColor":"#24292f","loopTextColor":"#24292f","activationBkgColor":"#fff8c5","activationBorderColor":"#9a6700","noteBkgColor":"#fff8c5","noteBorderColor":"#9a6700"}}}%%
sequenceDiagram
    participant Feed as WSFeed
    participant Tracker
    participant Detector
    participant Trader
    participant Risk
    participant Cost as CostModel
    participant Follower
    participant Leader

    Feed->>Tracker: normalized Tick
    Tracker->>Detector: latest prices and baseline
    Detector-->>Tracker: MarketEvent
    Tracker->>Trader: opportunity callback
    Trader->>Risk: eligibility and limits
    Trader->>Cost: quantity and expected net edge
    par concurrent submission
        Trader->>Follower: opening order
        Trader->>Leader: hedge order
    end
    alt both succeed
        Trader->>Trader: register position
    else one succeeds
        Trader->>Trader: reverse filled leg
    else both fail
        Trader->>Risk: record execution failure
    end
```

The cost model incorporates configured taker fees, expected holding-period funding, order-book depth or best-bid-and-offer spread slippage, contract multipliers, quantity precision, and minimum notionals.

## Position lifecycle

Positions store both legs, including venue, side, quantity, notional, order identifier, and actual fill information. Live startup queries existing exchange positions and loads matched leader-follower exposure into the monitoring path.

Protection layers overlap:

- incoming ticks trigger position checks;
- a periodic sweep re-evaluates positions from cached prices;
- reconnect callbacks request a position scan;
- a timer enforces the maximum holding period.

The current exit order is unfavorable funding near settlement, timeout, take-profit, and stop-loss. Close orders are submitted concurrently. A close-leg failure is logged as a critical event because residual exposure may remain.

## Risk and capital state

Risk is evaluated across four connected layers rather than as a single entry filter:

1. **Trade economics.** `CostModel` converts the signal into a common two-leg quantity and estimates round-trip taker fees, holding-period funding, and slippage. It uses order-book depth when available and a best-bid-and-offer spread fallback otherwise, then applies contract multipliers, quantity steps, venue minimums, and the configured net-return threshold.
2. **Admission and concentration.** `RiskManager`, `PositionManager`, and `Trader` check venue balances, order frequency, position counts by pair and symbol, symbol notional relative to equity, daily-loss state, post-stop-loss suspension, execution-failure cooldown, and whether rebalancing has paused new entries.
3. **Execution and position recovery.** Opening and closing legs are submitted concurrently. A one-leg opening fill triggers an immediate reversal attempt. Existing matched positions can be loaded during live startup, and close-leg failures are reported as critical events because residual exposure may remain.
4. **Capital distribution.** Balance refreshes support venue-level entry eligibility. `RebalanceSupervisor` checks cash concentration across exchanges, pauses entries while rebalancing, evaluates transfer routes and fee limits, and waits for transfers or deposits before resuming normal entry checks.

The daily balance halt stops new entries and lets open positions reach the flat state before the trader exits. A stop-loss sets a separate UTC-day entry suspension. Failure cooldowns are temporary and can expire without resetting the daily controls.

## Persistence and observability

| Artifact | Purpose |
|---|---|
| `logs/params.json` | Configuration snapshot |
| `logs/signals.csv` | Candidate market events |
| `logs/spread_snapshots.csv` | Spread and baseline observations |
| `logs/trades.csv` | Position lifecycle records |
| `logs/main.log`, `logs/tracker.log` | Operational events and failures |

Generated records are excluded from version control. File-based storage is transparent for a single-process research system but does not provide transactional persistence or multi-process coordination.

## Design boundaries

| Choice | Operational benefit | Boundary |
|---|---|---|
| One event loop | Direct in-memory coordination | CPU-heavy work can delay other tasks |
| Rolling median | Resistance to isolated spread spikes | Fixed windows adapt imperfectly to regime shifts |
| Direct callback | No queue backlog on the normal path | Callback work must remain short |
| Concurrent two-leg submission | Avoids deliberate sequential delay | Cannot create cross-venue atomicity |
| File records | Easy inspection and low dependency count | No transactional recovery guarantee |
| Common client interface | Central policy with localized venue logic | Exchange schema changes require maintenance |

Possible research extensions include alternative robust baselines, volatility-conditioned thresholds, replay adapters that emit `Tick` objects, calibrated slippage models, and structured metrics exporters. They are not implemented features.
