# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build & Test

This module is built and run as part of the parent project. See the root `CLAUDE.md` for repository-wide build and run instructions.

Run tests scoped to this package from the repo root:

```bash
# All tests in this module
go test -race ./source/...

# Single test file or function
go test -race ./source/ -run TestIngressSuite
go test -race ./source/wrappers/ -run TestDedupSource
```

Tests use `github.com/stretchr/testify` (assert/require/suite) and `k8s.io/client-go/kubernetes/fake` for fake Kubernetes clients. There are no integration tests; all tests are purely in-process.

## Architecture

### Core interface

`source.go` defines the `Source` interface that every source implementation must satisfy:

```go
type Source interface {
    Endpoints(ctx context.Context) ([]*endpoint.Endpoint, error)
    AddEventHandler(context.Context, func())
}
```

`eventHandlerFunc` in `source.go` is a bare `func()` that satisfies `cache.ResourceEventHandler` by calling itself on any add/update/delete event.

### Factory & configuration (`store.go`)

`BuildWithConfig(ctx, source, p, cfg)` is the central registry mapping source type name strings (defined in `types/types.go`) to constructors. Adding a new source type requires: a new constant in `types/types.go`, a `buildXXXSource` function in `store.go`, and a `case` in the `BuildWithConfig` switch.

`Config` (in `store.go`) is the single shared config object passed to all source constructors. It is built from the global `externaldns.Config` via `NewSourceConfig()`.

`SingletonClientGenerator` lazily initialises each Kubernetes client type exactly once using `sync.Once`. All source constructors receive a `ClientGenerator` interface, not concrete client types, enabling test injection.

### Source implementations (root package)

Each file (`ingress.go`, `service.go`, `node.go`, `pod.go`, `gateway_httproute.go`, etc.) contains a single private struct implementing `Source`. The struct embeds a shared informer for its resource type and filters by annotation/label selector at query time.

**Special cases noted in comments:**
- `buildSkipperRouteGroupSource` does not use `ClientGenerator`; it calls `GetRestConfig` directly to extract a bearer token.
- `buildGlooProxySource` does not forward `ctx` to its constructor (legacy design).

### Subpackages

| Package | Role |
|---|---|
| `types/` | String constants for all source type names (e.g., `types.Ingress = "ingress"`) |
| `annotations/` | Annotation key variables (all prefixed, rebuilt by `SetAnnotationPrefix`), TTL parsing, hostname/target/provider-specific extraction |
| `fqdn/` | Go `text/template` wrapper; `ParseTemplate` / `ExecTemplate` compile and execute FQDN templates against `metav1.Object` values. Template functions: `contains`, `trimPrefix`, `trimSuffix`, `trim`, `toLower`, `replace`, `isIPv6`, `isIPv4` |
| `informers/` | `WaitForCacheSync` / `WaitForDynamicCacheSync` (60 s timeout), generic `IndexerWithOptions[T]` indexer with `IndexWithSelectors` index name, `DefaultEventHandler` for dynamic informers |
| `wrappers/` | Decorator sources composed by `WrapSources` |

### Wrapper pipeline (`wrappers/`)

`WrapSources` always applies wrappers in this fixed order:

```
MultiSource → DedupSource → [NAT64Source] → [TargetFilterSource] → PostProcessor
```

- `MultiSource`: fans out `Endpoints()` across child sources; applies `defaultTargets` if the source returns no targets (or always if `forceDefaultTargets` is set).
- `DedupSource`: deduplicates on the composite key `RecordType/DNSName/SetIdentifier/Targets`.
- `NAT64Source`: synthesises A records from AAAA records for specified NAT64 networks (optional).
- `TargetFilterSource`: drops endpoints whose targets fall outside allowed CIDR ranges (optional).
- `PostProcessor`: enforces a minimum TTL via `endpoint.WithMinTTL`.

Each applied wrapper registers its name in `wrappers.Config.sourceWrappers` for instrumentation.

## Critical invariant

`annotations.SetAnnotationPrefix(prefix)` **must be called before any source is initialised**. All annotation key variables (`HostnameKey`, `TtlKey`, `TargetKey`, etc.) are package-level `var`s initialised to empty strings; they only gain their values when `SetAnnotationPrefix` is called. Calling any annotation lookup function before this results in matching the empty string.

## Important Files Reference

| File Path | Category | Purpose |
|---|---|---|
| `source.go` | entrypoint | `Source` interface definition; `eventHandlerFunc` adapter; annotation-filter helpers |
| `store.go` | entrypoint | `Config` struct, `ClientGenerator` interface, `SingletonClientGenerator`, `BuildWithConfig` factory, Kubernetes client constructors |
| `types/types.go` | build | String constants for all source type names; used as keys in `BuildWithConfig` |
| `annotations/annotations.go` | entrypoint | Annotation key variables and `SetAnnotationPrefix` |
| `annotations/processors.go` | entrypoint | TTL parsing, hostname/target extraction, `ProviderSpecificAnnotations` |
| `annotations/provider_specific.go` | entrypoint | Maps provider-prefixed annotations (aws-, scw-, webhook-, coredns-, cloudflare-) to `ProviderSpecificProperty` |
| `fqdn/fqdn.go` | entrypoint | FQDN template parsing and execution |
| `informers/informers.go` | entrypoint | Cache sync helpers with 60 s timeout |
| `informers/indexers.go` | entrypoint | Generic `IndexerWithOptions[T]` and `GetByKey[T]` with annotation+label filtering |
| `informers/handlers.go` | entrypoint | `DefaultEventHandler` for dynamic informers |
| `wrappers/types.go` | entrypoint | `WrapSources` pipeline; `Config` and functional options for wrapper configuration |

## Maintenance

| Field | Value |
|---|---|
| Last Updated | 2026-05-19 |
| Project Version | v0.15.1-0.20250516124452-b3273845de59 (from `go.mod` module path + git describe) |
| Maintained By | Unknown |
