# Configuration and Credential Handling

[Project overview](README.md) · [Operations](docs/OPERATIONS.md) · [Security](SECURITY.md)

This guide describes local configuration for demo and live exchange access. Do not commit credentials, withdrawal addresses, account identifiers, server addresses, or generated logs.

## Local files

```text
clients/
├── api_keys_demo.example.py
├── api_keys_demo.py              # local only
├── api_keys_live.py              # local only
├── withdrawal_addresses.example.py
└── withdrawal_addresses.py       # local only

trader/
├── config.example.py
└── config.py
```

Copy the relevant example file and keep the populated version outside version control.

## Demo credentials

Create `clients/api_keys_demo.py` from the example and add demo/testnet keys for the venues you intend to use. Exchange test environments differ in instrument coverage and API behavior, so validate each venue separately before running the combined workflow.

## Live credentials

Live keys should use the minimum permissions required:

- read permission for account and market checks;
- trading permission only when live execution is intended;
- withdrawals disabled unless an explicitly reviewed rebalancing workflow requires them;
- IP allowlisting when supported.

Do not place secret values directly in documentation, shell commands, screenshots, issue reports, or committed configuration files. When deploying, use a protected secret store or files readable only by the service account.

## Mode selection

The default checked-in configuration is demo/testnet mode. Live execution can be selected with:

```bash
python main.py --live
```

Live mode loads live credentials, runs a read-only preflight, and requires an exact `YES` confirmation before continuing. Review [Operations and validation](docs/OPERATIONS.md) first.

## Validation

Start with checks that do not place orders:

```bash
python -m tracker
python -m test_demo.run_all --skip-orders
python -m test_live.run_all
```

Authenticated read commands can still expose sensitive account information in terminal output or logs.

## Deployment placeholders

Documentation and committed scripts should use placeholders such as:

```text
<SERVER_HOST>
<SSH_USER>
<PROJECT_PATH>
<SECRET_STORE_PATH>
```

Never commit a real server IP, hostname, home-directory secret path, API key, passphrase, webhook, or withdrawal destination.

## Credential incident response

If a credential may have been exposed:

1. revoke it at the exchange immediately;
2. review account activity, orders, transfers, and withdrawal settings;
3. create a replacement with minimum permissions;
4. update the protected deployment secret;
5. remove the exposed value from the current tree and, when necessary, Git history.
