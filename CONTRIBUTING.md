# Contributing to Constellation

Thanks for your interest in the Constellation identity protocol. This document covers local development, testing, and PR workflow.

## Development setup

```sh
git clone https://github.com/myrgic/constellation.git
cd constellation
go mod download
go build ./...
```

Requirements: Go 1.24+.

## Running tests

```sh
go vet ./...
golangci-lint run
go test ./... -race
```

CI runs lint + race-enabled tests on every PR (`.github/workflows/ci.yml`).

## Project layout

The repo is flat, not split into `internal/`/`pkg/`. Everything in package `constellation` lives at the root:

- `cmd/constellation/`: thin entry point for `go install`, delegates to `Run()` in the root package
- `node.go`: node lifecycle (startup, key load/generate, graceful shutdown)
- `identity.go`: ECDSA P-256 identity (generate, load, sign, verify, NodeID derivation)
- `ledger.go`: hash-chained event ledger
- `gitstore.go`: git-backed event storage (events committed as JSON files in a bare repo)
- `protocol.go`: HTTP handlers for inter-node communication (`/heartbeat`, `/peers`, `/challenge`, `/join`, `/health`, `/state`)
- `heartbeat.go`: background heartbeat ticker and peer communication
- `coherence.go`: ledger validation (hash-chain integrity, schema, temporal monotonicity)
- `constellation.go`: EMA-weighted trust scoring and identity conflict detection
- `run.go`: CLI entry point (`node`, `inject`, `tamper`, `status` subcommands)
- `*_test.go`: unit tests, colocated with the code they cover
- `test/`: shell-driven integration scenarios (happy path, join, drift, theft)
- `docs/PAPER.md`: the research-paper writeup of the protocol

## Submitting changes

1. Fork the repo and create a branch from `main`
2. Make your changes
3. Run lint + tests above
4. Update `CHANGELOG.md` under the Unreleased section if user-visible
5. Open a pull request using the org PR template

Commit messages: conventional-commits (`feat:`, `fix:`, `chore:`, etc.) preferred.

## Reporting issues

Use the org-level [Bug Report](https://github.com/myrgic/constellation/issues/new?template=bug.yml) or [Feature Request](https://github.com/myrgic/constellation/issues/new?template=feature.yml) forms.

## License

By contributing, you agree that your contributions will be licensed under the [MIT License](LICENSE).
