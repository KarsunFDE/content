---
week: W06
day: Mon
topic_slug: ai-assist-project-audit
topic_title: "AI-Assist Project Audit — the per-PR honest record"
parent_overview: W06/pre-session/1-Monday/1-DailyTopicOverview.md
estimated_minutes: 13
sources:
  - url: https://codeslick.dev/blog/eu-ai-act-audit-trail-2026
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://bitloops.com/resources/governance/audit-trails-for-ai-assisted-development
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://codebrewtools.com/blogs/ai-code-provenance-compliance-tools-2026
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://blog.exceeds.ai/realtime-ai-code-audit-trails/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.knostic.ai/blog/ai-coding-assistant-governance
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-05-26
---

# AI-Assist Project Audit — the per-PR honest record

## 1. Learning Objectives

By the end of this reading, the learner can:

- Define AI code provenance and audit trail, and explain what an "honest record" of AI assistance must capture per PR.
- Distinguish at least three common annotation modes (e.g., human-led, AI-assisted, AI-led) and choose the correct mode for a given PR.
- Recognise the three common failure modes of AI-assistance disclosure (performative annotation, annotation drift, cross-link gaps) and how each is detected.
- Articulate why the *honest record* of how AI was used to produce a work product is increasingly a regulatory and contractual requirement, not just a hygiene practice.

## 2. Introduction

By 2026, AI-assisted code authoring is no longer a niche workflow — industry surveys put substantial-block AI generation at 30–40% of enterprise development teams, and analyst commentary characterises AI as the primary contributor on a significant fraction of production commits. This shift has produced a corresponding shift in how organisations think about *the audit trail of the code itself*: not just what was committed, but how it was authored, with what tool, against what prompt, and reviewed by whom.

The technical and regulatory community has converged on a single requirement: each AI-assisted code change carries metadata. The CodeSlick treatment of EU AI Act audit-trail requirements characterises the bar as recording "when and how AI suggestions are generated, when they are accepted or modified, and what inputs led to their creation." The Bitloops governance writeup calls this **compliance-by-design** in AI-assisted development. The shape varies by tool and organisation; the principle is uniform: provenance is now a first-class engineering artefact.

Within an engineering team, the most common and useful expression of this principle is a **per-PR annotation** — a small structured field in every pull request describing how AI was used. The annotation is dirt-cheap to add at PR time and disproportionately expensive to reconstruct after the fact. This reading covers the design of the annotation, the lifecycle around it, and the failure modes that make annotation discipline degrade over time.

## 3. Core Concepts

### 3.1 What "code provenance" means for AI-assisted work

**Provenance** in this context is the chain of facts about where a code change came from: which engineer initiated it, which tool was used, which model and version produced any AI suggestion, which prompt prompted it, which suggestion was accepted, which suggestion was modified, which suggestion was discarded.

CodeBrewTools' analysis of AI code-provenance tooling lists model-level traceability (timestamps, file/line ranges, developer ID) as the minimum bar. The Exceeds.AI guide to real-time AI code audit trails treats provenance as a stream that flows from suggestion through acceptance through review through merge, not a single field captured once.

For most engineering teams the practical question is not "do we capture every keystroke" but "do we capture enough that a downstream reader — auditor, successor engineer, regulator — can answer the question 'how was this authored?' without reconstructing memory."

### 3.2 The three-mode annotation

A widely-replicable pattern is to annotate each PR with one of three values describing the *predominant authoring mode*:

| Mode | Meaning | Typical signals |
|------|---------|-----------------|
| Human-led | The human authored the code; AI was used as an accelerant for autocomplete, refactor suggestions, doc lookups, or syntax help. | Most AI suggestions modified before accept; design choices made by the human; AI did not generate first drafts of structure. |
| AI-assisted (or pair-led) | Human and AI worked iteratively on a substantial section of the code; neither dominated. | AI generated drafts that the human revised; human-AI iteration over several rounds; human still owned design and review. |
| AI-led | AI generated the first draft of the PR; the human reviewed, verified, and revised but did not initiate the structure. | The first prompt produced >50% of accepted code; the human's edits were corrections, not structural changes. |

Some organisations add a fourth — **hand-authored** — for PRs explicitly authored without any AI involvement, used when the engineer deliberately turned the assistant off (e.g., when writing a security-sensitive section or a critique that needs unmediated thought).

The discipline is in the *honesty* of the choice. Bitloops' compliance-by-design treatment is explicit: if the annotation does not reflect reality, the audit trail is performative and the underlying compliance question is unanswered.

### 3.3 The three failure modes

Three failure modes recur across teams that adopt PR-level annotation:

#### Performative annotation
The team adopts the annotation as a checkbox. Every PR carries the default value ("human-led" or "AI-assisted") regardless of how the code was actually produced. The annotation looks compliant but tells the reader nothing. Detection: pick five PRs at random with the same annotation value; ask the author to walk through the design decisions in each. If they cannot, the annotation was performative.

#### Annotation drift
Early PRs (week 1–2 of an engagement) carry honest annotations; by week 6–8 the annotations are stale because nobody enforced. The team's actual AI usage may have increased, decreased, or shifted, but the annotations froze. Detection: plot the distribution of annotation values per week and look for a flat line where the underlying usage was likely changing.

#### Cross-link gaps
The PR annotation is fine in isolation but does not link to upstream artefacts (the prompt that produced it, the design discussion, the model version). A future auditor reading the annotation cannot follow the chain back. Detection: pick a PR labelled AI-led; ask whether the prompt and model version are recoverable. If not, the cross-links are missing.

### 3.4 What the annotation should reference (minimum and ideal)

| Field | Minimum | Ideal |
|-------|---------|-------|
| Authoring mode | One of {human-led, AI-assisted, AI-led, hand-authored} | Same, with reviewer cross-check |
| Tool used | Tool name (e.g., Copilot, Cursor, Claude Code) | Tool + version |
| Model | Model name | Model + version + temperature/config if applicable |
| Prompts | (omitted) | Link to a saved prompt log |
| Reviewer | Reviewer ID | Reviewer ID + their independent assessment of the annotation |
| Reasoning | (omitted) | One-paragraph note from the author about why this mode was chosen |

The minimum is what fits in a PR description today, the ideal is what mature compliance-by-design pipelines target. The CodeBrewTools tool inventory and the Knostic AI-coding-assistant governance writeup converge on the same shape.

### 3.5 Why this is becoming non-negotiable

Three forces are making this discipline non-negotiable:

1. **Regulatory.** The EU AI Act's high-risk-system requirements (effective August 2026 per the CodeSlick treatment) include record-keeping obligations. California's AB 2013 adds transparency mandates. Defence and regulated-industry customers increasingly require provenance disclosure in contracts.
2. **Contractual.** Enterprise customers buying engineering services are starting to require AI-usage disclosure as part of the engagement scope. The annotation is the artefact that lets the vendor answer the question without reconstructing memory.
3. **Operational.** When a bug appears in AI-led code six months after merge, the cheapest path to understanding it is to read the prompt and model context that produced it. Without the annotation chain, that context is unrecoverable.

The combined pressure means the engineering team that defers the annotation discipline pays the cost later, with interest.

## 4. Generic Implementation

Below is a generic PR-template snippet and a small CI check that together implement a minimum-viable AI-assistance audit trail. The example team is an **e-commerce platform engineering org** that ships across several markets — deliberately outside federal acquisitions.

```markdown
# Pull Request Template

## Summary
<one-paragraph summary of what this PR changes and why>

## AI Assistance Disclosure
<!-- Required. Fill honestly. CI checks for presence; reviewer checks honesty. -->

- **Authoring mode**: <one of: human-led | ai-assisted | ai-led | hand-authored>
- **Tool**: <tool name + version, or "n/a" for hand-authored>
- **Model**: <model name + version, or "n/a">
- **Reasoning** (1–2 sentences): <why this mode was chosen — what makes this PR fit this category>
- **Reviewer concurs**: <to be filled by reviewer: yes / no / reason>

## Test plan
- [ ] ...

## Risk
<one-paragraph risk note: what could go wrong in production from this change>
```

And a minimal CI check that fails if the section is absent or default-stubbed:

```yaml
# .github/workflows/check-ai-disclosure.yml
name: AI disclosure check
on:
  pull_request:
    types: [opened, edited, synchronize]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - name: Verify AI Assistance Disclosure block is filled
        uses: actions/github-script@v7
        with:
          script: |
            const body = context.payload.pull_request.body || "";
            // Required block headers
            const required = [
              "AI Assistance Disclosure",
              "Authoring mode",
              "Tool",
              "Model",
              "Reasoning",
              "Reviewer concurs"
            ];
            const missing = required.filter(h => !body.includes(h));
            if (missing.length > 0) {
              core.setFailed("Missing PR sections: " + missing.join(", "));
              return;
            }
            // Reject obvious stub values
            const stubs = ["<one of:", "<tool name", "<model name", "<why this"];
            const stubsLeft = stubs.filter(s => body.includes(s));
            if (stubsLeft.length > 0) {
              core.setFailed("Stub text left in disclosure block: "
                             + stubsLeft.join(", "));
            }
```

Annotations:

- The **template enforces structure**; the **CI check enforces presence and rejects unedited stubs**.
- The **reviewer-concurs field is the load-bearing line** — without an independent second opinion, the annotation is self-attestation.
- The check is intentionally **modest** — presence and not-stub. The "honesty" check is a human reviewer responsibility, not an automatable one. Pretending to automate it produces false confidence.
- The **reasoning field** is what defeats performative annotation: an author who marks `human-led` but cannot explain why has just self-flagged the PR for review.

## 5. Real-world Patterns

**Compliance-driven engineering at a European bank (fintech).** EU-resident banks subject to EU AI Act high-risk-system provisions and DORA operational-resilience requirements have started requiring per-PR AI-assistance disclosure. The annotation gets surfaced into the bank's central engineering compliance dashboard, where audit-rate metrics (% of PRs with valid disclosure, % with reviewer concurrence) become KPI. The dashboard is reviewed quarterly by the compliance officer; engineers know which metrics they're being measured on, and the discipline holds.

**Open-source-project provenance at large multi-contributor repos.** Several large open-source projects (kept generic here) have adopted contributor-DCO-style "signed-off-by" patterns extended to AI assistance: `AI-Assisted-By: copilot 2026.05` as a commit-trailer that survives squash-merge. The pattern is cheap to add, survives history rewrites, and gives downstream consumers of the codebase a verifiable provenance chain.

**Healthcare-clinical-deployment release notes (healthcare).** Hospital-IT teams deploying clinical software increasingly disclose AI assistance in *release notes*, not just internal PRs, because the clinical-deployment runbook is read by clinical leadership who want to know which parts of the workflow used AI. The pattern: each release-note item carries an AI-assistance tag visible to non-engineering readers. This forces the engineering team to maintain the chain from PR annotation to release note, which keeps annotation honesty up.

**Defence-contractor engagement disclosure (regulated industry).** Defence and intelligence-community customers are starting to require AI-usage disclosure as a contractual condition of the engagement. The War On The Rocks 2026 commentary on "your defense code is already AI-generated" treats this as a likely-imminent baseline. The vendor engineering teams that have adopted the annotation discipline in advance hand the customer a complete record at engagement close; the teams that have not face an after-the-fact reconstruction.

## 6. Best Practices

- **Annotate every PR — no exceptions.** A skipped annotation is a hole in the audit chain; the chain is no stronger than its weakest link.
- **Make the annotation a structured field, not free text.** A free-text field becomes inconsistent across authors and time; a structured field is queryable.
- **Require a reasoning sentence.** Free-form "why this mode" defeats performative annotation faster than any automated check.
- **Require reviewer concurrence on the annotation, not just on the code.** Self-attested annotation degrades; reviewer-concurred annotation does not.
- **Plot annotation distribution over time.** A flat line is a drift signal; an unexpected shift is worth a team conversation.
- **Link annotations to prompts where the prompts exist.** If your AI assistant logs prompts, link the log; if it does not, save them deliberately for high-stakes PRs.
- **Sample-audit randomly.** Periodically pick five recent PRs, ask the authors to defend the annotation. The sampling is what keeps the discipline honest.

## 7. Hands-on Exercise

**Code task (10–15 min).** Pick three recent commits or PRs you authored. For each, draft an AI Assistance Disclosure block using the template in §4 (Authoring mode / Tool / Model / Reasoning / Reviewer concurs).

Expected output:

- Three filled disclosure blocks, one per PR.
- The authoring-mode choices should not all be the same value (if they are, sanity-check yourself — is one of them wrong?).
- The reasoning sentence for each should be specific to the PR's content, not boilerplate.
- For one of the three, deliberately mark the annotation you would have *defaulted to* and the annotation you would have *honestly chosen* if forced to think about it. Note any gap.

**What good looks like.** A reader of your three disclosures should be able to predict, before reading the code, what kind of edits to expect in each PR. The reasoning sentences should pass a "could the author defend this if asked?" test. If one of the three honesty-checks revealed a default-vs-honest gap, that is exactly the failure mode the annotation discipline guards against — naming it is a growth signal, not a flag.

## 8. Key Takeaways

After this reading the learner should be able to answer:

- What is AI code provenance, and what is the minimum metadata needed to answer "how was this code authored?" without reconstructing memory? *(maps to LO 1)*
- What are the standard authoring-mode annotation values (human-led / AI-assisted / AI-led / hand-authored), and how do you choose the correct one for a given PR? *(maps to LO 2)*
- What are the three common failure modes of AI-assistance annotation (performative / drift / cross-link gaps), and how is each detected? *(maps to LO 3)*
- Why is per-PR AI-assistance disclosure transitioning from hygiene to regulatory and contractual requirement, and which forces are driving the transition? *(maps to LO 4)*
- What is the structural difference between an annotation that is self-attested and one that is reviewer-concurred, and why does the difference matter for audit credibility? *(maps to LO 1, 3)*

## Sources

1. [EU AI Act: What an Audit Trail for AI-Generated Code Actually Looks Like — CodeSlick](https://codeslick.dev/blog/eu-ai-act-audit-trail-2026) — retrieved 2026-05-26
2. [Audit Trails for AI-Assisted Development: Compliance by Design — Bitloops](https://bitloops.com/resources/governance/audit-trails-for-ai-assisted-development) — retrieved 2026-05-26
3. [10 Best AI Code Provenance Tools 2026: Avoid IP & Legal Risks — CodeBrewTools](https://codebrewtools.com/blogs/ai-code-provenance-compliance-tools-2026) — retrieved 2026-05-26
4. [How to Set Up Real-Time AI Code Audit Trails in 2026 — Exceeds.AI](https://blog.exceeds.ai/realtime-ai-code-audit-trails/) — retrieved 2026-05-26
5. [Governance for your AI Coding Assistant — Knostic](https://www.knostic.ai/blog/ai-coding-assistant-governance) — retrieved 2026-05-26

Last verified: 2026-05-26
