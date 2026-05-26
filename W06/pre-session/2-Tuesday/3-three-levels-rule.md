---
week: W06
day: Tue
topic_slug: three-levels-rule
topic_title: "Three-levels rule (90s / 5min / 20min)"
parent_overview: W06/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 9
sources:
  - url: https://www.liveplan.com/blog/funding/different-pitch-times
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://en.wikipedia.org/wiki/BLUF_(communication)
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://conversationsoncareers.com/2025/10/start-with-the-answer-the-minto-pyramid-principle/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://thepersimmongroup.com/bluf-how-these-4-letters-simplify-communication/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://fourweekmba.com/minto-pyramid-principle/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Three-levels rule (90s / 5min / 20min)

## 1. Learning Objectives

By the end of this reading, the learner can:

- Author a single technical narrative at three nested lengths (~90 seconds, ~5 minutes, ~20 minutes) without contradicting themselves across versions.
- Explain why nested-length structure (Minto pyramid + BLUF) outperforms three independent stories at the same content.
- Match version length to audience role (executive walk-by / manager review / auditor deep-dive) and switch registers cleanly mid-conversation.
- Identify the "two-stories problem" in a draft and collapse it back to a single nested story.
- Apply the subtract-don't-extend authoring discipline when moving from long to short.

## 2. Introduction

Most engineering teams that present to multiple audiences end up authoring three different stories — a slide for the CEO, a deck for the manager, a detailed write-up for the auditor — and discover too late that the three documents disagree in subtle ways. The auditor finds a number that does not appear in the manager's deck; the manager finds a tradeoff the CEO's summary glossed over; trust erodes audience by audience.

The three-levels rule is the discipline that prevents this. It says: there is *one* story. It has three nested lengths. The shorter versions are *prefixes* of the longer ones — not separate documents, not separate narratives. If the 5-minute version says something the 20-minute version contradicts, you have a structural defect, not a versioning problem.

The technique borrows from two well-established communication frameworks. The Minto Pyramid Principle, originated by Barbara Minto at McKinsey, structures every executive deliverable as a top-level answer supported by a small number of grouped arguments which are in turn supported by evidence ([Minto Pyramid Principle, Conversations on Careers, retrieved 2026-05-26](https://conversationsoncareers.com/2025/10/start-with-the-answer-the-minto-pyramid-principle/)). BLUF — Bottom Line Up Front — emerged from U.S. Army Regulation 25-50 and codifies the same idea for high-stakes military and operational writing ([BLUF, Wikipedia, retrieved 2026-05-26](https://en.wikipedia.org/wiki/BLUF_(communication))). Both frameworks share the structural insight: the answer comes first; depth is optional reading.

The three-levels rule is what you get when you apply Minto + BLUF *across time-budgets* rather than across paragraphs.

## 3. Core Concepts

### 3.1 The three lengths and why these three

- **~90 seconds.** The "hallway encounter" length. The audience is mobile, has limited attention, is choosing whether to invest more time. One sentence per element: the headline outcome, the cost shape, one named limitation. This length tracks the elevator pitch tradition: a single-minute pitch covers the heart — the problem, the solution, and proof — and nothing else ([LivePlan on pitch lengths, retrieved 2026-05-26](https://www.liveplan.com/blog/funding/different-pitch-times)).
- **~5 minutes.** The "standing review" length. The audience has committed to a short block; they want enough to make a decision but not enough to verify. Add three to four supporting points, each one or two sentences. The "rule of three" applies — beyond three supporting points, credibility *drops* in audit-oriented audiences ([Minto Pyramid, FourWeekMBA, retrieved 2026-05-26](https://fourweekmba.com/minto-pyramid-principle/)).
- **~20 minutes.** The "seated deep-dive" length. The audience is investing the time to verify, challenge, and document. Methodology, per-segment breakdowns, named open questions, reproducer steps. This is the length where evidence carries the argument; the headline is still the headline, but now every claim has a footnote.

These three lengths are not arbitrary. They correspond to the three most common audit-oriented stakeholder modes: *decision-makers walking by* (90s), *decision-makers reviewing* (5min), and *examiners verifying* (20min). A fourth length — 60 seconds or under — exists for very specific elevator-pitch contexts, but it tends to compress so hard that nuance vanishes. The 90-second target preserves space for the one named weakness.

### 3.2 Nesting, not duplication

The structural rule: **author the 20-minute version first; subtract to 5; subtract again to 90 seconds.** This guarantees the shorter versions are *subsets* of the longer one — anything in the 90-second version appears verbatim or near-verbatim in the 20-minute version. If the 90-second version contains a claim the 20-minute version does not support, you have two stories and one of them is wrong.

This inverts the usual instinct to "expand the executive summary into a full report." Expansion is unsafe — you tend to invent justifications mid-stream that contradict the original framing. Subtraction is safe — you can only remove what is already there.

The Minto pyramid makes the structural logic explicit: the top of the pyramid is the *answer*, supported by a small set of arguments, supported in turn by evidence ([StrategyPunk on Minto, retrieved 2026-05-26](https://www.strategypunk.com/the-minto-pyramid-principle-how-to-communicate-like-a-mckinsey-consultant-pdf/)). Each shorter version is a *higher slice* through the same pyramid:

```
           [ Answer / headline ]              <- 90s version sits here
            /         |         \
       [Arg A]    [Arg B]    [Arg C]          <- 5min version adds these
        / \        / \        / \
     [evidence underneath each arg]           <- 20min version drills here
```

The shorter version is never a different pyramid; it is the same pyramid viewed from higher up.

### 3.3 BLUF as the authoring discipline

BLUF (Bottom Line Up Front) is a writing convention but it acts as an authoring constraint. It forces the author to know, at draft time, what the *answer* is — which means it forces the author to pick. A draft that cannot put a single sentence at the top is a draft whose author has not yet decided what they are recommending ([BLUF, The Persimmon Group, retrieved 2026-05-26](https://thepersimmongroup.com/bluf-how-these-4-letters-simplify-communication/)).

Practically: every level — 90s, 5min, 20min — opens with the *same* one-sentence bottom line. The audience hearing the 90s version and the audience reading the 20-minute version both leave with the same headline. Only the supporting depth differs.

### 3.4 The two-stories problem

The most common defect is the *two-stories problem*: the 5-minute version reads as a summary of one system; the 20-minute version reads as the documentation of a slightly different system. Symptoms:

- The 5-minute version cites a number the 20-minute version does not contain (or contains with a different framing).
- The 5-minute version names a "primary tradeoff" the 20-minute version treats as one of several minor ones.
- The 5-minute version's "open question" list does not match the 20-minute version's `Findings` section.

Cause: the author wrote the long version first and then composed the short version from memory rather than by subtraction. The fix is mechanical — copy the long version, then delete (not rewrite) until the time budget is hit.

### 3.5 Register vs length

Length and register are independent axes. A 90-second story to an executive uses *executive register* (outcome + cost + one risk). A 90-second story to an auditor uses *audit register* (control + clause + one named gap). Same length, different surface. The three-levels rule constrains length; the audience determines register. Mixing them up — telling the auditor a marketing-style 90-second story, or telling the executive a controls-language 20-minute story — fails both audiences.

Pair this reading with the "technical storytelling for federal-acq audiences" reading for the controls-language register; pair it with the "tradeoff honesty" reading for the named-gap discipline.

## 4. Generic Implementation

A worked example from a generic SaaS company presenting a new email-deliverability feature to three audiences in one afternoon.

**90-second version (for the CEO between meetings):**

> Email-deliverability v2 launched to 10% of paid traffic Tuesday. Inbox-placement rate improved 6 percentage points in the canary; complaint rate held flat. One known limitation: the new IP-warming schedule under-performs for senders in the 1k–5k daily-volume band — we have a fix scheduled for next month.

Three sentences. Outcome, evidence, named gap.

**5-minute version (for the VP Engineering at standup):**

> Same opening sentence.
>
> Three supporting points: (a) the inbox-placement gain is consistent across the three largest mailbox providers we measure; (b) cost-per-send rose 2% due to additional dedicated-IP capacity, within the FY budget envelope; (c) the canary covered four customer segments — enterprise, mid-market, SMB-high-volume, SMB-low-volume — and the inbox-placement gain held in the first three but was flat in SMB-low-volume.
>
> Same named limitation, plus one detail: the 1k–5k band gap appears to be IP-reputation-driven, not algorithmic — the fix is operational (extending the warming window) not engineering-heavy.

Same headline. Three Minto-style supporting arguments. Same named limitation, slightly more depth.

**20-minute version (for the deliverability auditor visiting Thursday):**

> Same opening sentence.
>
> Six sections: methodology (canary cohort selection, statistical significance threshold, baseline period), per-mailbox-provider results table, per-segment results table, cost model with per-tenant breakdown, the open `Findings` list (with the SMB-low-volume gap as Finding F-2026-Q2-018, owner, ETA), and a reproducer-steps appendix for replicating any number cited in §1.
>
> Same named limitation, now with the full diagnostic write-up and the proposed fix's risk analysis.

Same headline. The depth is layered evidence supporting each of the 5-minute version's three arguments. Nothing in the 20-minute version contradicts the shorter versions. Nothing in the shorter versions is absent from the 20-minute version.

Notice the authoring discipline: the 20-minute version was written first. The 5-minute version is its top layer plus one paragraph per Minto argument. The 90-second version is its top layer alone. Subtraction, not expansion.

## 5. Real-world Patterns

**Aviation — pre-flight pilot briefings.** Commercial aviation uses a nested briefing pattern: the gate-agent announcement is the 90-second version (destination, flight time, weather summary), the cabin-crew briefing is the 5-minute version (specific routing details, expected turbulence windows, service plan), and the cockpit-tower briefing is the 20-minute version (taxiway, runway, climb profile, alternate airports, fuel state, all reproducible from the dispatch release). The three layers are nested by design — anything in the gate announcement is also in the cockpit briefing, just stripped of the operational depth.

**Medicine — clinical handoffs (SBAR).** The SBAR framework (Situation, Background, Assessment, Recommendation) supports a 90s nurse-to-nurse handoff, a 5-minute shift-change huddle, and a 20-minute multidisciplinary care-team review. The *headline* (the patient's current situation and the active recommendation) is identical across all three. The depth changes — the 20-minute review covers medication interactions, family preferences, and discharge-planning history that would never appear in the 90-second handoff — but no shorter version invents content the longer version contradicts. SBAR is essentially the three-levels rule applied to clinical communication.

**Investor pitches — startup fundraising.** Founders pitching at multiple lengths use this discipline by necessity: the 60-second pitch at a conference (problem + solution + traction), the 5-minute pitch at a partner meeting (adds market sizing + team + ask), and the 20-minute pitch at the partnership decision meeting (adds financials, comparable exits, defensibility). LivePlan and several pitch-coaching resources describe exactly the nested-not-duplicated structure: the 20-minute version expands the 5-minute version which expands the 1-minute version ([LivePlan, retrieved 2026-05-26](https://www.liveplan.com/blog/funding/different-pitch-times)). When founders fail this discipline, sophisticated investors notice the inconsistency and disengage.

**Incident response — postmortems.** Mature SRE practice publishes incident reports at three layers: a public-facing status-page summary (90s — what happened, what was affected, what we did), an internal all-hands summary (5min — adds the proximate cause and the immediate prevention work), and a full postmortem document (20min+ — adds the root-cause analysis, the contributing factors, the action items with owners). Google's SRE book and several incident-response writeups describe this as a *retelling discipline* — the same story, three depths, never contradicting itself.

## 6. Best Practices

- **Author the longest version first.** Then subtract — never expand — to reach the shorter versions. Expansion invents inconsistencies; subtraction preserves them.
- **Open all three versions with the same one-sentence headline.** That sentence is your BLUF. If you cannot write it, you do not yet have a story to tell.
- **Time-box each version when rehearsing.** A 5-minute version that runs 8 minutes is a 20-minute version in disguise; cut until it fits or restructure.
- **Use the rule of three for the 5-minute version.** Three supporting points, no more. Beyond three, credibility drops in audit-oriented audiences.
- **Map each shorter-version sentence to a longer-version location.** Every claim in the 90s should have a clear "this appears in §X of the 20-minute version" reference. This catches the two-stories problem before the audience does.
- **Keep the named-weakness in all three versions.** A weakness that survives the 90s cut signals honesty; a weakness that only appears in the 20-minute version reads as buried.
- **Separate length from register.** Pick the length from the audience's available time; pick the register from the audience's role. Do not conflate them.

## 7. Hands-on Exercise

**Time:** 15 minutes. **Format:** authoring + self-check.

Pick a project you've worked on in the last 12 months. Author the three nested versions of its current-state story.

**Steps:**

1. Draft the **20-minute version** first as a bulleted outline (not full prose) — about 12–18 bullets organised into 5–7 sections.
2. **Subtract** to a 5-minute version: keep the opening headline sentence + three Minto-style supporting points + one named weakness. About 8 bullets.
3. **Subtract again** to a 90-second version: keep the opening headline + the cost/outcome shape + one named weakness. Three sentences total.

**Self-check after writing:**

- Read the 90-second version. Can you point to where each of its sentences appears in the 20-minute outline?
- Read the 5-minute version. Does it open with the *same* sentence as the 90-second version?
- Is your named weakness identical (not paraphrased) across all three versions?
- Could you read the 90-second version to an executive and then, if asked "tell me more," cleanly continue into the 5-minute version without any contradiction?

**What good looks like:** the three versions visibly share their opening sentence, the named weakness is verbatim across all three, and the 20-minute outline contains every claim the shorter versions make. If your 5-minute version has any concept that does not appear in the 20-minute outline, you authored by expansion instead of subtraction — start over from the 20-minute version.

## 8. Key Takeaways

- Can I produce three nested versions of one story (90s / 5min / 20min) without any of them contradicting the others?
- Can I tell which audience needs which length, and switch between them in conversation without losing the headline?
- Did I author by subtraction (long → short) or by expansion (short → long)? If the latter, the two-stories defect is likely present.
- Does my named weakness appear in all three versions, verbatim?
- Is my BLUF — the single bottom-line sentence — the same first sentence of every length?

## Sources

1. [What to Say in Your 1, 5, 10, or 20-Minute Elevator Pitch (LivePlan)](https://www.liveplan.com/blog/funding/different-pitch-times) — retrieved 2026-05-26
2. [BLUF (communication) — Wikipedia](https://en.wikipedia.org/wiki/BLUF_(communication)) — retrieved 2026-05-26
3. [Start with the Answer: The Minto Pyramid Principle (Conversations on Careers)](https://conversationsoncareers.com/2025/10/start-with-the-answer-the-minto-pyramid-principle/) — retrieved 2026-05-26
4. [BLUF: The Four Letters That Transform Executive Communication (The Persimmon Group)](https://thepersimmongroup.com/bluf-how-these-4-letters-simplify-communication/) — retrieved 2026-05-26
5. [Minto Pyramid Principle: 2026 Guide to Structured Thinking (FourWeekMBA)](https://fourweekmba.com/minto-pyramid-principle/) — retrieved 2026-05-26

Last verified: 2026-05-26
