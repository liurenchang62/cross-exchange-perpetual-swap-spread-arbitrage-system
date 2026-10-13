# Execution, Positions, and Risk

The trader converts a detected market event into a constrained two-leg execution decision. Its cost model includes round-trip taker fees, expected funding, order-book depth or BBO-derived slippage, contract multipliers, quantity precision, minimum order sizes, and minimum notionals. Both legs use a common base-asset quantity, and an entry is rejected when its expected net return does not clear the configured threshold.

Before submission, the trader checks available balances, pair and symbol position limits, symbol-notional concentration, per-exchange order frequency, daily-loss state, stop-loss suspension, failure cooldown, and rebalance state. Opening orders are sent concurrently. If only one leg succeeds, the filled leg is submitted for immediate reversal; if both succeed, the actual fills are registered and monitored as one cross-exchange position.

Position checks use current prices and recorded fills. Exit reasons include unfavorable funding near settlement, maximum holding time, take-profit, and stop-loss. Close legs are also submitted concurrently, and a failed close leg is logged as a critical residual-exposure event. Live startup can load matched positions already present at the exchanges into the monitoring path.

See [Architecture](../docs/ARCHITECTURE.md).
