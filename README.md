# library-olm

A `v0` collection of Go libraries and CLIs for Operator Lifecycle Manager (OLM).

This repository is the basis for the eventual `operator-framework/library-olm`. It is
personal and pre-release: APIs may change without notice while at `v0`.

## Contents

- [`migration/`](migration/) — OLMv0 → OLMv1 migration library and CLIs
  ([OCPSTRAT-2693](https://redhat.atlassian.net/browse/OCPSTRAT-2693)). Start with
  [`migration/README.md`](migration/README.md).

The migration work follows a Specification-Driven Design (SDD) layout:

| Doc | Purpose |
|---|---|
| [migration/README.md](migration/README.md) | Overview of the feature |
| [migration/REQUIREMENTS.md](migration/REQUIREMENTS.md) | What must be built, field-by-field |
| [migration/PLAN.md](migration/PLAN.md) | How it will be built (phased) |
| [migration/VALIDATION.md](migration/VALIDATION.md) | How we prove it works |

## License

Apache 2.0 — see [LICENSE](LICENSE).
