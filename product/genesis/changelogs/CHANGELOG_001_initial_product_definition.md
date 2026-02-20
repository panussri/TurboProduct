# CHANGELOG 001 — Genesis: Initial Product Definition

**Date**: 2026-02-19
**Author**: @ARCHITECT
**Type**: New Product

---

## Summary

Established **Genesis** as the centralized Customer & Product Master Data platform for the enterprise. This is the foundational data layer that all future systems will query for up-to-date customer and product summary information.

## New Capabilities

### 1. Single View of Customer (Golden Record)
*   Unified customer identity profile across all business lines (Loans, Insurance).
*   System-generated `genesis_customer_id` as canonical key.
*   Product summary linkage (balances, statuses) — not full transactional detail.
*   Multi-subsidiary awareness on all records.

### 2. Consent-Based Data Visibility (PDPA Compliance)
*   Consent Registry as a first-class entity.
*   Directed consent model (A→B ≠ B→A).
*   API-level enforcement — all query results filtered by caller's subsidiary + consent records.
*   Consent audit trail.

### 3. Event-Driven Data Synchronization
*   Consumes domain events from Core Banking, Policy Admin, KYC/Matcha, Onigiri/LOS.
*   Idempotent event processing.
*   Downstream Query API — single point of query for all consumers.

### 4. Customer Data Change Management
*   Change Request workflow (not direct edits).
*   Approval routing by risk level.
*   Change propagation via `CustomerProfileChanged` events.
*   Duplicate detection on creation and change.

### 5. Collection Contact Compliance (Cross-Cutting)
*   Unified contact log across all products.
*   Frequency check API for Thai debt collection limits.
*   Cross-product coordination signals.

## Design Decisions (ADR)

| # | Decision | Rationale |
|---|----------|-----------|
| D1 | Centralized read model, not system of record | Simpler ownership. Source systems retain authority. Eventual consistency is acceptable. |
| D2 | Consent-based visibility, not role-based | PDPA requires entity-level consent, not job-function access control. |
| D3 | Directed consent model | Legally precise. A→B consent does not imply B→A. |
| D4 | Event-driven sync | Near-real-time needed for collection compliance. |
| D5 | Change requests, not direct edits | Governance for high-risk fields. Low-risk auto-approved. |
| D6 | Product name: Genesis | Foundational layer — everything builds on top. |
