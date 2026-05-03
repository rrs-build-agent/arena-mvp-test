# RRS_CADENCE_PROJECT.md

> **Project:** Roof Repair Specialists — Inbound Lead Follow-Up Cadence
> **Status:** Pre-Session 1 — gated on Salesforce baseline data pull
> **Last updated:** 2026-05-02

---

## 1. PROJECT FRAME

### One-line frame
Replace the voice-only inbound cadence with a multi-channel SMS + voice + email cadence — source-aware, persona-aware, with a global kill switch on any reply.

### Problem (quantified)
- ~80% of inbound leads never connect on first dial.
- Current cadence: 22 dials in first 9 days, then 1/day perpetually. Voice only.
- No SMS. No email. No channel variation. No persona switching.
- Voice = the channel consumers avoid. SMS = the channel they prefer (~98% open / ~45% reply).
- Lost revenue is invisible because these prospects never enter a CRM stage.

### Lead sources in scope
| Source | Posture | Notes |
|---|---|---|
| Third-party aggregators | Fast + aggressive | Shared with competitors. Speed-to-lead is everything. |
| Web form / website | Fast + warm | Intent declared. TCPA opt-in collected at form. |
| Google LSA | High-intent, phone-first | Often inbound calls; cadence triggers only on no-connect. |
| Facebook/Meta lead forms | Cold + slow-burn | Lower intent, longer nurture. |
| Referrals & call-ins | Soft touch | Owner/manager persona early. SMS only after explicit opt-in. |

### Design principles (locked)
- **Multi-channel:** SMS + voice + email, each doing what it does best.
- **Global kill switch:** any reply on any channel halts ALL outbound automation immediately. Centralized in n8n.
- **Persona-aware:** CSR → owner/manager handoff at the right point.
- **Source-aware pacing.**
- **Compliance-first:** 10DLC, TCPA, opt-out detection beyond keyword "STOP."
- **MVP first.** Prove on one source, then expand.

---

## 2. ARCHITECTURE DECISIONS (LOCKED)

- **n8n orchestrates.** All cadence state machines live there.
- **Salesforce holds state.** Lead object + custom fields:
  - `Cadence_Status__c` (Active / Paused / Killed / Cold)
  - `Cadence_Source__c` (Aggregator / Web / LSA / Meta / Referral)
  - `Last_Channel__c` (SMS / Voice / Email)
  - `Kill_Switch__c` (Boolean — ANY reply flips this)
  - `Opt_Out__c` (Boolean — TCPA-grade flag)
  - `Persona_Stage__c` (CSR / Owner)
  - Field names indicative; finalize at deploy.
- **Heymarket = SMS execution + shared inbox.** Inbound replies → webhook → n8n → Salesforce update + kill switch flip.
- **Aircall = voice execution.** Cadence triggers dial tasks; dispositions write back to Salesforce.
- **Email = Google Workspace via n8n** (Gmail API), tracked back to Lead.
- **Kill switch is single source of truth in n8n** — reused by Unsold Estimate Rehash later.
- **Custom objects/fields namespaced for reuse**, not hardcoded to "inbound."

---

## 3. MVP SCOPE & SUCCESS METRIC

### MVP scope (Phase 1)
- **One source live end-to-end:** aggregator leads (highest volume, highest pain).
- **Channels:** SMS + voice. Email = Phase 2.
- **Persona:** CSR → Owner handoff at Day 3 (revised from Day 4).
- **Kill switch:** fully functional across SMS + voice.
- **10DLC:** registered + approved.
- **Salesforce custom fields:** deployed and writing.

### MVP success metric
Connect rate on aggregator leads ≥30% improvement over baseline within 30 days of go-live. Baseline measured during deliverable #1 audit.

### Phase 2+ roadmap
- Add web form, LSA, Meta, referral cadences.
- Add email channel.
- Add persona switching for non-aggregator sources.
- Add AI-assisted SMS drafting in Heymarket for CSR.
- Bridge to Unsold Estimate Rehash project on shared infrastructure.

---

## 4. WORKING CADENCE INTERVALS (AGGREGATOR MVP)

| Day | Channel | Persona | Intent | Exit condition |
|---|---|---|---|---|
| 0, min 0 | SMS | CSR | Speed-to-lead intro + appt ask | Any reply → kill switch |
| 0, min 5 | Voice | CSR | First dial | Connect → handoff to booking |
| 0, +2hr | Voice | CSR | Second dial | — |
| 0, +4hr | SMS | CSR | Soft follow-up | — |
| 1 | Voice (AM) + SMS (PM) | CSR | Re-engage | — |
| 2 | Voice + SMS | CSR | Different angle (problem framing) | — |
| 3 | Voice (AM) + SMS (PM) | CSR → Owner handoff | Owner "looping in" intro text | — |
| 4 | Voice + SMS | Owner | Personal touch | — |
| 5–7 | Voice + SMS, 1/day | Owner | Personal touch | — |
| 8–13 | 1 touch/day, alternating channels | Owner | Light persistence | — |
| 14 | Final SMS + voicemail | Owner | "Closing the file" message | → Cold |
| Cold | No outbound | — | — | Re-entry only on inbound activity |

---

## 5. OPEN DECISIONS

- **Owner persona carrier on aggregator leads:** Brett, designated rep under Brett's name, or designated rep as "manager." Pending decision.
- **Final Salesforce field API names** (current names indicative).
- **Kill-switch latency target** — hard SLA <60s or guideline?
- **Aircall in-flight dial cancellation** — possible via API or accepted race condition?
- **Email sending mechanism** — Gmail API direct or n8n SMTP via Workspace?
- **Soft opt-out classifier model** — Claude Haiku (cheap/fast) or Sonnet (more accurate)?
- **Cold-file re-entry behavior** — resume, restart, or human-only escalation?

---

## 6. RISKS TRACKED

- **10DLC approval lag** (2–4 weeks). Mitigation: start registration day 1.
- **Heymarket discipline.** If reps reply from personal phones, kill switch breaks. Mitigation: all SMS through Heymarket, enforced.
- **Aircall + n8n race conditions.** Kill switch must beat the dialer. Mitigation: n8n-side queue check before each Aircall trigger, not after.
- **CSR capacity.** Multi-channel doubles inbound reply volume. Mitigation: response SLA + AI-assisted drafts may need to land sooner than Phase 2.
- **Source attribution drift.** Dirty `Lead_Source__c` routes wrong cadence. Mitigation: audit data integrity in deliverable #1.
- **TCPA exposure on referrals/call-ins.** Often lack documented consent. Mitigation: voice + email only until explicit opt-in.
- **Forward-compat.** All custom objects/logic must be reusable for Unsold Estimate Rehash.

---

## 7. PROJECT OUTLINE — ARCHITECTURE SPEC FOR LAURA

### 7.1 Deliverable definition
**Done =** a single architecture document Laura can read once and start building from with no follow-up questions to Brett.

**Format:** one markdown doc with three embedded Mermaid diagrams (system data flow, cadence state machine, kill-switch event flow).

**Acceptance criteria — Laura can answer all of these from the doc alone:**
1. What Salesforce object, fields, and field types do I create?
2. What n8n workflows do I build, in what order, with what triggers and error handling?
3. What webhooks do I configure between Heymarket / Aircall and n8n, and what payloads?
4. What is the pre-step guard logic before every outbound touch?
5. What does the kill switch do, and how does it handle race conditions with in-flight Aircall dials?
6. What is the opt-out detection logic — keyword list + Claude classifier prompt?
7. What gets logged where, and how are MVP success metrics instrumented?
8. What is explicitly Phase 2 and should not be built now?
9. What forward-compat hooks exist for Unsold Estimate Rehash?
10. Where do I push work and how is it tested before going live?

### 7.2 Workstream breakdown

- **WS-1: Salesforce Data Model** — Define every custom field, picklist, validation rule on the Lead object. Foundation. No dependencies. 1 session.
- **WS-2: Cadence State Machine** — Lifecycle, transitions, triggers. Mermaid diagram + transition table. Depends on WS-1. 1 session.
- **WS-3: Channel Integration Spec** — n8n ↔ Heymarket / Aircall / Gmail at API + webhook level. Depends on WS-1. 2 sessions.
- **WS-4: Kill Switch & Opt-Out Architecture** — Centralized halt logic, opt-out detection, latency target, race-condition handling. Depends on WS-1, WS-3. 1 session.
- **WS-5: Cadence Execution Workflows (n8n)** — Lead intake, step scheduler, channel dispatcher, inbound listener. Depends on WS-1 through WS-4. 2 sessions.
- **WS-6: Compliance & Consent Layer** — TCPA / 10DLC posture. Web form consent, opt-in audit, DNC sync. Depends on WS-1. 1 session.
- **WS-7: Instrumentation & Reporting** — How the system measures itself. Depends on WS-1, WS-5. 1 session.
- **WS-8: Phasing & Forward-Compat Notes** — MVP vs Phase 2, Unsold Estimate Rehash hooks. Synthesis. Last. 0.5 session.

**Total: 6–8 sessions to Laura handoff.**

### 7.3 Dependency map

```
WS-1 (Salesforce Data Model)
   ├─→ WS-2 (State Machine) ─┐
   ├─→ WS-3 (Channel Integ) ─┤
   ├─→ WS-6 (Compliance)     ├─→ WS-5 (Execution Workflows) ─→ WS-7 (Instrumentation) ─→ WS-8 (Phasing) ─→ DONE
   └─→ WS-4 (Kill Switch) ───┘
```

**Critical-path unblockers — make these two decisions first:**
1. Final Salesforce field schema (WS-1) — every workstream reads from this.
2. Kill-switch latency target + race-condition policy (WS-4) — drives every outbound workflow design.

### 7.4 Working cadence
- **Session length:** 45 min, hard stop.
- **Frequency:** 2–3/week.
- **Format:** 5 min Maximus presents prepared output + decisions needed. 30 min Brett decisions + pivots. 10 min Maximus commits + queues next session.

### 7.5 Kickoff decision — gates Session 1
Before WS-1 can produce a final field schema, Brett must deliver:
1. Current Salesforce Lead schema (full field list).
2. 30-day baseline connect rate on aggregator leads.
3. 10DLC registration status confirmation.

### 7.6 Data Maximus needs from existing systems
- **Salesforce:** full Lead schema, `Lead_Source__c` picklist + data integrity %, 30-day connect rate on aggregator, 30-day appointment-set rate on aggregator, current FSL config.
- **Heymarket:** 10DLC registration status, current sender numbers, API tier, webhook config.
- **Aircall:** full disposition list, current Salesforce write-back behavior, in-flight cancellation API capability.

---

## 8. ASSUMPTIONS NEEDING BRETT'S CONFIRMATION

- Aggregator is highest-volume source.
- Current connect rate ~20% on first dial (aggregator-specific).
- 10DLC registration started.
- Heymarket shared inbox configured.
- Aircall Salesforce write-back already functioning.
- CSR has bandwidth for ~2x reply volume from new SMS layer.
