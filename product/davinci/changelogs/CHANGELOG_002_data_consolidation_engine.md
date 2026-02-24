# CHANGELOG 002 — Data Consolidation Engine & Resolution Workflow

> **Date**: 2026-02-24
> **Author**: @ARCHITECT
> **Status**: 📝 Draft
> **Exploration Artifact**: [consolidation_engine_exploration.md](../artifacts/research/consolidation_engine_exploration.md)

---

## Summary

Adds two new capabilities to DaVinci, designed as **sequentially deliverable** units:

1. **Capability 6: Data Consolidation Engine** — Field-level source authority matrix, deterministic resolution rules, no-data-loss guarantee (Raw Event Store + FieldHistory + AlternativeValue), immutable consolidation log, admin override with mandatory logging, self-check mechanism, and a simple admin page. Deliverable standalone.

2. **Capability 7: Data Resolution Workflow & Integration** — Extends Capability 6's `CONFLICT_ESCALATED` path with a 3-tier resolution workflow (T1: CO Handle, T2: Needs Approval, T3: Needs Verification via Matcha). Unifies system conflicts and customer-initiated change requests into a single `ResolutionRequest` lifecycle owned by DaVinci. Integrates with Sensei (task execution), Matcha (document verification), and call center (request creation).

---

## New Capabilities

### Capability 6: Data Consolidation Engine

*   **Field Authority Matrix**: Per-field ranked source authority with authority-lock for identity/financial fields.
*   **Resolution Rules**: 7 deterministic rules covering initial set, confirmation, authority override, alternative storage, escalation, authority-lock rejection, and out-of-order handling.
*   **No-Data-Loss**: Raw Event Store (append-only, SHA-256 hashed), FieldHistory (archived superseded values), AlternativeValue (lower-authority values stored alongside primary).
*   **Consolidation Log**: Immutable audit trail for every resolution decision.
*   **Admin Override**: Force-set with mandatory reason. Logged as `ADMIN_OVERRIDE`. Zero silent changes.
*   **Self-Check**: Post-write integrity checks (hash, log-record consistency, completeness, temporal) + nightly batch reconciliation.
*   **Admin Page**: Simple CRUD for field authority, tier assignment, source registry, override panel, and audit log viewer.

### Capability 7: Data Resolution Workflow & Integration

*   **3-Tier Classification**: T1 🟢 CO Handle (same-day), T2 🟡 Needs Approval (3 days), T3 🔴 Needs Verification (5 days, via Matcha).
*   **ResolutionRequest Entity**: DaVinci-owned lifecycle — OPEN → IN_PROGRESS → RESOLVED/REJECTED, with escalation between tiers.
*   **Unified Entry**: System conflicts, customer change requests (call center/branch), and data quality alerts → same entity, same workflow.
*   **Product Boundary**: DaVinci = process owner (lifecycle, SLA, state). Sensei = task executor only. Matcha = T3 document verification.

---

## New Design Decisions

| # | Decision | Consequence |
|---|----------|-------------|
| D7 | **Field-level authority, not source-level** | Per-field matrix managed via admin page. More accurate than global ranking. |
| D8 | **Admin override with mandatory logging** | Immutable Consolidation Log entry per override. No silent changes. |
| D9 | **DaVinci as resolution process owner** | DaVinci manages `ResolutionRequest` lifecycle. Sensei only executes tasks. Clear separation. |
| D10 | **3-tier conflict classification** | T1/T2/T3 — configurable per field. Avoids over-engineering low-risk conflicts. |
