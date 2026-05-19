# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build & Test Commands

All commands are run from the repository root (parent of this directory):

```bash
# Build the binary
make build
# or directly:
CGO_ENABLED=0 go build -o build/external-dns -v -ldflags "-X sigs.k8s.io/external-dns/pkg/apis/externaldns.Version=$(git describe --tags --always --dirty --match 'v*') -w -s" .

# Run all tests with race detector
make test
# or: go test -race ./...

# Run tests for this package only
go test -race ./controller/...

# Run a single test
go test -race ./controller/... -run TestRunOnce

# Run tests with coverage
make cover

# Lint (requires golangci-lint installed via scripts/install-tools.sh --golangci)
make go-lint

# Format code
gofmt -l -s -w .
```

## Architecture

The `controller` package is the **orchestration layer** of external-dns. It does not contain DNS logic itself — it coordinates three interfaces (source, registry, provider) through a reconciliation loop.

### Core reconciliation flow (`controller.go`)

`Controller.RunOnce` executes one full sync cycle:
1. Fetch current DNS state from the **Registry** (`Registry.Records`)
2. Fetch desired DNS state from the **Source** (`Source.Endpoints`)
3. Call `Registry.AdjustEndpoints` to normalize desired endpoints to provider constraints
4. Build a `plan.Plan` combining both states with the configured `Policy` and `DomainFilter`
5. Call `plan.Calculate()` to diff current vs. desired
6. If changes exist, apply them via `Registry.ApplyChanges`; emit change events via `EventEmitter`

`Controller.Run` loops `RunOnce` on a 1-second ticker, gating execution with `ShouldRunOnce`. Errors that implement `provider.SoftError` are counted and logged without fatally exiting — all other errors from `RunOnce` call `log.Fatalf`.

### Throttling and event batching (`controller.go`)

`ScheduleRunOnce` / `ShouldRunOnce` implement a batching window:
- On a Kubernetes watch event, `ScheduleRunOnce` is called; it schedules the next run no earlier than `lastRunAt + MinEventSyncInterval`, but no later than `now + 5s`
- `ShouldRunOnce` gates execution and resets `nextRunAt` to `now + Interval` once it fires
- A `sync.Mutex` (`runAtMutex`) protects both `nextRunAt` and `lastRunAt`

### Application startup (`execute.go`)

`Execute()` is the top-level entry point (called from `main`). It:
1. Parses flags into `externaldns.Config`
2. Starts Prometheus metrics HTTP server (`/healthz`, `/metrics`) on a goroutine
3. Installs a SIGTERM handler that cancels the root context
4. Calls `buildSource`, `buildProvider`, `buildController` in sequence
5. Optionally runs in `--once` mode (single `RunOnce` then exit) or webhook server mode
6. Optionally registers a Kubernetes watch event handler that calls `ScheduleRunOnce`

`buildProvider` is a large `switch` over `cfg.Provider` that constructs one of ~25 provider implementations. Wraps the result in `provider.CachedProvider` if `cfg.ProviderCacheTime > 0`.

`selectRegistry` switches over `cfg.Registry` to build one of: `txt`, `dynamodb`, `noop`, or `aws-sd` registry implementations.

`buildSource` creates sources by name via `source.ByNames`, then wraps the combined source with filtering/transformation options (NAT64, target net filter, default targets, min TTL).

### Metrics (`controller.go`, `metrics.go`)

Prometheus metrics are registered in `controller.go`'s `init()` via `metrics.RegisterMetric`. The `metricsRecorder` struct in `metrics.go` is a local per-reconciliation-cycle counter (not Prometheus-facing directly) — it tallies endpoints by DNS record type and feeds the gauge vectors `registryRecords`, `sourceRecords`, and `verifiedRecords`.

Key Prometheus metrics emitted by this package:
- `registry_errors_total`, `source_errors_total` — error counters
- `source_endpoints_total`, `registry_endpoints_total` — current endpoint counts (gauges)
- `controller_last_sync_timestamp_seconds` — set only on successful apply
- `controller_last_reconcile_timestamp_seconds` — set at the start of every RunOnce
- `controller_no_op_runs_total` — incremented when plan has no changes
- `controller_consecutive_soft_errors` — gauge tracking current soft error streak
- `registry_records{record_type}`, `source_records{record_type}`, `controller_verified_records{record_type}` — per-type gauge vectors

### Events (`events.go`)

`emitChangeEvent` translates `plan.Changes` (Create/UpdateNew/Delete slices) into `events.Event` objects dispatched through the `EventEmitter` interface. This is a no-op when `EventEmitter` is nil.

### Testing patterns

- Tests use `github.com/stretchr/testify` (`assert`, `require`)
- `controller_test.go` uses hand-rolled `mockProvider` / `filteredMockProvider` structs; `execute_test.go` uses `fakeprovider.MockProvider` and `testutils.MockSource`
- Subprocess testing pattern: `TestHelperProcess` + `runExecuteSubprocess` run `Execute()` in a child process to test fatal-exit paths without killing the test binary
- `SoftError` resilience is tested via `toggleRegistry` which fails a fixed number of times before succeeding

### `gochecknoinits` exemption

The linter exempts `controller.go` from the `gochecknoinits` rule (`.golangci.yml` line 99) because this file uses `init()` to register Prometheus metrics.
