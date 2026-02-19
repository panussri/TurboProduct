# ATLAS - Product Capabilities & Strategic Blueprint

**Role**: `@ARCHITECT`
**Purpose**: This document defines the **Capabilities ("What")** of the Matcha Service, serving as the source of truth for Product and Business stakeholders.

**Technology Stack**: C# .NET 8, PostgreSQL, EF Core 9.0, HotChocolate 13.9 (GraphQL), React 19, TypeScript, Vite, Tailwind CSS, Apollo Client, Liquibase, Docker.

**API Strategy**: REST for external client APIs (task creation, status). GraphQL for internal Matcha Web (verification page).

---

## 1. Core Capability: Universal Document Verification Engine
**Goal**: A single, unified service for all document verification needs across the enterprise (Loans, Insurance, KYC).

### Key Features
*   **Domain Agnostic**: The system does not "know" it's processing a Loan application. It processes "Tasks" and "Documents" based on configuration.
*   **Seamless Integration**: Embeds directly into existing Operational Worklists (Solomon) via a simple URL (`/tasks/:taskUuid`), removing the need for verification teams to learn a new tool. Solomon = worklist page, Matcha = verification page.
*   **Centralized Audit**: A single immutable audit trail for all verification activities across the company.

### Task Lifecycle (4-State Model)
All tasks follow a universal lifecycle: `PENDING → IN_PROGRESS → COMPLETED → [END]`. The **outcome** (APPROVED, RETURNED, REFERRED) is stored as data on the completion event, not as a separate state. This ensures the state machine is reusable across all task types.

*   **PENDING_REVIEW**: A special re-entry state for when late-arriving external data or detected data changes need human review after a decision has already been made and communicated. On entry, a new SQS `work-entry` is published so the task re-appears in the Solomon worklist.

---

## 2. Core Capability: Flexible Logic Configuration
**Goal**: Allow Operations to define and change verification rules without requiring engineering code deployments.

### Key Features
*   **Two Types of Check Items**:
    *   **Data Items** (`item_type: data`): Field-level matching against `data` jsonb. The `check_name` matches a key in the document's data. Verifiers compare the document image against the system value.
    *   **Policy Items** (`item_type: policy`): Subjective boolean checks defined by business policy (e.g., "Is the signature forged?", "Document is clearly legible"). **Mandatory gates** — all must be checked before submission.
*   **Per-Document Decision: Correct / Incorrect Only**:
    *   Verifiers mark each document as **Correct** or **Incorrect**.
    *   When "Incorrect" is selected, a pre-configured `error_message` is shown and a `remark` field is available for case-specific notes.
    *   The `rejection_reason` (pre-configured) is sent to the client in the callback.
*   **Verification Result Persistence (Save-as-you-go)**:
    *   When the verifier makes a decision on a document, the **entire verification result** is persisted to the database in a single save operation per document:
        *   **Document-level decision**: Correct or Incorrect.
        *   **Check item outcomes**: Each data item (correct/incorrect) and each policy item (pass/fail).
        *   **Remarks**: Case-specific notes entered by the verifier (per check item or per document).
    *   Each document save is independent — progress survives browser close. The verifier can save one document and move on to the next without submitting the entire task.
    *   The pre-configured `rejection_reason` and `error_message` are associated with the result when a check item is marked incorrect.
*   **Task-Level Outcomes (Verifier-Driven)**:
    *   **APPROVED**: All documents are correct.
    *   **RETURNED**: Any document is incorrect — sent back to task originator for correction.
    *   **REFERRED**: Task-level escalation to senior approver for policy exceptions. Not a per-document status.
*   **Submission Guardrails**: The system validates that all required decisions are made before enabling submission. A **confirmation dialog** prevents accidental misclicks.
*   **Dynamic Rules**: Rules are stored as data (database configuration), allowing rapid updates for new document types without code changes.

---

## 3. Extended Capability: Asynchronous "Car Check" Integration
**Goal**: Support complex verification flows where data must be cross-referenced with external systems (e.g., vehicle databases) that may have latency.

### Key Features
*   **Hybrid Sync/Async Model**:
    *   **Real-time**: Verifiers can manually trigger a "Refresh" to pull the latest data.
    *   **Background**: The system listens for "Push" callbacks from external providers, updating the task data automatically.
*   **Car Check Status Tracking**: `none → waiting → received`. Displayed in the verification UI with appropriate visual indicators.
*   **Bypass with Cut-off**: If the external provider is slow or unresponsive, the verifier can **bypass** the wait and submit a decision (e.g., APPROVED). The callback fires immediately, unblocking the loan origination workflow. No `awaiting_car_check` state.
*   **Late Data Handling**: If external data arrives *after* a decision is already made (`COMPLETED`), the task transitions to `PENDING_REVIEW`. A new SQS `work-entry` is published so the task re-appears in Solomon. The verifier must circle back to review the new data and can re-confirm or change their decision.
*   **Full Decision Authority**: Upon re-entry from `PENDING_REVIEW`, the verifier retains full authority to **Refer** or **Return** the task. A new webhook callback is sent with the updated outcome and `isReDecision: true`.

---

## 4. Extended Capability: Safety & Integrity Guardrails
**Goal**: Guarantee that every decision is based on the *current* state of the data, preventing "approval leakage" on changed documents.

### Key Features
*   **Dual-Hash Change Detection**:
    *   **`context_hash`**: SHA-256 of the document's `data` + `file_urls`. Recalculated on any update.
    *   **`decision_hash`**: Snapshot of `context_hash` at the time the verifier made their decision.
    *   If `context_hash != decision_hash` → the data changed after the decision was made.
*   **Change Detection "Revised Box"**:
    *   If a document or data point changes *after* a verifier has viewed it, the system detects the diff via hash comparison.
    *   **Visual Alert**: The UI highlights exactly which documents changed since the last decision.
*   **Decision Invalidation**:
    *   **Mid-review**: If data changes while the task is `IN_PROGRESS`, the "Revised" alert appears on next load. Pre-filled decisions are cleared for changed documents. Submit is disabled until changed documents are re-reviewed.
    *   **Post-completion**: If data changes after `COMPLETED`, the task transitions to `PENDING_REVIEW`. A new SQS `work-entry` is published. Previous decisions are invalidated for changed documents.
*   **Immutable Audit Trail (`TaskCompletionEvent`)**:
    *   Every entry into `COMPLETED` creates an immutable audit event with: outcome, submitted_by, trigger_reason, and full callback status tracking.
    *   If a verifier changes their mind during `PENDING_REVIEW`, a **new** completion event is recorded (with `is_re_decision: true`) — the old one is never mutated.

---

## 5. Extended Capability: Re-flow (Same Reference Key)
**Goal**: When a returned task re-enters the system, minimize redundant re-work for verifiers by detecting what changed.

### Key Features
*   **Automatic Detection**: When a new task is created with the same `surrogateKey`, Matcha detects the previous completed task and increments the `version`.
*   **Smart Result Copying**: For documents with matching `context_hash` (same data + files), previous verification results are copied automatically. The verifier sees these as already decided.
*   **Change Flagging**: Documents with different hashes are marked `is_changed = true`. The verifier must re-verify only these documents.
*   **Previous Version Linking**: Each document links to its previous version (`previous_document_id`) for traceability.

---

## 6. Extended Capability: AI-First Document Verification

**Goal**: Use an LLM-based vision model to perform the **primary verification** of documents — checking both data accuracy and policy compliance. **All tasks flow through Matcha** for a complete audit trail. Human QA verifiers act as the **fallback** for uncertain tasks and as **spot-check auditors** for AI-confident tasks.

### Business Value
*   **QA Headcount Reduction**: Tasks where AI is fully confident are auto-completed inside Matcha without human distribution. At scale, this reduces QA staffing proportional to AI confidence rate.
*   **Speed**: Documents are verified asynchronously on upload — before the Matcha task is even created. By the time QA opens a task (if needed), AI results are already available.
*   **Error Prevention**: Document type mismatches and data discrepancies are caught at upload time, not during human review.
*   **Consistency**: Eliminates human variability in reading Thai characters, dates, and ID numbers.

### Key Design Principle: AI-First, Human Fallback, Full Audit Trail

> **All tasks flow through Matcha** — no bypass path. The AI is the **primary verifier**, but Matcha remains the single system of record for every verification decision (human or AI). This ensures a complete, immutable audit trail and enables **spot-check sampling** to validate AI accuracy.

### Decision Model

```
Document Uploaded (Onigiri)
       │
       ▼
  AI Processes (async, outside Matcha)
       │
       ▼
  Onigiri Worker creates Matcha Task
  (ALL tasks — AI results attached)
       │
       ▼
  ┌─────────────────────────────────┐
  │  Matcha: Verification Router    │
  │  (evaluates AI results + rules) │
  └──────────────┬──────────────────┘
                 │
     ┌───────────┼────────────────┐
     │           │                │
  Uncertain   Confident        Confident
  (any item   (all ≥99.99%)    + Spot-Check
   <99.99%)                    (sampled 0.1%)
     │           │                │
     ▼           ▼                ▼
  ┌────────┐  ┌──────────┐  ┌──────────────┐
  │ Human  │  │ Auto-    │  │ Human QA     │
  │ QA     │  │ Complete │  │ (audit the   │
  │ (full  │  │ in Matcha│  │  AI decision)│
  │ review)│  │ (no dist)│  │              │
  └────────┘  └──────────┘  └──────────────┘
     │           │                │
     ▼           ▼                ▼
  TaskCompletion TaskCompletion  TaskCompletion
  Event          Event           Event
  (human)        (ai_auto)       (human_audit)
```

### Key Features

#### 6.1 Document Type Classification
*   **What**: The LLM identifies the document type from the image (e.g., `THAI_ID_CARD`, `CAR_FRONT_PHOTO`, `HOUSE_DEED`).
*   **Mismatch → Immediate Fallback**: If the detected type does not match the expected `documentTypeKey`, the document is flagged as uncertain and the **entire task** routes to human QA.
*   **Confidence**: Classification confidence must be ≥99.99% to be considered auto-verified.

#### 6.2 Data Extraction & Matching (LLM Vision)
*   **What**: The LLM extracts structured fields from the document image and compares them against the system `data` (jsonb) provided by the client.
*   **Per-Item Result**:
    *   ✅ **Match** (≥99.99% confidence): Extracted value matches system value exactly.
    *   ❌ **Mismatch**: Values differ. Entire task → human QA.
    *   ⚠️ **Uncertain**: Confidence below threshold. Entire task → human QA.
*   **Thai + English**: Must support Thai script (primary), English, and numeric extraction.
*   **Output**: Stored per-document as `ai_extracted_data` (jsonb) — keys match `check_name` from `DocumentVerificationItem`.

#### 6.3 Policy Item Verification (LLM-as-Inspector)
*   **What**: Policy items (`item_type: policy`) are treated as **dynamic instructions** for the LLM. The `check_description` field is the prompt. The LLM inspects the document image and evaluates each policy item as pass/fail.
*   **Zero Prompt Engineering on Policy Change**: When a policy item's `check_description` is updated in the database, the LLM automatically uses the new instruction on the next task. No code change, no prompt template change.
*   **Examples**:
    *   `check_description: "ลายเซ็นลูกค้าถูกต้อง ไม่ใช่ลายเซ็นปลอม"` → LLM inspects signature area.
    *   `check_description: "เอกสารอ่านได้ชัดเจน ไม่เบลอ"` → LLM assesses document legibility.
*   **Conservative Threshold**: Policy items are inherently subjective. The 99.99% threshold means the LLM must be near-certain. Any doubt → human QA.

#### 6.4 Document Quality Assessment
*   **What**: Before extraction, the LLM assesses image quality — blurriness, poor lighting, truncation, obstruction.
*   **Quality Failure → Human QA**: Poor quality documents cannot be reliably verified. The entire task routes to human QA with a note: *"Image quality insufficient for AI verification."*

#### 6.5 AI Results in the Human QA UI
*   When a task routes to human QA (uncertain items OR spot-check), the AI results are **pre-filled as suggestions**:
    *   High-confidence items: pre-filled as `correct` — verifier can confirm or override.
    *   Low-confidence / mismatched items: highlighted for verifier attention.
    *   AI-Extracted Value shown alongside System Value for comparison.
*   **Spot-Check Mode**: For spot-checked AI-confident tasks, the verifier sees all AI decisions pre-filled. Their role is to **audit** the AI — confirm or flag errors. This provides the ground truth for measuring AI accuracy.
*   **Override Tracking**: Every verifier override of an AI suggestion is recorded: `{ aiSuggestion, aiConfidence, verifierDecision, overrideReason }`. This feeds the model improvement loop.

#### 6.6 Verification Routing (Matcha-Internal)
*   **What**: After a Matcha task is created with AI results, the **Verification Router** decides whether the task is auto-completed or distributed to human QA.
*   **Routing Rules** (evaluated in priority order):

| Priority | Rule | Action | Use Case |
|----------|------|--------|----------|
| 1 | ANY item < 99.99% confidence | **Distribute to Human QA** | AI is uncertain — human must verify |
| 2 | ANY mismatch detected | **Distribute to Human QA** | Data doesn't match — human must review |
| 3 | Random spot-check sample (configurable, default **0.1%**) | **Distribute to Human QA** | Audit AI accuracy on confident tasks |
| 4 | Rule-based override *(future)* | **Distribute to Human QA** | High-value, high-risk, or flagged applications |
| 5 | All items ≥ 99.99% + not sampled + no rule match | **Auto-Complete** | AI-verified, no human needed |

*   **Spot-Check Sampling**:
    *   Configurable percentage (default 0.1%) of AI-confident tasks are randomly selected for human QA audit.
    *   Sampling rate is stored as a system configuration — adjustable without code change.
    *   Spot-checked tasks are visually marked in the QA UI: *"AI Audit — Verify AI decisions"*.
*   **Rule-Based Override (Future-Ready)**:
    *   Routing rules are stored as database configuration, not code.
    *   Future rules can reference task metadata passed via `distributionConfig` (e.g., `loanAmount > 5M`, `riskLevel = HIGH`, `clientId = 'insurance'`).
    *   Rule engine evaluates conditions and forces human QA distribution when matched.
*   **Auto-Complete Flow**:
    *   Matcha creates an immutable `TaskCompletionEvent` with `trigger_reason: ai_auto_verified`.
    *   Webhook callback sent to client with outcome derived from AI results.
    *   No `work-entry` published to Raijin — task never appears in Solomon QA worklist.
    *   Full audit trail preserved: task + documents + AI results + completion event all in Matcha DB.

### Processing Architecture

AI verification operates in **two phases**, delivering value at different points in the application lifecycle:

```
  PHASE 1 — UNDERWRITER UPLOAD (application in progress)
  ─────────────────────────────────────────────────────────

  ┌──────────────┐    Upload     ┌──────────────────────┐
  │  Onigiri     │ ────────────► │  Wasabi              │
  │  (Underwriter│   (async)     │  (AI Verification)   │
  │   uploads    │               │                      │
  │   document)  │ ◄──────────── │  (LLM Vision Model)  │
  │              │   results     └──────────────────────┘
  │              │
  │  Surface     │
  │  warnings:   │
  │  ⚠️ Wrong    │    ← Underwriter can fix before proceeding
  │    doc type  │
  │  ⚠️ Data     │
  │    mismatch  │
  └──────────────┘

  ... credit approval ... create facility ...

  PHASE 2 — POST-APPROVAL (forwarded to Matcha)
  ─────────────────────────────────────────────────────────

  ┌──────────────────────┐
  │  Onigiri Worker       │
  │  POST /task to Matcha │
  │  (Wasabi results      │
  │   attached)           │
  └──────────┬───────────┘
             │
  ┌──────────▼───────────┐
  │  Matcha:              │
  │  Verification Router  │
  │  (rules + sampling)   │
  └──────────┬───────────┘
             │
  ┌──────────┼─────────────────────┐
  │          │                     │
  Uncertain /    Confident         Confident
  Mismatch /     (auto-complete)   + Sampled
  Rule-match                       (spot-check)
  │          │                     │
  ▼          ▼                     ▼
  ┌────────┐ ┌──────────────┐ ┌──────────────┐
  │Solomon │ │ Auto-Complete │ │ Solomon      │
  │(QA)    │ │ + callback    │ │ (AI Audit)   │
  └────────┘ └──────────────┘ └──────────────┘
```

*   **Phase 1 — Early Warning**: Wasabi results are returned to **Onigiri** during document upload. The underwriter sees warnings (wrong doc type, data mismatches) and can correct before proceeding. This reduces errors reaching Matcha.
*   **Phase 2 — Matcha Routing**: After credit approval, Onigiri Worker creates a Matcha task with Wasabi results attached. The Verification Router decides the path. Every task passes through Matcha regardless of AI confidence.
*   **Enterprise-Grade LLM**: Uses an enterprise-tier model (data privacy compliant, no PII leakage). Model selection optimizes for **lowest cost** while maintaining **correctness and completeness** as the primary constraint.
*   **Re-Processing**: If the underwriter re-uploads a document or updates data after seeing warnings, Wasabi re-processes. The latest results are forwarded to Matcha.

### Feedback Loop & Continuous Improvement

*   **Override Data Pipeline**: Every human override of an AI decision is stored with context (document image reference, AI extraction, AI confidence, verifier decision, override reason).
*   **Model Retraining**: Override data feeds back to the ML team for periodic model fine-tuning. Goal: increase the percentage of tasks that achieve 99.99% across all items → more auto-verification → fewer human QA tasks.
*   **Spot-Check as Ground Truth**: The 0.1% sampled tasks provide the **ground truth** for measuring AI false positive rate in production. If spot-checks reveal AI errors, the sampling rate can be increased.
*   **Metrics to Track**:
    *   **Auto-Verification Rate**: % of tasks auto-completed by AI (no human QA).
    *   **False Positive Rate**: % of AI auto-verified items overridden during spot-checks (must be ≤0.01%).
    *   **Spot-Check Agreement Rate**: % of spot-checked tasks where human agrees with AI.
    *   **Override Rate**: % of AI suggestions overridden by human verifiers (on uncertain tasks).
    *   **Cost per Task**: AI API cost vs. human QA cost per task.

### Resolved Design Decisions

| # | Question | Decision | Rationale |
|---|----------|----------|-----------|
| D1 | OCR Provider | LLM Vision Model (enterprise-grade) | Lowest cost, prioritize correctness and completeness. Model-agnostic — swappable via config. |
| D2 | Cost Model | Justify via QA headcount reduction | Every auto-verified task = one fewer human QA task. ROI scales with volume. |
| D3 | Confidence Threshold | **99.99%** — ultra-conservative | Minimize false positives above all else. Only near-certain items are auto-decided. |
| D4 | Processing Timing | **Async, on document upload** (outside Matcha) | Results ready before Matcha task creation. No latency impact on QA. |
| D5 | Data Privacy | Enterprise-grade model (compliant) | No PII concern — using enterprise tier with data processing agreements. |
| D6 | Policy Items | **Dynamic AI instructions** via `check_description` | Policy changes = DB update only. No prompt engineering. LLM uses `check_description` as the instruction. |
| D7 | Feedback Loop | **Yes** — overrides → model retraining | Any uncertainty → full task to human QA. Override data feeds improvement cycle. |
| D8 | Task Routing | **All tasks through Matcha** + Verification Router | Full audit trail. 0.1% spot-check sampling. Future: rule-based routing (high value, high risk). |
