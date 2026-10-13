# Command Reference

[Project overview](README.md) · [Operations](docs/OPERATIONS.md)

Read [Operations and validation](docs/OPERATIONS.md) before running authenticated or capital-moving commands.

## Public market data

```bash
python -m tracker
python -m tools.check_exchange_reachability
```

## Demo and testnet

```bash
python -m test_demo.run_all --skip-orders
python -m test_demo.run_all
```

The second command may place testnet orders or perform supported demo transfers.

## Live read-only checks

```bash
python -m test_live.run_all
python -m test_live.test_connectivity
python -m test_live.test_account
python -m test_live.test_orderbook
python -m test_live.test_funding
```

These commands load live credentials. Review terminal output and local logs for sensitive account data before sharing them.

## Live execution

```bash
python main.py --live
```

The command performs a read-only preflight and asks for an exact `YES` confirmation. It can transfer balances, submit orders, incur fees, and create market exposure.

## Rebalancing

```bash
python -m rebalance.check_balances
python -m rebalance.check_fees
python -m rebalance.run
python -m rebalance.run --execute
```

The final command can move assets or request withdrawals. Verify addresses, networks, fees, limits, and permissions independently.

## Inspection and recovery

```bash
python -m tools.check_account
python -m tools.cashflow
python -m tools.find_balance
python -m tools.close_naked_positions
```

Recovery tools may submit reduce-only orders. Inspect their arguments and confirmation behavior before use.
