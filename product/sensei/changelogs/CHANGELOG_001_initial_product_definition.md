# CHANGELOG 001 — Initial Product Definition

**Date**: 2026-02-24
**Author**: @ARCHITECT
**Status**: 📝 Draft

---

## Summary

Initial capability definition for **Sensei (先生)** — the Branch Operations Orchestration Platform. Defines the complete capability surface for managing branch field staff productivity, playbook-driven task orchestration, and supervisor visibility.

## New Capabilities

### 1. Playbook Engine
- Multi-step operational strategies (Playbooks) as ordered step sequences
- Step types: Call, Visit, Wait, Admin, Notify, Send Notification
- Playbook hierarchy: System Template (HQ) → Branch Variant (Supervisor)
- Compliance-locked steps: HQ can lock steps that cannot be removed
- Supervisor editing: reorder (drag & drop), add/remove steps, adjust timing
- Template version sync: branches notified on HQ template updates

### 2. Task Engine
- Task lifecycle: Created → Assigned → Active → Closed (with Overdue and Escalated states)
- Event-driven task generation from DaVinci, Core Banking, and Policy Admin events
- SLA enforcement with supervisor exception surfacing
- Supervisor controls: reassign, override playbook step, add manual task, bulk operations

### 3. Work Queue (High-Volume Processing)
- Grouped action buckets: Calls, Visits, Admin
- Priority sub-groups within each bucket: Overdue → High DPD → Normal → Scheduled
- Rapid-fire processing mode: single-card focus, quick outcomes, auto-advance
- Daily contact compliance enforcement (auto-skip if limit reached)

### 4. Performance & Visibility Dashboard
- Supervisor dashboard: team workload table, active playbooks, exception alerts, daily scorecard
- Staff self-service: personal metrics, monthly objectives, branch rank, leaderboard, supervisor feedback
- Management by exception: alerts for idle staff, low performance, contact blocks, SLA breaches

### 5. Contact Compliance (Cross-Cutting)
- Daily contact limits per customer (Thai Debt Collection Act)
- Cross-product contact counting via DaVinci integration
- Contact logging with timestamp, channel, and outcome

### 6. Template Library (Cross-Cutting)
- Reusable action type definitions with typed outcomes
- Required fields per outcome, SLA defaults, escalation rules
- Managed by HQ, referenced by Playbook Steps

## Design Decisions

| # | Decision | Rationale |
|---|----------|-----------|
| D1 | Vertical step pipeline for playbook editing | Sequential steps; simpler mental model than Kanban |
| D2 | Branch-variant model for customization | HQ control + branch flexibility; fork-and-merge pattern |
| D3 | Compliance-locked steps | Legally mandated steps cannot be removed by supervisors |
| D4 | Event-driven task generation | Real-time response to delinquency, payments, renewals |
| D5 | Grouped queue with rapid-fire mode | Reduces cognitive load for 300-500 customer/CO ratio |
| D6 | Gamified leaderboard | Motivation for high-volume roles; balanced with coaching |
| D7 | Contact compliance at Sensei layer | DaVinci = data, Sensei = rules; clear separation |
| D8 | Product name: Sensei (先生) | Guides and structures; aligns with Japanese naming |
