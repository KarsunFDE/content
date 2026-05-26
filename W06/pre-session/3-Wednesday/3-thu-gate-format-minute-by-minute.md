---
week: W06
day: Wed
topic_slug: thu-gate-format-minute-by-minute
topic_title: "Thu gate format — minute-by-minute"
parent_overview: W06/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://www.smartsheet.com/content/design-review-process
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://cehhs.utk.edu/elps/wp-content/uploads/sites/9/2024/12/Doctoral-Defense-Handout-Final.pdf
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.noota.io/meeting-templates/design-meeting
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://demosecret.com/articles/how-to-demo-to-technical-audience
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Thu gate format — minute-by-minute

## 1. Learning Objectives

By the end of this reading, the learner can:

- Describe why time-boxed, phase-segmented review meetings outperform open-ended discussions for high-stakes technical defenses.
- Identify the canonical phases of a structured technical defense (presentation, examiner Q&A, audience Q&A, private deliberation, results) and what each phase is designed to accomplish.
- Explain why per-individual evaluation in a paired defense is a fundamentally different format from per-team evaluation, and why the transitions between phases matter as much as the phases themselves.
- Articulate the failure modes of unstructured review meetings (dominant-voice capture, off-course drift, incomplete-capture) and how minute-by-minute structure mitigates each.
- Map a generic time-boxed defense skeleton (intro / demo / structured Q&A / per-individual / retro / deliberation) to any specific defense format.

## 2. Introduction

A high-stakes technical defense — whether a doctoral dissertation, a product design review, a customer-facing solution review, or a final-stage interview panel — is a meeting type in which the *structure* is load-bearing. Two defenses of the same work can produce two different outcomes purely because of how the time was carved up. This is not a presentation-skills observation; it is a meeting-design observation. The form of the review shapes what the panel sees, what they remember, and what they decide.

Industry guidance on design review meetings is direct on this point. The traditional unstructured round-table format is described as having predictable failure modes: "the discussion can go off-course quickly if a participant interrupts the designer before they finish briefing... it is also challenging to capture all information during an open discussion, and louder participants may silence those quieter members" [1]. The contemporary alternative — variously called a "feedback workshop" or a "structured defense" — replaces ad hoc time allocation with explicit minute-by-minute phases and dedicated speaking turns. This is the meeting design that has become standard across academic dissertation defenses [2], engineering design reviews [1], and product design critiques [3].

For an engineer learning to defend their own work, the structure is not bureaucratic overhead. It is the framework that protects the defense from the modal failure modes — domination by one panelist, time running out before the strongest section, the candidate getting flustered and losing their place. This reading covers the generic shape of a phase-segmented defense and the design intent behind each phase. The daily overview connects this generic shape to the specific Thu format the cohort will face.

## 3. Core Concepts

### 3.1 Why structure beats open discussion

The Smartsheet design-review guide names three failure modes of unstructured review meetings: (a) off-course drift when a panelist interrupts before context is set, (b) information-capture loss when the discussion flows freely without a designated capture mechanism, and (c) volume bias when louder panelists crowd out quieter ones [1]. A minute-by-minute structure addresses each:

- A fixed presentation phase prevents interruption before the panel has a shared context.
- A fixed Q&A phase with a chair role enforces capture and prevents drift.
- A round-robin or per-individual phase ensures every panelist gets airtime and every candidate is evaluated.

The cost of structure is a small loss of spontaneity. The benefit is dramatically lower variance in outcomes — the defense reflects the work, not the personalities in the room.

### 3.2 The canonical five-phase defense

Across domains where structured defenses are mature — academic dissertation defenses, engineering design reviews, product-design critiques, and final-stage interviews — five phases recur [1][2][3]:

1. **Welcome + introductions (5–10 min).** The chair sets the agenda, introduces the candidate(s) and panel, and states the time budget. Purpose: anchor everyone to the same time-budget mental model so no one panics at minute 20 thinking they have only 5 left.
2. **Presentation / demo (15–25 min).** The candidate presents the work uninterrupted. Standard doctoral-defense guidance: "the candidate's presentation typically takes 20-30 minutes" [2]. The cap exists because attention is finite — beyond ~25 minutes uninterrupted, panel retention drops sharply.
3. **Examiner Q&A (15–25 min).** The expert panel probes the work. Questions can be structured (each panelist gets a turn) or chair-led. Industry design-review guidance recommends a chair-led structure: the chair calls on each examiner, captures the question and answer, and time-boxes follow-ups so one examiner cannot consume the entire phase.
4. **Audience / observer Q&A (5–15 min, optional).** Non-committee attendees get the floor. This phase is intentionally last and time-boxed because the most-load-bearing evaluation happens in phase 3; audience Q&A is enrichment, not assessment.
5. **Private deliberation + results delivery (10–30 min).** Non-committee members are excused; the panel deliberates; the candidate is invited back for the result [2]. The privacy of deliberation is structural — panelists need to negotiate without political pressure, and the candidate hears one consolidated decision rather than the deliberation itself.

This five-phase skeleton can compress (a 60-minute design review uses the same phases at smaller sizes) or expand (a 3-hour doctoral defense uses the same phases at larger sizes). The phases themselves are robust across scale.

### 3.3 Per-individual vs per-team evaluation

A defense involving multiple candidates (a paired project, a team capstone, a panel-of-three interview) makes one additional design decision: does evaluation happen *per team* or *per individual*?

- **Per-team evaluation.** One verdict for the whole team. Optimizes for evaluating the *artifact* — did the team produce a deliverable that meets the bar? Used in product design reviews where the question is "is this design ready to ship," not "is each designer individually ready for promotion" [1].
- **Per-individual evaluation.** Each team member is evaluated separately, often in a dedicated per-individual phase. Optimizes for evaluating the *people* — can each individual carry weight independently in a future engagement? Used in dissertation defenses (the degree is granted to a person, not a team) [2] and final-stage interviews (the hire decision is per-candidate).

The two modes require different *phase structures*. A per-individual defense inserts a phase where each candidate defends a sub-section of the work alone — long enough for the panel to form an independent read on each person, short enough to fit within the overall time budget. The transition into and out of this phase is the most fragile moment in the defense: the candidates have to switch from "we built this" to "I personally own this aspect," and the panel has to recalibrate from team-level to individual-level evaluation.

### 3.4 The transitions

The phase transitions are where panel attention resets and energy can be lost. A well-designed defense buffers them explicitly:

- **Presentation → examiner Q&A.** A 1–2 minute pause for the panel to consult their notes and pick their opening question. Without it, the first question is whichever panelist speaks fastest, not whichever question is most material.
- **Examiner Q&A → audience Q&A.** A chair-led handoff: "Committee questions are complete; we have 10 minutes for audience questions." This signals the gear shift and prevents committee members from continuing to dominate.
- **Audience Q&A → deliberation.** Non-committee members leave the room. This is the most ritualized transition in the dissertation format and exists for a reason: it gives the candidate emotional space and the committee political space [2].
- **Deliberation → results.** The candidate is invited back. The result is delivered cleanly — one decision, one set of next steps, no panel-internal disagreement aired.

Each transition is ~30–60 seconds in real time but signals a content shift the panel and candidate both need to absorb.

### 3.5 The chair's role

The chair is the meeting-design enforcer. Their job is not to evaluate (that's the panel's) and not to present (that's the candidate's) — it is to *protect the structure*. Specifically: keep the clock, call on panelists in turn, intervene when one panelist is consuming disproportionate time, mark the phase transitions verbally, and capture the consolidated result for delivery.

In a well-designed defense the chair is invisible to the candidate — they hear questions, they answer, the meeting flows. In a poorly designed defense the chair is also invisible, but the meeting drifts off-course at minute 30 and never recovers. The chair role is felt by absence: when it's done well, no one notices; when it's missing, everyone notices.

## 4. Generic Implementation

A 75-minute structured technical defense for a paired engineering team (two engineers presenting a product they built together) might look like this:

```
00:00–00:05  Chair welcome + intros + time-budget statement
00:05–00:20  Paired demo (15 min, uninterrupted)
00:20–00:30  Examiner Q&A round 1 — domain examiner (10 min)
              Chair calls on examiner; one main question + one follow-up
00:30–00:40  Examiner Q&A round 2 — operations examiner (10 min)
              Chair-led; same structure
00:40–00:50  Per-individual segment — Engineer A defends solo (10 min)
              Panel probes Engineer A on specific design decisions they personally owned
00:50–01:00  Per-individual segment — Engineer B defends solo (10 min)
              Same shape, different engineer
01:00–01:10  Paired retrospective Q&A (10 min)
              Open discussion across the panel about lessons learned
01:10–01:15  Chair closing; candidates excused for deliberation
              [Deliberation happens off-clock; result delivered separately]
```

Each block has a phase name, a duration, and an explicit role allocation. The chair's clock-keeping makes each transition crisp — the demo doesn't run to 22 minutes; the per-individual segment doesn't accidentally become a paired conversation; the retrospective doesn't bleed into the deliberation.

If this defense runs 8 minutes long because the domain examiner's questions land particularly well, the chair cuts time from the audience or retrospective phase, not from the per-individual phases — the per-individual phases are the load-bearing evaluation surface for individual hiring or staffing decisions and cannot be compressed without compromising the outcome.

## 5. Real-world Patterns

**Healthcare — Morbidity & Mortality (M&M) conferences.** Hospital departments run weekly or monthly M&M conferences in which a case with an adverse outcome is reviewed by the clinical staff. The structured format is direct lineage from the dissertation defense: case presenter (10–15 min), panel of expert physicians' Q&A (15–20 min), open floor (5–10 min), private deliberation on lessons-learned and any required action items. Published case-management literature credits the format's structure — not its content — for the consistent quality of lessons extracted, because the structure prevents the discussion from collapsing into blame [3].

**Aerospace — preliminary design review (PDR) and critical design review (CDR).** NASA and DoD aerospace programs have for decades used structured design-review gates with explicit phases [1]. A typical CDR for a subsystem allocates a fixed presentation block, a fixed expert-board Q&A block, an action-items capture block, and a closed deliberation. The phase budget is documented in the project's review plan before the meeting; deviations are noted as risk items. The minute-by-minute discipline is not optional — it is what makes the multi-billion-dollar gate decisions auditable and defensible.

**Venture capital — investment committee (IC) meetings.** Venture firms' weekly IC meetings have a structured format with strong parallels to a defense: the deal partner presents the opportunity (15–20 min, uninterrupted), the rest of the partnership probes (20–30 min), the deal partner is excused while the partnership deliberates (variable), and the result is delivered to the deal partner with the conditions or rejection rationale. The phases are time-boxed by tradition and by calendar; partnerships that drift from the format report decision quality dropping [3].

**Product design — Apple-style design review.** Public accounts of Apple's design-review culture under Jony Ive describe a structured weekly cadence: each designer presents a prototype (uninterrupted), then receives feedback in a strict order — peers first, then senior designers, then the design lead — with the lead explicitly silencing herself for the first rounds so the room hears junior voices first. The minute-by-minute discipline is what protects the most-valuable design voices in the room from being crowded out by the loudest ones [1].

## 6. Best Practices

- **Publish the minute-by-minute agenda in advance.** Candidates rehearse to the actual structure; panelists arrive with the right mental model; the chair has a script.
- **Cap the presentation at 25 minutes regardless of how much there is to show.** Attention is finite; depth beyond 25 minutes lands in Q&A, not in monologue [2].
- **Always insert a 1–2 minute buffer at each phase transition.** Panelists need it to consult notes; candidates need it to mentally reset.
- **Assign a chair whose only job is structure.** This is *not* an evaluating role. Conflating chair and evaluator weakens both.
- **Distinguish per-team and per-individual evaluation phases when the format requires both.** Don't try to do per-individual evaluation inside a paired demo — the candidates can't switch frames that fast, and the panel can't either.
- **Time-box examiner Q&A turns explicitly.** One examiner consuming 20 of 25 minutes is the modal failure of the format; the chair's job is to intervene.
- **Run deliberation in private with non-panel members excused.** Public deliberation politicizes the outcome and erodes the candidate's emotional space [2].

## 7. Hands-on Exercise

**Whiteboarding prompt (20 min).** You have been asked to design the format for a final-stage technical interview at a software company. Three candidates have made it to the final stage; each is interviewed back-to-back on the same day. Each interview gets 90 minutes with a panel of four (engineering manager, two senior engineers, a product manager). The hiring decision is per-candidate.

On paper or in a markdown file, design the **90-minute minute-by-minute agenda** for one interview. Your design must explicitly name:

- Each phase, its duration, and the role of the chair vs. panel vs. candidate in that phase.
- The transitions between phases and what signals or words the chair uses to make them.
- Which panelist is responsible for what subject matter (e.g., the EM owns the leadership questions; the senior engineers split deep-technical vs. system-design; the PM owns the cross-functional collaboration).
- The buffer for unexpected derailment (e.g., a question that lands and produces a 10-minute organic conversation — where do you cut to absorb it?).
- The closing — how the candidate exits the room and when/how they receive the decision.

**What good looks like:** The agenda has five clearly named phases (welcome / candidate-led problem walk-through / structured-Q&A by panelist role / candidate's own questions back to the panel / chair-led close). Phase durations sum to 90 minutes with at least 5 minutes of explicit buffer. The chair role is held by a single named panelist whose job is described as "protect the clock, ensure each panelist gets their slot, capture the consolidated read for deliberation." The panelists' subject-matter ownership is explicit so no two panelists ask overlapping questions. The buffer is named — e.g., "if the system-design conversation runs hot, the panel's own back-channel slack message during the next phase tightens the next examiner's slot to compensate."

## 8. Key Takeaways

- *Why is the minute-by-minute structure of a technical defense load-bearing rather than ceremonial?* (Because structure prevents the three modal failures — drift, capture loss, volume bias — that turn the same defense into different outcomes.)
- *What are the canonical five phases of a structured defense and what does each accomplish?* (Welcome / presentation / examiner Q&A / audience Q&A / private deliberation; the phases are robust across scale from 30-min design reviews to 3-hour dissertation defenses.)
- *How is a per-individual defense structurally different from a per-team defense?* (It inserts a dedicated phase where each candidate defends alone, and demands explicit transitions in and out of that phase so the panel and candidates both recalibrate frames.)
- *What is the chair's role and why is it distinct from the evaluator role?* (The chair protects the structure — clock, turns, transitions, capture — without evaluating; conflating the two roles weakens both.)
- *What is the load-bearing reason to time-box the presentation at 25 minutes?* (Attention is finite; depth past 25 minutes lands in Q&A, not in monologue, so giving the candidate more time hurts the panel's retention and the defense's outcome.)

## Sources

1. [The Critical Components of Preparing for and Conducting a Product Design Review (Smartsheet)](https://www.smartsheet.com/content/design-review-process) — retrieved 2026-05-26
2. [Doctoral Defense Handout (University of Tennessee — ELPS)](https://cehhs.utk.edu/elps/wp-content/uploads/sites/9/2024/12/Doctoral-Defense-Handout-Final.pdf) — retrieved 2026-05-26
3. [Design Meeting Agenda & Minutes Template (Noota.io)](https://www.noota.io/meeting-templates/design-meeting) — retrieved 2026-05-26
4. [How to Demo to Technical Audience (DemoSecret)](https://demosecret.com/articles/how-to-demo-to-technical-audience) — retrieved 2026-05-26

Last verified: 2026-05-26
