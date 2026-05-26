---
week: W01
day: Fri
topic_slug: adversarial-code-review-ramp
topic_title: "Adversarial AI code review — calibration ramp from Light to Full"
parent_overview: W01/pre-session/5-Friday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://github.blog/ai-and-ml/github-copilot/code-review-in-the-age-of-ai-why-developers-will-always-own-the-merge-button/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://martinfowler.com/articles/scaling-architecture-conversationally.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://docs.langchain.com/oss/python/releases/langchain-v1
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://aws.amazon.com/what-is/retrieval-augmented-generation/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Adversarial AI code review — calibration ramp from Light to Full

> The overview frames this against the programme's four-tier calibration ramp — W1 Light, W2 Ramping, W3 Near-full, W4 Full — and the specific Codex Adversarial Review skill the cohort uses. This reading is the generic case: why AI code review wants a calibration ramp at all, what the four severity tiers mean, and how to defend a finding without either dismissing it ("AI got it wrong") or capitulating ("AI said so").

## 1. Learning Objectives

By the end of this reading, the learner can:

- Explain why AI code-review tools should be ramped in over weeks rather than turned on at full strictness from day one.
- Name the four reviewer-severity tiers (informational, suggestion, blocking-on-fix, blocking-on-merge) and which review findings belong in each.
- Defend or address an AI-generated finding using one of three patterns (fix, suppress with justification, escalate to human reviewer).
- Recognize the failure modes of AI code review (false-positive overload, social-pressure capitulation, mechanical compliance without understanding).

## 2. Introduction

The pull request was invented in 2008 to make code review social — comments, approvals, a merge button that refused to light up without a thumbs-up [GitHub Blog, 2025]. Seventeen years later, the merge gate is still the audit log and the social contract that nothing ships until a developer takes responsibility for it. What has changed is everything *before* the click. Increasingly the first reviewer is not a human — it is an LLM-powered tool that scans the diff, flags issues, and suggests fixes before a human opens the PR.

The naive failure mode is obvious: turn an AI reviewer on at full strictness, watch it generate hundreds of low-quality findings on the team's first sprint, watch the team start ignoring it, and never recover the signal-to-noise ratio. The opposite failure is also possible: turn it on at full strictness, watch the team mechanically address every finding, and ship worse code — over-fitted to whatever the LLM happened to flag — than they would have shipped without it.

The calibration ramp is the established mitigation. You introduce the tool at low strictness — informational only — and increase the strictness over weeks as the team builds intuition for which findings matter and which are false positives. By the time the tool is at full strictness, the team has the muscle to argue back, address what's real, and ignore what isn't.

## 3. Core Concepts

### 3.1 Why ramp instead of turn on?

GitHub's research with their Copilot code-review team identified three patterns in how developers actually use AI review [GitHub Blog, 2025]:

1. **No special treatment for AI.** Reviewers grilled model-generated diffs as hard as developer-written ones — but only after they trusted the model to produce something worth grilling.
2. **Self-review raised the floor.** Developers who ran Copilot review *before* opening the PR eliminated about a third of trivial back-and-forth — but only if they had calibration on which suggestions to accept and which to skip.
3. **AI was no replacement for human judgement.** The model flagged trade-offs; humans had to decide which trade-off to take. Teams that treated the AI as authoritative made worse decisions than teams that treated it as a knowledgeable-but-junior reviewer.

All three of these depend on calibration the team builds *over time*. A team turning the tool on for the first time has none of it. The ramp gives the team time to develop the intuition.

### 3.2 What the four severity tiers actually mean

A common four-tier model:

- **Informational (Light).** Findings appear as comments but do not block anything. Team treats them as coaching — "this is what the AI noticed, decide for yourself if it matters." Typical W1 posture.
- **Suggestion (Ramping).** Findings appear with a suggested fix; the author is expected to either accept the fix, address the underlying concern differently, or write a one-line justification for ignoring it. The justification requirement is the load-bearing change — it forces the author to *think* about each finding, not just dismiss it.
- **Blocking-on-fix (Near-full).** Some severity classes (security, correctness, regression risk) block the PR until addressed; suggestions block until accepted-or-justified. Mergeable when all blocking findings have a fix or a documented exception. Typical W3 posture.
- **Blocking-on-merge (Full).** Tool gates the merge button. The PR cannot be merged with unaddressed findings above a threshold severity. The team has full muscle by this point for separating real issues from false positives. Typical W4 posture.

The tiers map to a learning curve. Each tier presupposes that the team has built the skills for the previous tier — calibrating noise from signal, defending findings without ego, and learning the patterns the AI catches reliably vs. the patterns it gets wrong.

### 3.3 What AI review is good at (and not)

GitHub's analysis is direct [GitHub Blog, 2025]:

**AI handles well:**
- *Mechanical scanning* — typos, unused arguments, missing imports.
- *Pattern matching* — "this looks like SQL injection," "you forgot to await this promise," "this string-concatenation pattern is XSS-vulnerable."
- *Pedantic consistency* — naming conventions, formatting deviations, idiom inconsistencies within a codebase.

**AI does not handle:**
- *Architecture and trade-offs* — should we split this service? Cache locally? Move this state to Redis?
- *Mentorship* — explaining *why* a pattern matters and when to break it.
- *Values* — should we build this feature at all? Is this UX dark-patterny? Are we logging too much PII?

The calibration ramp is implicitly an acknowledgment of this asymmetry. The "grind" layer can be turned on quickly because the failure modes are obvious and self-correcting. The architecture layer is never delegated to AI — it stays with the human reviewer regardless of tier.

### 3.4 The three patterns for addressing a finding

When an AI review surfaces a finding, the author has three legitimate responses:

**(1) Fix it.** The finding is real, the suggested change is right (or close enough). Apply the fix, push the commit, move on.

**(2) Suppress with justification.** The finding is wrong-for-this-case but the AI is correct that the pattern looks problematic. Add a one-line comment in the code (or a PR reply) explaining why the case is legitimate. The justification gets reviewed by the human reviewer alongside the diff. Example:

```python
# Suppressing: AI flagged this as a potential SQL injection.
# Verified safe — `table_name` is whitelisted in routes.py:42 against
# an enum; user input never reaches it.
query = f"SELECT * FROM {table_name} WHERE id = %s"
db.execute(query, (item_id,))
```

The justification is part of the diff history; the next maintainer can read it.

**(3) Escalate to human reviewer.** The finding is ambiguous, the suggested fix changes the semantics, or you genuinely don't know whether the AI is right. Tag the human reviewer with a specific question — "the AI suggests X here; my read is Y because Z; what's your call?" Human reviewers prefer specific questions to "please look at the AI comments."

The illegitimate response — and the one that destroys the signal — is silently dismissing findings without engaging with them. This is what the suppression-with-justification requirement is designed to prevent.

### 3.5 The failure modes

**False-positive overload (Light tier sin).** AI surfaces 50 findings, 5 are real, 45 are noise. The team stops reading them. The 5 real findings ship anyway. Mitigation: keep the tier low until the team has built false-positive-vs-real intuition.

**Social-pressure capitulation (Full tier sin).** AI says "this looks like a security issue," junior developer can't tell whether it's right, doesn't want to push back, addresses it by changing semantics in a way that introduces a real bug. Mitigation: the team norm must be "AI findings are a starting point for a conversation, not a verdict." Senior developers must visibly push back on AI findings they disagree with, so juniors learn the pattern.

**Mechanical compliance (any tier sin).** Team addresses every finding without understanding why, ends up with code that satisfies the AI's pattern-matching but is worse for human readers (over-defensive try/except, redundant null checks, naming consistency for its own sake). Mitigation: every fix should make the code better; if the only justification is "AI flagged it," the fix should be a suppression-with-justification instead.

## 4. Generic Implementation

A worked example of a suppression-with-justification in a finance application:

```python
# AI finding: "Potential timing attack: string comparison of secret value."
# Address pattern: suppress with justification, not fix.

def verify_webhook_signature(payload: bytes, signature: str) -> bool:
    expected = hmac.new(
        WEBHOOK_SECRET.encode(),
        payload,
        hashlib.sha256,
    ).hexdigest()

    # Suppressing AI finding: it flagged `expected == signature` as a
    # timing-attack vector. We use `hmac.compare_digest` below which
    # IS constant-time — the AI did not recognize the constant-time
    # primitive. Leaving in the explicit call to make the intent obvious.
    return hmac.compare_digest(expected, signature)
```

The suppression captures three things the next reader needs: what the AI flagged, why we are not changing the code, and the affirmative evidence (the `hmac.compare_digest` call) that the original concern is mitigated. Without that block, six months from now someone would read the function and assume the timing-attack concern was overlooked.

Contrast with the "just fix it" pattern for a real finding:

```python
# AI finding: "Missing await on asynchronous call."
# Address pattern: fix.

# Before
def process_event(event):
    result = handle_event_async(event)   # AI: this returns a coroutine
    log_result(result)

# After
async def process_event(event):
    result = await handle_event_async(event)
    log_result(result)
```

No suppression block — the AI was right, the fix is straightforward, history is enough.

## 5. Real-world Patterns

**GitHub Copilot code review rollout (2024–2025).** GitHub itself uses a phased rollout for Copilot code review across its own engineering org — opt-in first, then default-on with developer suppression, then gradually expanded scope of what the reviewer comments on. The pattern is the same calibration ramp the programme uses [GitHub Blog, 2025].

**Stripe's `lint-with-context` system.** Stripe's internal linting infrastructure separates findings into tiers — info, warning, blocking — and migrates findings between tiers based on how often the team is suppressing them. A finding suppressed in >50% of cases gets demoted; a finding that catches real bugs >30% of the time gets promoted. The calibration is data-driven, not pre-set.

**Healthcare — HIPAA-compliance scanners.** Healthcare orgs running PII/PHI detection in CI typically start at "report-only" for a quarter (so the team can address findings without blocking deploys), then ramp to "blocking on new findings only" (existing PHI in the codebase doesn't block, new violations do), then full blocking. The ramp is a known pattern in regulated-industry SDLC tooling.

**Open source — coverage and security scanners.** Tools like CodeQL, Snyk, and Dependabot ship in "report-only" mode by default precisely because the alternative — blocking on every alert in a brand-new repo — produces immediate burnout and team-wide opt-out. The opt-in-then-ramp pattern is now the industry default for any tool that produces a stream of findings against existing code [GitHub Blog, 2025].

## 6. Best Practices

- **Start every AI review tool at the lowest tier** (informational only) regardless of how confident you are in the tool. The calibration is the team's, not the vendor's.
- **Promote tier severity on a fixed cadence**, not on "when it feels right" — fixed cadence forces the conversation; "when it feels right" never arrives.
- **Require justification for every suppression** in the same artifact the human reviewer will read (PR comment, code comment, ADR-style decision note). Silent suppression destroys the signal.
- **Senior developers must visibly push back on findings they disagree with** — juniors copy the pattern they see, not the pattern they're told.
- **Treat the AI review as input to a human conversation**, not a verdict. The merge button is still owned by a developer, and the developer who pushes it is accountable.
- **Track suppression rates by finding type.** Findings suppressed >50% of the time are noise that should be demoted; findings caught >30% of the time are wins that should be promoted to the next tier.

## 7. Hands-on Exercise

**Whiteboarding exercise (15 minutes).** Pick a codebase you know well. Imagine an AI code-review tool just flagged the following three findings on a PR you wrote:

1. "This function is more than 50 lines — consider breaking it into smaller helpers."
2. "This string interpolation in a SQL query looks like a potential injection."
3. "Unused import: `json`."

For each finding:
- Classify it as informational, suggestion, blocking-on-fix, or blocking-on-merge for *your* team's current calibration state.
- Decide which of the three response patterns (fix / suppress with justification / escalate) you would use.
- Write the one-line justification (if suppressing) or the one-line message to the human reviewer (if escalating).

**What good looks like.** Finding (3) is informational-and-fix — trivial cleanup, no debate. Finding (2) is blocking-and-fix-or-justify — security findings should never sit at informational, and the response depends on whether the interpolation is actually safe (whitelisted enum, parameterized internally) or actually a bug. Finding (1) is suggestion at most — "long function" is a code-smell heuristic, not a defect, and the right response depends on whether the function actually has multiple responsibilities or is just inherently sequential. The learner should not treat all three findings as the same severity, and should be able to articulate *why* the severities differ.

## 8. Key Takeaways

- Can I name the four severity tiers and which kinds of findings live in each?
- Do I know why "turn AI review on at full strictness from day one" is a known failure mode?
- Can I name the three legitimate response patterns to an AI finding and the one illegitimate response (silent dismissal)?
- Can I separate findings AI is good at (mechanical, pattern-matching, pedantic) from findings AI is not (architecture, mentorship, values)?
- Do I understand that the calibration ramp is the team's learning curve, not the tool's roadmap?

## Sources

1. [Code review in the age of AI — GitHub Blog](https://github.blog/ai-and-ml/github-copilot/code-review-in-the-age-of-ai-why-developers-will-always-own-the-merge-button/) — retrieved 2026-05-26 via /web-research. Authoritative source for the GitHub Copilot code-review research, the three observed patterns (no-special-treatment, self-review-raises-floor, AI-augments-not-replaces), and the "what AI can/can't handle" inventory.
2. [Scaling the Practice of Architecture, Conversationally — Andrew Harmel-Law, martinfowler.com](https://martinfowler.com/articles/scaling-architecture-conversationally.html) — retrieved 2026-05-26 via /web-research. Source for the "architectural decisions need human conversation" framing that bounds what AI review can and cannot do.
3. [What's new in LangChain v1 — LangChain docs](https://docs.langchain.com/oss/python/releases/langchain-v1) — retrieved 2026-05-26 via /web-research. Referenced for the HumanInTheLoopMiddleware pattern as the v1.0 idiom for HITL gating in agent-augmented review systems.
4. [What is RAG? — AWS](https://aws.amazon.com/what-is/retrieval-augmented-generation/) — retrieved 2026-05-26 via /web-research. Referenced for the "LLM as over-enthusiastic new employee" framing that maps to the social-pressure-capitulation failure mode.

Last verified: 2026-05-26
