---
title: Mock Interview Prep Guide (Entry Level) — Forward Deployed Engineer (Full Stack + GenAI)
audience: learners — Entry Level / fresh graduates
phase: pre-interview
last_verified: 2026-06-11
source_of_truth:
  - Mock Interview Panel Alignment Document (Entry Level)
---

# Mock Interview Prep Guide (Entry Level) — Forward Deployed Engineer (Full Stack + GenAI)

For the **Entry-Level / fresh-graduate** cohort. This guide tells you what to expect in
your mock interview, what the panel is looking for, and how to prepare. Built from
(1) the panel's Entry-Level alignment document, (2) current research on how Forward
Deployed Engineer interviews are run in 2026, and (3) the skills you built across
Weeks 1–3 of training.

> **Your bar (Entry Level):** the panel is validating that you *understand and can
> explain* the concepts — not that you can architect an enterprise system solo. Clear,
> correct conceptual explanations and honest reasoning beat trying to sound senior.

Last verified: 2026-06-11

---

## 1. The format — what to expect

- **One round, 60 minutes.** This is a conversation, not a coding test.
- **Not a LeetCode interview.** No algorithm puzzles. The panel cares about how you
  think and explain — not whether you can invert a binary tree.
- **Centered on GenAI**, with your Full Stack training as supporting context.
- **Two halves: technical concepts + behavioral.** The topic scope was aligned with the
  client over the last six weeks, so questions stay inside the areas you trained on
  (Weeks 1–3). Expect the panel to spend real time on the **behavioral / how-you-work**
  side as well — it is not an afterthought.
- **Conducted over video conferencing.**
- The panel wants to see you **connect individual concepts into a simple, coherent
  solution** — not recite definitions.

**The single biggest thing the panel rewards:** thinking before answering. The
number-one reason candidates stumble is jumping straight to an answer without first
asking a clarifying question or stating an assumption. On any open question: ask one or
two clarifying questions, say your assumptions out loud, then walk through your thinking.

---

## 2. What the panel evaluates (six areas)

Know all six. The GenAI areas (2–4) carry the most weight; Full Stack (1) is validated
at a **fundamentals / awareness** level.

1. **Enterprise Engineering & Full Stack Foundations** — microservices fundamentals,
   distributed-systems basics, OAuth2 and JSON Web Tokens overview, structured logging,
   correlation IDs, continuous integration and delivery awareness, repository onboarding.
   *(Entry bar: explain the fundamentals.)*
2. **Large Language Model Engineering Fundamentals** — model behavior and limits,
   hallucination failure modes, model selection, Amazon Bedrock invocation, streaming,
   retry strategies, structured JSON output and validation, prompt lifecycle, context
   engineering and compression.
3. **Retrieval-Augmented Generation & Retrieval Engineering** — RAG architecture,
   chunking, embeddings, vector databases, dense / sparse / hybrid retrieval, re-ranking,
   citation grounding, retrieval evaluation (and RAGAS awareness), index management,
   RAG security awareness.
4. **Agentic Systems & AI Workflow Design** — agent architecture, ReAct workflows,
   tool calling, memory, LangGraph fundamentals, agentic retrieval, multi-agent and
   supervisor-worker awareness.
5. **Architecture, Solution Design & Technical Reasoning** — architecture mapping,
   Architecture Decision Record awareness, tradeoff discussions, solution decomposition,
   design planning. *(Entry bar: explain design decisions and discuss system design at a
   conceptual level — logical problem-solving, not enterprise architecture ownership.)*
6. **Communication, Project Understanding & Technical Storytelling** — explaining your
   training assignments and prototypes, justifying your decisions, telling a clear story.

---

## 3. How they score you (the behaviors that win)

The panel prioritizes **understanding, clear reasoning, and communication over advanced
implementation**. Demonstrate these and you signal real potential:

- **Think before you answer.** Ask what success looks like and what the constraints are
  before you start designing.
- **Explain tradeoffs simply.** "I'd pick X over Y because…" — even a simple, correct
  tradeoff is a strong signal at this level.
- **Show evaluation awareness.** Be ready for **"How do you know your AI system is
  actually working?"** with concrete signals (faithfulness, context recall/precision,
  answer relevance, a held-out test set) — not "it looked right."
- **Name an obvious failure mode.** Hallucination, a retrieval miss, an agent doing
  something it shouldn't. Naming one unprompted shows you think past the happy path.
- **Use ownership language.** "I did / I built / I figured out," not a vague "we."
- **Narrate your thinking.** Silence reads as being stuck. Think out loud.
- **Explain to a non-expert.** Practice describing one concept (e.g. what RAG is) in
  plain language.

---

## 4. Practice questions by area

Drawn from current Forward Deployed Engineer / GenAI question banks, scoped to the
Entry bar. Practice answering each out loud in 2–3 minutes.

**Large Language Model Engineering**
- In plain terms, what is a large language model and why does it sometimes hallucinate?
- When would you use retrieval versus just prompting the model?
- How do you get reliable structured JSON out of a model, and how would you check it?
- What would make you pick one Bedrock model over another?

**Retrieval-Augmented Generation**
- Walk me through what a RAG pipeline does, step by step.
- Why do we chunk documents? What makes a good chunk?
- What is an embedding, and what is a vector database for?
- How would you tell whether your retrieval is returning good results?

**Agentic Systems**
- What is an agent, and how is it different from a single LLM call?
- Explain the ReAct loop (reason → act → observe).
- What is tool calling, and why does an agent need memory?
- When might you use more than one agent (supervisor-worker)?

**Architecture & Solution Design**
- For a simple business scenario, what are the main pieces and how does data flow
  between them? *(Ask a clarifying question first.)*
- What is an Architecture Decision Record, and what goes in one?

**Communication**
- Explain to a non-technical person why an AI model can give different answers to the
  same question — and why that isn't necessarily a bug.
- Walk me through a project you built in training. What did you decide, and why?

**Behavioral (expect a real chunk of the hour here)**

Answer with the STAR shape — Situation, Task, Action, Result — in about 60–90 seconds,
using ownership language ("I did"). Have a few stories ready you can flex:
- Tell me about a time you took ownership of a problem that wasn't strictly yours.
- Describe a time requirements changed partway through a project. What did you do?
- Tell me about debugging something in code or a tool you'd never seen before.
- Give an example of a decision you made without all the information you wanted.
- Tell me about something that didn't go as expected. What did you learn?
- How do you handle a teammate disagreeing with your approach?
- Why this role / why a client-facing engineering path?

Tie at least one story back to your training work, so technical and behavioral reinforce
each other.

---

## 5. Out of scope — do not over-prepare these

These are **not** assessed (they are later-week training material). Don't spend prep
time here:

- AI-Native Software Development Lifecycle and Brownfield Modernization
- Requirement decomposition frameworks, integration mapping, modernization strategy
- AI Security Engineering, prompt-injection testing, personally-identifiable-information
  protection frameworks
- Governance and compliance frameworks
- OpenTelemetry, drift detection and monitoring, explainability frameworks
- Operational readiness, client deliverables and final solution defense
- Advanced deployment and production support

**Do not expect** (per the Entry-Level panel guidance): deep React expertise,
Kubernetes administration, production deployment experience, advanced system design, or
enterprise architecture ownership.

### Worth a quick refresh: React and authentication

Two areas are worth a short refresher before the mock — they may come up and are quick
wins. As always, lead with what you know and reason honestly rather than bluffing:

- **React.** Not a primary criterion at this level. Be comfortable with the fundamentals
  (components, state, props, basic data flow).
- **OAuth2 and JSON Web Tokens.** Be ready for the *concepts* — what a JSON Web Token is,
  why it's signed, and the idea of logging in to get a token you send on later requests.

A one-hour refresher on each is the highest-leverage prep outside the GenAI core.

---

## 6. The night before — a short checklist

- Pick **one or two training projects** you can walk through in ~3 minutes each: the
  problem, what you decided, what you'd do differently.
- Prepare **3–4 short stories** (60–90 seconds): a problem you owned, debugging
  something unfamiliar, a decision with incomplete information.
- Rehearse a one-sentence answer to **"How do you know it works?"** for RAG, for an
  agent, and for an LLM output.
- Practice the opening move on any open question: **ask a clarifying question → state an
  assumption → walk through your thinking.**
- Be ready to explain **what RAG is** in plain language.
- Breathe. The panel wants to see how you think, not catch you on trivia. Think out loud.
