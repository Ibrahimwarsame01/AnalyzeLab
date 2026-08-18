# AnalyzeLab — Product Requirements Document

**Hackathon:** All Things Agentic (Gemini / Google Cloud)
**Track:** Fortified Enterprise Fleet
**Build window:** 2 weeks
**Version:** 1.0

---

## 1. Problem Statement

Counterfeit and diverted drugs are actively entering the legitimate U.S. pharmaceutical supply chain from counterfeit Ozempic containing non-sterile needles, to gray-market Botox traced through a Texas med spa in an FDA case that produced the first-ever DSCSA dispenser warning letter.

Under the Drug Supply Chain Security Act (DSCSA), trading partners (manufacturers, distributors, dispensers) are legally required to exchange package-level verification data electronically and investigate suspect products within a strict window, notifying the FDA of illegitimate product within 24 hours. This exchange largely does happen electronically today via EPCIS 2.0 and the Verification Router Service (VRS) network — but in practice it's fragile: delayed messages, mismatched master data, broken system handoffs, and middleware failures mean many organizations still fall back on manual checks and after-the-fact reconciliation. The failures are frequently caught only after a shipment has already arrived.

**The gap:** existing automation verifies data mechanically (does this serial number match?) but doesn't reason about it — investigate anomalies, correlate signals across a multi-day case, or produce a defensible, auditable decision trail a regulator or partner could inspect. That reasoning-and-governance layer is what AnalyzeLab adds.

## 2. Solution Overview

AnalyzeLab is a governed fleet of AI agents, built on Google ADK and deployed on the Gemini Enterprise Agent Platform (GEAP), that sits on top of a trading partner's existing verification infrastructure. It verifies incoming pharmaceutical products against a mock EPCIS/VRS-style registry, investigates anomalies with persistent multi-day case memory, resists tampering/spoofing attempts, and auto-drafts the FDA-required suspect/illegitimate product notification — all while producing a full, inspector-reconstructable audit trail.

**Note on positioning:** Commercial "AI agent for counterfeit detection" products already exist in the market. AnalyzeLab's differentiation is not "AI detects counterfeits" — it's the **governed multi-agent architecture**: distinct cryptographic identity per agent, a discoverable agent registry, policy-enforced routing, guardrails against poisoned/spoofed data, and a full reasoning-chain audit trail built specifically on GEAP's Fortified Enterprise Fleet primitives. Lead every pitch with this framing.

## 3. Goals & Success Criteria

| Goal | Success Metric |
|---|---|
| Verify legitimate products fast | Sub-few-second verification response for a clean product |
| Catch and investigate suspect products | Agent correctly flags a mismatched/anomalous product and opens an investigation case |
| Resist adversarial input | Model Armor blocks a spoofed/tampered verification response live in demo |
| Produce regulator-grade audit trail | Every agent action traceable to identity, reasoning, and human-visible log entry |
| Persist multi-day investigations | Memory Bank correctly recalls case context across two separate invocations |
| Demonstrate full GEAP governance stack | Agent Identity, Registry, Gateway, Model Armor, Observability, Memory Bank all visibly working in the demo |
| Deploy on real GCP infra | Cloud Run / Agent Runtime deployment provably running, shown in Cloud Console |

## 4. Non-Goals (Explicitly Out of Scope)

- Real integration with an actual manufacturer's live EPCIS/VRS system (simulated instead — this is legitimate per DSCSA architecture research: real trading partners are not required to be AI agents themselves)
- Real FDA submission of any notification (draft/mock only)
- Physical/chemical counterfeit detection (spectroscopy, packaging analysis) — this is a data/verification play, not a lab-testing play
- Multi-jurisdiction support (EU FMD, etc.) — U.S. DSCSA only
- Production-grade security hardening beyond what's needed to demo the governance stack convincingly

## 5. User & Stakeholder Personas

- **Distributor compliance officer** — needs fast verification and a defensible audit trail for regulators
- **Pharmacy/dispenser staff** — needs anomalies flagged before dispensing to a patient
- **FDA inspector (implicit persona)** — needs to be able to reconstruct any decision after the fact

## 6. System Architecture

### 6.1 Agent Fleet

| Agent | Role | Identity Scope |
|---|---|---|
| **Orchestrator Agent** | Receives incoming product "receipt events," routes to Verification, escalates to Investigation | Read-only on events; write to routing log only |
| **Verification Agent** | Checks product identifiers (GTIN, lot, serial) against the mock EPCIS/VRS registry | Read-only on registry |
| **Investigation Agent** | Triggered on verification failure or anomaly; reasons over context, decides false-alarm vs. genuine suspect product | Read on registry + case history; write to case records |
| **Reporting Agent** | Drafts the FDA illegitimate-product notification when Investigation confirms a suspect product | Only agent authorized to generate/finalize the notification artifact |

### 6.2 Data Layer
- **Firestore** — product/lot/serial registry (simulated EPCIS data), transaction/receipt event log, investigation case records
- **Vertex AI Memory Bank** — persistent cross-session context for investigations spanning multiple days/invocations

### 6.3 Governance Layer (GEAP)
- **Agent Identity** — distinct scoped GCP IAM identity per agent; Verification is read-only, Reporting is the sole write-authorized agent for filings
- **Agent Registry** — all four agents cataloged, versioned, discoverable
- **Agent Gateway** — routes all inter-agent and external (simulated trading-partner) calls, enforces policy
- **Model Armor** — screens all inputs/outputs for prompt injection, tool poisoning, and PII leakage; specifically demoed against a spoofed verification response attack
- **Agent Observability** — OpenTelemetry traces to Cloud Trace/Cloud Logging; powers an "inspector mode" reconstruction view

### 6.4 Communication Pattern
- **A2A protocol** between AnalyzeLab's own agents
- A **plain REST/Cloud Function endpoint** simulates the counterparty trading partner's EPCIS/VRS system — intentionally NOT another AI agent, matching real-world DSCSA infrastructure

### 6.5 Frontend
- Lightweight dashboard (React or plain HTML/JS) showing: incoming products, verification status, flagged investigations, generated reports, and the audit trail/reasoning-chain view
- Hosted on Firebase Hosting or Cloud Run, calling the backend API

### 6.6 Model
- **Gemini 3.5 Flash** as default for all agents
- **Gemini Pro** reserved only if Investigation Agent reasoning needs deeper analysis

## 7. Key Demo Scenarios (Must-Have for Video)

1. **Happy path** — legitimate product received, verified instantly, no escalation
2. **Suspect product** — mismatched lot/serial triggers Investigation Agent, which opens a case and correctly concludes it's a genuine suspect product
3. **Adversarial spoofing** — a tampered/spoofed verification response is sent to the Verification Agent; Model Armor / Agent Gateway blocks it live on camera
4. **Multi-day investigation** — a case is opened on "day 1," a second invocation on "day 2" (new evidence) shows Memory Bank correctly recalling prior case state
5. **Inspector mode** — click into any of the above actions and show the full reasoning-chain / audit-trail entry, including which agent identity performed it and why

## 8. 2-Week Build Plan

### Week 1 — Core agent logic + working end-to-end flow

| Day | Deliverable |
|---|---|
| 1 | GCP project set up, billing alerts on, ADK installed (`pip install google-adk`), Gemini API key working. Data model defined (product, transaction, verification request — simplified EPCIS-style fields: GTIN, lot, serial, expiration). |
| 2 | Verification Agent built — takes a product ID, checks against mock registry (Firestore), returns pass/fail. Working request→response loop locally. |
| 3 | Investigation Agent built — triggered on verification failure/anomaly, reasons over context, classifies as false-alarm / needs-more-info / genuine suspect. |
| 4 | Orchestrator Agent built and wired to both. Full chain deployed to Cloud Run via Agent Runtime. |
| 5 | Reporting Agent built — drafts FDA-style illegitimate-product notification when Investigation confirms suspect product. |
| 6–7 | Buffer / catch-up. Begin Memory Bank wiring for multi-day case persistence. |

### Week 2 — Governance layer + polish

| Day | Deliverable |
|---|---|
| 8 | Agent Identity — distinct scoped IAM permissions per agent. |
| 9 | Agent Registry — all agents cataloged/versioned. |
| 10 | Agent Gateway + Model Armor — build and test the spoofed-verification-response attack scenario; confirm it's blocked. |
| 11 | Agent Observability — OpenTelemetry traces visible in Cloud Trace; build the "inspector mode" reconstruction view. |
| 12 | Frontend dashboard finished; README + architecture diagram written. |
| 13 | Record ~4-minute demo video: open with the real counterfeit-Ozempic/med-spa stakes → happy path → suspect product caught → spoofing attempt blocked → report generated → GCP dashboard proof. |
| 14 | Buffer day — fix issues from recording, finalize public repo, submit. |

## 9. Submission Checklist (per hackathon requirements)

- [ ] Hosted project URL (if available)
- [ ] Text description: problem, features, technologies used, other data sources, findings/learnings
- [ ] Public (or shared) code repository URL
- [ ] README.md with spin-up instructions (local + cloud deploy)
- [ ] Architecture diagram (Gemini ↔ agents ↔ backend ↔ database ↔ frontend)
- [ ] ~4-minute demo video showing problem, value prop, live demo, and proof of Google Cloud backend (Cloud Console / Cloud Run dashboard / Vertex AI logs)

## 10. Risks & Mitigations

| Risk | Mitigation |
|---|---|
| Scope too large for 2 weeks | Non-goals section strictly enforced; only 4 demo scenarios required, not full coverage |
| GEAP components in preview/shifting | Confirm GA/preview status of each service (Agent Gateway, Memory Bank) early in Week 1; have a fallback (e.g., custom IAM policy) if a component is unavailable |
| Demo video fails to convey stakes | Open with the real counterfeit-Ozempic/med-spa narrative before any technical explanation |
| Judges unfamiliar with DSCSA | Keep the in-video explanation to under 20 seconds, plain language, before diving into the system |
| Running out of cost budget | Min instances = 0 on Cloud Run, budget alerts on Day 1, turn off resources immediately after demo recording |

## 11. Open Questions

- Exact schema for the simulated EPCIS/VRS counterparty endpoint — finalize on Day 1
- Whether to show two AnalyzeLab agents talking A2A to each other AND to the plain EPCIS/VRS endpoint (recommended, if time allows, to prove heterogeneity)
