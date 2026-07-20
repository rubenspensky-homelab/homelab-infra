# Feature: Improve infrastructure documentation

## Status

- Type: Documentation feature
- Priority: High
- State: Proposed
- Owner: Unassigned

## Summary

Create accurate, maintainable, and operational documentation for the homelab
infrastructure. The documentation must explain not only what is installed, but
also how to bootstrap, validate, operate, recover, upgrade, and troubleshoot the
cluster.

This feature addresses outdated component notes, missing disaster recovery
procedures, undocumented operational decisions, and the lack of automated
documentation validation.

## Problem

The repository provides a useful component inventory and a high-level bootstrap
flow, but it is not yet sufficient to operate or rebuild the cluster without
tribal knowledge.

Current issues include:

- `README.md` describes the architecture but only provides a four-step bootstrap
  outline.
- `docs/authentik.md` references templates and assets that no longer match the
  current implementation.
- `docs/nvidea.support.md` has an incorrect filename and describes an NVIDIA GPU
  Operator deployment that is not the current implementation.
- `docs/nvidia.for.ansible.md` contains a non-portable local file link.
- There are no tested runbooks for backup restoration, PostgreSQL recovery,
  node loss, credential rotation, GitOps failures, or runner compromise.
- Architectural and operational decisions are not recorded separately from
  procedural documentation.
- No automated check detects broken links, invalid commands, or documentation
  drift.

## Goals

- Make a clean cluster bootstrap reproducible from repository documentation.
- Document the actual configuration represented by the manifests.
- Provide recovery procedures for critical data and platform services.
- Record important design decisions and accepted limitations.
- Give each operational procedure clear prerequisites, validation, rollback,
  and expected results.
- Introduce lightweight automated checks that prevent obvious documentation
  regressions.

## Non-goals

- Change Kubernetes resources, Helm values, sync waves, or application behavior.
- Implement backup schedules, high availability, NetworkPolicies, or security
  controls as part of this feature.
- Document the internal behavior of every upstream chart.
- Duplicate upstream product documentation when a maintained link is enough.
- Guarantee disaster recovery before the related infrastructure changes are
  implemented and tested.

## Target information architecture

```text
README.md
docs/
  architecture/
    overview.md
    dependency-order.md
  components/
    authentik.md
    external-secrets.md
    gpu.md
  runbooks/
    bootstrap.md
    backup-restore.md
    postgres-recovery.md
    node-loss.md
    credential-rotation.md
    argocd-recovery.md
    runner-incident.md
    storage-capacity.md
    upgrade-rollback.md
  decisions/
    0001-gitops-app-of-apps.md
    0002-local-storage.md
    0003-public-access-cloudflare.md
  specs/
    documentation-improvements.md
```

The exact number of files may change during implementation, but architecture,
component reference, runbooks, and decision records must remain distinct.

## Documentation standard

Every component document must include:

- Purpose and reason for inclusion.
- Owning Argo CD `Application` and source directory.
- Namespace and important dependencies.
- Data persisted by the component.
- Secrets consumed and their source, without secret values.
- Public and internal exposure.
- Health and validation commands.
- Upgrade considerations.
- Links to applicable runbooks and upstream documentation.

Every runbook must include:

- Purpose and triggering conditions.
- Preconditions and required access.
- Impact and safety warnings.
- Ordered commands with placeholders clearly identified.
- Expected result after each major step.
- Validation and success criteria.
- Rollback or abort procedure.
- Escalation notes and related runbooks.
- Last-tested date and tested environment.

Commands must be safe to paste by default. Destructive commands must be clearly
marked and must include a verification step before execution.

## Work items

### DOC-001: Create documentation index

Priority: P0

Update `README.md` so it remains a concise entry point and links to architecture,
component references, runbooks, and decision records.

Acceptance criteria:

- A reader can locate bootstrap, recovery, component, and architecture
  documentation from `README.md`.
- The component inventory links to the relevant local document when one exists.
- The README distinguishes implemented behavior from planned improvements.
- No credentials, account identifiers, or environment-specific secret values are
  added.

### DOC-002: Document architecture and dependency order

Priority: P0

Document the GitOps flow from `bootstrap/root-app.yaml` through the child Argo CD
applications and the external `homelab-apps` repository. Include the intended
dependency graph for CRDs, operators, secret stores, databases, workloads, and
routing.

Acceptance criteria:

- The document identifies all platform categories and their namespaces.
- It explains Argo CD sync waves and their limitations across applications.
- It records the current dependency-order risks as known issues rather than
  describing the current order as reliable.
- It includes a diagram in Mermaid or a maintained image with source.
- Every referenced file exists in the repository.

### DOC-003: Replace the bootstrap outline with a complete runbook

Priority: P0

Create a bootstrap runbook covering host prerequisites, Kubernetes assumptions,
Argo CD installation prerequisites, out-of-band secrets, root application
installation, reconciliation order, and post-install validation.

Acceptance criteria:

- A new operator can identify every prerequisite not managed by GitOps.
- AWS, Cloudflare, DNS, LAN addresses, storage paths, and GitHub App dependencies
  are identified without exposing secret values.
- Validation covers nodes, storage, operators, databases, routing, monitoring,
  authentication, backups, and runners.
- Known bootstrap blockers caused by the current sync waves are explicitly noted
  until fixed.
- The runbook contains recovery guidance for a partially completed bootstrap.

### DOC-004: Correct component documentation drift

Priority: P0

Bring existing component documentation in line with the current manifests.

Required changes:

- Correct Authentik template, blueprint, asset, hostname, database, and secret
  references.
- Correct External Secrets store names, AWS remote keys, target Secret names, and
  `refreshPolicy: CreatedOnce` behavior.
- Replace the NVIDIA documentation with a document for the current device plugin
  and DCGM exporter deployment.
- Rename `docs/nvidea.support.md` to a correctly spelled and stable name.
- Delete or replace the non-portable content in `docs/nvidia.for.ansible.md`.

Acceptance criteria:

- Every path, resource name, namespace, hostname, and configuration statement can
  be traced to a current manifest.
- No documentation claims that NVIDIA GPU Operator is installed.
- No non-portable local file links remain.
- No two documents provide contradictory instructions for the same component.

### DOC-005: Document backup and restore capabilities honestly

Priority: P0

Create a backup and restore runbook that distinguishes Kubernetes object backup,
PVC data backup, database-consistent backup, and external dependencies.

The first version must document current limitations even if backup infrastructure
is not yet complete.

Acceptance criteria:

- The runbook states that current Velero configuration has no schedules,
  snapshots, or node agent.
- It inventories PostgreSQL, Loki, Tempo, Authentik media, BuildKit cache, and
  persistent volumes by recovery importance.
- It distinguishes disposable cache and telemetry from critical application data.
- It defines target RPO and RTO fields, allowing `TBD` with an assigned follow-up
  task where a decision is pending.
- It includes a restore test checklist and a place to record test evidence.
- It does not claim a recovery path has been validated until a restore has been
  executed successfully.

### DOC-006: Create PostgreSQL recovery runbook

Priority: P0

Document diagnosis and recovery of the shared CloudNativePG cluster, including
the impact on Authentik, Grafana, and Umami.

Acceptance criteria:

- The runbook covers pod failure, node loss, PVC loss, logical database issues,
  and credential mismatch as separate scenarios.
- It identifies which scenarios cannot currently be recovered from repository
  state alone.
- Validation includes database readiness and application-level connectivity.
- Destructive recovery steps require an explicit backup and identity check.
- Future CloudNativePG backup/recovery implementation can extend the runbook
  without replacing its structure.

### DOC-007: Create platform incident runbooks

Priority: P1

Create focused runbooks for:

- Loss of a Kubernetes node.
- Argo CD stuck, degraded, or unable to reconcile.
- Local storage capacity or filesystem pressure.
- External Secret synchronization failure.
- Cloudflare Tunnel or Gateway routing failure.
- GitHub runner or BuildKit compromise.
- Credential rotation for AWS, Cloudflare, GitHub App, databases, Grafana, and
  Authentik.

Acceptance criteria:

- Each scenario has detection, containment, recovery, and validation sections.
- The runner incident procedure includes stopping runner scale-up and rotating
  credentials.
- The storage procedure explains the implications of local-path storage and
  `reclaimPolicy: Delete`.
- Credential procedures account for secrets using `CreatedOnce` and explain how
  synchronization is triggered safely.

### DOC-008: Document upgrades and rollback

Priority: P1

Document the update process for Helm dependencies, vendored CRDs, container
images, and Kubernetes itself.

Acceptance criteria:

- The procedure requires compatibility and release-note review.
- It documents regeneration and verification of `Chart.lock` files.
- It explains how vendored manifests are sourced and verified.
- It contains pre-upgrade backup, render diff, health check, and rollback steps.
- It warns against treating a Git revert as a complete data rollback.

### DOC-009: Record architecture decisions

Priority: P1

Introduce short Architecture Decision Records for decisions that materially
affect reliability or operations.

Initial ADRs:

- Argo CD app-of-apps and automatic synchronization.
- Local-path storage as the default StorageClass.
- Cloudflare public TLS termination and the absence of cert-manager.
- Shared PostgreSQL cluster versus per-application databases.
- Optional status of GPU components.

Acceptance criteria:

- Each ADR records context, decision, consequences, alternatives, and status.
- Accepted risks and assumptions are explicit.
- Superseded decisions link to their replacement.

### DOC-010: Add documentation validation

Priority: P1

Add lightweight validation to CI after the repository has a CI workflow.

Minimum checks:

- Markdown linting.
- Local link and image validation.
- Detection of non-portable and absolute local filesystem links.
- Spelling allowlist for product and Kubernetes-specific names.
- Verification that paths written as repository file references exist.

Acceptance criteria:

- Validation runs on pull requests that change Markdown or documentation assets.
- A broken local link fails the check.
- Product names and intentional technical terms can be added to a repository
  allowlist.
- Checks do not require access to cluster credentials or external secrets.

### DOC-011: Establish review and ownership rules

Priority: P2

Define how documentation remains synchronized with infrastructure changes.

Acceptance criteria:

- Pull requests changing a component must update its documentation when behavior,
  dependencies, secrets, persistence, exposure, or recovery changes.
- Runbooks include `last tested` metadata.
- Recovery runbooks are reviewed after incidents and restore exercises.
- Documentation ownership is represented in `CODEOWNERS` or an equivalent policy.

## Delivery phases

### Phase 1: Accuracy and discoverability

- DOC-001 documentation index.
- DOC-002 architecture and dependency order.
- DOC-003 bootstrap runbook.
- DOC-004 correction of existing documents.

Exit condition: no known stale paths, resource names, or descriptions remain in
the existing documentation.

### Phase 2: Recovery and operations

- DOC-005 backup and restore.
- DOC-006 PostgreSQL recovery.
- DOC-007 platform incident runbooks.
- DOC-008 upgrades and rollback.

Exit condition: every critical service has a documented diagnosis and recovery
entry point, with current limitations stated explicitly.

### Phase 3: Governance and maintenance

- DOC-009 architecture decisions.
- DOC-010 automated validation.
- DOC-011 ownership and review rules.

Exit condition: documentation regressions are checked in pull requests and major
operational decisions have an auditable record.

## Definition of done

This feature is complete when:

- All P0 and P1 work items meet their acceptance criteria.
- Existing stale documentation has been corrected, moved, or removed.
- The README provides a clear entry point into the documentation set.
- Bootstrap and restore documentation has been followed by someone other than its
  author or reviewed step by step against a clean environment.
- At least one restore exercise has produced recorded evidence, or the
  documentation clearly marks restore as unvalidated with an open infrastructure
  task.
- Documentation validation runs automatically on relevant pull requests.
- Remaining documentation debt is represented by explicit tasks rather than
  untracked notes.

## Follow-up infrastructure features

The documentation work is expected to produce or refine separate implementation
features for:

- Correcting Argo CD dependency ordering and sync waves.
- Implementing CloudNativePG backups and recovery from object storage.
- Enabling scheduled Velero backups with PVC data coverage.
- Configuring Alertmanager receivers and backup alerts.
- Hardening and isolating BuildKit and GitHub runners.
- Adding NetworkPolicies and namespace security controls.
- Pinning container images and adding infrastructure validation CI.

Those changes are deliberately outside this documentation feature so that each
can have its own risk assessment, implementation plan, and verification criteria.
