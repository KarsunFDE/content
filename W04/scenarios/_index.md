---
week: W04
scenarios_released_at: "Mon W4 16:00 (in Plan Day spec) — pairs research Wed PM dedicated /web-research slot per D-040"
adr_due: "EOD Thu W4"
live_defense_scheduled: "Fri W5 (NOT W4 Fri — W4 Fri is Mid-Sprint Surprise, Live Defense slot repurposed to defend amended plan-spec)"
last_verified: 2026-05-23
---

# W04 scenario-alternatives index

> 3 scenarios per D-040 (1:1 mapping to 3 pairs). Each carries 2–3 candidate techs. Pairs research collaboratively (Senior + Entry together, not split).

| ID | Title | Default tech | Candidate alternatives | Pair owner |
|----|-------|--------------|------------------------|------------|
| W04-SA-1 | Circuit breaker library for Item 3 + W5 AIOps hand-off | Resilience4j | Hystrix (legacy), hand-rolled | Pair 1 (TBD W4 Mon AM assignment) |
| W04-SA-2 | Prompt-injection defenses for the 4 stored-content surfaces | Pydantic-v2 input/output validators (in-house) | LLM-level guardrails (Bedrock Guardrails — managed, deferred W5+), sandboxing | Pair 2 (TBD W4 Mon AM) |
| W04-SA-3 | Modernization sequencing — J17 first vs SB3 first | J17 first then SB 3.0 (D-054 default) | SB 3.0 first then J17, combined single-stage | Pair 3 (TBD W4 Mon AM) |

## Cadence notes

- Scenarios released in Mon W4 Plan Day spec (per D-040). Pairs research Wed PM in dedicated /web-research slot (which doubles as the AI Security Day practical block).
- ADR due Thu W4 EOD — same day as the Modernization Execution Day deadlines. Plan accordingly.
- **Live Defense slot moves to W5 Fri** — W4 Fri's Live Defense is repurposed for the amended plan-spec defense post-Mid-Sprint-Surprise. Scenario ADRs sit until W5 Fri to avoid colliding with the surprise.
