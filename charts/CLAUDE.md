# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Module Context

This module is built and run as part of the parent project. See the root `CLAUDE.md` for repository-wide build and run instructions.

This directory contains the Helm chart for deploying ExternalDNS to Kubernetes. It is a standalone Helm chart — not a Go module — and has its own independent lifecycle: versioned separately from the Go binary (`Chart.version` vs `appVersion`), linted and tested with Helm tooling, and published to the Helm repository at `https://kubernetes-sigs.github.io/external-dns/`.

## Commands

All chart commands are driven through `scripts/helm-tools.sh` from the repo root, or via `make` targets:

```bash
# Install required Helm plugins (helm-values-schema-json, helm-unittest, helm-docs)
scripts/helm-tools.sh --install

# Run all Helm unit tests
make helm-test
# equivalent: helm unittest -f 'tests/*_test.yaml' --color charts/external-dns

# Run a single test suite file
helm unittest -f 'tests/deployment-flags_test.yaml' --color charts/external-dns

# Lint the chart (helm lint + chart-testing)
scripts/helm-tools.sh --lint
# equivalent: helm lint charts/external-dns --debug --strict --values values.yaml --values ci/ci-values.yaml
#             ct lint --target-branch=master --check-version-increment=false

# Regenerate values.schema.json from schema/values.yaml + values.yaml annotations
make helm-lint
# schema step only: scripts/helm-tools.sh --schema  (run from repo root; script cd's internally)

# Validate schema has not drifted from source (CI gate)
scripts/helm-tools.sh --diff

# Regenerate README.md from README.md.gotmpl using helm-docs
scripts/helm-tools.sh --docs
# equivalent: cd charts/external-dns && helm-docs

# Render templates to _scratch/ for inspection
make helm-template
# equivalent: helm template external-dns charts/external-dns --output-dir _scratch -n kube-system
```

> `scripts/helm-tools.sh --schema` and `--docs` internally `cd charts/external-dns` before running — always invoke them from the repo root.
> `values.schema.json` is generated, never edit it manually. Edit `schema/values.yaml` or the `# @schema` annotations in `values.yaml`, then re-run `--schema`.
> `README.md` is generated from `README.md.gotmpl` via `helm-docs`. Never edit `README.md` directly.
> The CRD at `crds/dnsendpoints.externaldns.k8s.io.yaml` is also generated — it is copied from `config/crd/standard/` by `make crd` (a root-level target).

## Architecture

### Chart Structure

```
charts/
├── OWNERS
└── external-dns/
    ├── Chart.yaml              # chart metadata, chart version & appVersion
    ├── values.yaml             # default values with @schema annotations
    ├── values.schema.json      # generated JSON Schema (draft-07) for values validation
    ├── schema/values.yaml      # supplemental schema constraints merged with values.yaml
    ├── .schema.yaml            # helm-values-schema-json plugin config
    ├── README.md               # generated docs
    ├── README.md.gotmpl        # helm-docs template
    ├── .helmignore
    ├── CHANGELOG.md
    ├── crds/
    │   └── dnsendpoints.externaldns.k8s.io.yaml   # generated DNSEndpoint CRD (v1alpha1)
    ├── ci/
    │   └── ci-values.yaml      # values used during CI linting
    ├── templates/
    │   ├── _helpers.tpl        # named templates (fullname, labels, providerName, etc.)
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   ├── serviceaccount.yaml
    │   ├── clusterrole.yaml    # dynamic RBAC: Role or ClusterRole depending on .Values.namespaced
    │   ├── clusterrolebinding.yaml
    │   ├── secret.yaml         # only rendered when secretConfiguration.enabled=true (deprecated)
    │   ├── servicemonitor.yaml # only rendered when serviceMonitor.enabled=true
    │   └── NOTES.txt
    └── tests/
        ├── common-metadata_test.yaml
        ├── deployment-config_test.yaml
        ├── deployment-flags_test.yaml
        ├── deployment-scheduling_test.yaml
        ├── json-schema_test.yaml
        ├── notes-deprecation_test.yaml
        ├── notes_test.yaml
        ├── rbac_test.yaml
        ├── secret_test.yaml
        └── serviceaccount_test.yaml
```

### Key Architectural Decisions

**Provider configuration:** The `provider` value accepts either a plain string (legacy, deprecated) or an object with `provider.name`. The `external-dns.providerName` helper in `_helpers.tpl` normalises both forms. When `provider.name: webhook`, a second sidecar container named `webhook` is injected into the Deployment alongside `external-dns`, communicating on port `8080` (`http-webhook`). This is the only provider with first-class Helm support; all others pass configuration via `extraArgs`.

**RBAC generation is source-driven:** `clusterrole.yaml` generates RBAC rules conditionally based on which sources are listed in `.Values.sources`. Each source type requires specific Kubernetes API groups and resources. Adding a source to `values.yaml` automatically adds the corresponding RBAC rules; removing it removes them. When `namespaced: true`, a `Role`/`RoleBinding` is created instead of `ClusterRole`/`ClusterRoleBinding`, with a separate `ClusterRole` emitted only for Gateway API namespace listing if needed.

**Schema is two-layer:** `values.schema.json` is generated by merging `schema/values.yaml` (structural constraints not expressible inline) with the `# @schema` annotations embedded directly in `values.yaml`. The `.schema.yaml` config file tells the `helm-values-schema-json` plugin to read both files and output draft-07 JSON Schema. CI enforces that the committed `values.schema.json` matches what would be generated (`--diff` gate).

**Affinity/topology auto-injection:** The `_helpers.tpl` defines `external-dns.labelSelector`, which is used in `deployment.yaml` to automatically inject the pod selector `matchLabels` into `podAffinity`, `podAntiAffinity`, and `topologySpreadConstraints` entries that omit an explicit `labelSelector`. This means users can specify spread constraints without repeating selectors.

**`txtPrefix` and `txtSuffix` are mutually exclusive:** The deployment template calls `fail` at render time if both are non-empty, making this a hard chart-level constraint rather than a runtime error.

**`secretConfiguration` is deprecated:** The `secretConfiguration.enabled` path creates a `Secret` and mounts it into the container. New deployments should use `extraArgs` and environment variables (`env`) instead. The deprecation is tested in `notes-deprecation_test.yaml`.

**`enabled` value is a no-op stub:** Present solely for sub-charting compatibility (parent chart can conditionally enable/disable this chart). It has no effect on rendered templates.

## Section 3 — Integrations & Network Topology

### A. Core Infrastructure

| System | Protocol | Role |
|---|---|---|
| Kubernetes API | HTTPS/REST (in-cluster) | Chart deploys a `Deployment` that reads Services, Ingresses, and other source resources from the API; source-specific permissions are generated dynamically in `clusterrole.yaml` |
| DNS providers (external) | Provider-specific (CLI flags via `--provider`) | ExternalDNS binary writes DNS records to the configured provider at each reconciliation interval |

### B. Downstream Services — APIs We Consume

No direct external service integrations. All downstream calls are routed through intra-repo service modules.

The ExternalDNS binary (deployed by this chart) calls DNS provider APIs at runtime, but those integrations are defined in the parent Go module (`provider/`), not in this Helm chart. This chart only configures which provider is used via `provider.name` and `extraArgs`.

### C. Exposed Interfaces — APIs We Provide

#### Deployment HTTP endpoint (port `7979`)

The `external-dns` container exposes a single HTTP port used for probes and metrics. The `Service` resource exposes it as `ClusterIP`.

| Path | Description |
|---|---|
| `/healthz` | Liveness and readiness probe endpoint |
| `/metrics` | Prometheus metrics (scraped by `ServiceMonitor` when `serviceMonitor.enabled=true`) |

#### Webhook sidecar HTTP endpoint (port `8080`, only when `provider.name: webhook`)

The `webhook` sidecar container exposes its own port for probe and metrics scraping.

| Path | Description |
|---|---|
| `/healthz` | Liveness and readiness probe endpoint for the webhook container |
| `/metrics` | Prometheus metrics for the webhook container (scraped by `ServiceMonitor`) |

## Section 4 — Important Files Reference

| File Path | Category | Purpose |
|---|---|---|
| `external-dns/Chart.yaml` | `build` | Chart name, version (`1.19.0`), and appVersion (`0.19.0`); version must be bumped on every chart change |
| `external-dns/values.yaml` | `config` | Default values with inline `# @schema` annotations that feed schema generation |
| `external-dns/values.schema.json` | `schema` | Generated JSON Schema (draft-07) for Helm values validation; never edit manually |
| `external-dns/schema/values.yaml` | `schema` | Supplemental schema constraints merged with `values.yaml` by the schema plugin |
| `external-dns/.schema.yaml` | `build` | Config for `helm-values-schema-json` plugin: specifies input files, draft version, and output path |
| `external-dns/README.md.gotmpl` | `spec` | Source template for `README.md`; edit this, not `README.md` |
| `external-dns/crds/dnsendpoints.externaldns.k8s.io.yaml` | `schema` | Generated CRD for `DNSEndpoint` (v1alpha1); source of truth is `apis/` in repo root |
| `external-dns/ci/ci-values.yaml` | `test-config` | Overrides applied during `helm lint` CI run to exercise non-default code paths |
| `external-dns/templates/_helpers.tpl` | `entrypoint` | All named templates: fullname, labels, selectorLabels, providerName, webhookImage, labelSelector, hasGatewaySources |
| `external-dns/templates/deployment.yaml` | `entrypoint` | Core workload; builds the `args` list from values; injects webhook sidecar when provider is `webhook` |
| `external-dns/templates/clusterrole.yaml` | `spec` | Dynamic RBAC: rules are generated per-source; emits `Role` or `ClusterRole` based on `namespaced` flag |
| `external-dns/tests/deployment-flags_test.yaml` | `test-config` | Unit tests for CLI flag construction and `extraArgs` handling (slice vs map forms) |
| `external-dns/tests/json-schema_test.yaml` | `test-config` | Unit tests for schema validation edge cases (legacy string provider, null values) |

## Section 5 — Maintenance

| Field | Value |
|---|---|
| Last Updated | 2026-05-19 |
| Project Version | 1.19.0 |
| Maintained By | Unknown |
