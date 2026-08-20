# Requirements — OLMv0 → OLMv1 Migration

Synthesized from the "OLMv0 to OLMv1 Migration" RFC,
[OCPSTRAT-2693](https://redhat.atlassian.net/browse/OCPSTRAT-2693), and design review.
Requirements are labeled `R1`–`R10`; [VALIDATION.md](VALIDATION.md) traces each to a
verifiable check.

API groups referenced:
- OLMv0: `operators.coreos.com/v1alpha1` (`Subscription`, `CatalogSource`, `ClusterServiceVersion`, `InstallPlan`), `operators.coreos.com/v1` (`OperatorGroup`, `Operator`, `OperatorCondition`)
- OLMv1: `olm.operatorframework.io/v1` (`ClusterExtension`, `ClusterObjectSet`, `ClusterCatalog`)

---

## R1. Functional requirements

**R1.1 — Library API.** A Go package exposes, at minimum:
- `ScanAll(ctx)` → all OLMv0 `Subscription`s classified into the four `OperatorStatus` states (R1.3), including a per-operator catalog-availability check.
- `Check(ctx, opts)` → run all readiness & compatibility checks for one operator; no cluster mutations.
- `Gather(ctx, opts)` → collect and return everything that would be migrated; no cluster mutations (backs the CLI `convert --dry-run`).
- `Migrate(ctx, opts)` → perform the full migration (phased, with recovery); backs the CLI `convert`.
- `Rollback(ctx, opts)` → restore an operator to OLMv0 management.
- `Cleanup(ctx, opts)` → finish a partial migration (Conflict state).
- A separate catalog-migration API for `CatalogSource` → `ClusterCatalog`.

**R1.2 — CLI command surface.** Two binaries. `migrate-catalogs-v0-to-v1` (with
`--dry-run`) migrates catalogs. `migrate-operators-v0-to-v1` follows a kubectl/`oc`-style
verb-plus-target model: every action is a verb subcommand taking either an operator name
**or** `--all`. Because the target is an argument, an operator named `check`/`convert`/etc.
is never ambiguous.

| Command | Target | Library call | Mutating? |
|---|---|---|---|
| `check <operator>` / `check --all` | Subscription(s) | `Check` / `ScanAll` | no |
| `convert <operator>` / `convert --all` | Subscription(s) | `Migrate` (`Gather` when `--dry-run`) | yes (no when `--dry-run`) |
| `rollback <ce-name>` / `rollback --all` | ClusterExtension(s) | `Rollback` | yes |
| `cleanup <ce-name>` / `cleanup --all` | ClusterExtension(s) | `Cleanup` | yes |

Flags: `-n/--namespace`, `--all`, `--dry-run` (on `convert`), `--backup <directory>` (on
`convert`; writes OLM-related objects to disk before deletions — see R2.6),
`--continue-on-error` (on `convert --all`), `--acknowledge-installed` (on `rollback`), and
the eligibility-override flags (R3): `--acknowledge-watch-scope-change`,
`--acknowledge-operator-condition`, `--acknowledge-olmv0-api-access`,
`--acknowledge-scoped-serviceaccount`, `--acknowledge-not-steady-state`.
`check`/`convert` target a `Subscription` (name + `-n` namespace);
`rollback`/`cleanup` target the resulting `ClusterExtension`.

**R1.3 — Four-state classification.** Every `Subscription` is `Eligible`, `Ineligible`,
`AlreadyMigrated`, or `Conflict`, each with a specific human-readable reason.

**R1.4 — `--all` output ordering.** For `check --all` and `convert --all`, sections are
printed in order: **Conflict** (warn prominently; never auto-migrate) → **Ineligible**
(reason per operator) → **AlreadyMigrated** → **Eligible**. `convert --all` then migrates
the Eligible operators sequentially.

**R1.5 — Batch failure handling.** `convert --all` stops on the first failure by default;
pass `--continue-on-error` to log the failure and continue with the remaining operators.

**R1.6 — Non-interactive.** No prompts. `convert --dry-run` is the preview mechanism; all
overrides are explicit `--acknowledge-*` flags.

**R1.7 — Downtime.** Close-to-zero downtime when the install namespace is unchanged (only
the management plane changes; workloads keep running). When a namespace change is required,
downtime is unavoidable but must be minimized.

**R1.8 — Recovery.** Every mutating phase has a recovery path. Deletions use orphan
cascading to preserve operator workloads. The `Subscription` and `OperatorGroup` specs are
backed up as CE annotations (R2.5) so `rollback` is self-contained. A `--backup <directory>`
flag (R2.6) saves all OLM-related objects to disk for auditing and manual recovery.

---

## R2. Architectural requirements

**R2.1** — Standalone `v0` Go module (`github.com/operator-framework/library-olm`);
breaking API changes permitted while `v0`. Two binaries. Prow + GitHub Actions CI
(build, test, lint, api-diff).

**R2.2** — Target OLMv1's `ClusterObjectSet` (COS). The `perdasilva` prototype uses the
old `ClusterExtensionRevision` name throughout; all references must be updated to
`ClusterObjectSet` / `ClusterObjectSetList` / `ClusterObjectSetTypeSucceeded`.

**R2.3** — The migration tool **creates** the COS, then **waits** for the COS controller
to set `Succeeded=True`, then creates the `ClusterExtension` (which adopts the COS via
`olm.operatorframework.io/owner-kind` + `owner-name` labels). The tool **must not** write
status on any OLMv1 API. No operator-controller changes are required (verified — see
[PLAN.md](PLAN.md) prerequisite).

**R2.4** — Collected objects are stored via boxcutter's **SecretPacker** (Secret-backed,
not inline in the COS spec) to support large bundles, with
`CollisionProtection: IfNoController` so OLMv1 can adopt pre-existing resources — including
CRDs — without conflict, while still refusing to stomp resources owned by another
controller.

**R2.5 — Migration annotations.** The `migrated-from-subscription` annotation is set on **both** the `ClusterObjectSet` and the `ClusterExtension`:
- On the **COS**: `olm.operatorframework.io/migrated-from-subscription: <ns>/<name>` — ties the revision to its OLMv0 origin; makes migrated COSes discoverable independently of the CE and provides provenance even if the CE annotation is lost.
- On the **CE**: `olm.operatorframework.io/migrated-from-subscription: <ns>/<name>` — the key for `AlreadyMigrated`/`Conflict` detection during scan.

The CE also carries:
- `olm.operatorframework.io/migration-subscription-backup: <json>` — the original `Subscription` spec (package, channel, source, sourceNamespace, installPlanApproval, startingCSV), so `rollback` is self-contained and machine-independent. The `Subscription` is deleted during migration, so it cannot be used as the backup.
- `olm.operatorframework.io/migration-operatorgroup-backup: <json>` — the original `OperatorGroup` spec (selector, targetNamespaces, serviceAccountName, upgradeStrategy) from the Subscription's namespace, so `rollback` can also restore the OperatorGroup if needed.
- `olm.operatorframework.io/acknowledged-<flag>: "true"` — one per acknowledgment flag used at migration time, for audit.

**R2.6 — `--backup <directory>` flag.** On `convert`, optionally write all OLM-related objects for the operator to YAML files under the specified directory before any deletions occur. Files written:
- `subscription.yaml` — full `Subscription` object
- `operatorgroup.yaml` — full `OperatorGroup` object from the Subscription's namespace
- `clusterserviceversion.yaml` — the installed `ClusterServiceVersion`
- `installplans/` — one YAML per `InstallPlan` associated with the installed CSV

The directory is created if it does not exist. Backup does not gate migration — it is informational and aids manual recovery if the CE annotation backup is insufficient. If the directory write fails, `convert` warns and continues (the CE annotation backup is the authoritative recovery path).

**R2.7 — Boxcutter phase 2.** Upcoming boxcutter changes may introduce a
`ClusterObjectDeployment` resource. The implementation must track this and be prepared to
adapt which OLMv1 objects it creates.

---

## R3. Eligibility & compatibility rules

Blocks are **soft** (overridable by an explicit `--acknowledge-*` flag that records a CE
annotation, R2.5) or **hard** (must be remediated first — no override).

| # | Check | Ineligible when… | Override flag |
|---|---|---|---|
| C1 | AllNamespaces watch scope | OperatorGroup targets specific namespaces (Own/Single/Multi) | `--acknowledge-watch-scope-change` |
| C2 | No dependency resolution *(hard)* | CSV declares `olm.package.required` or `olm.gvk.required` | none — OLMv1 fundamentally does not resolve dependencies; migrating without them would leave the operator broken |
| C3 | No APIService definitions *(hard, temporary)* | CSV `spec.apiservicedefinitions.owned` is non-empty | none — OLMv1's registry+v1 renderer currently has **no** `apiregistration.k8s.io` generator; when [OPRUN-4723](https://redhat.atlassian.net/browse/OPRUN-4723) merges, OLMv1 will manage APIService objects natively and **C3 is removed entirely** (no override flag; operators with APIService definitions become Eligible) |
| C4 | No active OperatorCondition | `OperatorCondition.status.conditions` has entries (see R9) | `--acknowledge-operator-condition` |
| C5 | OLMv0-API RBAC without OLMv1 RBAC | The installed RBAC (from live cluster, sourced from bundle manifests or CSV) grants access to `operators.coreos.com` resources (`subscriptions`/`installplans`/`clusterserviceversions`/`catalogsources`, **excluding** `operatorconditions`) **and** does not also grant equivalent OLMv1 API access — operators updated for OLMv1 compatibility will carry both and pass | `--acknowledge-olmv0-api-access` |
| C6 | No scoped ServiceAccount | OperatorGroup `spec.serviceAccountName` is set | `--acknowledge-scoped-serviceaccount` |
| C8 | Catalog availability *(hard)* | Package not served by any `ClusterCatalog` | none — run `migrate-catalogs-v0-to-v1` first |
| C9 | Steady state | CSV not `Succeeded`, or Subscription state not `AtLatestKnown`/`UpgradePending` | `--acknowledge-not-steady-state` |

**C7 removed:** `SubscriptionConfig` representability is no longer a check. The `DeploymentConfig`
feature gate will be promoted at the same time as the Boxcutter feature gate, so all target
clusters that support migration will support `deploymentConfig`. `spec.config` maps cleanly
to `CE.spec.config.inline.deploymentConfig` (R4/R7) without a gate check.

---

## R4. Subscription field handling (`operators.coreos.com/v1alpha1`)

The on-wire JSON keys differ from the Go field names — a tool reading raw objects must key
off the JSON names shown. `spec` is a pointer and required.

| Field (JSON) | Go name | Handling |
|---|---|---|
| `spec.source` | `CatalogSource` | Locate the `CatalogSource` (with `sourceNamespace`) to obtain its image; used to resolve the target `ClusterCatalog`. Not copied to the CE directly. |
| `spec.sourceNamespace` | `CatalogSourceNamespace` | Namespace of the `CatalogSource`; used with `source`. |
| `spec.name` | `Package` | → `CE.spec.source.catalog.packageName`. Also forms the `migrated-from-subscription` annotation and the `Operator` CR name `<package>.<ns>`. |
| `spec.channel` | `Channel` | When set: → `CE.spec.source.catalog.channels` (single-element list). When **empty**: OLMv0 resolves via the catalog's `defaultChannel` — a concept OLMv1 does not carry forward. OLMv1 with no `channels` considers upgrade edges across *all* channels, which may differ from OLMv0's default. Mitigation: query the resolved `ClusterCatalog` for the package's declared default channel and set it explicitly on the CE. Warn the admin if the default channel cannot be determined. |
| `spec.startingCSV` | `StartingCSV` | Not carried to the CE. Preserved in the backup; on `rollback`, reset to `status.installedCSV`. |
| `spec.installPlanApproval` | `InstallPlanApproval` | `Manual` → pin `CE.spec.source.catalog.version` to the installed version (preserve manual upgrade control). `Automatic` (or empty, which defaults to `Automatic`) → leave version unset for channel-based auto-upgrade. |
| `spec.config` | `Config` (`SubscriptionConfig`) | **Maps directly** to `CE.spec.config.inline.deploymentConfig`. OLMv1's `DeploymentConfig` is a Go **type alias** of `SubscriptionConfig` (`internal/operator-controller/config/config.go`), and the registry+v1 renderer applies it to the operator Deployment on **every** render — installs *and* upgrades — so overrides persist. Sub-fields map 1:1 — `env`, `envFrom`, `volumes`, `volumeMounts`, `tolerations`, `resources`, `nodeSelector`, `affinity`, `annotations` — **except `selector`**, which OLMv1 omits (never honored in v0; drop it, harmless). Feature-gated by `NewOLMConfigAPI`: if the target cluster lacks the feature, the overrides can't be applied → `Ineligible` (C7), overridable with `--acknowledge-subscription-config` (migrate without them). |
| `status.installedCSV` | — | **Primary input.** The migration operates on the CSV actually installed. |
| `status.currentCSV` | — | May differ from `installedCSV` during `UpgradePending` (manual approval). Acceptable; ignore it and operate on `installedCSV`. |
| `status.state` | — | Readiness gate (C9): must be `AtLatestKnown` or `UpgradePending`. |
| `status.installPlanRef` | — | Used as a **supplementary** source for resource collection (see R5). Not the primary source; see note below. `status.install` is the deprecated equivalent — fall back if the ref is absent. |
| `status.installPlanGeneration`, `status.catalogHealth`, `status.conditions`, `status.reason`, `status.lastUpdated` | — | Controller-managed; read-only signals, not migrated as user intent. |
| annotation `olm.generated-by` | — | If present, the Subscription was generated as a **dependency** of another operator → readiness flags it (dependency operators are out of scope, R10). |

### R5. Resource collection strategy

The migration tool must collect all resources that belong to the operator's installation so
they can be placed into the `ClusterObjectSet` for OLMv1 to manage. No single OLMv0
mechanism is complete; the strategy combines multiple sources and deduplicates by
GVK+namespace+name.

**Primary source — `Operator` CR `status.components.refs`:** The `Operator` CR
(`operators.coreos.com/v1`, named `<package>.<namespace>`) is maintained by OLMv0 via
label-based selection on `operators.coreos.com/<package>.<namespace>`. It represents the
**live cluster state** of everything OLMv0 currently associates with the operator, including
resources created or managed dynamically by the CSV controller after initial install. This is
more comprehensive than the InstallPlan (which is an install-time plan only) and should be
treated as the primary source of truth.

**Supplementary sources** (add anything not already in the Operator CR refs):
1. **`olm.owner` label query** — list all resources of the eligible kinds matching
   `olm.owner=<csv-name>` across the cluster; catches resources the Operator CR may have
   missed if the label propagation lagged.
2. **OwnerReference query** — in the Subscription namespace, list namespace-scoped resources
   with an ownerReference pointing to the CSV; catches resources (e.g., ServiceAccounts) that
   may lack the `olm.owner` label.
3. **InstallPlan steps** — parse the InstallPlan's `status.plan[]` steps where `resolving`
   matches the CSV name, and fetch each live object. The InstallPlan does **not** cover
   resources managed by the CSV controller after install, so it is a fallback supplement
   rather than a primary source.

**Excluded from collection:** `ClusterServiceVersion`, `Subscription`, `InstallPlan`,
`Operator`, `OperatorGroup`, `OperatorCondition` — these are OLMv0 management resources
cleaned up separately.

---

## R6. OperatorGroup field handling (`operators.coreos.com/v1`)

| Field | Handling |
|---|---|
| `spec.targetNamespaces` | If set → not AllNamespaces (Single/Multi) → C1 watch-scope block. When set, `selector` is ignored. |
| `spec.selector` | If set/non-empty → selector-based targeting → C1 watch-scope block. Empty/nil `selector` **and** empty `targetNamespaces` ⇒ AllNamespaces (eligible). |
| `spec.serviceAccountName` | Scoped install SA → C6. OLMv1 runs via operator-controller's cluster-admin SA; a scoped SA cannot be represented. Override `--acknowledge-scoped-serviceaccount` (operator will run with OLMv1's privileges). There is **no** CE target field for this — CE `spec.serviceAccount` is deprecated and ignored (see R7). |
| `spec.upgradeStrategy` | `Default` → fine. `TechPreviewUnsafeFailForward` → no OLMv1 equivalent; informational warning, ignored (OLMv1 has its own upgrade-constraint model). |
| `spec.staticProvidedAPIs` | No OLMv1 equivalent (OLMv1 does not use the `olm.providedAPIs` annotation). Ignored (note only). |
| `status.namespaces` | Read to compute the effective install mode (AllNamespaces vs Own/Single). |
| `status.serviceAccountRef`, `status.conditions`, `status.lastUpdated` | Controller-managed; not migrated. |
| Cleanup | Delete the OperatorGroup **only if no other `Subscription`s remain** in the namespace. First strip `olm.owner` / `olm.owner.namespace` / `olm.owner.kind` / `olm.managed` labels from the OperatorGroup aggregation ClusterRoles (`olm.og.<name>.<view\|admin\|edit>-<hash>`) so RBAC is retained without OLMv0 ownership. |

---

## R7. ClusterExtension result — how each CE spec field is populated

The migration produces one `ClusterExtension` (`olm.operatorframework.io/v1`) from the
Subscription + OperatorGroup inputs above.

| CE field | Source | Notes |
|---|---|---|
| `metadata.name` | Subscription name (default; `--ce-name` override) | |
| `metadata.annotations` | migration metadata | See R2.5 (`migrated-from-subscription`, `migration-subscription-backup`, `acknowledged-*`). |
| `spec.namespace` | Subscription namespace (default) or `--install-namespace` | Required, immutable today. **Phase 6 / [PR #2825](https://github.com/operator-framework/operator-controller/pull/2825):** may become optional/omitted → OLMv1 resolves it from bundle metadata. |
| `spec.serviceAccount` | **do not set** | Deprecated and **ignored** in current OLMv1 (operator-controller uses its own cluster-admin SA). The RFC/prototype `<ce>-installer` SA concept is obsolete — do not create or set it. |
| `spec.source.sourceType` | constant `Catalog` | Only implemented source type. |
| `spec.source.catalog.packageName` | Subscription `spec.name` | Required, immutable. |
| `spec.source.catalog.channels` | Subscription `spec.channel` | Single-element list if set. If empty, resolve the package's `defaultChannel` from the `ClusterCatalog` content and set it explicitly (see R4 channel row). |
| `spec.source.catalog.version` | installed CSV version — only if `installPlanApproval == Manual` | Pin for manual control; unset for Automatic. |
| `spec.source.catalog.selector` | resolved `ClusterCatalog` | `matchLabels: {olm.operatorframework.io/metadata.name: <catalog>}` — pins to the catalog resolved from the CatalogSource image (ties to R8). |
| `spec.source.catalog.upgradeConstraintPolicy` | default `CatalogProvided` | OperatorGroup `TechPreviewUnsafeFailForward` has no exact equivalent; `SelfCertified` is the closest permissive analogue but is **not** applied automatically. |
| `spec.install.preflight.crdUpgradeSafety` | not mapped | No OLMv0 equivalent; leave default (`Strict`). |
| `spec.config.inline.deploymentConfig` | Subscription `spec.config` (`SubscriptionConfig`) | 1:1 — `DeploymentConfig` is a type alias of `SubscriptionConfig`; all sub-fields except `selector` (R4). Applied to the operator Deployment on every render. Requires the `NewOLMConfigAPI` feature on the target cluster. |
| `spec.config.inline.watchNamespace` | *(future)* | OLMv1's inline config also carries `watchNamespace`, the emerging mechanism for Own/Single-namespace watch scope. Not populated by the initial migration (watch scope is handled per C1/R6), but a candidate mapping once that feature stabilizes — potentially relaxing C1. |

---

## R8. CatalogSource → ClusterCatalog mapping (`migrate-catalogs-v0-to-v1`)

Produces one `ClusterCatalog` (`olm.operatorframework.io/v1`) per eligible `CatalogSource`
(`operators.coreos.com/v1alpha1`).

| CatalogSource field | ClusterCatalog target | Notes |
|---|---|---|
| `spec.sourceType` | `spec.source.type` | Only `grpc` **with** `spec.image` maps to OLMv1 `Image`. `configmap` / `internal` / address-only sources have **no** OLMv1 equivalent → report as **not migratable** and skip. |
| `spec.image` | `spec.source.image.ref` | Required for `Image` type. |
| `spec.updateStrategy.registryPoll.interval` | `spec.source.image.pollIntervalMinutes` | Duration string (default 15m) → integer minutes. **Forbidden with digest-based refs** → drop the poll if the ref is a digest. |
| `spec.priority` (`int`) | `spec.priority` (`int32`) | Direct. |
| `spec.secrets` | — | No equivalent (OLMv1 uses the cluster global pull secret). Note if present. |
| `spec.grpcPodConfig.*` | — | No equivalent (catalogd manages the serving pod). Note if present. |
| `spec.displayName` / `description` / `publisher` / `icon` | — | Metadata; dropped. |
| `spec.configMap`, `spec.address` | — | Non-image sources; not migratable (see `sourceType`). |
| *(n/a)* | `spec.availabilityMode` | OLMv1-only; default `Available`. |
| `metadata.name` | `metadata.name` | Reuse the CatalogSource name; becomes the `olm.operatorframework.io/metadata.name` value that the CE selector (R7) pins to. |

---

## R9. Edge cases

- **Multiple operators in one namespace** → OperatorGroup cleanup skipped while other Subscriptions remain (R6).
- **Multiple operators sharing a cluster resource** (e.g. the same CRD across Own/Single installs) → `CollisionProtection: IfNoController` allows adoption without conflict.
- **Operator not at steady state** → C9 hard block.
- **Dependency relationships** → an operator that depends on others is blocked by C2; migrating an operator that *others depend on* proceeds but must warn about dependents.
- **OperatorCondition detection** → OLMv0 stamps OperatorCondition RBAC onto **every** operator's service account, so RBAC is **not** a usage signal. Usage is detected **only** via `OperatorCondition.status.conditions` (C4); C5 explicitly excludes `operatorconditions` from the OLMv0-API RBAC check.
- **Certificate handling** → OLMv0 manages TLS certs directly; OLMv1 delegates to cert-manager (upstream) / service-ca (downstream). Expect pod restarts across the pivot; document as known behavior.
- **Large bundles** → SecretPacker (R2.4) avoids Kubernetes object-size limits.
- **Namespace change** → copy PSA labels (`pod-security.kubernetes.io/*`) and the OpenShift SCC sync label (`security.openshift.io/scc.podSecurityLabelSync`) from the old namespace to the new one. Delete the old namespace **only** with `--acknowledge-namespace-delete` (it may contain non-operator resources).
- **Disconnected / mirrored** → catalogs must be migrated first; the operator tool never auto-creates catalogs.

---

## R10. Non-goals / out of scope

- Dependency resolution (operators declaring `olm.package.required` / `olm.gvk.required`).
- Operators relying on scoped service accounts (without explicit acknowledgment).
- OwnNamespace / SingleNamespace as a *permanent* mode (migration converts to AllNamespaces).
- Hosted Control Planes — OLMv1 is not supported on HCP yet; the design must not foreclose it (e.g. don't hardcode a single kubeconfig).
- Binary packaging / `oc` plugin / container images — deferred to the downstream TP phase ([OCPSTRAT-2692](https://redhat.atlassian.net/browse/OCPSTRAT-2692)).
- Console UI.
