# Product Catalog

> Master registry of all products managed in this repository.

| Product | Codename | Status | ATLAS | Summary |
|---------|----------|--------|-------|---------|
| Document Verification Service | **Matcha** (抹茶) | ✅ Active | [ATLAS](matcha/ATLAS.md) | Universal, domain-agnostic document verification engine. 4-state task lifecycle, flexible rule configuration, async car check integration, SHA-256 change detection, re-flow support, and AI-first verification routing. Includes [ARCHITECTURE](matcha/ARCHITECTURE.md). |
| AI Document Verification Service | **Wasabi** (わさび) | 📝 Draft | [ATLAS](wasabi/ATLAS.md) | Stateless LLM-based document verification engine. Processes document images against Matcha-owned verification instructions. 4-stage pipeline: quality assessment → type classification → instruction-based verification → report assembly. Operates outside Matcha, returns results to Onigiri for early warnings and downstream Matcha routing. |
| Customer & Product Master Data | **DaVinci** (ダヴィンチ) | 📝 Draft | [ATLAS](davinci/ATLAS.md) | Centralized CRM master data store — the enterprise Golden Record. Consent-based data visibility (PDPA), event-driven sync from Core Banking & Policy Admin, customer data change management, and collection contact compliance. Single point of query for all downstream services. Includes [ARCHITECTURE](davinci/ARCHITECTURE.md). |
| Branch Operations Orchestration | **Sensei** (先生) | 📝 Draft | [ATLAS](sensei/ATLAS.md) | Branch staff productivity and operations platform. Playbook-driven task orchestration (multi-step strategies with drag-and-drop editing), event-driven task engine, high-volume work queue with rapid-fire processing, supervisor dashboard with exception management, gamified performance tracking, and Thai debt collection contact compliance via DaVinci integration. |
