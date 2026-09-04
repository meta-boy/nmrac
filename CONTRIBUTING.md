# Contributing

Small, focused pull requests are easiest to review.

## Ground rules

- Keep it a single dependency-light bash script. `curl`, `jq` and `wireguard-tools`
  are the budget; please do not add a runtime.
- `bash -n nmrac` must pass, and `shellcheck nmrac` should stay quiet.
- The script runs as root and writes WireGuard configs. Anything touching
  `wg-quick`, key material, or file modes deserves a note in the PR describing
  what you tested.
- When a change depends on Netmaker server behaviour, cite the endpoint and, if
  you can, the handler in the Netmaker source. The existing comments do this;
  it is what makes the tool maintainable across server versions.

## Reporting a bug

Include:

- your Netmaker server version (`GET /api/server/status` reports it)
- the command you ran and the full error
- whether your Netmaker user is an admin or a regular user

## Security

Do not open a public issue for anything exploitable. Report it privately via
GitHub Security Advisories on this repository.
