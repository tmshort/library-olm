# Implementation Plan — OLMv0 → OLMv1 Migration

Seven phases, consolidated from an earlier 13-story breakdown. Each phase lists its goal,
key files, dependencies, and a one-line exit criterion. Requirement references (`Rn`) point
at [REQUIREMENTS.md](REQUIREMENTS.md); validation references at [VALIDATION.md](VALIDATION.md).

Base for the port: [perdasilva/operator-controller `migration`](https://github.com/perdasilva/operator-controller/tree/migration)
(`internal/operator-controller/migration/` + `hack/tools/migrate/`).

### CLI as prototype / library exercise

The CLIs in this repo (`migrate-operators-v0-to-v1`, `migrate-catalogs-v0-to-v1`) are
**example consumers of the library**, not production delivery artifacts. Their purpose is to
exercise and test the library API during the prototype phase; binary packaging and production
delivery are deferred to the downstream TP ([OCPSTRAT-2692](https://redhat.atlassian.net/browse/OCPSTRAT-2692)).

To make this intent obvious in the repo layout, CLIs live under **`migration/examples/cmd/`**
rather than `migration/cmd/`:

```
migration/
  pkg/migration/           ← the library (canonical; the real deliverable)
  pkg/catalogmigration/    ← catalog migration library
  examples/
    cmd/
      migrate-operators-v0-to-v1/   ← example CLI exercising pkg/migration
      migrate-catalogs-v0-to-v1/    ← example CLI exercising pkg/catalogmigration
```

Downstream consumers (e.g., an `oc` plugin or the Console) will import `pkg/migration`
directly and build their own CLI surface.

## Prerequisite (cross-repo, runs in parallel)

**Verify COS adoption end-to-end in `operator-controller`.** No controller changes are
expected: the COS controller reconciles any COS regardless of origin, and the CE controller
discovers a pre-created COS via `olm.operatorframework.io/owner-name` and adopts it (SSA
patch) on first reconcile. Confirm a manually pre-created COS (owner labels, `revision: 1`,
correct bundle annotations) reaches `Succeeded=True` and is adopted by a subsequently created
CE **without** producing a duplicate COS. Track boxcutter phase 2 / `ClusterObjectDeployment`
(R2.6). *Exit:* documented confirmation the flow works with no operator-controller changes.
Verified during Phase 7 E2E.

---

## Phase 1 — Repo bootstrap & prototype port
**Goal:** Stand up the `v0` module and port the prototype onto OLMv1's current APIs.
- Module `github.com/operator-framework/library-olm` (`v0.x`); GitHub Actions (build, test,
  golangci-lint, go-apidiff); prow config (tide, lgtm/approve, hold) mirroring operator-controller.
- Port to `migration/pkg/migration/` and `migration/examples/cmd/migrate-operators-v0-to-v1/`.
- Rename `ClusterExtensionRevision*` → `ClusterObjectSet*` (`clusterobjectset.go` apply-config,
  `ClusterObjectSetTypeSucceeded`) (R2.2).
- Replace inline COS objects with boxcutter **SecretPacker**; set `CollisionProtection: IfNoController` (R2.4).
- `go.mod` deps: `operator-framework/operator-controller` (`ocv1`), `operator-framework/api` (OLMv0).

**Depends on:** prerequisite (for the adoption contract). **Exit:** `go build ./...` and
`go test ./...` pass; CI green on the skeleton.

## Phase 2 — Scan & classification
**Goal:** Four-state classification with catalog availability at scan time (R1.3, R4, C8).
- `types.go`: `OperatorStatus` enum (`Eligible`/`Ineligible`/`AlreadyMigrated`/`Conflict`),
  replacing `OperatorScanResult.Eligible bool`.
- `scan.go` `ScanAllSubscriptions`: detect `Conflict` (Sub + annotated CE) and
  `AlreadyMigrated` (no Sub + annotated CE); call catalog resolution per operator.
- `migration.go`: set the `olm.operatorframework.io/migrated-from-subscription: <ns>/<name>`
  annotation on **both** the COS and the CE (replacing the prototype's `migrated-from-v0: "true"`).
  Setting it on the COS makes migrated revisions discoverable by cluster admins independently of
  the CE, and provides provenance even if the CE annotation is lost.

**Depends on:** Phase 1. **Exit:** unit tests classify one fixture operator into each state.

## Phase 3 — Compatibility checks & acknowledgment framework
**Goal:** All eligibility rules (R3) and the override mechanism (R2.5).
- `compatibility.go`: implement C1–C7 as overridable checks; add `checkNoOLMv0APIAccess`
  (C5, **excluding** `operatorconditions`) and keep the OperatorCondition-**status** check (C4, R8).
- `types.go`: `Options` gains one `bool` per flag — `AcknowledgeWatchScopeChange`,
  `AcknowledgeDependencies`, `AcknowledgeAPIServices`, `AcknowledgeOperatorCondition`,
  `AcknowledgeOLMv0APIAccess`, `AcknowledgeScopedServiceAccount`, `AcknowledgeSubscriptionConfig`,
  `AcknowledgeNamespaceDelete`, `AcknowledgeInstalled` — plus `ContinueOnError`.
- On use, record `olm.operatorframework.io/acknowledged-<flag>: "true"` on the CE.
- Collector places all objects (incl. CRDs) into the COS with `IfNoController`.

**Depends on:** Phases 1, 2. **Exit:** each check flips Ineligible→Eligible when its flag is set;
CE carries the matching annotation.

## Phase 4 — Migration & recovery commands
**Goal:** The operator CLI surface (R1.1, R1.2, R1.4, R1.5, R1.8) — verb-plus-target
(kubectl/`oc`-style); each verb takes an operator name or `--all`.
- `check <op> | --all`: readiness + compatibility + four-state classification; `--all` scans the cluster (calls `Check`/`ScanAll`).
- `convert <op> | --all`: profile → resolve catalog → back up Subscription spec to CE annotation
  → collect (5-source, dedup) → create COS (wait `Succeeded=True`) → create CE → cleanup.
  `--dry-run` previews via `Gather` (replaces the former `gather` subcommand). `--all` prints the
  four-section summary, then converts each Eligible operator; `--continue-on-error` to keep going.
- `rollback <ce-name> | --all`: require `--acknowledge-installed` when CE is `Installed=True`; delete CE
  then COS with orphan cascade (fallback: new COS revision → `Succeeded=True` → orphan delete);
  restore Subscription from the backup annotation (`startingCSV` → `installedCSV`).
- `cleanup <ce-name> | --all`: for `Conflict`; delete Subscription (orphan) + `CleanupOLMv0Resources`.
- CLI files under `migration/examples/cmd/migrate-operators-v0-to-v1/` (one file per verb).

**Depends on:** Phases 1–3. **Exit:** `check`, `convert` (single + `--all`), `rollback`, and
`cleanup` each pass their VALIDATION per-command checks on kind.

## Phase 5 — Catalog migration CLI *(parallelizable with 2–4)*
**Goal:** `migrate-catalogs-v0-to-v1` (R7).
- `migration/pkg/catalogmigration/` + `migration/examples/cmd/migrate-catalogs-v0-to-v1/`.
- List CatalogSources; skip already-migrated (matching image); create `ClusterCatalog` from
  the image and wait `Serving=True`; report per source; `--dry-run`.
- Report non-image sources (configmap/internal/address) as not migratable.

**Depends on:** Phase 1. **Exit:** N CatalogSources → N serving ClusterCatalogs; operator scan
then reports catalog-available.

## Phase 6 — Install-namespace change ⚠️ blocked
**Goal:** Support `--install-namespace` differing from the Subscription namespace (R5, R8).
- **Blocked on** [OCPSTRAT-2690](https://redhat.atlassian.net/browse/OCPSTRAT-2690) /
  [OPRUN-4505](https://redhat.atlassian.net/browse/OPRUN-4505) /
  [PR #2825](https://github.com/operator-framework/operator-controller/pull/2825) (making
  `spec.namespace` optional / COS-managed). Once it lands, the tool may omit `spec.namespace`.
- Move namespace-scoped resources to the new namespace; copy PSA (`pod-security.kubernetes.io/*`)
  and `security.openshift.io/scc.podSecurityLabelSync` labels; delete the old namespace only with
  `--acknowledge-namespace-delete`.

**Depends on:** Phases 1, 3 + PR #2825. **Exit:** resources land in the new namespace with PSA/SCC
labels copied; old namespace deleted only when acknowledged.

## Phase 7 — Testing
**Goal:** Confidence across unit and E2E (R-wide).
- Unit: ≥80% of `migration/pkg/...` with `controller-runtime/pkg/client/fake` — readiness,
  compatibility (each ack flag), scan (4 states), catalog parsing, collector (CRD/IfNoController,
  namespace rewrite, dedup), rollback/cleanup.
- E2E on kind (OLMv0 + OLMv1): all four states, each acknowledgment override, rollback, cleanup,
  catalog migration, and the COS-adoption prerequisite.

**Depends on:** Phases 2–5 (Phase 6 tests gated on that phase). **Exit:** unit coverage target met;
E2E scenarios in VALIDATION pass in CI.

---

## Dependency summary

```
prerequisite (operator-controller) ──┐  (parallel; needed before Phase 7)
                                      ▼
Phase 1 ──► Phase 2 ──► Phase 3 ──► Phase 4 ──► Phase 7
   │           └──────────────────► (Phase 3) ──► Phase 6 (blocked on PR #2825)
   └──► Phase 5 (parallel) ─────────────────────► Phase 7
```
