---
week: W06
day: Wed
topic_slug: dry-run-discipline
topic_title: "Dry-run discipline"
parent_overview: W06/pre-session/3-Wednesday/1-DailyTopicOverview.md
estimated_minutes: 10
sources:
  - url: https://newventure.org.uk/index.php/production-manual/technical-rehearsal-process
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://livestorm.co/blog/webinar-dry-run
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://sreschool.com/blog/comprehensive-tutorial-on-graceful-degradation-in-site-reliability-engineering/
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
  - url: https://docs.cloud.google.com/architecture/framework/reliability/graceful-degradation
    retrieved_on: 2026-05-26
    recency_category: foundation-stable
last_verified: 2026-05-26
---

# Dry-run discipline

## 1. Learning Objectives

By the end of this reading, the learner can:

- Describe why a single full-pass dry-run delivers more usable feedback than two partial passes.
- Distinguish between *cue-to-cue* style preparation and *full-run* preparation, and explain when each is appropriate.
- Articulate how the graceful-degradation principle from SRE transfers to the live-demo failure-handling design.
- Identify the three failure modes a dry run is designed to surface (timing, sequence, environmental) and how each is detected.
- Apply the "name-it-when-it-breaks" rule that converts a live failure from a credibility loss into a credibility signal.

## 2. Introduction

A dry run is the *evidence-collection* event upstream of the prep checklist. It is the moment where the actual demo, rehearsed in the actual room (or as close as possible) under actual time constraints, surfaces what is not yet ready. Without a dry run, the team enters the live defense relying on the optimism bias of the rehearsal in their head — every transition smooth, every demo step clean, every Q&A answer crisp. With a dry run, the team enters with a small set of named gaps and a small set of named fallbacks. The size of the gap-and-fallback set is bounded by the dry-run discipline; the cost of an unbounded gap-and-fallback set is paid live, in front of the panel.

Theatre production captures the principle precisely: technical rehearsals are organized as distinct *staged* events — first technical, then cue-to-cue, then dress/tech — with the explicit recognition that "this is the last chance to solve problems" [1]. The same principle applies to a technical defense, where a structured dry run is the last chance to surface problems while there is still time to act. Webinar-dry-run guidance is equally direct: dry runs exist to "test technology, refine the agenda, and practice audience engagement prior to a live event," and their value is in identifying and resolving issues "before the live event" [2].

This reading covers the discipline of a single-pass dry run — its design intent, the failure modes it is built to surface, and the graceful-degradation framing that converts dry-run findings into live-demo resilience. The daily overview specifies what the cohort's Wednesday-afternoon dry run will look like in detail; this reading covers the underlying principle.

## 3. Core Concepts

### 3.1 Why one full pass beats multiple partial passes

The strongest argument for a *single, full, end-to-end* dry run rather than two or three partial passes is that the failure modes a dry run is designed to surface only appear under the full-pass timing. Theatre production guidance frames this as the distinction between *cue-to-cue* and *dress/tech*: cue-to-cue runs each technical cue in sequence but does not run the show in real time, while dress/tech is "a run in real time, with all elements of the production, with only essential stops in cases where adjustments need to be made" [1]. Both are valuable; they surface different problems.

For a technical defense, the equivalent distinction is between *fragment rehearsal* (running individual demo segments to polish them) and *full-pass dry run* (running the whole demo end-to-end at the actual time budget). Fragment rehearsal surfaces vocabulary and pacing problems within a segment. Only a full pass surfaces:

- **Cumulative time drift.** Each segment is 30 seconds over budget; in fragments this looks fine; in a full pass the demo runs 3 minutes long.
- **Energy decay.** The demo's energy at minute 10 is a function of how minute 1 went, which fragment rehearsal cannot capture.
- **Transition friction.** The handoff from segment A to segment B has its own latency cost that only shows up when you do it in order.
- **Q&A pacing.** Real Q&A at the end of a 15-minute demo lands differently than Q&A at the end of a 3-minute fragment.

The single full pass is the *only* signal source for these failure modes, and the modes are exactly the ones a panel will notice live.

### 3.2 What a dry run is designed to detect

A well-designed dry run surfaces three classes of problem [1][2]:

- **Timing.** Does the demo fit the budget? Is the cut crisp? Does Q&A have room to breathe? Webinar-dry-run guidance is explicit that "presenters and moderators become familiar with the platform, allowing for smoother delivery and better interaction with attendees" [2] — and the "smoother delivery" is the dry-run-acquired knowledge of where you have time to slow down and where you must move on.
- **Sequence.** Are the demo steps in the right order? Does the panel have the prerequisite mental model when each step lands? Is there a step that depends on something you forgot to show?
- **Environmental.** Does the projector show what you expect? Does the network reach the API? Does the laptop's screen-share configuration cooperate? These are the items that a dry run in the *actual* environment catches and a dry run on your laptop misses.

Each class is detected by a different observation discipline. Timing needs a stopwatch. Sequence needs an observer who is following the logic, not just the screen. Environmental needs the actual physical setup. A dry run that lacks any of these surfaces only a subset of its possible findings.

### 3.3 The single-iteration constraint

When time is compressed (a four-day instead of five-day week, a defense the morning after the dry run), the dry run is a single iteration, not a multi-iteration loop. This constraint is *not* a degradation of the discipline — it is the discipline shaping itself to the available time. Theatre's dress/tech is described as "the last chance to solve problems" [1] precisely because there is no second dress/tech.

The implication is operational: every feedback signal the dry run produces is acted upon or not within the available evening. The team cannot defer to "the second dry run we'll do tomorrow morning" because there is no such second dry run. This forces the prioritization discipline covered in the prep-checklist reading — top-1-or-2 material items get acted on; the rest become honestly-named known weaknesses.

### 3.4 Graceful degradation as a live-demo principle

Site Reliability Engineering's graceful-degradation principle is a near-perfect transfer to live-demo failure handling. Google Cloud's reliability framework defines graceful degradation as a system's ability to maintain limited but useful functionality when parts of it fail or become overloaded [4]. The SRE School definition is more pointed: "Instead of crashing entirely, the system sacrifices non-critical features or performance to ensure core services remain available" [3].

For a live demo, the same pattern applies at the human level. The components are: the live system, the speaker's narrative, and the audience's attention. When the live system fails (rate-limit, network drop, projector glitch), the graceful-degradation response is:

- **Throttle / drop the non-critical part.** Don't insist on completing the failing live call; pivot.
- **Serve from a fallback.** The pre-recorded video, the static screenshot, the printed handoff page — these are the "cached results" in the SRE analogy [3].
- **Preserve the core service.** The narrative must continue. The audience's mental model of what the demo is *about* should not break.

The dry run is when the graceful-degradation paths are chosen and rehearsed. A graceful-degradation path that has never been rehearsed is operationally a half-built feature — it exists on paper but has never been exercised, and the first exercise will be live.

### 3.5 The "name-it-when-it-breaks" rule

A subtle but high-value piece of dry-run discipline: when something breaks live, *name it*. The DemoSecret guidance on live technical demos is direct: "When they ask about something you do not support, say so... Deflection erodes trust. Honesty builds it" [4 — DemoSecret, referenced in the Tue Live-Demo Discipline reading]. The same principle extends from "things we don't support" to "things just broke live" — naming the failure converts it from a credibility loss into a credibility signal.

The practical phrasing pattern: *"That endpoint just hit a rate-limit — let me show you the same query from the pre-recorded run instead."* This phrasing does three things at once: (a) names what broke, (b) tells the audience what's coming next, (c) signals that the failure was anticipated and handled. The audience's attention recovers; the demo continues; trust holds. The opposite phrasing — hand-waving the failure, blaming the demo gods, pretending nothing happened — converts a recoverable moment into a sustained credibility leak.

The dry run is where this phrasing is *practiced*. The first time you fire your fallback should not be live; it should be in the dry run, where you can hear yourself say the words and adjust the cadence.

## 4. Generic Implementation

For a topic about dry-run discipline, the implementation is the dry-run protocol itself rather than a code snippet. A 25-minute structured dry-run protocol for a 15-minute live demo might look like this:

```
00:00–00:01  Setup: project to the actual screen (not the laptop screen)
00:01–00:16  Full-pass demo (15 min — same content, same script as live)
              Stopwatch runs. No restarts. If you stumble, recover live —
              this is the rehearsal of recovery.
00:16–00:20  Q&A simulation (4 min — observer poses 2-3 likely questions)
              Same rule: no restarts. Practice the answers in real time.
00:20–00:23  Fallback rehearsal (3 min)
              Trigger one fallback intentionally. Practice the phrasing.
              "That endpoint just hit a rate-limit — let me show you..."
00:23–00:25  Observer debrief (2 min — verbal only)
              Observer surfaces the top 3 notes, ranked by materiality.
              Team acknowledges; full debrief and prioritization happens
              in Layer 1 of the prep checklist immediately after.
```

What this protocol deliberately does *not* include: a second pass, an iteration on a section that didn't land, an in-flight redesign of the demo flow. Those are out of scope by design. The dry run's job is detection; the prep checklist's job is action.

The "no restarts" rule is the load-bearing piece. The dry run is *testing recovery*, not *avoiding the need for it*. A dry run that pauses and restarts every time something stumbles produces a polished fragment-rehearsal output and zero data about how the team recovers under live pressure — which is the failure mode the actual defense will exercise.

## 5. Real-world Patterns

**Theatre — dress/tech rehearsal.** Professional theatre's dress/tech rehearsal is the canonical full-pass-with-no-restarts discipline. The New Venture Theatre production manual describes it as "a run in real time, with all elements of the production, with only essential stops in cases where adjustments need to be made" [1]. The "essential stops" carve-out is narrow — a stop is justified only when an adjustment cannot wait for the post-rehearsal debrief. Most stumbles, missed cues, and partial-prop failures are run through, not paused; the cast and crew practice recovering live because that is the discipline the actual performance will require.

**Aviation — line-oriented flight training (LOFT).** Commercial aviation's LOFT exercises are full-pass simulator sessions in which the flight crew runs an entire mission, including induced failures, without instructor intervention. The intent is identical to a theatrical dress/tech: rehearse recovery in real time. Post-LOFT debriefs review the recovery patterns rather than the technical failures; the failures are induced precisely so the crew has something to recover from. Published FAA training data credits LOFT for measurable improvements in crew resource management — the recovery skill itself.

**Esports — scrim matches.** Professional esports teams rehearse with scrim matches: full game-length practice runs against other professional teams, played as if they were tournament matches. The discipline is strict — no in-game coaching, no pause-to-discuss, post-match review only. The intent is to practice the team's coordination *under tournament conditions*, including the failures (lost teamfights, mis-rotations, draft mistakes). Teams that scrim heavily but only with in-game coaching report worse tournament outcomes than teams that scrim less but follow the no-pause discipline.

**Healthcare — high-fidelity simulation.** Hospital and emergency-medicine training programs use high-fidelity simulators (mannequins with computer-controlled physiology) for full-pass code-blue and trauma-resuscitation simulations. Published outcomes data shows that simulation programs with the "no-pause, debrief-after" structure produce measurably better real-world resuscitation team performance than programs that pause to teach mid-scenario. The discipline transfers: the failure of the dry-run-with-pauses model is the same across domains [3].

## 6. Best Practices

- **One full pass beats multiple partial passes** when the time budget is constrained — partial passes cannot surface cumulative timing drift, energy decay, or transition friction.
- **Run the dry run in the actual environment** (or as close to it as possible) — the same projector, the same room, the same network path. Environmental failure modes are otherwise undetectable [1][2].
- **Stopwatch the full pass.** The single most useful artifact a dry run produces is the actual elapsed time per segment, which feeds the prep checklist's "where do I cut?" decision.
- **Enforce the no-restart rule.** Stumbles are run-through, not paused; recovery is what is being rehearsed.
- **Intentionally fire at least one fallback during the dry run.** A fallback that has never been exercised is half-built — verify it works and that the phrasing transitions feel natural [3][4].
- **Have an observer following the logic, not the screen.** The observer's job is to notice whether the panel will track the argument, not whether the screen looks polished.
- **Cap dry-run debrief output at top-3 notes.** Beyond three, the team cannot meaningfully act on all of them in the remaining time; the rest become honestly-named known weaknesses.
- **Practice the "name-it-when-it-breaks" phrasing** so the words feel natural when something fails live. The dry run is where the phrasing's cadence is rehearsed.

## 7. Hands-on Exercise

**Protocol-design prompt (15 min).** You are leading a team of three engineers who will demo a new internal data pipeline tomorrow at 10:00 AM to a panel of two senior architects and a director. The demo is 12 minutes; the demo environment requires a VPN to access the production-like staging cluster. You have one 30-minute window today at 17:00 for a dry run.

On paper or in a markdown file, design the **30-minute dry-run protocol** for that window. Your design must explicitly include:

- The full-pass discipline (no restarts, stopwatch, actual environment).
- Which fallback you will intentionally fire during the dry run, and the phrasing the demoer will practice when it fires.
- The role of the observer — who plays the role, what they are watching for, how they capture notes without interrupting.
- The 5-minute debrief at the end, capped at three actionable notes.
- The hard line between dry-run scope and post-dry-run prep-checklist scope (what stays in this 30 minutes vs. what waits for the evening).

**What good looks like:** The full-pass demo gets ~15 minutes (12 for demo, 3 for buffered Q&A). At least 3 minutes is reserved for an intentional fallback (e.g., disconnect the VPN and verify that the pre-recorded video plays cleanly and that the demoer's "the VPN just dropped — let me show you the recorded version" phrasing comes out naturally). The observer is named, sitting at the back of the room watching the panel's likely view, and capturing notes silently (the laptop or paper, not interrupting). The 5-minute debrief is structured (top-3 notes, ranked, materiality-tagged). Items beyond top-3 are explicitly *deferred* to the prep checklist or to the known-weaknesses doc, not absorbed into Wed-evening re-rehearsal.

## 8. Key Takeaways

- *Why does a single full-pass dry run beat multiple partial passes when time is constrained?* (Because cumulative-timing, energy-decay, transition-friction, and Q&A-pacing failure modes only appear under the full-pass timing — partial passes cannot surface them.)
- *What are the three failure-mode classes a dry run is designed to detect, and how is each detected?* (Timing via stopwatch; sequence via an observer following the logic; environmental via the actual physical setup.)
- *How does the SRE graceful-degradation principle map onto live-demo failure handling?* (The live system is throttled or dropped on the non-critical part; the fallback serves the equivalent of cached results; the narrative — the core service — must continue.)
- *Why is the no-restart rule load-bearing in a dry run?* (The dry run rehearses recovery, not avoidance; pausing to restart erases the only signal about how the team behaves under live pressure.)
- *What is the "name-it-when-it-breaks" phrasing pattern, and why does naming the failure convert credibility loss into credibility signal?* (Naming what broke + signaling what's next + showing the failure was anticipated; this preserves audience trust where hand-waving would erode it.)

## Sources

1. [Technical Rehearsal Process (New Venture Theatre Production Manual)](https://newventure.org.uk/index.php/production-manual/technical-rehearsal-process) — retrieved 2026-05-26
2. [Webinar Dry Run: Getting Ready for Your Event (Livestorm)](https://livestorm.co/blog/webinar-dry-run) — retrieved 2026-05-26
3. [Comprehensive Tutorial on Graceful Degradation in Site Reliability Engineering (SRE School)](https://sreschool.com/blog/comprehensive-tutorial-on-graceful-degradation-in-site-reliability-engineering/) — retrieved 2026-05-26
4. [Design for graceful degradation (Google Cloud Architecture Center)](https://docs.cloud.google.com/architecture/framework/reliability/graceful-degradation) — retrieved 2026-05-26

Last verified: 2026-05-26
