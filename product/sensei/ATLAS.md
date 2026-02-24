# ATLAS - Product Capabilities & Strategic Blueprint

**Role**: `@ARCHITECT`
**Purpose**: This document defines the **Capabilities ("What")** of the Sensei Service — the Branch Operations Orchestration Platform for field staff productivity and management.

> **Sensei (先生)** — The master who provides structure, guidance, and discipline. Sensei does not do the work — it orchestrates *how work is done* across branches, ensuring alignment with policy, visibility for leadership, and productivity for field staff.

---

## 1. Core Capability: Playbook Engine

**Goal**: Provide a structured, reusable system for defining multi-step operational strategies (Playbooks) that drive how field staff handle specific business objectives.

### Why It Exists (First Principles)

*   **Policy Alignment Problem**: Thousands of branch staff across hundreds of branches must execute consistent strategies. Without a structured playbook, each branch invents its own approach, leading to inconsistent outcomes and compliance risk.
*   **Knowledge Codification**: Effective collection strategies, renewal campaigns, and recovery workflows are institutional knowledge. They must be captured as executable templates, not tribal knowledge.
*   **Adaptability**: HQ defines the default strategy, but local conditions (branch size, customer demographics, staffing) require branch-level customization within guardrails.

### Key Concepts

*   **Playbook**: An ordered sequence of **Steps** that defines how to handle an objective (e.g., "Delinquency Recovery", "Insurance Renewal", "KYC Re-verification"). Each Playbook targets a specific **Objective Type**.
*   **Step**: A single action within a playbook. Each step has:
    *   **Action Type**: `📞 Call`, `🏠 Visit`, `⏳ Wait`, `📋 Admin`, `🔔 Notify Supervisor`, `📧 Send Notification`
    *   **Timing**: When the step triggers relative to the previous step (e.g., "Day 1", "3 days after previous step", "Immediately on failure")
    *   **Outcome Transitions**: Each step defines what happens for each possible outcome (see below)
    *   **Assignee Rule**: Who executes (`Same CO`, `Branch Supervisor`, `Auto-escalate`)
    *   **Compliance Lock**: Whether HQ has locked this step (cannot be removed or reordered below a certain position)

### Outcome-Based Transition Model

Playbooks follow a **linear default path** (Step 1 → 2 → 3 → ...) but each step can define **outcome transitions** that override the default next step. This allows branching behavior without the complexity of a full flowchart.

Each action type defines a set of **possible outcomes**. The number and type of outcomes varies per action:

| Action Type | Possible Outcomes |
|-------------|------------------|
| 📞 Call | PTP, No Answer, Refused, Callback, Wrong Number, Line Busy, Voicemail |
| 🏠 Visit | Met Customer, Not Home, Address Invalid, PTP (in-person), Refused |
| 📋 Admin | Completed, Incomplete, Escalated |
| ⏳ Wait | (auto-advances after duration) |
| 🔔 Notify | Acknowledged, No Response |

**Each outcome maps to one transition rule:**

| Transition Type | Meaning |
|----------------|--------|
| `→ Next Step` | Continue to the next step in sequence (default) |
| `→ Specific Step` | Jump to a named step (e.g., skip to Visit, skip to Report) |
| `→ Retry` | Repeat the same step (with max attempt limit, e.g., max 3) |
| `→ End (Success)` | Close the playbook as succeeded |
| `→ End (Failed)` | Close the playbook as failed |
| `→ Escalate` | Route to supervisor for manual decision |

**Example: Step "📞 โทรศัพท์" with outcome transitions:**

```
📞 โทรศัพท์ครั้งที่ 1
  ├── PTP           → ✅ End (Success)
  ├── No Answer     → 🔄 Retry (max 3) then → Step 5 (🏠 Visit)
  ├── Refused       → ⏭️ Step 6 (📋 Report)
  ├── Callback      → ⏭️ Step 3 (⏳ Wait) with scheduled date
  ├── Wrong Number  → ⏭️ Step 6 (📋 Report)
  └── Default       → ⏭️ Next Step
```

Supervisors can **drag outcomes to different target steps** in the editor. The transition list is always visible per step, and each outcome row can be independently configured.

### Playbook Hierarchy

```
  ┌──────────────────────────────────┐
  │   System Template (HQ-owned)     │  ← Published by HQ, read-only for branches
  │   e.g., "ติดตามหนี้ค้างชำระ v3"   │
  └────────────┬─────────────────────┘
               │ fork
  ┌────────────▼─────────────────────┐
  │   Branch Variant                 │  ← Supervisor customizes for their branch
  │   e.g., "สาขารัชดา — แก้ไข"      │
  └──────────────────────────────────┘
```

*   **System Templates** are generic, reusable blueprints owned by HQ. Published to all branches. Cannot be edited by supervisors.
    *   Templates define: step sequence, action types, outcome transitions, timing defaults, compliance locks
    *   Templates **do not** contain: branch-specific assignee names, branch-specific timing, or customer data
    *   Where data lands: Template definitions are stored centrally. When a playbook is **instantiated** for a customer, the resulting tasks are stored in the **Task Engine** (Capability 2). Outcomes and notes are recorded on each task record. Reports aggregate from tasks, not from the template.
*   **Branch Variants** are created when a supervisor modifies a system template. The variant tracks which template version it was forked from.
*   **Compliance-Locked Steps**: HQ can mark specific steps as locked (🔒). These steps:
    *   Cannot be removed
    *   Cannot be reordered past a defined boundary (e.g., "ส่งรายงาน" must always be the final step)
    *   Can have their timing adjusted within limits
    *   Outcome transitions on locked steps can be viewed but not modified
*   **Template Version Sync**: When HQ publishes a new template version, branches with variants receive a notification to review changes. The system highlights what changed so the supervisor can merge.

### Supervisor Edit Capabilities

| Allowed | Not Allowed |
|---------|-------------|
| Drag & drop to reorder steps | Delete compliance-locked (🔒) steps |
| Drag outcome transitions to different target steps | Edit system-level templates directly |
| Add optional steps (extra call, admin task) | Remove audit trail / compliance logging |
| Add/remove outcomes on non-locked steps | Modify outcome transitions on locked steps |
| Adjust wait timing (e.g., 3 days → 2 days) | Reorder locked steps past their boundary |
| Change assignee rules (same CO → escalate) | Bypass publishing workflow |
| Set retry limits on outcomes | |
| Remove non-locked steps | |

### Example Playbook: ติดตามหนี้ค้างชำระ (Delinquency Recovery)

| Step | Action | Timing | Outcome Transitions | Lock |
|------|--------|--------|---------------------|------|
| 1 | 📞 โทรศัพท์ครั้งที่ 1 | Day 1 | PTP → End ✅ · No Answer → Retry (max 3) then → Step 5 · Refused → Step 6 | — |
| 2 | ⏳ รอ | 3 days | (auto) → Next | — |
| 3 | 📞 โทรศัพท์ครั้งที่ 2 | Day 4 | PTP → End ✅ · No Answer → Step 5 · Refused → Step 6 | — |
| 4 | ⏳ รอ | 3 days | (auto) → Next | — |
| 5 | 🏠 ลงพื้นที่ | Day 7 | Met & PTP → End ✅ · Not Home → Step 6 · Refused → Step 6 | — |
| 6 | 📋 ส่งรายงาน | Day 8 | Completed → Next | 🔒 |
| 7 | 🔔 แจ้งหัวหน้า | Auto | Acknowledged → End | 🔒 |

---

## 2. Core Capability: Task Engine

**Goal**: Translate playbook steps and business events into discrete, assignable tasks with lifecycle tracking, SLA enforcement, and outcome recording.

### Why It Exists (First Principles)

*   **Accountability**: Every interaction with a customer must be recorded. Without a task record, there is no audit trail of attempts, outcomes, or compliance adherence.
*   **Throughput Visibility**: Management cannot measure what is not tracked. Tasks provide the atomic unit of measurement for staff productivity.
*   **Automation**: Tasks generated automatically from events and playbooks eliminate reliance on supervisors manually assigning work.

### Task Lifecycle

```
  ┌──────────┐   assign   ┌──────────┐   start   ┌──────────┐   record    ┌──────────┐
  │ CREATED  │ ─────────► │ ASSIGNED │ ────────► │ ACTIVE   │ ──────────► │ CLOSED   │
  └──────────┘            └──────────┘           └──────────┘            └──────────┘
       │                       │                      │
       │                       │ reassign             │ escalate
       │                       ▼                      ▼
       │                  ┌──────────┐          ┌──────────┐
       │                  │ ASSIGNED │          │ ESCALATED│
       │                  │ (new CO) │          └──────────┘
       │                  └──────────┘
       │
       │ expire (SLA)
       ▼
  ┌──────────┐
  │ OVERDUE  │ → surfaces in supervisor exception panel
  └──────────┘
```

### Task Properties

*   **Source**: `playbook_step`, `event_rule`, `manual` (supervisor-created)
*   **Action Type**: `Call`, `Visit`, `Admin`, `Review`, `Notify`
*   **Priority**: Derived from DPD bucket, playbook urgency, and SLA proximity
*   **Outcome**: Typed per action (e.g., Call outcomes: `PTP`, `No Answer`, `Refused`, `Callback`, `Wrong Number`)
*   **SLA**: Configurable deadline. Breached tasks surface in supervisor exception panel.
*   **Linked Entities**: `customer_id` (DaVinci), `contract_id`, `playbook_id`, `playbook_step_id`

### Event-Driven Task Generation

*   **Event Rules**: System or branch-defined rules that create tasks from external events:
    *   `Delinquency > 30 DPD` → Create Call task from "Delinquency Recovery" playbook
    *   `Insurance expiry within 15 days` → Create Call task from "Insurance Renewal" playbook
    *   `KYC expiry within 30 days` → Create Admin task for document collection
*   **Event Sources**: DaVinci (customer events), Core Banking (delinquency events), Policy Admin (renewal events)

### Supervisor Task Controls

*   **Reassign**: Move a task from one CO (Collection Officer) to another
*   **Override Playbook Step**: Skip or insert a step for a specific customer's playbook instance
*   **Add Manual Task**: Create a one-off task not tied to any playbook
*   **Bulk Operations**: Reassign all tasks from one CO to another (e.g., staff absence)

---

## 3. Core Capability: Work Queue (High-Volume Processing)

**Goal**: Present field staff with a prioritized, grouped work queue optimized for processing 300-500 customers per CO per day with minimal friction.

### Why It Exists (First Principles)

*   **Cognitive Load**: A flat list of 300 tasks is overwhelming. Grouping by action type reduces cognitive switching (calls together, visits together).
*   **Speed**: Collection officers must process tasks rapidly. The UI must minimize clicks, pre-load context, and provide quick outcome entry.
*   **Prioritization**: Not all tasks are equal. The queue must surface overdue items first, then high-DPD customers, then normal flow.

### Queue Structure

```
  ┌─────────────────────────────────────────┐
  │  📊 Overview: 28 tasks | 12 done | 3 ⚠️ │
  ├─────────────────────────────────────────┤
  │                                         │
  │  📞 Calls (22)          [Start Queue ▶] │
  │    🔴 Overdue (3)                       │
  │    ⚡ High Priority — DPD > 60 (5)      │
  │    📋 Normal — DPD 30-60 (8)            │
  │    🔔 Insurance Renewal (4)             │
  │    📞 Scheduled Callbacks (2)           │
  │                                         │
  │  🏠 Visits (4)          [Plan Route 📍] │
  │    ⚡ Contact Failed × 3 (3)            │
  │    📋 Scheduled Follow-up (1)           │
  │                                         │
  │  📖 Admin (2)           [View All]      │
  │    📖 Read & Confirm Policy v2.3        │
  │    📋 Submit Weekly Report              │
  │                                         │
  └─────────────────────────────────────────┘
```

### Primary Processing: One-by-One Task Execution

The **primary method** of working through the queue is **one task at a time**. A CO selects a task from any bucket and processes it individually:

1. **Select a task** from the grouped queue list
2. **View full context**: Customer name, phone, loan summary, DPD, contact history, previous notes
3. **Execute the action** (make the call, conduct the visit, complete the admin task)
4. **Record the outcome**: Select from the action's defined outcomes (e.g., PTP, No Answer, Refused)
5. **Fill conditional fields**: Required fields based on outcome (e.g., PTP → amount + date)
6. **Save and return** to the queue, where the completed task is removed and the next appears

This ensures staff always have full context and can handle complex cases that require judgment.

### Extended Capability: Rapid-Fire Processing Mode

> **Note**: This is an **extended capability** — an accelerated processing mode for experienced COs who need to work through high-volume queues quickly. It is not the default method.

When a CO clicks "Start Queue ▶" on a bucket (e.g., Calls), the system enters rapid-fire mode:

1. **Single-card focus**: One customer at a time, full context pre-loaded
2. **Quick outcome entry**: Large tap targets for common outcomes
3. **Auto-advance**: "Save → Next ▶" progresses to the next customer immediately
4. **Progress bar**: "3 of 22" with completion percentage
5. **Contact compliance**: Real-time display of daily contact count. Blocks further calls if limit reached.

Rapid-fire mode can be exited at any time to return to the standard one-by-one queue view.

### Key Business Rules

*   **Daily Contact Limit**: Maximum contacts per customer per day (configurable, default: 2). Enforced by the system — task is auto-skipped if limit reached, with visual indicator.
*   **Priority Sorting**: Within each bucket, tasks are sorted by: Overdue → High DPD → Playbook urgency → Registration date
*   **Supervisor Message**: Daily motivational/tactical message from supervisor displayed at queue top

---

## 4. Core Capability: Performance & Visibility Dashboard

**Goal**: Provide real-time visibility into team operations for supervisors and self-service performance tracking for individual staff.

### Why It Exists (First Principles)

*   **Management by Exception**: Supervisors managing 10-15 staff and 3,000+ tasks cannot review everything. They need to see *exceptions* — who is behind, which playbooks are failing, what needs intervention.
*   **Staff Motivation**: Transparency in personal performance metrics and gamified rankings (leaderboard) drive healthy competition and self-improvement.
*   **Accountability Chain**: Branch → Area → Region aggregation enables consistent performance reporting upward.

### Supervisor Dashboard

| Component | Purpose |
|-----------|---------|
| **Team Workload Table** | Each staff member: queue size, completed, completion rate, PTP amount, alert flags |
| **Active Playbooks** | Each playbook: case count, on-track / at-risk / failed / succeeded, progress bar |
| **Exception Panel** | Alerts requiring supervisor action: idle staff, blocked contacts, pending approvals |
| **Daily Scorecard** | Metrics aggregated at team level: today / this week / this month / vs. target |
| **Contact Compliance** | Real-time compliance status: 100% = no violations |

### Exception Alert Types

| Alert | Trigger | Supervisor Action |
|-------|---------|-------------------|
| Idle Staff | No activity for > 1 hour during work hours | Review, send message |
| Low Performance | Completion rate < 40% at midday | Intervene, reassign |
| Contact Blocked | Customer reached daily contact limit | Acknowledge, plan next-day |
| Playbook Stuck | Playbook step awaiting supervisor approval | Approve / reject |
| SLA Breach | Task overdue by > 4 hours | Reassign or escalate |

### Staff Self-Service (My Performance)

| Component | Purpose |
|-----------|---------|
| **Metrics Dashboard** | Personal progress bars: tasks completed, PTP rate, visit success, SLA compliance |
| **Monthly Objectives** | Count of succeeded / in-progress / failed objectives from assigned playbooks |
| **Branch Rank** | Gamified ranking within the branch with medal indicators (🥇🥈🥉) |
| **Team Leaderboard** | Ranked list of team members with task count and PTP rate |
| **Supervisor Feedback** | Messages and coaching notes from supervisor |

---

## 5. Cross-Cutting Concern: Contact Compliance & Action Verification

**Goal**: Ensure all customer interactions comply with Thai debt collection regulations regarding contact frequency, timing, and methods — and verify that recorded actions actually occurred.

### Rules Enforced

| Rule | Source | Enforcement |
|------|--------|-------------|
| Max contacts per debtor per day | Thai Debt Collection Act | Task auto-skipped if limit reached |
| No contact outside business hours | Thai Debt Collection Act | Tasks not assignable outside hours |
| Contact logging | PDPA / Internal policy | Every contact attempt recorded with timestamp + channel + outcome |
| Contact frequency across products | Cross-product (via DaVinci) | DaVinci provides combined contact history; Sensei checks before allowing task |

### Action Verification (External System Cross-Check)

To ensure recorded outcomes are genuine (not fabricated), Sensei cross-references task outcomes against external system logs where available:

| Action Type | Verification Source | What Is Checked |
|-------------|--------------------|-----------------|
| 📞 Call | **3CX Phone System** (call logs) | Call was actually placed, duration > 0s, caller ID matches CO, timestamp matches |
| 🏠 Visit | GPS / check-in system (future) | Location matches customer address within radius |
| 📧 Notification | Email/SMS gateway logs | Message was actually sent and delivered |

**3CX Integration**:
*   Sensei queries the 3CX call log API to verify that a call task marked as "PTP" or "No Answer" corresponds to an actual outbound call record.
*   **Verification statuses**: `✅ Verified` (call log matches), `⚠️ Unverified` (no matching log found), `❌ Mismatch` (log contradicts recorded outcome — e.g., CO recorded "PTP" but call duration was 0 seconds).
*   Unverified and mismatched tasks are surfaced in the **Supervisor Exception Panel** for review.
*   This is a **trust but verify** model — it does not block the CO from recording outcomes, but flags discrepancies after the fact.

### Integration with DaVinci

Sensei queries DaVinci for the **combined contact count** across all products for a given customer before allowing a contact task to proceed. This prevents scenarios where Subsidiary A calls a customer twice and Subsidiary B calls them twice on the same day — exceeding the legal limit.

---

## 6. Cross-Cutting Concern: Template Library

**Goal**: Provide a library of reusable task and step definitions that standardize how work is described and measured across the organization.

### Templates Define

*   **Action Types and Their Outcomes**: e.g., "Call" has outcomes {PTP, No Answer, Refused, Callback, Wrong Number}
*   **Required Fields per Outcome**: e.g., PTP requires amount + date
*   **SLA Defaults**: e.g., Call tasks default to 4-hour SLA
*   **Escalation Rules**: e.g., After 3 failed calls → auto-escalate to Visit

Templates are managed by HQ and referenced by Playbook Steps. Supervisors cannot modify templates directly — they customize through Playbook Variants.

---

## Resolved Design Decisions (ADR Style)

| # | Decision | Context | Consequence |
|---|----------|---------|-------------|
| D1 | **Vertical step pipeline, not Kanban board** | Playbook steps are strictly sequential; order matters. | Simpler mental model for supervisors. Drag & drop is vertical reordering, not category movement. |
| D2 | **Branch-variant model, not in-place editing** | Supervisors need flexibility, but HQ needs control over the canonical strategy. | Supervisors fork a system template into a branch variant. HQ can push template updates; branches decide whether to merge. Adds version management complexity but prevents policy drift. |
| D3 | **Compliance-locked steps** | Certain steps (reporting, supervisor notification) are legally or operationally mandated. | Locked steps cannot be removed or reordered past boundaries. Reduces supervisor flexibility slightly but ensures compliance. Reversible: HQ can unlock a step if policy changes. |
| D4 | **Linear default + outcome routing (Option C)** | Pure linear sequences can't handle failed conditions (No Answer vs PTP lead to different next steps). Full flowcharts are too complex for supervisors. | Each step carries outcome transition rules that override the default next step. Visual remains a vertical pipeline; outcomes appear as a sub-list per step. Supervisors drag outcome targets to different steps. Balances expressiveness with simplicity. |
| D5 | **Task Engine is event-driven, not batch-scheduled** | Branch operations are real-time — delinquency changes, payments, and renewals happen continuously. | Tasks are created from events (via DaVinci), not nightly batch jobs. Higher infrastructure complexity but enables same-day response. |
| D6 | **One-by-one as primary, rapid-fire as extended** | COs handle 300-500 customers/day but complex cases need full context. | One-by-one processing is the default. Rapid-fire is an optional accelerated mode for experienced COs. Prevents mistakes from rushing while enabling throughput for skilled users. |
| D7 | **Gamified leaderboard** | Staff motivation in high-volume roles benefits from visible competition. | Leaderboard with medals drives engagement. Risk: may create unhealthy competition — mitigated by combining rank with supervisor feedback and collaborative metrics. |
| D8 | **Contact compliance at Sensei layer, not DaVinci** | DaVinci provides raw data (contact history); compliance enforcement is an operational concern. | Sensei checks DaVinci before every contact task. Clear separation: DaVinci = data, Sensei = rules. If compliance rules change, only Sensei needs updates. |
| D9 | **Trust-but-verify via 3CX cross-check** | COs could fabricate outcomes (record "called" without calling). Direct blocking would slow down legitimate work. | Sensei cross-references 3CX call logs after the fact. Flags discrepancies in supervisor exception panel. Does not block COs in real-time — maintains throughput while enabling accountability. |
| D10 | **Product name: Sensei** | The system guides and structures branch operations — it teaches the organization how to work effectively. | Clear metaphor. Aligns with Japanese-themed naming convention (先生). |
