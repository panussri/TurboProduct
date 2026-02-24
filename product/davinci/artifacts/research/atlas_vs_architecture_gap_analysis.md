# DaVinci: ATLAS vs Architecture — Gap Analysis

> **Date**: 2026-02-24
> **Author**: @ARCHITECT
> **Purpose**: Compare the capability vision ([ATLAS.md](file:///Users/panussriwattana/TurboProduct/product/davinci/ATLAS.md)) against the technical blueprint ([Architecture.md](file:///Users/panussriwattana/TurboProduct/product/davinci/Architecture.md)), surface gaps, and recommend future capabilities.

---

## Document Identity

| Dimension | ATLAS (The "What") | Architecture (The "How") |
|-----------|--------------------|--------------------------|
| **Project label** | DaVinci — enterprise platform | Blueprint 879: CreditMaster — single project phase |
| **Scope horizon** | Enterprise-wide (Loans + Insurance + future products, multi-subsidiary) | Lending only (Operation System = Loan Origination) |
| **Upstream sources** | Core Banking, Policy Admin, KYC/Matcha, Onigiri/LOS | Operation System (single source) |
| **Downstream consumers** | Branch Dashboard, Collection System, Risk Analytics, Future Services | CRM (single consumer) |
| **Domain breadth** | Customer + Product Summaries (lightweight) | Customer + Contract + Collateral (deep, field-level) |

> [!IMPORTANT]
> The two documents describe **different scopes**. ATLAS is the enterprise product vision; Architecture is the delivery blueprint for **Phase 1 (lending master data only)**. Many "gaps" are actually **future phases**, not missing work.

---

## Gap Matrix

### 🔴 ATLAS Capabilities Missing from Architecture

These capabilities are defined in ATLAS but have **zero technical counterpart** in Architecture.

| # | ATLAS Capability | Gap in Architecture | Impact | Suggested Blueprint |
|---|-----------------|--------------------|---------|--------------------|
| G1 | **Consent-Based Data Visibility (PDPA)** — Consent Registry, directed consent, API-level filtering | No mention of consent, PDPA, subsidiary visibility, or data filtering rules | 🔴 **Critical** — legally required before multi-subsidiary rollout | Blueprint: "Consent Engine & PDPA Visibility Layer" |
| G2 | **Customer Data Change Management** — Change Request workflow, approval routing, duplicate detection | Architecture directly upserts from events; no human-approved change flow | 🟡 **Medium** — needed when branch users can edit customer data | Blueprint: "Change Request Workflow & Duplicate Detection" |
| G3 | **Collection Contact Compliance** — Unified contact log, frequency check API, block signal | No contact logging entity, no frequency check API, no block signal flow | 🔴 **Critical** — Sensei depends on this for Thai Debt Collection Act compliance | Blueprint: "Contact Compliance Engine" |
| G4 | **Insurance domain events** — `PolicyIssued`, `PremiumPaid`, `PolicyLapsed`, etc. | Architecture only handles Operation System (Loan Origination) events | 🟡 **Medium** — needed when insurance subsidiaries onboard | Part of multi-source event expansion |
| G5 | **Multi-Subsidiary Awareness** — `originating_subsidiary_id` on every record | No subsidiary concept in the domain model | 🔴 **Critical** — prerequisite for G1 (consent visibility) | Include in Consent Engine blueprint |
| G6 | **`davinci_customer_id`** — system-generated canonical key | Architecture uses National ID as unique key (`ADR #4`); no system-generated surrogate | 🟡 **Medium** — National ID works short-term, but foreigners/corporate entities may need surrogate key | Revisit in Golden Record v2 |

### 🔵 Architecture Depth Missing from ATLAS

The Architecture models domains at a depth the ATLAS deliberately omits (ATLAS = "What", not "How"). These are **not gaps to fix in ATLAS** but worth noting for capability planning.

| # | Architecture Detail | ATLAS Coverage | Observation |
|---|--------------------|--------------------|-------------|
| A1 | **Contract domain** — Disbursement, PaymentStatus, Litigation, History, Document, CollectionFee | ATLAS mentions "product summary records" only | Architecture goes far deeper than ATLAS describes. ATLAS may need to acknowledge Contract Master Data as a first-class capability. |
| A2 | **Collateral domain** — 4 types, Registration, Blacklist, Damage, Image, Appraisal, Ownership | No mention in ATLAS | Entire Collateral vertical is absent from ATLAS. This is a significant gap — Collateral is 1 of 3 Architecture pillars. |
| A3 | **Batch Migration** — S3-based event replay for historical data backfill | No mention in ATLAS | Operational capability needed for go-live. Not a "product capability" per se, but should be referenced as an operational concern. |
| A4 | **CRM as exclusive downstream** — Architecture only outputs to CRM | ATLAS lists 4+ downstream consumers | Architecture is correctly scoped to Phase 1; downstream expansion is future work. |

---

## Alignment Issues

### 1. Unique Key Conflict

| | ATLAS | Architecture |
|-|-------|--------------|
| **Customer Key** | `davinci_customer_id` (system-generated) | National ID (เลขบัตรประชาชน) — ADR #4 |

> [!WARNING]
> These are contradictory. National ID works for Thai individuals but fails for:
> - **Foreign nationals** (no Thai National ID)
> - **Corporate entities** (Tax ID is different)
> - **Cases where one person has changed National IDs** (rare but legally possible)
>
> **Recommendation**: Use National ID as a **natural key for deduplication** but generate `davinci_customer_id` (UUID) as the **canonical system key**. Both can coexist.

### 2. Event Contract Mismatch

ATLAS defines specific domain events (`LoanDisbursed`, `PolicyIssued`, etc.) while Architecture uses generic event contracts (`eventType`, `domain`, `payload`). These are compatible — Architecture's generic envelope can carry ATLAS's typed events — but the **event catalog** should be explicitly documented.

### 3. Downstream Consumer Model

ATLAS envisions DaVinci as a **query API provider** to many systems. Architecture only defines **Event Trigger B (push to CRM)**. The **REST Query API** that ATLAS describes (customer lookup, product summary, contact history) has no API contract in Architecture.

---

## Future Capability Recommendations

Based on the gaps and the Sensei integration requirements, here are recommended future capabilities in priority order:

### Phase 2 — Must-Have Before Multi-Product Rollout

| Priority | Capability | Rationale | Dependencies |
|----------|-----------|-----------|--------------|
| **P0** | **Consent Engine & PDPA Visibility Layer** (G1, G5) | Legally required before any insurance subsidiary uses DaVinci. Without this, multi-subsidiary = PDPA violation. | Requires `originating_subsidiary_id` on all entities |
| **P0** | **Contact Compliance Engine** (G3) | Sensei is already designed to query DaVinci for contact frequency. Without this, Sensei cannot enforce Thai Debt Collection Act. | Requires contact log entity, frequency check API |
| **P1** | **Downstream Query API** | ATLAS promises a query layer; Architecture only has event push. Branch Dashboard, Sensei, and Risk Analytics all need a REST query surface. | Requires consent filtering in query responses |

### Phase 3 — Growth Capabilities

| Priority | Capability | Rationale |
|----------|-----------|-----------|
| **P1** | **Change Request Workflow** (G2) | Needed when branch users can submit customer edits (not just consume upstream events). Prevents data corruption from uncontrolled edits. |
| **P1** | **Insurance Event Domain** (G4) | Expand event sources to include Policy Admin. Enables unified product view. |
| **P2** | **Collateral ATLAS Definition** (A2) | Architecture models Collateral deeply but ATLAS doesn't acknowledge it. Add Collateral Master Data as a first-class ATLAS capability. |
| **P2** | **`davinci_customer_id` Surrogate Key** (G6) | Introduce UUID-based canonical key alongside National ID. Supports foreign nationals and corporate entities. |
| **P3** | **Cross-Product Intelligence** | DaVinci as the unified data layer can power: cross-sell scoring, risk concentration alerts, portfolio analytics. Not in ATLAS yet — pure future opportunity. |

---

## Recommended Next Steps

1. **Reconcile the Customer Key** — Decide whether `davinci_customer_id` or National ID is the canonical key. Document as an ADR.
2. **Add Collateral to ATLAS** — The Architecture already builds it; ATLAS should acknowledge it as a capability.
3. **Design the Consent Engine** — This is the highest-risk gap. PDPA compliance is non-negotiable for multi-subsidiary operations.
4. **Define the REST Query API contract** — Architecture only covers event flows. The query surface that ATLAS promises needs an API contract.
5. **Build the Contact Compliance API** — Sensei is blocked on this for debt collection compliance.
