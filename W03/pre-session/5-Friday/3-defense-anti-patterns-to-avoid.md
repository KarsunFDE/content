---
week: W03
day: 5-Friday
topic_slug: defense-anti-patterns-to-avoid
topic_title: Defense Anti-Patterns to Avoid
parent_overview: W03/pre-session/5-Friday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://martinfowler.com/articles/exploring-gen-ai/i-still-care-about-the-code.html
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://martinfowler.com/articles/exploring-gen-ai/humans-and-agents.html
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://github.com/joelparkerhenderson/architecture-decision-record
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://staffeng.com/guides/staff-archetypes/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://lethain.com/getting-in-the-room/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Defense Anti-Patterns to Avoid

## 1. Learning Objectives

By the end of this reading, the learner can:

- Name the five most common anti-patterns engineers fall into when defending AI-assisted work to senior reviewers, and recognize each one in their own draft answers.
- Distinguish between owning a decision and explaining a decision someone else made, and articulate why the distinction matters in a per-individual defense.
- Lead an answer with the load-bearing fact (the risk, the trade-off, the limitation) instead of with framing or context.
- Cite a source — a standard, a regulation, a benchmark — only after verifying that they have read enough of it to answer a follow-up.
- Use the evidence trail produced during build (traces, logs, ADRs, eval reports) as the defense's primary artifact rather than relying on recall.

## 2. Introduction

The defense is the part where someone asks you, "why did you do it this way?" — and your answer is the artifact being evaluated. Senior reviewers, auditors, and stakeholders are not testing whether the system works in the demo. They are testing whether *you* understand the trade-offs you made, whether *you* can spot the risks you carry, and whether *you* would catch the failure mode they are about to ask about. This is true in a tier-4 promotion panel, a regulator's audit, a venture-firm diligence interview, and the in-class gate you are sitting tomorrow.

What makes AI-assisted work specifically hard to defend is that the artifact in front of the reviewer was not produced one line at a time by the person who has to defend it. Some of the code came from a model. Some of the decisions came from a model's suggestion that you accepted. Birgitta Böckeler at Thoughtworks frames this as constant risk assessment — every AI-accepted change is a bet with three dimensions: impact, probability, and detectability. The defense is where you make that bet legible to someone who wasn't sitting next to you when you placed it.

The good news: the anti-patterns are well-known. The same five mistakes recur across audit defenses, architecture reviews, and post-incident interviews. If you internalize the list, you can catch yourself before you commit each one. This reading is the catalog; tomorrow is the practice.

## 3. Core Concepts

### 3.1 The five defense anti-patterns

These are catalog entries. Each has a name, a tell, and a counter-move.

**Anti-pattern 1: Defending decisions you don't own.**

The tell: you find yourself saying "we decided" when the decision was actually made by your pair-partner, your tech lead, or a model whose suggestion you accepted without strong opinions. You then have to invent justification on the spot when the reviewer presses.

The counter-move: separate ownership cleanly. "My partner owned the framework choice; I owned the retrieval layer. For the framework choice, here's what I understand of the trade-off — but they're the one to ask for the deep rationale." Reviewers respect calibrated ownership far more than they respect performed omniscience.

**Anti-pattern 2: Citing sources you haven't read.**

The tell: you drop a regulation citation (FAR 15.308, OWASP LLM06, NIST SP 800-218) into an answer because it sounds load-bearing, and you cannot answer the follow-up question about what the cited section actually says.

The counter-move: if you cite it, you have read it. If you haven't read it, paraphrase the underlying principle and say so. "The principle here is that delegation has to be documented and accountable — I haven't read the specific clause text, but that's the discipline we applied." Reviewers will trust the second answer more than they will trust an unsourced authority you can't defend.

**Anti-pattern 3: Buried-the-lede answers.**

The tell: a reviewer asks "what's your biggest production risk?" and you respond with two minutes of context-setting before naming the risk.

The counter-move: lead with the load-bearing sentence. "The biggest risk is that the retrieval layer ranks stale documents above fresh ones in 3% of queries — here's why, and here's what we'd do about it." Context comes second. Reviewers in audit settings, in particular, are usually working from a question list and time-boxing each answer; if you bury the lede they may move on before you reach it.

**Anti-pattern 4: Pretending the work was clean.**

The tell: when asked what went wrong, you produce a sanitized story in which the project went roughly to plan. This is the most expensive anti-pattern because it discards the most valuable artifact you produced — the discoveries.

The counter-move: name the discovery in plain terms. "The plan budgeted 1.5 days for the multi-tenant filter; it took 3 days because we hit a row-level-security edge case the design didn't anticipate. Here's what we learned. Here's what we'd do differently." A discovery you can name is a competency the reviewer recognizes. A discovery you hide reads as a competency you lack.

**Anti-pattern 5: Skipping the evidence.**

The tell: you have observability traces, eval reports, ADR documents, prompt-tuning history — and you do not show any of them during the defense, relying instead on verbal recall.

The counter-move: the evidence is your defense. The LLM trace screenshot is more credible than your description of what the LLM did. The eval-report graph is more credible than your characterization of accuracy. Auditors are trained to weight evidence over assertion; engineers are trained to weight reasoning over evidence. Bring both, lead with the evidence.

### 3.2 Why AI-assisted work amplifies each anti-pattern

Böckeler's framing — risk = impact × probability × (1 / detectability) — is a useful lens. Each defense anti-pattern is a way the *detectability* of a problem collapses.

- Defending a decision you don't own means the person who could detect the trade-off is not in the conversation.
- Citing a source you haven't read means the authority that could detect the misuse is not actually consulted.
- Burying the lede means the failure-shaped sentence never reaches the reviewer's attention.
- Pretending the work was clean means the discovery that would have triggered a deeper review never surfaces.
- Skipping the evidence means the artifact that would have made a problem visible stays in the trace folder.

In each case, the defense itself becomes the choke-point — the place where the system's actual risk profile fails to reach the people whose job is to weigh it.

### 3.3 Owning vs. explaining

A defense has two kinds of statements interleaved: things you own, and things you can explain but did not own. Reviewers grade *ownership* differently from *recall*.

When you own a decision, you can answer "why this and not the alternative" in your own words. You know the second-best option you rejected and the trade-off that tipped the choice. When you can explain a decision someone else made, you can repeat the rationale, but you may not be able to defend it under pressure. That's fine — staff archetypes literature is clear that no senior engineer owns every decision in a system they ship. What gets you in trouble is mis-claiming ownership of something you can only explain.

The clean version: "I own this. My partner owns that. Here's our shared understanding of the third thing." Calibrated. Specific. Honest.

### 3.4 Evidence as the primary artifact

Kief Morris's "on-the-loop" framing is that the human's job around AI-assisted work is to build and supervise the feedback loop — on the loop, not in it for every line. The artifact of that supervision is *the loop's output*: tests passing, evals passing, traces showing the agent did what it was supposed to do.

In a defense, those loop outputs are your evidence. They are stronger than your recall because they were captured when the system was actually running. The disciplined defender opens with the strongest evidence available — the eval report, the trace screenshot, the test-coverage delta — and walks the reviewer through it. The verbal answer accompanies the evidence; it does not replace it.

## 4. Generic Implementation

This is a non-code topic. Here is a worked example of the same Q&A done two ways — first as an anti-pattern, then as a counter-move. The setting is a fintech engineering review of an AI-assisted fraud-detection feature.

**Reviewer's question:** "What's your worst false-negative case in production?"

**Anti-pattern answer (buries the lede, can't cite, skips evidence):**

> "So our fraud-detection pipeline is built on a few different layers, the most important of which is the rules engine, and we layer the LLM-based classifier on top of that. We follow industry standard practices around evaluation. There's some research we used as guidance — I believe it was a Stripe paper or something similar — and we ran a lot of tests. Overall I think the false-negative rate is acceptable."

What the reviewer hears: no specific case, no specific number, no specific source, no specific evidence. The candidate is performing competence rather than demonstrating it.

**Counter-move answer (leads with the case, cites only what was read, shows evidence):**

> "Worst case in production is card-not-present transactions under $35 from a new merchant, where the rules engine fires first and the LLM doesn't get to weigh in. We caught two false negatives there last quarter — both were chargebacks. The eval report on screen shows the cohort: 4.1% false-negative rate, versus our 1.2% target. The principle we applied for the layered design is defense-in-depth — I haven't read the specific OWASP page on it, that was our security lead's call — but the trade-off was bounding LLM latency against catching marginal cases."

What the reviewer hears: a specific case, a specific number, a specific cohort, an honest cite-vs-paraphrase line, and an evidence artifact on screen. The candidate is demonstrating competence.

The structural difference is that the counter-move answer's first sentence carries the lede; everything else supports it. The anti-pattern answer's lede never lands.

## 5. Real-world Patterns

**Healthcare: FDA pre-submission defense.** A team building an AI-assisted radiology triage tool met with the FDA for a pre-submission consultation in 2024. Their preparation playbook addressed anti-pattern 2 explicitly: every cite to a 510(k) precedent had to be by submission number, and the engineer presenting had to have read the summary of safety and effectiveness document for that precedent. They built a cite-and-read tracker and rehearsed off it. The reviewer's first probe was a follow-up on a cited precedent. The team answered cleanly.

**Gaming: post-incident review at scale.** A multiplayer-game company's weekly post-incident review enforces anti-pattern 3 at the format level: every presenter must lead with a single sentence naming customer impact in concrete units (users affected, minutes of degradation, revenue lost). The chair cuts off context-first openings. The team's MTTR improved measurably after the format change — not because incidents got easier, but because the handoff to the review audience got faster.

**Fintech: model-risk-management defense.** A US bank's model-risk-management process for an LLM-assisted credit-decisioning workflow required defending the model to independent validators. The team's post-validation retro names anti-pattern 4 as the one that hurt them: validators caught two production cases the engineering team had quietly fixed without naming the underlying class of failure in their submission. The submission had to be redone. A discovery you name and remediate is documented competence; a discovery you fix and don't name reads as a discovery you tried to hide.

## 6. Best Practices

- Write down the five anti-pattern names on a notecard you keep with you during the defense; seeing them in your peripheral vision is enough to catch the pattern in your own mouth.
- Build an "owned vs. explained" two-column list of every load-bearing decision; rehearse the owned column out loud, and prepare honest "ask my partner" handoffs for the explained column.
- If you intend to cite a regulation, standard, or paper, read the actual cited section the day before — not a summary, the source itself.
- Practice leading each answer with the failure-shaped sentence; rehearse with a partner who interrupts the moment you bury the lede.
- Build the evidence pack — traces, eval reports, ADR list, prompt-history snapshots — as a small set of named artifacts you can open within five seconds.
- Calibrate honesty above performance: a defender who says "I don't know, but here's how I'd find out" is rated higher than one who confabulates.

## 7. Hands-on Exercise

**Mock-defense exercise — 15 minutes, pair format.**

Partner A plays the defender. Partner B plays a senior reviewer.

Partner A presents — in three minutes — a system they recently built, with at least one AI-assisted component. Partner B asks **three questions**, drawn from this list, in this order:

1. "What's the worst failure mode this system has in production today?"
2. "Cite one source — a standard, a regulation, a paper — that informed a key design decision. Then tell me what that source actually says."
3. "Walk me through a specific trade-off you made, and tell me which of you on the team owned that decision."

Partner A answers each in under 90 seconds. Partner B does not coach during the answers, but writes down which of the five anti-patterns (if any) the answer commits.

After all three answers, Partner B reads the anti-pattern observations to Partner A. The pair discusses each for two minutes: did the defender catch it themselves? What was the counter-move that would have been stronger?

Swap roles. Repeat.

**What good looks like.** A strong defender catches themselves committing an anti-pattern mid-answer and self-corrects. They say "I don't know" to at least one part of one question, then describe how they would find out. They reach for a specific evidence artifact at least once. A weak defender delivers three smooth, generic, evidence-free answers that could apply to any system.

## 8. Key Takeaways

- Can you name all five defense anti-patterns and recognize each one in a transcript of your own answer?
- Can you separate cleanly, in a single sentence, what you own from what you can explain but did not decide?
- Can you lead an answer with the load-bearing sentence — the risk, the failure mode, the trade-off — before any framing?
- Can you describe the discipline you follow before citing a source, and the alternative phrasing you use when you have not read the source?
- Can you name the three to five pieces of evidence you would bring to your next defense and explain why each one is stronger than the verbal description it replaces?

## Sources

1. [I still care about the code (Birgitta Böckeler, Thoughtworks)](https://martinfowler.com/articles/exploring-gen-ai/i-still-care-about-the-code.html) — retrieved 2026-05-26
2. [Humans and Agents in Software Engineering Loops (Kief Morris, Thoughtworks)](https://martinfowler.com/articles/exploring-gen-ai/humans-and-agents.html) — retrieved 2026-05-26
3. [Architecture Decision Record (Joel Parker Henderson et al.)](https://github.com/joelparkerhenderson/architecture-decision-record) — retrieved 2026-05-26
4. [Staff Archetypes (Will Larson, StaffEng)](https://staffeng.com/guides/staff-archetypes/) — retrieved 2026-05-26
5. [Getting in the Room (Will Larson)](https://lethain.com/getting-in-the-room/) — retrieved 2026-05-26

Last verified: 2026-05-26
