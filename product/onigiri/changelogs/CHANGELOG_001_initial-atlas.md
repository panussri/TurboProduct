# CHANGELOG 001 — Initial ATLAS: Smart Form, Workflow, Campaign Configuration

**Date**: 2026-02-27
**Author**: @ARCHITECT
**Status**: Draft

---

## Summary

Initial capability definition for Onigiri — the all-in-one loan origination system. This changelog establishes the foundational ATLAS covering four core capabilities and one cross-cutting concern.

## New Capabilities

### 1. Smart Form (Application Intake)
- Compositional form architecture: **Page → Section → Field**
- Fixed stage sequence: Borrower → Guarantor → Loan Setup → Summary → Document Upload
- Configurable sections within each stage (add, reorder, split across pages)
- JSON data persistence model — persists to RDS only on workflow transitions
- Section-level properties: validation, external integration, document requirements, completion tracking

### 2. Underwriting Workflow
- Four-phase state machine: Origination → Underwriting → Decision → Terminal
- **Draft state includes document upload** — single entry state for all data capture and document collection
- States: Draft, Risk Assessment, Approval + Risk Level, Create Facility, Cash? routing, Confirmation, Create Loan + Disbursement, QA, Funded, Rejected, Withdrawn, Expired
- Cash vs. non-cash loan path divergence after Create Facility via Cash? decision
  - Cash: Confirmation → Create Loan + Disbursement → QA → Funded
  - Non-cash: QA → Confirmation → Create Loan + Disbursement → Funded
- Multiple return/pull-back paths — all return to Draft
- Fixed topology, configurable execution steps within each state

### 3. Loan Campaign Configuration
- Campaign-driven product definition (eligibility, pricing, fields, documents)
- Eligibility criteria: rule-based gateway with operator-value pairs
- Loan pricing: amount range, interest, tenor, LTV
- Per-campaign field and document requirement lists

### 4. Risk Assessment Engine (Preview)
- Hierarchical structure: Strategy → Policy → Rule → Decision
- Risk level output mapped to approval authority (10–70 scale)
- Deviation flags and conditional document requirements
- Detailed spec deferred to future iteration

### 5. External Integrations (Cross-Cutting)
- Integration map: Matcha, Wasabi, DaVinci, Sensei, Core Banking, NCB

## Design Decisions

| # | Decision | Rationale |
|---|----------|-----------|
| D1 | JSON application data, persist on workflow transition | Rapid product iteration without DDL changes |
| D2 | Fixed workflow topology, configurable execution steps | Maintainability + compliance vs. flexibility |
| D3 | Cash vs. Non-Cash divergence after Create Facility | Matches real-world operational risk profiles |
| D8 | Merged Draft + Document Upload into single Draft state | Simpler state machine, all returns go to Draft |
| D4 | Page → Section → Field composability | Reusable sections across loan products |
| D5 | Campaign-driven configuration | Product managers define campaigns without dev involvement |
| D6 | Strategy → Policy → Rule hierarchy for risk | Hierarchical decision logic, policy reuse across strategies |
| D7 | Product name: Onigiri (おにぎり) | Self-contained, all-in-one metaphor |
