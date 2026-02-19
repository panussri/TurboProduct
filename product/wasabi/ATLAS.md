# ATLAS — Wasabi: AI Document Verification Service

> **Service:** Wasabi (わさび — sharp clarity)
> **Role:** LLM-based document verification engine — external to Matcha
> **Owner:** AI/ML Platform Team
> **Status:** Draft
> **Last Updated:** 2026-02-19

---

## Executive Summary

**Wasabi** is an independent, stateless AI service that uses an LLM vision model to verify document images against structured verification instructions. It operates **outside Matcha**, triggered asynchronously on document upload during the **underwriter's application phase** in Onigiri.

Wasabi does **not** make final decisions. It produces a structured verification report for each document, which serves two purposes:

1. **Immediate feedback to the underwriter** — Onigiri surfaces Wasabi results (wrong document type, data mismatches) to the underwriter while they are still working on the application. Issues can be corrected before submission.
2. **Pre-filled AI results for Matcha** — When the application later reaches Matcha (after credit approval), Wasabi results are forwarded with the task. Matcha's Verification Router uses them to decide auto-complete vs human QA.

**Key principle:** Wasabi has **no knowledge** of Matcha's task lifecycle, QA workflow, or routing logic. It receives images and instructions → returns results to **Onigiri**. Onigiri owns the underwriter feedback and later forwards results to Matcha.

---

## System Context & Timing

Wasabi runs during the **underwriter's document upload phase** — long before the Matcha task exists. Results flow back to Onigiri first, then travel to Matcha when the application is eventually forwarded.

```
  UNDERWRITER PHASE (application in progress)
  ────────────────────────────────────────────────────────────

  ┌──────────────┐    (1) Upload    ┌──────────────┐
  │  Onigiri     │ ───────────────► │   Wasabi      │
  │  (Underwriter│   image + meta   │   (AI Engine) │
  │   uploads    │                  │               │
  │   document)  │                  │   (2) Fetch   │
  │              │                  │   instructions│
  │              │                  │       │       │
  │              │                  │       ▼       │
  │              │                  │   ┌────────┐  │
  │              │                  │   │ Matcha  │  │
  │              │                  │   │ Config  │  │
  │              │                  │   └────────┘  │
  │              │                  │               │
  │              │  (3) Results     │   (LLM       │
  │              │ ◄─────────────── │   processes)  │
  │              │                  │               │
  │  (4) Surface │                  └──────────────┘
  │  to under-   │
  │  writer:     │
  │  ⚠️ Wrong    │
  │    doc type  │
  │  ⚠️ Name     │
  │    mismatch  │
  └──────────────┘

  ────────────────────────────────────────────────────────────
  ... time passes ... credit approval ... create facility ...
  ────────────────────────────────────────────────────────────

  POST-APPROVAL PHASE (application forwarded to Matcha)
  ────────────────────────────────────────────────────────────

  ┌──────────────┐   POST /task     ┌──────────────┐
  │  Onigiri     │ ───────────────► │   Matcha      │
  │  Worker      │  (Wasabi results │   (Verification│
  │              │   attached)      │    Router)     │
  └──────────────┘                  └──────────────┘
```

| Step | When | From → To | What |
|------|------|-----------|------|
| 1 | Underwriter uploads document | Onigiri → Wasabi | Document image URL + expected `documentTypeKey` + system `data` |
| 2 | Immediately after | Wasabi → Matcha (config only) | Fetch verification instructions for the document type |
| 3 | After LLM processing | Wasabi → **Onigiri** | Structured verification report (per document) |
| 4 | Underwriter still working | Onigiri → Underwriter | Surface warnings: wrong doc type, data mismatches, policy failures |
| 5 | After credit approval | Onigiri Worker → Matcha | Create Matcha task with Wasabi results attached |

> **Critical timing:** Steps 1–4 happen during the underwriter phase. Step 5 happens much later — only after credit approval and facility creation. By then, Wasabi results are already stored in Onigiri, ready to be forwarded.

---

## Two-Phase Value Delivery

### Phase 1: Early Warning to Underwriter (Onigiri-side)

Wasabi results are surfaced to the underwriter **while they are still editing the application**:

*   **Wrong document type** → *"⚠️ Expected: บัตรประจำตัวประชาชน — Detected: ทะเบียนรถ"* — underwriter can re-upload the correct document.
*   **Data mismatch** → *"⚠️ ชื่อ-สกุล: document says สมชัย but system has สมชาย"* — underwriter can correct the application data or re-upload a correct document.
*   **Policy instruction failed** → *"⚠️ ลายเซ็นลูกค้า: No signature found"* — underwriter can prompt the customer for a proper document.

This **prevents errors from reaching Matcha** in the first place. Fewer errors at upload = higher auto-verification rate downstream.

### Phase 2: AI-Assisted Routing in Matcha

When the application later reaches Matcha, the stored Wasabi results are attached to the task. Matcha's Verification Router uses them to decide:

*   **All items pass** → candidate for auto-complete (subject to spot-check + routing rules).
*   **Any uncertainty remains** → route to human QA with AI results pre-filled.

> **Changes between phases:** The underwriter may re-upload documents or update application data after seeing Wasabi warnings. If so, Wasabi should re-process the updated documents. The latest Wasabi results are what gets forwarded to Matcha.

---

## Processing Pipeline

Wasabi processes each document through a **4-stage pipeline**. Each stage feeds the next. Failures at any stage mark the document as `requires_human_review`.

### Stage 1: Document Quality Assessment

*   Assess image quality before any extraction — blurriness, poor lighting, truncation, obstruction, watermarks.
*   If image is unusable → skip remaining stages. The underwriter is prompted to re-upload a clearer image.
*   Quality score (0–100) is included in the report.

### Stage 2: Document Type Classification

*   The LLM identifies the document type and compares it to the expected `documentTypeKey`.
*   **Type matches** → proceed to Stage 3.
*   **Type doesn't match** → include a `suggestedType`. The underwriter is prompted before they proceed: *"This looks like a ทะเบียนรถ, not a บัตรประจำตัวประชาชน."*
*   **Confidence < 99.99%** → flag for downstream human review.

### Stage 3: Instruction Fetch & Verification

Wasabi fetches the verification instructions from Matcha for the document type, then processes each item using the LLM:

#### Data Items (`item_type: data`)

The `check_name` is the key in the system `data` jsonb. Wasabi:
1. **Extracts** the corresponding value from the document image.
2. **Compares** the extracted value against the system value.
3. **Reports** result per item:

| Status | Meaning |
|--------|---------|
| `match` | Extracted value matches system value. Confidence ≥ 99.99%. |
| `mismatch` | Values differ. Highlighted for underwriter + downstream verifier. |
| `low_confidence` | Confidence < 99.99%. Cannot determine match reliably. |
| `not_extractable` | Field could not be extracted (obscured, missing, unreadable). |

#### Policy Items (`item_type: policy`)

The `check_description` is treated as a **natural-language instruction** for the LLM:
1. **Reads** the `check_description` as an inspection instruction.
2. **Inspects** the document image against that instruction.
3. **Reports** result with reasoning:

| Status | Meaning |
|--------|---------|
| `instruction_completed` | Instruction satisfied. Confidence ≥ 99.99%. |
| `instruction_failed` | Instruction NOT satisfied. Highlighted for attention. |
| `instruction_uncertain` | LLM cannot determine pass/fail with sufficient confidence. |

> **Why fetch instructions from Matcha?** Verification items are Matcha-owned configuration. Policy changes = DB update only. Wasabi automatically uses updated instructions — zero prompt engineering, zero deployment.

### Stage 4: Report Assembly

All stage results are assembled into a single **Document Verification Report**, containing:
*   Quality assessment (score + usability)
*   Type classification (expected vs detected, match status, suggested type if wrong)
*   Per-item results (data matches + policy evaluations, each with confidence and reasoning)
*   Summary (counts + overall confidence + whether human review is required)

---

## The Three Core Questions

Wasabi answers exactly three questions per document:

### 1. Is the document type correct? If not — suggest.

→ Underwriter sees: **"⚠️ Expected: บัตรประจำตัวประชาชน — Detected: ทะเบียนรถ"** — can re-upload before proceeding.

### 2. Is any data incorrect? If so — highlight.

→ Underwriter sees: Per-field mismatch warnings with extracted values vs system values. Can correct data or re-upload.

### 3. Are any instructions not completed? If so — highlight.

→ Underwriter sees: Policy instruction failures with reasoning. Can prompt customer for proper documents.

---

## Result Interpretation

### By Onigiri (Underwriter Phase)

| Wasabi Result | Onigiri Action |
|---------------|----------------|
| Type mismatch | Prompt underwriter: wrong document, suggest correct type |
| Data mismatch | Warn underwriter: field X doesn't match system value |
| Policy instruction failed | Warn underwriter: document doesn't meet requirement |
| Image quality unusable | Prompt underwriter: re-upload clearer image |
| All items pass | No warnings — application can proceed normally |

### By Matcha (Verification Router, post-approval)

| Wasabi Result | Matcha Action |
|---------------|---------------|
| All items pass, all confidence ≥ 99.99%, type matches | **Candidate for auto-complete** (subject to spot-check + routing rules) |
| Any data `mismatch` | Route to **human QA** — mismatched items highlighted |
| Any `low_confidence` or `instruction_uncertain` | Route to **human QA** — uncertain items flagged |
| Any `not_extractable` or `instruction_failed` | Route to **human QA** — failed items flagged |
| Type mismatch | Route to **human QA** — wrong document type shown |
| Image quality unusable | Route to **human QA** — quality warning shown |
| Processing failure | Route to **human QA** — as if no AI results exist |

> **Wasabi never decides routing.** It reports findings. Onigiri surfaces them early. Matcha uses them for routing later.

---

## LLM Prompt Strategy

*   **Dynamic Assembly**: Prompt constructed from `DocumentVerificationItem` records fetched from Matcha. No hardcoded checks.
*   **Thai-First**: All field names, instructions, and values are in Thai. The LLM must natively handle Thai script.
*   **Structured Output**: The LLM returns structured results. Wasabi parses and validates.
*   **Single-Shot per Document**: One LLM call per document (all items in one prompt). Minimizes cost.
*   **Re-processing on Change**: If the underwriter re-uploads a document or updates data, Wasabi re-processes and returns updated results.

---

## Architectural Properties

| Property | Description |
|----------|-------------|
| **Stateless** | Processes request, returns result, stores nothing. Onigiri stores results. |
| **Independent Deployment** | Separate repo, deployment pipeline, and scaling. No shared database. |
| **Horizontally Scalable** | Each document is an independent unit of work. Scales linearly with volume. |
| **Model-Agnostic** | LLM provider swappable via config. Prompt template is model-neutral. |
| **Per-Document Processing** | One message per document (not per task). Enables parallel processing. |
| **Idempotent** | Duplicate requests (same `requestId`) return cached results. |
| **Fail-Safe** | LLM failures → `requires_human_review`. Matcha defaults to human QA if no results. |
| **Re-Processable** | Document re-upload triggers re-processing. Latest results are authoritative. |

---

## Boundary Responsibilities

| Concern | Owner |
|---------|-------|
| Verification instructions (what to check) | **Matcha** (config) |
| LLM processing (how to check) | **Wasabi** |
| Storing Wasabi results during underwriter phase | **Onigiri** |
| Surfacing warnings to underwriter | **Onigiri** |
| Forwarding results to Matcha on task creation | **Onigiri Worker** |
| Verification results storage (post-task creation) | **Matcha** |
| Routing decisions (auto-complete vs human QA) | **Matcha** |
| QA task lifecycle | **Matcha** |
| Document image hosting | **Onigiri (S3)** |
| LLM model selection & optimization | **Wasabi / ML Team** |
| Override feedback data for retraining | **Matcha** (stores) → **Wasabi** (consumes) |

---

## Cost Optimization

| Strategy | Description |
|----------|-------------|
| **Single LLM call per document** | All items in one prompt. Reduces per-call overhead. |
| **Parallel document processing** | Documents within an application processed concurrently. |
| **Model selection** | Cheapest model achieving ≥99.99% accuracy on Thai document extraction. |
| **Config caching** | Cache verification items per `documentTypeKey`. |
| **Early exit on quality** | Stage 1 quality gate avoids expensive LLM calls on unusable images. |
| **Skip re-processing** | If document + data haven't changed, return previous results. |

---

## Open Questions

> [!NOTE]
> These are implementation-level decisions, not blockers for `@BUILDER`.

1. **Message Queue**: SQS (consistent with Hephaestus flow) or RabbitMQ (consistent with Onigiri Worker)?
2. **Config Caching TTL**: How long should Wasabi cache verification items?
3. **Multi-Page Documents**: Multiple `fileUrls` per document — one LLM call with all pages or separate calls?
4. **Confidence Calibration**: Single LLM call or consensus voting (multiple calls + compare)?
5. **Re-Processing Trigger**: Should Wasabi automatically re-process when the underwriter updates application data (not just document re-upload)?
