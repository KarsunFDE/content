---
week: W06
day: Wed
title: "Pre-session — Karsun-manager calibration + Thu gate format + Final Defense prep + Phase 3 commitment framing"
audience: All cohort members
time_on_task_minutes: 48
last_verified: 2026-05-26
---

# W6 Wed Pre-Session — Karsun-Manager Calibration + Thu Gate Format + Final Defense Prep + Phase 3 Commitment Framing

> Read before Wed morning. ~48 min. Wed = the documentation finalization push + HITL synthesis capstone + dry-run + **Final Defense pre-flight prep** (absorbed from original Thu morning per Cohort #1's Fri 3 Jul loss). This pre-session preps you for the Thu gate format + the Karsun-manager audience calibration (per D-060) so Wed's dry-run is the rehearsal, not the discovery — and Wed-evening's pre-flight checklist is the last surface before you defend.

## 1. Why D-060 picked Karsun managers (5 min)

For Cohort #1, the W6 Final Showcase audience is **Karsun managers** (3-5 internal, federal-acq-literate) — per `pipeline/DECISIONS.md` D-060.

The rationale matters because it shapes how you defend:

- **Realistic stakeholder pressure without external-client risk.** A Karsun manager is the closest available proxy to the people who'd actually staff your cohort onto a real federal engagement next quarter.
- **Federal-acq-literate, NOT pair-stack-deep.** Karsun managers know FAR, DFARS, FedRAMP, ATO, IDIQ, Source Selection, OIG, the difference between Section L and Section M of a solicitation. They do NOT know Resilience4j config knobs, LangGraph node-ID semantics, or your specific Pydantic schema choices.
- **Cohort #2 graduates to mock-client panel.** Karsun-manager-tier is the calibration floor for Cohort #1; future cohorts may face a sharper external panel.

What this means for your defense framing on Thu:

| Defend in these terms (Karsun manager will track) | Don't lead with (Karsun manager will glaze) |
|--------------------------------------------------|---------------------------------------------|
| "FAR 15.206 compels amendment-acknowledgement, so our HITL gate sits at the publish step" | "We chose `interrupt_before` over `interrupt_before_tool` because LangGraph's checkpoint semantics..." |
| "FedRAMP Moderate AC-2 + AU-2 are the controls our audit-log + RBAC work attests to" | "Spring Security 5 vs 6 differs in JwtDecoder configuration..." |
| "Cost shape per tenant is $0.X / 1k tokens; 12-month forecast at projected agency volumes is $Y" | "Bedrock invokeModel returns ResponseStream..." |
| "Two known weaknesses, both tracked as Findings, both ETA Phase 3 week 2" | "We didn't have time to implement..." (no Finding ID, no ETA) |

The fallback panel framing (per `assessments/client-showcase-rubric.md` §Fallback): if Karsun-manager panel can't be staffed Thu 2 Jul, degrade to instructor + 2 senior FDEs from another Karsun project. Same rubric, substitute panel briefed on D-060 calibration before showcase.

## 2. Thu gate format — minute-by-minute (15 min)

Per `weeks/W06/PLAN.md` §Thu + `war-room/D4-defense-showcase-retro.md`:

**09:00–12:00 — Final Defense per pair (~75 min × 3 pairs = 225 min, with 5-min rotation buffers).**

Per pair:
- **0:00–15:00 — Paired demo.** Walk through the *modernised* system. Phase 1 discoveries → Phase 2 changes → production-ready state. End with the eval report headline + the 2 known-weaknesses ranked by closure ETA.
- **15:00–25:00 — Agency CIO Q&A.** Instructor plays Agency CIO. Probes FedRAMP posture, deployment readiness, cost shape, security + failure-handling. Likely-asked: "What's our ATO timeline if we wanted to push this to GovCloud production tomorrow?" "What's the per-tenant 12-month cost forecast?"
- **25:00–35:00 — OIG Auditor Q&A.** Instructor plays OIG Auditor. Probes reproducibility, audit-trail completeness, who-decided-what. Likely-asked: "Show me your audit-log for the W4 Fri Workflow 4 incident — every state transition." "Who signed off on the eval-set construction?"
- **35:00–55:00 — Per-individual architecture defence (10 min each learner × 2 in the pair = 20 min).** Gate decisions are **per-individual**, not per-pair (D-035). Questions span Phase 1 + Phase 2 architecture choices. Each learner defends separately.
- **55:00–70:00 — Per-pair retrospective Q&A (15 min).** Cohort observes; observer pair poses 2 clarifying questions.
- **70:00–75:00 — 5-min rotation buffer.**

**13:00–14:30 — Client Showcase to Karsun managers.**

Per pair:
- **0:00–15:00 — Polished demo to Karsun manager panel.** Same demo as AM but reframed for staffing-decision-maker audience (per §1 above).
- **15:00–25:00 — Karsun-manager Q&A.** Panel probes: "Could a new FDE come up to speed on this in a week using just `docs/`?" "What would you de-scope if we cut the contract by 30%?" "What's the 30-second pitch to a federal CIO who's never heard of this?"

**14:30–15:30 — Gate decisions delivered (per-pair + per-individual).**

**15:30–17:00 — Cohort Retro + Phase 3 framing + cohort celebration.**

## 3. Final Defense Prep checklist (8 min) — *PDF Thu (Prep) absorbed into Wed*

Cohort #1 lost Fri 3 Jul to Independence Day observed, which moved Final Defense from Fri → Thu and absorbed the **PDF Thu morning Prep block** into Wed evening. There is no Thu-morning warm-up. You walk in Thu 09:00 and defend. This section is the pre-flight checklist you execute Wed evening so Thu morning is logistics-only.

The Prep block has three layers. Treat them sequentially — don't start layer 2 before layer 1 is locked.

### Layer 1 — Rehearsal close-out (immediately post-dry-run, ~30 min)

The dry-run (§4 below) is the **practice** of Tue's Live-Demo Discipline briefing. Prep is what you do with the dry-run notes:

- **Rank the 4 peer notes + 4 instructor notes by materiality.** Address the top 1–2 only. The rest go into `known-weaknesses.md` as honest-named gaps, not Wed-night rewrites.
- **Cut anything that ran long.** If your demo hit 17 min Wed it'll hit 20 Thu. Identify the 2-min cut you'll make and rehearse the trimmed version once.
- **Lock the demo-script reference card.** One page, large font, the 5–7 demo beats in order with their time-targets (e.g., `0:00 intro · 0:45 Phase 1 finding · 3:00 Phase 2 modernization hop · 7:00 eval headline · 9:00 known-weaknesses · 12:00 HITL boundaries · 14:30 close`). This card sits next to the laptop Thu morning — it is not the demo, it is the lifeline if you lose your place.

### Layer 2 — Graceful-degradation rehearsal (Wed evening, ~20 min)

Per Tue Live-Demo Discipline §"have-a-fallback": you decided Wed which fallback fires when. Prep is the actual **rehearsal** of that fallback:

- **Bedrock rate-limit fallback.** Pre-recorded video of the 3 LLM call paths cued up. Open the video file once Wed evening, confirm it plays, leave it minimised in the dock. If you've never invoked the fallback, the first time it fires will be Thu in front of Karsun managers — don't let that happen.
- **Network-degradation fallback.** Static screenshots of the same 3 call paths, saved locally (not Google Drive, not Confluence). One folder on the desktop.
- **Audit-trail-walkthrough fallback.** If `/admin/findings` UI fails to render Thu, the printed handoff package §"Findings index" is your substitute. Print it Wed evening — don't promise yourself you'll print Thu morning.

### Layer 3 — Logistics pre-flight (Wed evening, ~15 min)

Wed-evening final checklist. Each item is a yes/no — if any answer is "I'll do it Thu morning," redo Layer 3 now:

- **Laptop tabs pre-loaded.** All 8 handoff-package artifacts open in browser tabs in next-FDE reading order (`handoff.md` → `known-weaknesses.md` → `runbook.md` → `adrs/INDEX.md` → `eval/REPORT.md` → `security-attestation.md` → `hitl-authority-boundaries.md` → `threat-model.md`). Bookmark the tab group so a crash-recovery doesn't cost you 5 min Thu morning.
- **Handoff package printed.** One full printed copy per pair, in a binder or stapled set. Karsun managers may ask to flip through during Q&A — paper is faster than scrolling.
- **Demo-script reference card printed.** Layer 1's output, sitting next to the laptop.
- **Power + adapter + HDMI/USB-C confirmed.** Test the projector connection Wed evening if room access permits.
- **Arrival time locked.** 08:30 Thu for 09:00 kickoff. Room setup at 08:30 is non-negotiable per `war-room/D4-defense-showcase-retro.md` §Pre-event checklist.
- **Karsun-manager panel confirmation reviewed.** Instructor confirms panel status by Wed 17:30 (final lock per `war-room/D3.md` walkthrough). Fallback panel (instructor + 2 senior FDEs) is the substitute if confirmation fails — you do not need to act on this, but you should know which panel you're facing Thu PM.
- **Sleep.** Wed-evening intensive revision is capped at the top 1–2 dry-run notes. Past that, sleep beats polish — Karsun managers calibrate on whether you can think under pressure, not on the 3rd-decimal-place polish of your eval report.

> Cross-link to Tue Live-Demo Discipline: that section was the **briefing** ("here's how a federal-acq demo holds together"). This section is the **execution checklist** ("here's what you actually do Wed evening so Thu doesn't surprise you"). One without the other is a coin flip.

Source: composed from in-repo artifacts (`war-room/D3.md`, `training-resources/W06/D3-wed/walkthrough.md` §Wrap 17:15–17:30, `war-room/D4-defense-showcase-retro.md` §Pre-event checklist, `PLAN.md` §Wed + §Thu, canonical FDE-v2 programme spec §W6 Thu Prep) per `RESEARCH-PROTOCOL.md` §"What 'research' means here" — composing from cohort artifacts is not subject to /web-research routing.

## 4. Dry-run discipline (5 min) — *also absorbs PDF Thu Dry-run framing*

Wed late afternoon's dry-run is your **one** full pass. No second iteration (Cohort #1 compression cost the Thu morning buffer). What this means:

- **Time the demo.** 15 minutes. Practice the cut. If your demo runs 22 minutes Wed, it'll run 25 on Thu — over the Karsun-manager attention window.
- **Have a graceful-degradation plan.** If Bedrock is rate-limited Thu, what do you fall back to? Pre-recorded video of the demo? Static screenshots? Decide Wed, not Thu morning.
- **Bring the printed handoff package** (or laptop with all 8 artifacts already open in browser tabs). Karsun managers may ask to see the runbook live during Q&A — you don't want to be `grep`ing for it.
- **Each pair gets 1 critical + 1 supportive note from the other 2 pairs.** That's 4 notes total per pair from peers + 4 from instructor. Use the Wed-evening push to address the 1-2 most material.

The "broken-but-named-broken" rule from W6 Tue practical: if something breaks live Thu, name it. "That endpoint is rate-limited — let me show you the eval report on the same query instead." Beats hand-waving.

## 5. Phase 3 commitment card framing (10 min)

Thu EOD (15:30–17:00 block) each learner delivers a **Phase 3 commitment card** — a 1-pager naming the ONE improvement they'd ship in their first Phase 3 week.

Phase 3 = 6-month on-the-job development after the programme. Each pair's pair-project repo is archived as a cohort artifact and revisited across Phase 3.

Commitment card structure:

```
Phase 3 Commitment — <Learner Name>, <Pair>

Improvement: <one sentence — concrete, ticketed, deliverable>
Pair-project: <repo + branch>
Estimated effort: <day-range>
Tied to W6 known-weakness: Finding/F-2026-W6-NNN <yes / no — if no, name the Phase 3 motivation>
First Phase 3 week ship date: <date>
Karsun-manager review trigger: <when does this become showable?>
```

Examples of right-sized commitments (not aspirational, not trivial):
- "Close `Finding/F-2026-W6-002` — fix the Item 2 audit-log race in the SSDD-sign event by moving to outbox pattern. ~3 days."
- "Tune the W2 chunking strategy from sentence-window-10 to sliding-window-300-with-50-overlap; re-run RAGAS; if faithfulness improves > 5pp, promote. ~2 days."
- "Add a Pydantic schema to `/eval/ssdd-draft` to close the Item 4 share-drift surface. ~1 day."

Examples of WRONG-sized commitments:
- "Rewrite the whole agentic flow in CrewAI." (aspirational; not a first-week ship)
- "Add a docstring to `legacy_chain.py`." (trivial)

## 6. Cohort retro probes (5 min)

The cohort retro (Thu 15:30–17:00) starts from 8 anchor questions in `retros/cohort-retro-template.md`. Three you should pre-think:

1. *"Did the 6-week intensive feel like 6 weeks of intensive, or did it feel like 12 weeks of normal pace?"*
2. *"Where did Claude Code accelerate you the most? Where did it actively hurt?"*
3. *"Looking back at your W1 Tue diagnostic answer — would the W6-you give the same answer? What changed?"*

The retro is **honest, not polite.** Cohort #2 inherits your candor. The single best gift you give Cohort #2 is the things that didn't work + why.

## What you'll do W6 Wed

- 09:00–12:00 — War-room: FDE SME Q&A + codex handoff-package adversarial review fires.
- 13:00–16:00 — Documentation finalization push (eval report + known-weaknesses + handoff README + security attestation + **HITL authority-boundaries capstone**) + runbook/ADR catalog finalization.
- 16:00–17:30 — Dry-run (25 min × 3 pairs).
- 17:30 — Wed-evening push on the 1-2 most-material dry-run notes.
- Wed evening — **Final Defense Prep checklist (§3)**: rehearsal close-out → graceful-degradation rehearsal → logistics pre-flight. Layer 1 → Layer 2 → Layer 3 in order. Sleep beats polish.

---

Last verified: 2026-05-26
