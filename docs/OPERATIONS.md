# Operations and Validation

[Project overview](../README.md) · [Architecture](ARCHITECTURE.md) · [Configuration](../CONFIG_GUIDE.md)

This runbook separates public observation, demo interaction, authenticated reads, and live capital operations. A filename beginning with `test_` does not imply that the command has no financial side effects.

## Operating modes

| Mode | Credentials | Possible effects | Entry point |
|---|---|---|---|
| Tracker only | None | Network requests and local logs | `python -m tracker` |
| Demo/testnet | Demo keys | Testnet orders and transfers | `python main.py` |
| Live read-only suite | Live keys | Authenticated account and market reads | `python -m test_live.run_all` |
| Live trading | Live keys | Orders, fills, fees, and market exposure | `python main.py --live` |
| Manual rebalance | Live keys and withdrawal setup | Transfers or withdrawals with `--execute` | `python -m rebalance.run` |

Exchange-facing checks depend on API availability, account permissions, clock synchronization, network region, and IP allowlists.

## Setup checklist

1. Use Python 3.10 or newer in an isolated environment.
2. Install dependencies with `pip install -r requirements.txt`.
3. Create local credential files from the templates described in [CONFIG_GUIDE.md](../CONFIG_GUIDE.md).
4. Confirm that credential, address, environment, and generated log files are ignored by Git.
5. Use least-privilege API keys and keep withdrawal permission disabled unless explicitly needed.
6. Start with public-data and demo checks.

## Recommended validation order

```bash
python -m tracker
python -m test_demo.run_all --skip-orders
python -m test_demo.run_all
python -m test_live.run_all
```

The first command uses public market data only. The demo suite can interact with testnet accounts. The live suite loads live credentials but defaults to its read-only group; review its flags before enabling order or transfer checks.

## Live-mode guard

`python main.py --live` performs a read-only preflight before asking for an exact `YES` confirmation. Continuing may:

- redeem supported earn balances;
- transfer spot balances into futures accounts;
- load existing matched positions;
- submit opening and closing orders;
- incur fees, funding payments, and market exposure.

A failed preflight can still be overridden by an explicit confirmation. Investigate failures before continuing rather than treating the prompt as a routine step.

## Execution failure handling

Cross-exchange orders are not atomic.

- If both opening legs fill, the position is registered for monitoring.
- If one opening leg fills, the system attempts to reverse it immediately.
- If both opening legs fail, the failure is recorded and may contribute to cooldown.
- If a closing leg fails, the event is logged at critical severity and may leave residual exposure.

After any interrupted run, inspect balances and positions before restarting. The repository includes account inspection and residual-position recovery utilities under `tools/`.

## Rebalancing

The rebalancing package evaluates cash distribution across venues and can construct transfer routes. Commands using `--execute` can move assets or request withdrawals. Run planning and fee checks first, verify destination addresses and networks independently, and keep withdrawal permissions disabled when rebalancing is not required.

## Records

Runtime records are written under `logs/` and may include signals, spread snapshots, positions, balances, and operational errors. Generated logs should remain local because they may reveal account activity or infrastructure details.

## Validation interpretation

The repository contains integration checks, not a deterministic research benchmark. A passing run confirms that selected interfaces and account operations worked in that environment at that time. It does not establish profitability, latency guarantees, or future API compatibility.

See [COMMANDS.md](../COMMANDS.md) for the concise command reference.
