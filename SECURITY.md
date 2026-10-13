# Security Policy

[Project overview](README.md) · [Configuration](CONFIG_GUIDE.md)

## Sensitive information

Do not commit or publish:

- API keys, secret keys, passphrases, or authentication headers;
- withdrawal addresses or transfer destinations;
- live account identifiers, balances, positions, or transaction records;
- server IP addresses, hostnames, SSH credentials, or secret-store paths;
- Feishu or other notification webhooks;
- generated logs containing operational or account data.

Use least-privilege exchange keys. Keep withdrawal permission disabled unless an explicitly reviewed workflow requires it, and apply IP allowlists where supported.

## Live-operation boundary

Several commands can place orders, transfer balances, or request withdrawals. Read [Operations and validation](docs/OPERATIONS.md) and inspect command flags before running them. Test and demo labels do not guarantee that a script is side-effect free.

## Reporting a vulnerability

Do not open a public issue containing credentials, account data, infrastructure details, or an exploitable proof of concept. Contact the repository owner privately through the contact information on the GitHub profile and include only the minimum information needed to reproduce the issue safely.

## Credential exposure

If a secret may have been exposed, revoke it immediately, review account activity and permissions, rotate the credential, update the protected deployment environment, and remove the value from the current tree and Git history when required.
