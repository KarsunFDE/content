---
week: W06
day: Tue
topic_slug: live-demo-discipline
topic_title: "Live-demo discipline"
parent_overview: W06/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 9
sources:
  - url: https://www.ringcentral.com/guides/events/technical-setup-and-rehearsals/dry-runs.html
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://livestorm.co/blog/webinar-dry-run
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://dev.to/hxlnt/use-this-obs-workflow-to-seamlessly-integrate-live-demos-into-pre-recorded-tech-talks-5ff9
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://www.getunleash.io/blog/graceful-degradation-featureops-resilience
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://theappslab.com/2009/07/17/murphys-law-of-demos/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Live-demo discipline

## 1. Learning Objectives

By the end of this reading, the learner can:

- Plan a live software demo around a time-boxed script (target time + hard stop) rather than an open-ended exploration.
- Build a tiered fallback ladder (live → pre-recorded clip → screenshot → verbal walkthrough) for each demo step.
- Pre-stage the demo environment so no live typing, searching, or navigation happens during the actual demonstration.
- Run an effective dry run (timing pass + failure-injection pass + handoff pass) at least 24 hours before the demo.
- Distribute demo-driving responsibility across presenters so any one person's failure does not stop the demo.

## 2. Introduction

Live demos are the highest-leverage and highest-variance segment of any technical presentation. A demo that works lands the entire talk; a demo that fails undermines arguments that would have stood on their own. The variance is largely controllable — most demo failures trace to a small set of preventable patterns (live typing, live searches, live network calls to flaky services, undertested fallback paths).

The discipline that controls variance has three components: a pre-staged environment, a time-budgeted script, and a tiered fallback ladder. None of these are exotic; what is exotic is teams that consistently apply all three. Conference-talk and webinar-rehearsal literature is unanimous on the importance of dry runs — Livestorm recommends a dry run *days* before the live event, with a full timing pass and failure-handling pass ([Livestorm, retrieved 2026-05-26](https://livestorm.co/blog/webinar-dry-run)); RingCentral's guide emphasises rehearsing *both* the success path and the most plausible failure paths ([RingCentral on dry runs, retrieved 2026-05-26](https://www.ringcentral.com/guides/events/technical-setup-and-rehearsals/dry-runs.html)).

Murphy's Law of demos is real, and it has a documented corollary: the degree of disaster scales with the seniority of the audience ([Murphy's Law of Demos, The AppsLab, retrieved 2026-05-26](https://theappslab.com/2009/07/17/murphys-law-of-demos/)). The disciplines below exist because the times you most need a live demo to work are the times most likely to make it fail.

This reading covers the technique generically. The day's overview applies it to the W6 Thu defence demo; this reading is the underlying craft.

## 3. Core Concepts

### 3.1 Time-budget your demo as a script, not an exploration

A demo is not a live debugging session. It is a *scripted performance* of a known-good path through the system. The first discipline is to write the script as a numbered sequence of steps with per-step time budgets, then sum them and confirm the total fits inside your slot with margin.

A useful structure:

| # | Step | Target time | Hard stop |
|---|------|-------------|-----------|
| 1 | Open the dashboard at pre-loaded tab | 0:30 | 1:00 |
| 2 | Trigger the workflow (one click) | 0:15 | 0:30 |
| 3 | Walk the audit row that appears | 2:00 | 3:00 |
| 4 | Pull the corresponding log entry | 1:00 | 2:00 |
| 5 | Show the human-approval gate | 1:30 | 2:30 |
| | **Total** | **5:15** | **9:00** |

The *target* is what you aim for in rehearsal; the *hard stop* is when you abort that step and move on. If a step blows through its hard stop, the script has a branching rule: skip to the next step, deliver the segment verbally, recover later. A common mistake is treating the target as the budget — by the time you discover you're over, you have no recovery margin.

### 3.2 The fallback ladder

Every demo step should have a four-rung fallback ladder:

1. **Live, in production-like environment.** First choice. Real system, real data (or realistic synthetic data), real network.
2. **Live, in pre-staged local environment.** Same flow, but pointed at a local stack you control. The network dependency is removed.
3. **Pre-recorded video clip.** A 30-to-60-second screen recording of the same workflow, captured the day before, played from a local file (not from a hosted video service that might buffer).
4. **Annotated screenshot.** Static, but present. The clearest path through the workflow as a single image with arrows or callouts.
5. **Verbal walkthrough.** The fallback of last resort. Practised once during rehearsal so the cadence is rehearsed, not improvised.

The ladder works because each rung degrades the experience but preserves the *information*. The audience still learns what the system does — they just see less of it live. This is the same principle as graceful degradation in production systems: the user can still complete the task even when some components fail ([Graceful Degradation, Unleash, retrieved 2026-05-26](https://www.getunleash.io/blog/graceful-degradation-featureops-resilience)).

Each rung needs to be tested. An untested fallback is not a fallback; it's a second possible failure. The most embarrassing demo failures are typically the cascade where the live demo dies, the pre-recorded clip won't play, and the screenshot is on a hard drive on the wrong laptop. Test every rung in the dry run.

### 3.3 Pre-stage everything

The biggest source of demo failure is live typing, searching, and navigation. Each introduces latency, typo risk, and the chance of pop-up notifications at the wrong moment. Pre-stage every input:

- **Tabs pre-loaded** with exact URLs in a dedicated demo browser profile (no other tabs, history, or notifications visible).
- **Terminal commands pre-pasted** in a multiplexer pane, ready with one keystroke. Typing accuracy under pressure is lower than you assume.
- **API requests pre-built** in Postman or HTTPie with body, headers, auth ready.
- **Notifications silenced** (Do-Not-Disturb on; Slack and email closed; calendar muted).
- **Dashboard time windows pre-set** to a fixed window — not "last hour" auto-refresh that may roll past your data.

A dedicated demo user profile with only demo-relevant tools eliminates a whole failure class ([Livestorm, retrieved 2026-05-26](https://livestorm.co/blog/webinar-dry-run)).

### 3.4 Distribute the demo across presenters

If only one person can drive, any failure affecting that person stops the demo. Every step should be drivable from at least two laptops or by at least two presenters. Both presenters speak to every step; both laptops are logged in; both have the pre-staged tabs and pre-pasted commands. The two-presenter discipline doubles as a comprehension test — if presenter B cannot drive step 4 cold, shared understanding hasn't been built.

### 3.5 The dry run, properly run

A dry run is not "we ran through the demo once." A proper dry run has three passes:

1. **Timing pass.** Stopwatch on. Run the demo exactly as you would live. Record per-step times. Compare against the target and hard-stop columns. Identify any step that drifted; tighten or restructure.
2. **Failure-injection pass.** Deliberately fail each step in turn (kill the network mid-step 2, force a Bedrock rate-limit on step 3, close the terminal on step 4). Verify the fallback ladder works. Note which rung was reached.
3. **Handoff pass.** Run the demo with each presenter driving each step. Verify both presenters can speak fluently to every section.

Industry guidance is consistent: dry runs should happen days before the live event, not minutes ([Livestorm webinar dry-run checklist, retrieved 2026-05-26](https://livestorm.co/blog/webinar-dry-run); [Pega on dry-run execution, retrieved 2026-05-26](https://academy.pega.com/topic/execution-successful-dry-run-session/v1)). The 24-hour gap is the minimum that lets you fix what the rehearsal surfaced.

### 3.6 Recording the live demo

A common pattern is to integrate a pre-recorded clip *inside* a live demo, then transition back to live for Q&A. OBS workflows support this hybrid pattern — pre-record risky steps, play as scene cuts, stay live for Q&A and unscripted exploration ([OBS hybrid workflow, retrieved 2026-05-26](https://dev.to/hxlnt/use-this-obs-workflow-to-seamlessly-integrate-live-demos-into-pre-recorded-tech-talks-5ff9)). For high-stakes audit demos this is often the right answer.

## 4. Generic Implementation

A short script and fallback ladder for a generic e-commerce live demo: "show how a refund request flows from customer chat to finance reconciliation."

**Pre-stage checklist:**

- [ ] Demo browser profile: 4 tabs open in order — customer chat UI, refund-admin dashboard, finance ledger, audit log viewer.
- [ ] Terminal: 3 panes — kill-switch script (in case live data needs to be reset), tail of the refund-service log, tail of the audit log.
- [ ] Test customer account `demo-acct-04` pre-loaded with a $42.00 charge from earlier today.
- [ ] Refund-admin user `demo-reviewer` logged in on the second laptop.
- [ ] OBS scene 1 = live laptop screen; OBS scene 2 = pre-recorded clip of step 3.
- [ ] Do-Not-Disturb on, Slack closed, email closed, calendar reminders off.

**Demo script:**

| # | Step | Target | Hard stop | Fallback ladder |
|---|------|--------|-----------|-----------------|
| 1 | Customer asks for refund in chat (typed by demo-customer account on phone) | 0:30 | 0:45 | Skip phone, paste pre-staged message |
| 2 | Refund-admin dashboard surfaces the request | 0:30 | 1:00 | Refresh dashboard manually; if blank, switch to pre-recorded clip |
| 3 | Admin approves; ledger entry appears | 1:00 | 2:00 | Pre-recorded 45s clip ready in OBS scene 2 |
| 4 | Audit log shows row with admin identity | 1:00 | 1:30 | Static screenshot in slide 14 |
| 5 | Finance reconciliation report includes the refund | 1:00 | 2:00 | Verbal walkthrough; show last week's report instead |
| | **Total** | **4:00** | **7:15** | |

**Dry-run protocol (24 hours before):**

- Timing pass: run all five steps with stopwatch. Steps 3 and 5 are most likely to overrun — pre-trim if they do.
- Failure-injection pass: kill the refund-service Docker container after step 2; verify the pre-recorded clip plays cleanly from local file (not from cloud video).
- Handoff pass: presenter A drives steps 1-3, presenter B drives steps 4-5; swap and repeat. Verify both can fluently narrate every step.

After this protocol, the live demo's failure modes are mostly the network — and even network failure has a known fallback (clip + screenshot + verbal). The chance of a wholesale demo failure approaches the chance of a complete equipment failure (laptop bricked, projector dead), which is a different category of risk handled by the venue.

## 5. Real-world Patterns

**Apple keynote demos.** Apple's product-keynote demos are rehearsed for weeks. The "live" demos run on re-imaged machines on event-reserved Wi-Fi, executing scripts that look natural but are rehearsed sequences. The 2007 iPhone demo was the product of dozens of rehearsals and a chosen "golden path" — any step that had failed was fixed or removed. The discipline travels: pick the path, rehearse it, remove what fails.

**Twitch and YouTube product launches.** Platform companies record demo videos under controlled conditions (same hardware, same network, identical input) and layer live presenter narration on top. The pattern protects against streaming-environment variance while preserving real-time feel ([OBS scene-collection workflow, retrieved 2026-05-26](https://dev.to/hxlnt/use-this-obs-workflow-to-seamlessly-integrate-live-demos-into-pre-recorded-tech-talks-5ff9)).

**Aviation — pilot procedural checklists.** Commercial pilots do not improvise during takeoff; they run a script (pre-flight, taxi, take-off, climb) rehearsed in thousands of simulator hours. Failure branches (engine failure, bird strike, hydraulic failure) are also rehearsed scripts, not improvisations. Demo dry-run failure-injection is the engineering analogue.

**Surgery — pre-op briefings.** Surgical teams run a structured pre-op briefing (WHO Surgical Safety Checklist) naming patient, procedure, allergies, expected blood loss, and contingency plans for the two most likely complications. A verbal rehearsal of the script and its branches; fewer surprises during the procedure.

## 6. Best Practices

- **Write the demo as a numbered script with per-step time budgets and hard stops.** No open-ended exploration during the live demo.
- **Build a 4-or-5-rung fallback ladder for every step.** Live → local → recorded clip → screenshot → verbal. Every rung tested in dry run.
- **Pre-stage everything**: tabs, terminals, API requests, dashboard time windows. No live typing or live searching during the demo.
- **Dedicate a demo user profile** with only demo-relevant tools and bookmarks. Notifications silenced.
- **Run the dry run 24 hours in advance** with three passes: timing, failure-injection, handoff. Fix what surfaces; don't carry it into the live event.
- **Distribute drive-ability across at least two presenters and two laptops.** Single-point-of-failure presenters are a single point of failure for the demo.
- **Reserve the live-demo slot for the steps that need to be live.** Pre-record the fragile steps and integrate them with OBS or equivalent; stay live for Q&A.

## 7. Hands-on Exercise

**Time:** 15 minutes. **Format:** plan + rehearse one micro-demo.

Pick a feature you've worked on. Plan a 5-minute live demo of it for an audit-oriented audience (someone who wants to verify the system behaves as claimed, not someone who wants to be wowed).

**Steps:**

1. **Script the demo as 4-6 numbered steps**, each with a target time and a hard stop. Total should be ≤ 5 minutes.
2. **Build a fallback ladder for each step**: name what you would do if that step failed live. At minimum identify rungs 1, 3, and 5 (live / pre-recorded / verbal); rungs 2 and 4 are optional.
3. **List the pre-stage checklist** — every tab, terminal, account, and notification setting you would touch before the demo.
4. **Identify one place where you would pre-record** rather than live — the step most likely to fail or take too long.

**Self-check:**

- Does the total target time leave at least 20% margin against the 5-minute slot?
- Does every step have at least three rungs in its fallback ladder?
- Is there any step that *only one presenter* can drive? If so, why?
- If a notification popped up during step 3, would the audience see it? (Did you remember Do-Not-Disturb?)

**What good looks like:** the script reads as five numbered steps, each two or three sentences; each step has a named fallback at the recorded-clip rung; the pre-stage checklist names every tab and terminal you'll touch; you've identified the step most worth pre-recording. If your script reads like "first I'll show the dashboard, then I'll click around and show some interesting things" — you do not have a script; you have an exploration. Re-script.

## 8. Key Takeaways

- Can I produce a numbered script with per-step time budgets and hard stops, totalling under my slot with margin?
- For every step, can I name at least three fallback rungs (live / pre-recorded / verbal)?
- Have I tested every rung in a real dry run, not just visualised it?
- Can both presenters drive every step fluently — and could either continue alone if the other's laptop failed?
- Have I pre-staged the environment so no live typing or searching is required during the demo?

## Sources

1. [Conducting Dry Runs and Virtual Event Rehearsals (RingCentral)](https://www.ringcentral.com/guides/events/technical-setup-and-rehearsals/dry-runs.html) — retrieved 2026-05-26
2. [Webinar Dry Run: Getting Ready for Your Event (Livestorm)](https://livestorm.co/blog/webinar-dry-run) — retrieved 2026-05-26
3. [Use this OBS workflow to seamlessly integrate live demos into pre-recorded tech talks (DEV)](https://dev.to/hxlnt/use-this-obs-workflow-to-seamlessly-integrate-live-demos-into-pre-recorded-tech-talks-5ff9) — retrieved 2026-05-26
4. [Graceful degradation in practice (Unleash)](https://www.getunleash.io/blog/graceful-degradation-featureops-resilience) — retrieved 2026-05-26
5. [Murphy's Law of Demos (The AppsLab)](https://theappslab.com/2009/07/17/murphys-law-of-demos/) — retrieved 2026-05-26

Last verified: 2026-05-26
