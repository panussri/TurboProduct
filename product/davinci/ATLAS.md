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

## 6. Core Capability: Data Consolidation Engine

**Goal**: Deterministically assemble a single Golden Record from data arriving from multiple upstream sources — with field-level authority, no data loss, self-check integrity, and an admin override function with full logging.

> **Delivery Note**: This capability is self-contained and deliverable **before** the Resolution Workflow (Capability 7). At minimum, it provides: authority-based consolidation, admin override with audit trail, and self-check.

### Why It Exists (First Principles)

DaVinci receives customer data from multiple systems (Operation System, Core Banking, Policy Admin, KYC/Matcha, Onigiri, Data Warehouse). These sources are each authoritative for *different fields* of the same customer. Without explicit consolidation logic:

*   **Last-write-wins** → silently loses data.
*   **Reject conflicts** → creates an ever-growing backlog.
*   **Keep everything** → no one trusts the record.

Authority is **field-level**, not source-level. CRM knows phone numbers best; Core Banking knows balances best. Recency alone is insufficient — a CRM address from 2024 may be the correct *home* address while a LOS address from 2026 is the *work* address.

### Key Features

#### 6.1 Field Authority Matrix

Each field in the Golden Record has a **ranked list of authoritative sources** and a **conflict classification tier** (used by Capability 7).

*   **Authority Ranking**: Per-field ordered list of sources. Higher-ranked source wins on conflict.
*   **Authority Lock** (🔒): Some fields (National ID, DOB, KYC Status, Loan Balance) can only be set by their Rank 1 source. Lower-ranked sources are always stored as alternatives, never promoted automatically.
*   **Conflict Tier**: Each field is assigned `T1` (CO Handle), `T2` (Needs Approval), or `T3` (Needs Verification). This classification routes conflicts to the appropriate resolution workflow (Capability 7). Default for new fields: `T2`.

#### 6.2 Resolution Rules

| Scenario | Rule | Outcome |
|----------|------|---------|
| New field, no existing value | **Accept** | Write the value. Log as `INITIAL_SET`. |
| Same value from different source | **Confirm** | Keep value, update `last_confirmed_at`. Log as `CONFIRMED`. |
| Different value, higher-authority source | **Override** | Replace value. Archive previous. Log as `AUTHORITY_OVERRIDE`. |
| Different value, lower-authority source | **Store alternative** | Keep current value. Store incoming as `alternative_value`. Log as `LOWER_AUTHORITY_STORED`. |
| Different value, same authority rank | **Escalate** | Flag for resolution (Capability 7). Log as `CONFLICT_ESCALATED`. |
| Authority-locked field, below Rank 1 | **Reject override** | Keep current. Log as `AUTHORITY_LOCKED_REJECTED`. |
| Out-of-order event | **Evaluate + flag** | Apply rules but log as `OUT_OF_ORDER_PROCESSED`. Self-check triggers. |

#### 6.3 No-Data-Loss Guarantee

*   **Raw Event Store** (append-only): Every incoming event is persisted verbatim *before* the consolidation engine processes it. Fields: `event_id`, `source_system`, `received_at`, `event_timestamp`, `payload_hash` (SHA-256), `raw_payload`. Never deleted.
*   **FieldHistory**: When a Golden Record field is updated, the previous value moves to a history table. Never deleted.
*   **AlternativeValue**: Lower-authority or conflicting values are stored alongside the primary value. Can be promoted via admin override or the Resolution Workflow.

#### 6.4 Consolidation Log (Audit Trail)

Every consolidation decision produces an immutable log entry: `log_id`, `customer_id`, `event_id`, `field_name`, `source_system`, `resolution_type`, `previous_value`, `incoming_value`, `resolved_value`, `authority_rank`, `rule_applied`, `created_at`.

This log enables:
*   **Compliance audit**: Trace how any field arrived at its current value.
*   **Debugging**: Understand exactly why a value was set or rejected.

#### 6.5 Admin Override (Logged)

> **Critical**: This is the basic manual intervention function that ships with the Consolidation Engine *before* the full Resolution Workflow (Capability 7) exists.

*   **Who**: DaVinci administrators and designated data stewards.
*   **What**: Force-set any Golden Record field to a specified value, overriding the current authority-based result.
*   **Logging**: Every admin override creates a Consolidation Log entry with `resolution_type = ADMIN_OVERRIDE`, recording: `overridden_by` (user ID), `override_reason` (mandatory free-text), `previous_value`, `new_value`, `timestamp`.
*   **Alternative Promotion**: An admin can promote a stored `AlternativeValue` to the Golden Record. Logged as `ADMIN_PROMOTED`.
*   **Reversibility**: Admin overrides can be reversed by a subsequent override. The full history is preserved in the Consolidation Log.
*   **No Silent Changes**: There is no path to modify the Golden Record without a Consolidation Log entry. Zero exceptions.

#### 6.6 Self-Check Mechanism

**Post-write checks** (after every consolidation):

| Check | Validates | On Failure |
|-------|-----------|------------|
| Hash verification | SHA-256 of resolved values matches DB write | 🔴 Halt + alert |
| Log-record consistency | Golden Record value matches latest `resolved_value` in log | 🔴 Flag for investigation |
| Field completeness | Required fields are not null | 🟡 Warning + log |
| Temporal consistency | `last_updated_at` ≥ previous value | 🟡 Warning (possible out-of-order) |

**Nightly batch checks**:

*   **Log reconciliation**: Replay consolidation logs for sampled customers (1%/night) and verify result matches current Golden Record.
*   **Raw Event completeness**: Every event in Raw Event Store has a corresponding consolidation log entry.
*   **Alternative value staleness**: Alternatives > 90 days with `pending_review` are flagged.
*   **Cross-field consistency**: Business rules (e.g., `KYC_status = VERIFIED` → `kyc_verified_date` must not be null).

Results published as a **Data Quality Report**.

#### 6.7 Admin Page

A simple DaVinci-owned page for managing the Consolidation Engine:

*   **Field list**: View all tracked fields with authority ranking, lock status, and conflict tier.
*   **Authority ranking**: Reorder source authority per field. Toggle authority-lock.
*   **Tier assignment**: Set T1 / T2 / T3 per field.
*   **Source registry**: Add/remove upstream sources.
*   **Override panel**: Search customer → view Golden Record → force-set with logged reason.
*   **Audit log viewer**: Filter consolidation log by customer, field, resolution type, or date.

---

## 7. Core Capability: Data Resolution Workflow & Integration

**Goal**: Provide a structured, tiered process for resolving data conflicts and processing customer change requests — owned by DaVinci, executed via Sensei, and verified via Matcha where evidence is required.

> **Delivery Note**: This capability depends on Capability 6 (Consolidation Engine) being in place. It extends the `CONFLICT_ESCALATED` path with structured resolution workflows and adds customer-initiated change request support.

### Why It Exists (First Principles)

The Consolidation Engine (Capability 6) auto-resolves most conflicts via authority rules. But some conflicts cannot be resolved by authority alone:

*   **Same-rank sources disagree** → the system cannot choose.
*   **Customer requests a change** → no upstream event exists; a human initiates the change.
*   **Identity-affecting fields** → require documentary evidence, not just assertion.

Without a structured workflow, these cases accumulate as unresolved flags that no one owns, and customer-initiated changes bypass the consolidation audit trail entirely.

### Key Concepts

*   **ResolutionRequest**: A DaVinci-owned entity representing a data conflict or change request that requires human action. DaVinci manages its lifecycle, SLA, and state.
*   **Sensei = Task Executor Only**: Sensei receives resolution events from DaVinci, creates `📋 Admin` tasks for COs, and returns outcomes. Sensei does not own or track resolution state.
*   **Unified Entry**: System conflicts, customer change requests (via call center or branch), and data quality alerts all create the same `ResolutionRequest` entity and follow the same tiered workflow.

### 3-Tier Resolution

| Tier | Name | Fields | Resolver | SLA |
|------|------|--------|----------|-----|
| **T1** 🟢 | CO Handle | Phone, address, email | CO at customer's responsible branch | Same day |
| **T2** 🟡 | Needs Approval | Name, employer, marital status, income | CO submits → Supervisor/Data Steward approves | 3 business days |
| **T3** 🔴 | Needs Verification | National ID, DOB, KYC, passport | Evidence submitted → Matcha verifies → DaVinci accepts | 5 business days |

### Resolution Lifecycle

```
ResolutionRequest States:

  T1:  OPEN → IN_PROGRESS → RESOLVED
                           → ESCALATED (bumps to T2)

  T2:  OPEN → IN_PROGRESS → PENDING_APPROVAL → RESOLVED (approved)
                                               → REJECTED (denied)
                           → ESCALATED (bumps to T3)

  T3:  OPEN → EVIDENCE_REQUIRED → VERIFICATION_IN_PROGRESS → RESOLVED
                                                             → REJECTED
                                                             → MANUAL_REVIEW → RESOLVED / REJECTED
```

### Entry Channels

| Channel | Actor | Flow |
|---------|-------|------|
| **System conflict** | Consolidation Engine | Auto-creates `ResolutionRequest` when `CONFLICT_ESCALATED` |
| **Customer call** | Call center agent | Creates `ResolutionRequest` via DaVinci API; T3 fields require branch visit with documents |
| **Branch walk-in** | Branch staff | Creates `ResolutionRequest` via DaVinci API; can attach evidence immediately |
| **Data quality alert** | Self-check batch | Auto-creates `ResolutionRequest` for inconsistencies |

### Cross-Product Integration

| System | Role | Integration |
|--------|------|-------------|
| **Sensei** | Task executor | Receives `customer.resolution_required` events → creates tasks → returns `customer.resolution_completed` |
| **Matcha** | Document verifier (T3) | DaVinci creates Matcha verification task for evidence → Matcha returns pass/fail/inconclusive |
| **Call Center** | Request creation channel | Creates `ResolutionRequest` via DaVinci API when customer contacts them |

### Product Boundary

*   **DaVinci owns**: `ResolutionRequest` entity, lifecycle state, SLA tracking, resolution outcome recording, Golden Record update on resolution.
*   **Sensei owns**: Task creation, CO assignment, task outcome recording. Sensei does *not* track resolution state.
*   **Matcha owns**: Document verification for T3 requests. Returns result to DaVinci.

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
| D7 | **Field-level authority, not source-level** | Different sources are authoritative for different fields (CRM for phone, Core Banking for balance). A global source ranking would produce incorrect overrides. | Per-field authority matrix requires more configuration (managed via admin page) but is far more accurate. |
| D8 | **Admin override with mandatory logging** | Before the Resolution Workflow is built, admins need a way to correct data. Unlogged overrides would violate the no-data-loss guarantee. | Every admin action produces an immutable Consolidation Log entry. No silent changes. Adds friction but preserves audit trail. |
| D9 | **DaVinci as resolution process owner, Sensei as task executor** | DaVinci owns the data and the resolution state. Splitting process ownership across products would create state synchronization problems. | DaVinci manages `ResolutionRequest` lifecycle; Sensei only creates and completes tasks. Clear boundary. Sensei stays focused on operational orchestration. |
| D10 | **3-tier conflict classification** | Not all conflicts are equal. Phone number conflicts are low-risk (CO can verify during next call). National ID conflicts require documentary evidence. A single workflow for all would over-engineer simple cases and under-govern critical ones. | T1 (CO Handle), T2 (Needs Approval), T3 (Needs Verification via Matcha). Configurable per field via admin page. |

