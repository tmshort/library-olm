# Validation — OLMv0 → OLMv1 Migration

Verifiable acceptance criteria drawn from the RFC,
[OCPSTRAT-2693](https://redhat.atlassian.net/browse/OCPSTRAT-2693), and
[REQUIREMENTS.md](REQUIREMENTS.md). Each item is phrased so it can be asserted in a unit or
E2E test. IDs (`V*`) are referenced by the traceability table at the end.

Unit tests use `sigs.k8s.io/controller-runtime/pkg/client/fake`; E2E runs on a kind cluster
with both OLMv0 and OLMv1 installed.

## V1. Per-command behavior

- **V1.1** `migrate-operators-v0-to-v1 check` on a healthy AllNamespaces operator with an available catalog reports all checks green and exits 0; makes no cluster changes.
- **V1.2** `gather` lists every resource that would be migrated (grouped by kind), including CRDs, and reports them as COS objects with `CollisionProtection: IfNoController`; makes no cluster changes.
- **V1.3** `migrate` (single) results in a `ClusterExtension` reaching `Installed=True`; the `Subscription` and `CSV` are deleted; CRDs remain and are adopted by OLMv1; the COS reached `Succeeded=True` **before** the CE was created.
- **V1.4** `rollback <ce-name> --acknowledge-installed` deletes the CE and COS (orphan cascade), recreates the `Subscription` from the backup annotation, and the operator returns to OLMv0 management (`AtLatestKnown`/`UpgradePending`). Without `--acknowledge-installed` on an `Installed=True` CE, rollback refuses and exits non-zero.
- **V1.5** `cleanup <ce-name>` on a Conflict state deletes the `Subscription` and OLMv0 artifacts (Operator CR, OperatorCondition, copied CSVs, OperatorGroup if last) and leaves the CE intact.
- **V1.6** `all` prints sections in order Conflict → Ineligible → AlreadyMigrated → Eligible; Conflicts are never auto-migrated; only Eligible operators are migrated.
- **V1.7** `all` stops on the first failure by default; with `--continue-on-error` it logs the failure and continues, exiting non-zero if any operator failed.
- **V1.8** No command ever blocks on interactive input.

## V2. Eligibility matrix (one case per check, R3)

For each, a fixture operator produces the expected state + reason, and setting the override
flag flips it to `Eligible`:

- **V2.1 (C1)** OperatorGroup with `targetNamespaces` → Ineligible "watch scope"; `--acknowledge-watch-scope-change` → Eligible (migrates to AllNamespaces).
- **V2.2 (C2)** CSV with `olm.package.required` → Ineligible "dependencies"; `--acknowledge-dependencies` → Eligible.
- **V2.3 (C3)** CSV with owned APIServices → Ineligible "apiservices"; `--acknowledge-api-services` → Eligible.
- **V2.4 (C4)** OperatorCondition with `status.conditions` entries → Ineligible "operator-condition"; `--acknowledge-operator-condition` → Eligible.
- **V2.5 (C5)** CSV `.clusterPermissions` granting `operators.coreos.com/subscriptions` → Ineligible "olmv0-api-access"; `--acknowledge-olmv0-api-access` → Eligible.
- **V2.6 (C6)** OperatorGroup with `serviceAccountName` → Ineligible "scoped serviceaccount"; `--acknowledge-scoped-serviceaccount` → Eligible.
- **V2.7 (C7)** Subscription with non-empty `spec.config` → Ineligible "subscription-config"; `--acknowledge-subscription-config` → Eligible.
- **V2.8 (C8, hard)** Package absent from all ClusterCatalogs → Ineligible "package not found; run migrate-catalogs-v0-to-v1 first"; no override.
- **V2.9 (C9, hard)** CSV not `Succeeded` → Ineligible "not at steady state"; no override.

## V3. Field-mapping assertions (R4/R5/R6/R7)

- **V3.1** Manual approval → `CE.spec.source.catalog.version` pinned to the installed version.
- **V3.2** Automatic approval → CE version unset (channel-based upgrades allowed).
- **V3.3** Subscription `spec.channel` → single-element `CE.spec.source.catalog.channels`; empty channel → omitted.
- **V3.4** `CE.spec.source.catalog.selector` pins to the resolved catalog via `olm.operatorframework.io/metadata.name`.
- **V3.5** `CE.spec.serviceAccount` is never set (deprecated/ignored), even when the OperatorGroup had a `serviceAccountName`.
- **V3.6** CE carries `migrated-from-subscription`, `migration-subscription-backup`, and one `acknowledged-<flag>` annotation per flag used.
- **V3.7** OperatorGroup deleted only when no other Subscriptions remain; aggregation ClusterRoles retained with `olm.*` labels stripped.
- **V3.8** CatalogSource `spec.image` → `ClusterCatalog.spec.source.image.ref`; ClusterCatalog `metadata.name` equals the CatalogSource name.
- **V3.9** CatalogSource `registryPoll.interval` → `pollIntervalMinutes` (integer minutes); dropped when the image ref is a digest.
- **V3.10** CatalogSource `priority` carried to ClusterCatalog `priority`.
- **V3.11** A `configmap`/`internal`/address-only CatalogSource is reported not-migratable and skipped.

## V4. Edge-case tests (R8)

- **V4.1** Two operators in one namespace: migrating one leaves the OperatorGroup intact.
- **V4.2** Two operators sharing a CRD: `IfNoController` lets the second adopt without a collision error.
- **V4.3** Operator not at steady state → Ineligible (C9) with a clear reason.
- **V4.4** Dependency operator (`olm.generated-by` present / declares requirements) → flagged; an operator others depend on migrates but emits a dependents warning.
- **V4.5** OperatorCondition disambiguation: an operator with OLMv0-stamped OperatorCondition RBAC but **empty** `status.conditions` is **Eligible** (RBAC is not treated as usage).
- **V4.6** Large bundle exceeding inline size limits migrates successfully via SecretPacker.
- **V4.7** Namespace change copies `pod-security.kubernetes.io/*` and `security.openshift.io/scc.podSecurityLabelSync` to the new namespace; old namespace deleted only with `--acknowledge-namespace-delete`.

## V5. End-to-end scenario (kind)

1. Bootstrap kind with OLMv0 + OLMv1 side-by-side.
2. Install an AllNamespaces operator via an OLMv0 Subscription; confirm healthy.
3. `migrate-catalogs-v0-to-v1` → CatalogSource becomes a serving ClusterCatalog.
4. `migrate-operators-v0-to-v1 check` → all green (catalog now found).
5. `gather` → lists resources incl. CRDs (`IfNoController`).
6. `migrate` → CE `Installed=True`; Subscription/CSV deleted; CRDs adopted.
7. Upgrade via OLMv1 → CRDs updated through the normal bundle lifecycle.
8. `rollback <ce-name> --acknowledge-installed` → Subscription restored, CE deleted.
9. Fresh cluster with operators in all four states → `all` → correct four-section output; Conflicts warned and skipped.

## V6. Non-functional (Jira deployment considerations)

- **V6.1** Works on all node topologies: SNO, compact (3-node), and multi-node.
- **V6.2** No architecture-specific behavior (x86_64, aarch64, ppc64le, s390x).
- **V6.3** Works connected and in restricted networks (catalog resolution by package name against pre-mirrored catalogs).
- **V6.4** Only AllNamespaces install mode is produced (Own/Single converted with acknowledgment).
- **V6.5** Self-managed, classic (standalone) clusters. HCP explicitly not covered (R9).

## V7. Deliverable checks

- **V7.1** Repo at `/home/tshort/git/operator-framework/library-olm` with a pushed public personal remote; `migration/` holds README, REQUIREMENTS, PLAN, VALIDATION.
- **V7.2** `REQUIREMENTS.md` covers every Subscription (R4), OperatorGroup (R5), and CatalogSource (R7) spec field, plus the full ClusterExtension target mapping (R6).
- **V7.3** `PLAN.md` has 7 phases + the cross-repo prerequisite, each with an exit criterion.
- **V7.4** This traceability table has no requirement without at least one validation item.

---

## Traceability

| Requirement | Validated by |
|---|---|
| R1.1 Library API | V1.1–V1.7 (via library calls), V3.*, V4.* |
| R1.2 Two CLIs | V1.*, V3.8–V3.11, V5 |
| R1.3 Four-state classification | V1.6, V2.*, V5.9 |
| R1.4 `all` ordering | V1.6 |
| R1.5 Batch failure / `--continue-on-error` | V1.7 |
| R1.6 Non-interactive | V1.8 |
| R1.7 Downtime posture | V5.6 (namespace unchanged), V4.7 (namespace change) |
| R1.8 Recovery | V1.4, V1.5 |
| R2.1 v0 module + CI | V7.1, V7.3 |
| R2.2 ClusterObjectSet rename | V1.3 (COS created), V4.6 |
| R2.3 Wait for COS Succeeded; no status writes | V1.3 |
| R2.4 SecretPacker + IfNoController | V1.2, V4.2, V4.6 |
| R2.5 CE annotations | V3.6 |
| R2.6 Boxcutter phase 2 | Prerequisite note (PLAN) |
| R3 C1–C9 | V2.1–V2.9 |
| R4 Subscription fields | V3.1–V3.3, V2.7, V4.4 |
| R5 OperatorGroup fields | V2.1, V2.6, V3.5, V3.7 |
| R6 ClusterExtension mapping | V3.1–V3.6 |
| R7 CatalogSource→ClusterCatalog | V3.8–V3.11 |
| R8 Edge cases | V4.1–V4.7 |
| R9 Non-goals | V6.4, V6.5 |
