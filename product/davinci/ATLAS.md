# ATLAS - Product Capabilities & Strategic Blueprint

**Role**: `@ARCHITECT`
**Purpose**: This document defines the **Capabilities ("What")** of the DaVinci Service — the centralized Customer & Product Master Data platform for the enterprise.

**Codename**: DaVinci (ダヴィンチ)
**Status**: 📝 Draft

---

## 1. Core Capability: Single View of Customer (Golden Record)

**Goal**: Provide a single, authoritative source of truth for all customer identity and contact information across every business line (Loans, Insurance, future products).

### Why It Exists (First Principles)

The company operates **multiple business lines** (Lending, Insurance via multiple subsidiaries) through **shared branch networks**. The same branch employee may originate loans, sell insurance, and perform collections. Without a centralized customer record:

*   **Regulatory Risk**: Thai debt collection regulations limit contact frequency per day. Without a combined view, separate teams may unknowingly exceed this limit by contacting the same customer for different products.
*   **Operational Friction**: Address changes, phone number updates, and KYC re-verifications must be repeated across every system.
*   **Missed Opportunity**: Cross-sell and upsell opportunities are invisible when customer data is siloed by product.

### Key Features

*   **Golden Record**: A single, deduplicated customer profile aggregating identity from all source systems.
    *   **Unique Identifier**: A system-generated `davinci_customer_id` that becomes the canonical customer key across all downstream systems.
    *   **Identity Core**: Name (TH/EN), Date of Birth, National ID (encrypted), Passport, Tax ID.
    *   **Contact Profile**: Phone numbers (ranked primary/secondary), Email, Physical addresses (with type: home, work, mailing).
    *   **KYC / AML Status**: Current verification status, last verification date, source system of verification.
*   **Product Linkage (Summary, Not Detail)**:
    *   DaVinci does **not** store full loan agreements or insurance policies. Those live in their respective transactional systems (Core Banking / LOS, Policy Admin).
    *   DaVinci stores **product summary records** linked to the customer:
        *   Loan: Account number, product type, current balance, status (active/closed/delinquent), originating subsidiary.
        *   Insurance: Policy number, product type, premium status, status (active/lapsed/cancelled), originating subsidiary.
    *   These summaries are kept current via **event consumption** (see Capability 3).
*   **Multi-Subsidiary Awareness**: Every customer record and every product linkage carries an `originating_subsidiary_id`, identifying which legal entity owns that relationship.

---

## 2. Core Capability: Consent-Based Data Visibility (PDPA Compliance)

**Goal**: Enforce Thai PDPA consent rules at the data access layer, ensuring that customer information is only visible to users whose subsidiary has obtained the customer's consent.

### Why It Exists (First Principles)

The company has **multiple legal subsidiaries** that sell insurance products. A customer who purchases insurance from *Subsidiary A* has not necessarily consented to share their data with the *parent lending entity (Subsidiary H)*. Thai PDPA mandates that data sharing across legal entities requires **explicit customer consent**.

This is **not** role-based security (e.g., "Loan Officers can't see Insurance data"). It is **entity-level consent gating**: a user employed by *Subsidiary H* cannot see a customer's insurance products held at *Subsidiary A* **unless the customer has given cross-entity consent**.

### Key Features

*   **Consent Registry**: Each customer-subsidiary relationship has a consent record:
    *   `customer_id`, `subsidiary_id`, `consent_type` (e.g., `CROSS_ENTITY_DATA_SHARING`), `granted_at`, `revoked_at`, `consent_channel` (branch, app, web), `consent_document_ref`.
*   **Visibility Rules (Enforcement Layer)**:
    *   **Default: Restricted**. A user sees only data originating from their own subsidiary.
    *   **With Consent**: If the customer has granted `CROSS_ENTITY_DATA_SHARING` consent between Subsidiary A → Subsidiary H, then users from Subsidiary H can see Subsidiary A's product summaries for that customer.
    *   **Symmetric vs. Directed**: Consent is **directed** (A→H ≠ H→A). Each direction requires separate consent.
    *   **Revocation**: Consent can be revoked. Upon revocation, visibility is immediately restricted. Existing data is not deleted but becomes inaccessible to the non-consented entity.
*   **Consent Audit Trail**: All consent grants, revocations, and access attempts are logged immutably.
*   **API Enforcement**: All DaVinci APIs filter results based on the requesting user's subsidiary and the applicable consent records. There is no "see everything" bypass outside of a designated compliance/audit role.

### Regulatory Context

| Regulation | Requirement | DaVinci Response |
|------------|-------------|------------------|
| PDPA (Thailand) | Cross-entity data sharing requires explicit consent | Consent Registry + Visibility Rules |
| Debt Collection Act (Thailand) | Max contact frequency per day per debtor | Combined product view enables contact frequency tracking across all products |
| AML / KYC | Customer due diligence across all relationships | Golden Record aggregates KYC status from all subsidiaries |

---

## 3. Core Capability: Event-Driven Data Synchronization

**Goal**: Keep DaVinci current with upstream transactional systems by consuming domain events, making it the **single point of query** for all downstream services needing customer or product summary data.

### Why It Exists (First Principles)

DaVinci is **not** the system of record for loans or insurance policies — Core Banking and Policy Admin systems are. However, **downstream consumers** (branch dashboards, collection systems, risk analytics, reporting) should not query multiple source systems. DaVinci acts as the **materialized read model** — a centralized, pre-joined, always-current view.

### Key Features

*   **Event Consumption**: DaVinci subscribes to domain events from upstream systems:
    *   **Core Banking**: `LoanDisbursed`, `LoanPaymentReceived`, `LoanStatusChanged`, `LoanBalanceUpdated`, `LoanClosed`.
    *   **Policy Admin**: `PolicyIssued`, `PremiumPaid`, `PolicyLapsed`, `PolicyCancelled`, `PolicyRenewed`.
    *   **KYC Service / Matcha**: `CustomerVerified`, `CustomerKYCExpired`.
    *   **Onigiri (LOS)**: `ApplicationCreated`, `ApplicationApproved`, `CustomerProfileUpdated`.
*   **Idempotent Processing**: All event handlers are idempotent (keyed on event ID + version). Duplicate or out-of-order events are handled gracefully.
*   **Event Schema Registry**: Upstream systems publish events conforming to a shared schema contract. DaVinci validates incoming events against the registry.
*   **Downstream Query API**: Other services query DaVinci (not the source systems) for:
    *   Customer profile lookup (by ID, phone, name).
    *   Product summary by customer (all loans, all policies — filtered by consent).
    *   Contact history and frequency (for collection compliance).

### Data Flow

```
   ┌─────────────┐   Events    ┌───────────────┐   Query API   ┌─────────────────┐
   │ Core Banking│ ──────────► │               │ ◄──────────── │ Branch Dashboard │
   └─────────────┘             │               │               └─────────────────┘
                               │               │
   ┌─────────────┐   Events    │    DAVINCI    │   Query API   ┌─────────────────┐
   │ Policy Admin│ ──────────► │  (Golden      │ ◄──────────── │ Collection Sys   │
   └─────────────┘             │   Record +    │               └─────────────────┘
                               │   Product     │
   ┌─────────────┐   Events    │   Summaries)  │   Query API   ┌─────────────────┐
   │ KYC/Matcha  │ ──────────► │               │ ◄──────────── │ Risk Analytics   │
   └─────────────┘             │               │               └─────────────────┘
                               │               │
   ┌─────────────┐   Events    │               │   Query API   ┌─────────────────┐
   │ Onigiri/LOS │ ──────────► │               │ ◄──────────── │ Future Services  │
   └─────────────┘             └───────────────┘               └─────────────────┘
```

---

## 4. Core Capability: Customer Data Change Management

**Goal**: Provide a governed workflow for creating, updating, and merging customer records, ensuring data quality and audit compliance.

### Why It Exists (First Principles)

Customer data changes (address updates, phone number corrections, name changes after marriage) have downstream impact across all products. An uncontrolled update could:

*   Break collection correspondence (wrong address).
*   Create compliance gaps (outdated KYC).
*   Generate duplicate records (new vs. existing customer).

### Key Features

*   **Change Request Workflow**:
    *   Branch users submit **Change Requests** (not direct edits) for customer profile modifications.
    *   Change types: `UPDATE_CONTACT`, `UPDATE_ADDRESS`, `UPDATE_IDENTITY`, `MERGE_DUPLICATES`.
    *   Each request captures: `requested_by`, `requested_at`, `change_type`, `field_changes` (before/after), `supporting_document_ref`.
*   **Approval Routing** (configurable):
    *   Low-risk changes (e.g., update secondary phone) → auto-approved.
    *   High-risk changes (e.g., National ID correction, address change) → requires reviewer approval.
    *   Merge requests → always require approval + supporting evidence.
*   **Change Propagation**: Upon approval, DaVinci:
    *   Updates the Golden Record.
    *   Publishes a `CustomerProfileChanged` event for downstream systems to consume.
    *   Logs the change in the immutable audit trail.
*   **Duplicate Detection**: On new customer creation or change requests, DaVinci runs matching rules (name + DOB + ID number fuzzy match) to flag potential duplicates before creation.

---

## 5. Cross-Cutting Concern: Collection Contact Compliance

**Goal**: Enable collection teams to comply with Thai debt collection contact frequency limits by providing a unified view of all customer products and contact history.

### Key Features

*   **Unified Contact Log**: All collection contacts (call, SMS, visit) across all products for a customer are logged in DaVinci.
*   **Frequency Check API**: Before initiating contact, the collection system queries DaVinci: *"How many times has this customer been contacted today across all products?"*
*   **Block Signal**: If the daily limit is reached, DaVinci returns a block signal. The collection system must respect this.
*   **Cross-Product Coordination**: Because DaVinci knows all active delinquent products for a customer, it can inform the collection system: *"This customer has 2 delinquent loans and 1 lapsed insurance policy — coordinate your contact."*

---

## Resolved Design Decisions (ADR Style)

| # | Decision | Context | Consequence |
|---|----------|---------|-------------|
| D1 | **Centralized read model, not system of record** | DaVinci consumes events from source systems (Core Banking, Policy Admin). It does not own loan or policy lifecycle. | Simpler ownership boundaries. DaVinci never conflicts with source systems. Trade-off: eventual consistency (acceptable for read-model use cases). |
| D2 | **Consent-based visibility, not role-based** | Security requirement is driven by PDPA cross-entity consent, not by job function. A loan officer from Subsidiary H can see insurance data IF customer consented. | More granular than RBAC. Requires consent registry as a first-class entity. More complex query layer but legally correct. |
| D3 | **Directed consent model** | Consent from A→H does not imply H→A. | Each direction must be explicitly granted. More consent records to manage, but legally precise. |
| D4 | **Event-driven sync (not ETL batch)** | Near-real-time currency is needed for collection compliance (contact frequency) and branch operations. | Requires event infrastructure (SQS/SNS or Kafka). More complex than nightly ETL but meets SLA. |
| D5 | **Change requests, not direct edits** | High-risk fields (National ID, address) need governance. | Adds friction for branch users but prevents data quality issues. Low-risk changes can be auto-approved to minimize friction. |
| D6 | **Product name: DaVinci** | Centralized master data layer — the Renaissance polymath who unified art, science, and engineering. | Clear metaphor for unifying all data domains. |
