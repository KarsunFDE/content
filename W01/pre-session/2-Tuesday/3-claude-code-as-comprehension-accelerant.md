---
week: W01
day: Tue
topic_slug: claude-code-as-comprehension-accelerant
topic_title: "Claude Code as comprehension accelerant — install, auth, plugins, custom skills"
parent_overview: W01/pre-session/2-Tuesday/1-DailyTopicOverview.md
estimated_minutes: 12
sources:
  - url: https://code.claude.com/docs/en/skills
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://alexop.dev/posts/understanding-claude-code-full-stack/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://blakecrosley.com/guides/claude-code
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://www.morphllm.com/claude-code-extensions
    retrieved_on: 2026-05-26
    recency_category: hot-tech
  - url: https://ofox.ai/blog/claude-code-hooks-subagents-skills-complete-guide-2026/
    retrieved_on: 2026-05-26
    recency_category: hot-tech
last_verified: 2026-05-26
---

# Claude Code as comprehension accelerant — install, auth, plugins, custom skills

## 1. Learning Objectives

By the end of this reading, the learner can:

- Distinguish four Claude Code extensibility surfaces (MCP servers, custom Skills, Subagents, Hooks) by **what they are, where they live, and when to use which**.
- Read a `SKILL.md` frontmatter block and predict whether Claude will auto-invoke it or wait for a slash-command trigger.
- Explain why "verifying Claude's output against the source" is the engineering discipline the cohort is being trained in — not "Claude writes the code for you."
- Identify the file-system path patterns (`.claude/skills/`, `.claude/settings.json`, `.mcp.json`) Claude Code uses to discover extensions.

## 2. Introduction

Claude Code is an agentic CLI from Anthropic that reads codebases, executes commands, edits files, and orchestrates multi-step work via an extensible plug-in system. The 2026 official documentation (code.claude.com/docs) frames extensibility around four layered surfaces: **MCP servers** for external-system connectivity, **Skills** for reusable workflows, **Subagents** for isolated parallel work, and **Hooks** for deterministic event-driven automation. Each surface lives in a different place, has different invocation semantics, and is appropriate for different jobs.

The wider 2026 industry guides (morphllm.com, alexop.dev, blakecrosley.com, ofox.ai) all converge on the same framing: these are not interchangeable. Picking the wrong surface for a job — putting a security check in a Skill instead of a Hook, putting a stateful workflow in a Hook instead of a Skill — produces the wrong outcome. This reading walks the four surfaces so you can read an unfamiliar repo's `.claude/` directory at a federal client and immediately understand what's set up and why.

The deeper point — the one that matters more than any single surface — is the engineering discipline Claude Code enables, not the tool itself. Federal codebases are often the first time anyone has read a system in two years. The README is stale. The original engineer left. Documentation pages are dead links. Claude Code lets you **ask the codebase questions in English and verify the answer against the actual source file**. That muscle — asking, verifying, noticing divergence — is the comprehension-accelerant skill. Treat the tool as the means, not the end.

## 3. Core Concepts

### 3.1 The four extensibility surfaces

The 2026 official docs (`code.claude.com/docs/en/skills`) and the four secondary guides agree on the following decomposition:

| Surface | What it is | File location | Invocation | Context behaviour |
|---------|------------|---------------|------------|-------------------|
| **MCP server** | External-system bridge (Postgres, GitHub, Jira, Slack, browser). Tools show up as `mcp__<server>__<tool>` in Claude's tool list. | `.mcp.json` at repo root, or `~/.claude/mcp.json` globally | Configured via `claude mcp add` or `.mcp.json` entry; tools auto-available | Tools become available to the main conversation |
| **Custom Skill** | Reusable workflow as markdown. SKILL.md with frontmatter + body. Becomes a slash-command. | `.claude/skills/<name>/SKILL.md` (project) or `~/.claude/skills/<name>/SKILL.md` (personal) | Auto (Claude picks it when the description matches) **or** manual (`/<name>`) — controlled by frontmatter | Body loads only when invoked; doesn't cost context unless used |
| **Subagent** | Sub-conversation with its own context window, system prompt, and tool allowlist | `.claude/agents/<name>.md` | Delegated by main Claude when work needs isolation | Separate context window — prevents "context poisoning" of main conversation |
| **Hook** | Event-driven shell script. PreToolUse / PostToolUse / UserPromptSubmit / Stop | `.claude/settings.json` (declarative) pointing to scripts (typically in `.claude/hooks/`) | Automatic — fires on every matching event, no Claude judgement involved | Deterministic; doesn't depend on model interpretation |

The 2026 `code.claude.com/docs` explicitly notes a recent consolidation: **custom commands and skills have been merged**. A file at `.claude/commands/deploy.md` and a skill at `.claude/skills/deploy/SKILL.md` both create `/deploy`. The skills surface is the recommended path going forward — it adds frontmatter for invocation control, supporting files in the skill directory, and the ability for Claude to auto-load when relevant.

### 3.2 Why this matters for federal clients

The morphllm.com 2026 guide names the principle: "Hooks guarantee execution; prompts do not." If a federal-systems integrator needs to **guarantee** that no commit ever contains an AWS access key, that guarantee cannot live in a Skill (which Claude may or may not invoke) or a CLAUDE.md instruction (which the model may de-prioritise under context pressure). It must live in a `PreToolUse` hook on the Bash tool that pattern-matches `git commit` invocations and reads the staged diff for AWS-key patterns. That is the difference between "Claude tries to comply" and "the system refuses to comply."

The same logic applies inversely: a multi-step workflow for "create a Jira-linked PR with appropriate reviewers" is not a hook — it's a Skill. Hooks have no judgement; Skills do.

### 3.3 SKILL.md anatomy

The official docs spec a SKILL.md as YAML frontmatter plus a markdown body. A typical structure:

```markdown
---
name: deploy-staging
description: Deploy the current branch to staging. Run when the user asks
  to "ship to staging", "push to staging", or "deploy this branch".
---

# Deploy to Staging

You are deploying to the staging environment. Steps:
1. Confirm tests pass on the current branch.
2. Verify the user has confirmed they want staging (not prod).
3. ...
```

The `description` field is load-bearing — it's what Claude reads to decide whether to auto-invoke. A vague description (`"deploy stuff"`) gets ignored; a specific one (`"deploy current branch to staging when user says ship to staging"`) gets picked up reliably. The 2026 ofox.ai guide warns explicitly: "Write skill descriptions like you're writing API tool definitions — they're the contract Claude uses for routing."

### 3.4 MCP servers — when to add one

MCP (Model Context Protocol) is the Anthropic-led open standard, originally released in late 2024, for connecting LLMs to external systems. The alexop.dev 2026 piece calls it "USB for AI: one protocol, many tools, many clients." MCP servers expose tools to Claude with names like `mcp__atlassian-rovo__searchJiraIssuesUsingJql` or `mcp__github__create_pull_request`.

At a federal client, MCP is often **the leverage point**. The codebase is bounded by Karsun's contracts; the *integration* between Claude Code and Karsun-internal systems (a custom Jira queue, an internal ticket router, a Confluence corpus) is where Karsun-specific value lives. This is why `.mcp.json` at the repo root is the artefact you'll see most often at engagement — it's the manifest of "what external systems does this project let Claude talk to?"

### 3.5 Hooks as security guardrails

The 2026 guides converge on a small set of canonical hook use cases:

- **`PreToolUse` on Read** — block reads of `.env*`, `*.pem`, `*.key`, `secrets/`.
- **`PreToolUse` on Bash** — block `cat .env`, `printenv`, `aws configure list`, `curl ${IMDS_URL}`.
- **`PreToolUse` on Write/Edit** — block writes containing JWT / AWS-key / Stripe-secret / PEM-block regexes.
- **`PostToolUse` (any)** — scan tool output for PII or secrets and alert if found.

The cohort will trip one or two of these in W1 — that's a feature, not a bug. A blocked Read on `.env` is the hook telling you "you forgot we don't read secrets directly; go look at `.env.example` instead."

## 4. Generic Implementation

A minimal `.claude/` scaffold for a non-Karsun project — say, a small e-commerce SaaS — might look like this:

```
.claude/
├── settings.json           # hook declarations
├── hooks/
│   ├── pre-bash.sh         # block dangerous bash patterns
│   └── pre-read.sh         # block secret-file reads
├── skills/
│   ├── deploy-staging/
│   │   └── SKILL.md        # /deploy-staging slash command
│   └── code-review/
│       └── SKILL.md        # /code-review slash command
└── agents/
    └── perf-investigator.md  # subagent for performance investigations
```

A representative `settings.json` (the declarative hook registry) for the above might be:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": ".claude/hooks/pre-bash.sh" }
        ]
      },
      {
        "matcher": "Read",
        "hooks": [
          { "type": "command", "command": ".claude/hooks/pre-read.sh" }
        ]
      }
    ]
  }
}
```

And a tiny `pre-bash.sh` (executable, returns non-zero to block):

```bash
#!/usr/bin/env bash
# Read the proposed tool input from stdin
INPUT="$(cat)"
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command')

# Block reads of .env files
if echo "$COMMAND" | grep -qE '\bcat\s+.*\.env\b|\bprintenv\b'; then
  echo "Blocked: do not cat .env or use printenv. Use .env.example instead." >&2
  exit 1
fi

exit 0
```

The hook is deterministic — it cannot be talked out of refusing by clever prompting, because it's a shell script, not an LLM call. That property is precisely why hooks are appropriate for security guardrails and inappropriate for stateful business workflows.

## 5. Real-world Patterns

**E-commerce (Shopify-Plus migrations).** A migration team at a large agency runs a Claude Code workflow that wires an MCP server into the merchant's Shopify Admin API, plus a Skill for "convert merchant's legacy theme to Online Store 2.0." The Skill is multi-step (parse theme.liquid, identify deprecated patterns, rewrite). The MCP is the external surface. This is the canonical pattern: **MCP for system access, Skill for the workflow operating on it**.

**Fintech (Stripe-integration consulting).** Independent Stripe consultants frequently maintain personal `~/.claude/skills/` libraries — skills for "audit a webhook implementation for idempotency," "convert Charges-API code to PaymentIntents." These travel with the engineer between client engagements, instantiated at the new repo via `claude skill copy`. The pattern is **personal-skill-library as portable knowledge capital** — a benefit of the new role: your craft accretes in version-controlled files instead of in your head.

**Healthcare IT (EHR-integration vendors).** Hooks dominate. Reading patient data in PHI-tagged tables must be auditable; Claude Code in a HIPAA-bounded environment runs `PostToolUse` hooks on every Read tool call that touches `*_phi` tables, appending an immutable audit record to a CloudTrail-style log. The hook is *deterministic auditability* — the LLM cannot bypass it because the hook fires regardless of the LLM's intent.

**Gaming (live-ops at mid-sized studios).** Some studios run Subagents heavily — a "balance-tester" subagent that simulates 10,000 PvP matches with a proposed weapon-stat change and returns win-rate deltas. The main Claude conversation stays focused on the design conversation; the heavy computation happens in the subagent's isolated context. The pattern is **subagent-as-batch-job** — sensible whenever the work is parallelisable, voluminous, and you don't want it polluting the main context.

## 6. Best Practices

- **Use a Hook when the requirement is "this must execute every time, regardless of the model's interpretation."** Security, compliance, PII detection.
- **Use a Skill when the requirement is "this is a multi-step workflow we want Claude to follow consistently."** Deploys, code reviews, refactors.
- **Use an MCP server when the requirement is "give Claude tool-level access to an external system."** Databases, ticket systems, browsers.
- **Use a Subagent when the requirement is "do heavy parallel work without polluting the main context."** Investigations, multi-file refactors, batch operations.
- **Write skill descriptions like API tool definitions.** Specific triggers and use cases — vague descriptions get ignored at routing time.
- **Always read `.mcp.json` and `.claude/settings.json` first when landing in an unfamiliar repo.** They reveal what external systems are wired in and what guardrails are active.
- **Treat hook trips as information, not errors.** A blocked tool call is the system telling you something. Read the block message, adjust, move on.

## 7. Hands-on Exercise

**Reading-the-extension-surface (12-15 min):**

Open the `acquire-gov` repo in your editor and locate the following four files (these will exist by W1 Tue afternoon — if your stack isn't up yet, do this from memory / the diagrams):

1. `.mcp.json` — list the MCP servers configured. For each, name **one external system** Claude can now access.
2. `.claude/settings.json` — find the `hooks` section. For each hook entry, identify whether it's `PreToolUse` or `PostToolUse` and what file pattern or command it blocks.
3. `.claude/skills/` — list the skill directories. For each, open the `SKILL.md` and read the frontmatter `description` field. Predict whether Claude will auto-invoke it.
4. `.claude/agents/` (if present) — note which subagents exist.

**What good looks like:** a one-page note of the form `acquire-gov extends Claude Code with: <N> MCP servers (X, Y, Z), <M> hooks (security A, security B, audit C), <K> skills (workflow-1, workflow-2), <J> subagents`. That note is the "what's set up here" map you'd carry forward in any federal-client repo. The reading skill itself is the goal — same exercise on a client repo Day 1 of an engagement is week-1 work.

## 8. Key Takeaways

- *Can I distinguish MCP, Skill, Subagent, and Hook by where they live and when each fires?* (Maps to LO 1.)
- *Can I read a SKILL.md frontmatter and predict auto-invoke vs manual?* (Maps to LO 2.)
- *Can I articulate why "verify Claude against source" is the discipline, not "Claude writes code for me"?* (Maps to LO 3.)
- *Can I name the file paths Claude Code uses to discover extensions?* (Maps to LO 4.)

## Sources

1. [Extend Claude with skills — Claude Code official docs](https://code.claude.com/docs/en/skills) — retrieved 2026-05-26
2. [Understanding Claude Code's Full Stack: MCP, Skills, Subagents, Hooks (alexop.dev)](https://alexop.dev/posts/understanding-claude-code-full-stack/) — retrieved 2026-05-26
3. [Claude Code CLI: The Complete Guide — Hooks, MCP, Skills (blakecrosley.com)](https://blakecrosley.com/guides/claude-code) — retrieved 2026-05-26
4. [Claude Code Extensions: MCP, Skills, Agents & Hooks Guide 2026 (morphllm.com)](https://www.morphllm.com/claude-code-extensions) — retrieved 2026-05-26
5. [Claude Code: Hooks, Subagents, and Skills — Complete Guide 2026 (ofox.ai)](https://ofox.ai/blog/claude-code-hooks-subagents-skills-complete-guide-2026/) — retrieved 2026-05-26

Last verified: 2026-05-26
