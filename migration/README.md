# OLMv0 → OLMv1 Migration

A `v0` Go library plus two CLIs that migrate operators from **OLMv0** (`Subscription` /
`ClusterServiceVersion` / `CatalogSource`) to **OLMv1** (`ClusterExtension` /
`ClusterObjectSet` / `ClusterCatalog`) management — with minimal downtime and full
rollback at every step.

> Status: upstream **prototype** for
> [OCPSTRAT-2693](https://redhat.atlassian.net/browse/OCPSTRAT-2693). The library is
> versioned `v0` — breaking API changes are expected. Binary packaging, Console UI, and
> Hosted Control Planes are out of scope (see [REQUIREMENTS.md](REQUIREMENTS.md) §R9).

## Why

With OLMv1 reaching general availability, cluster administrators need a supported path to
move operators currently managed by OLMv0 onto OLMv1. Without tooling, adoption means
manually decommissioning and reinstalling each operator — risky, error-prone, and likely
to cause downtime or data loss for workloads that depend on those operators. This tooling
automates the transition, reduces operational risk, and accelerates OLMv1 adoption.

## The two CLIs

Catalogs are migrated **first**, then operators — the operator tool refuses to migrate an
operator whose package is not served by a `ClusterCatalog`.

### `migrate-catalogs-v0-to-v1`
Migrates OLMv0 `CatalogSource` resources to OLMv1 `ClusterCatalog` resources.

| Flag | Purpose |
|---|---|
| `--dry-run` | Print what would be created without modifying the cluster |

### `migrate-operators-v0-to-v1`
Migrates OLMv0 `Subscription`/`CSV` installations to OLMv1 `ClusterExtension`/`ClusterObjectSet`.

| Subcommand | Purpose |
|---|---|
| `check` | Report readiness & compatibility for one operator (no changes) |
| `gather` | Show everything that would be migrated (dry-run for one operator) |
| *(default)* | Migrate a single named operator |
| `all` | Scan the cluster, classify every operator, migrate all Eligible ones |
| `rollback <ce-name>` | Restore an operator to OLMv0 management |
| `cleanup <ce-name>` | Finish a partial migration (Conflict state) |

The tool is **non-interactive**: there are no prompts. `gather` / `--dry-run` are the
preview mechanisms, and every risk override is an explicit `--acknowledge-*` flag (see
[REQUIREMENTS.md](REQUIREMENTS.md) §R3).

## Operator states

`migrate-operators-v0-to-v1 all` classifies every OLMv0 `Subscription` into one of four states:

| State | Meaning | Action |
|---|---|---|
| **Eligible** | Passes all readiness & compatibility checks; package available in a `ClusterCatalog`; no `ClusterExtension` yet | Migrate |
| **Ineligible** | Fails one or more checks | Report the specific reason; skip (override with the matching `--acknowledge-*` flag) |
| **AlreadyMigrated** | No `Subscription`, but a `ClusterExtension` annotated `migrated-from-subscription` exists | Report as done; skip |
| **Conflict** | Both a `Subscription` **and** an annotated `ClusterExtension` exist — indicates a failed cleanup | Warn prominently; block; resolve with `cleanup` or `rollback` |

## High-level flow

```
migrate-catalogs-v0-to-v1        (prerequisite: CatalogSource -> ClusterCatalog)
        │
        ▼
scan ──► check ──► gather ──► migrate ──────────────────────────────► cleanup
 (all)   (compat)  (preview)   │                                        (delete Sub/CSV,
                               │                                         OperatorGroup if last)
                               ├─ profile Subscription/CSV/InstallPlan
                               ├─ resolve target ClusterCatalog (by image)
                               ├─ back up Subscription spec (CE annotation)
                               ├─ collect owned resources (5 sources, dedup)
                               ├─ create ClusterObjectSet (SecretPacker, IfNoController)
                               │     └─ wait for COS controller: Succeeded=True
                               └─ create ClusterExtension (adopts the COS)
                               ▲
                               └── rollback: delete CE+COS (orphan cascade), restore Subscription
```

Workloads keep running throughout when the install namespace is unchanged — only the
management plane changes (**close-to-zero downtime**). When a namespace change is required,
some downtime is unavoidable while the deployment restarts in the new namespace, but it is
minimized.

## Architecture at a glance

- **Standalone `v0` module**, two binaries, prow + GitHub Actions CI.
- Targets OLMv1's `ClusterObjectSet` (COS) — the current name for the revision object
  (formerly `ClusterExtensionRevision`).
- The migration tool **creates** the COS and **waits** for the COS controller to set
  `Succeeded=True`; it never writes status on OLMv1 APIs. It then creates the
  `ClusterExtension`, which adopts the COS via owner labels.
- Collected objects are stored via boxcutter's **SecretPacker** (not inline) to support
  large bundles, with `CollisionProtection: IfNoController` so OLMv1 can adopt existing
  resources (including CRDs) without conflict.

## Prototype lineage

- [joelanford/operator-controller `0-to-1`](https://github.com/joelanford/operator-controller/tree/0-to-1) — single-file proof of concept
- [perdasilva/operator-controller `migration`](https://github.com/perdasilva/operator-controller/tree/migration) — the base this work is ported from

## Documents

| Doc | Contents |
|---|---|
| [REQUIREMENTS.md](REQUIREMENTS.md) | Functional/architectural requirements, eligibility rules, per-field handling for every Subscription, OperatorGroup, and CatalogSource field, and the ClusterExtension/ClusterCatalog target mappings |
| [PLAN.md](PLAN.md) | The seven implementation phases |
| [VALIDATION.md](VALIDATION.md) | Verifiable acceptance criteria and a requirement-traceability table |
