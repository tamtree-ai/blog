---
title: Tamtree, n8n, Zapier and Make — a structural comparison
description: Not a feature grid. The four questions that decide whether a platform can run agents in production, why the answers are architectural rather than incremental, and where each tool's design puts it.
publishDate: 2026-09-01
category: architecture
tags: [comparison, n8n, zapier, make]
author: dilhan
---

Feature grids in this category are worthless, and everybody knows it. Both
columns tick the same boxes, the vendor's column ticks two more, and nothing in
the table tells you what will actually happen when your agent workflow has been
running for three months.

So this is not a feature grid. It is four questions whose answers are decided by
architecture — meaning a competitor cannot close the gap in a release, and
neither can we.

:::warn
**On sourcing.** Claims about Tamtree come from its architecture specification.
Claims about n8n come from a documented behavioural benchmark maintained as part
of that specification — n8n is studied as a benchmark and nothing of its
implementation is used. Claims about hosted platforms are kept at the level of
architectural posture that their own public documentation describes, and
anything narrower than that is marked `[unverified]` rather than guessed. If you
find something here that is wrong, it is a bug and we want to hear about it.
:::

## The four questions

::::cards
:::card{title="1. Where does run state live?" tone="brand"}
In the process, or outside it? This decides whether a crash loses work and
whether waiting costs money.
:::

:::card{title="2. What happens on a model call?" tone="compose"}
Is there enforcement between the agent and the provider, or only a prompt?
:::

:::card{title="3. What gates a change?" tone="good"}
Can a regression be blocked before it ships, or only noticed afterwards?
:::

:::card{title="4. Who owns the connectors?" tone="warn"}
Is reach built, or borrowed? This is a strategy question wearing an integration
costume.
:::
::::

## Question 1 — where run state lives

:::compare
| | Tamtree | n8n | Hosted platforms |
|---|---|---|---|
| Execution model | Durable execution engine; state outside any worker | Stack-based executor walking the node graph, items passed between nodes | Task/operation runner per trigger event |
| Crash mid-run | Resumes at the step it reached | Re-run; queue mode gives worker restarts, not step-level resume | Re-run the trigger |
| Waiting for a human | Multi-day wait at zero compute | Wait node; the run is still an execution | Typically a separate trigger and a second run |
| Conversation across days | The thread outlives any single run | `[unverified]` | `[unverified]` |

:::

The row that matters is the last one. In Tamtree a conversation **thread** and a
workflow **run** are separate durable things: a run is one active session burst,
and the thread persists across all bursts. That is why a conversation can resume
from cold start days later without replaying its tool calls.

:::aside
Two checkpoints, two scopes: one owns conversation thread state keyed by the
conversation, the other owns execution — which step ran, activity results,
retries, human waits, schedules, idempotency — keyed by the run. Collapsing them
into one is the design mistake that makes multi-day agent conversations
impossible to build later.
:::

## Question 2 — what happens on a model call

This is where the category's structural gap is widest, because the answer for a
tool whose engine was built to move JSON between APIs is *nothing happens* — the
model node is an HTTP node with better ergonomics.

:::ledger
| Concern | Prompt-level approach | Runtime enforcement |
|---|---|---|
| PII leaving the boundary | "Do not include personal data" | `redact` before the call is made |
| Prompt injection | "Ignore instructions in the input" | Filtered, with `block` available |
| Groundedness | "Only answer from context" | `regenerate` with the failure as context |
| Judgement calls | Not addressed | `escalate` to a human, mid-run |

:::

:::error
A prompt-level rule is not a control. The model may decline it, and there is no
event when it does — the failure is invisible to the run, to the ledger and to
the operator. This is the single most common way an agent pilot passes review
and then fails in production.
:::

## Question 3 — what gates a change

Everyone in this category can show you a dashboard of agent performance. Very
few can refuse to promote a version.

:::steps
1. **Suites** — cases with expected behaviour, run before a change ships.
2. **Scoring** — LLM-as-judge for prose, deterministic checks for structure.
3. **Promotion gates** — a version that regresses does not reach production.
4. **Online evals** — scoring continues after release, with feedback and drift
   alerts, because the world moves even when the prompt does not.
:::

Step 3 is the one that changes outcomes and the one most commonly absent. A
dashboard describes the past; a gate decides the future.

## Question 4 — who owns the connectors

Here the honest answer runs against us, and saying so is more useful than not.

::stat{value="400+" label="integrations documented for n8n — a lead a new entrant does not close by building, which is why Tamtree does not try"}

Tamtree is MCP-native: it consumes any MCP server as tools and exposes its own
workflows as MCP tools. Existing automation platforms are registered **as tools**
rather than reimplemented — which means an n8n or Zapier workflow you already
have is something Tamtree calls, not something it asks you to migrate.

:::tip
If your problem is "connect these two SaaS apps and copy a row", the tools built
for exactly that will do it faster, and you should use them. The argument for a
different engine starts when the workflow has to reason, wait, be corrected by a
human, and be safe to run unattended.
:::

## The composition question nobody asks

One more, because it is the difference that is easiest to demonstrate and
hardest to retrofit: can a deterministic workflow and an autonomous agent be
composed, or must you pick one?

:::compare
| | Pipeline | Crew |
|---|---|---|
| Control flow | Fixed, author-defined | The lead agent decides each turn |
| Cost and latency | Low, predictable | Higher, variable |
| Nesting | A pipeline node can *be* a crew | A crew can call a pipeline as a tool |
| Shares with the other | Run, ledger, guardrail policy, credentials | The same four |

:::

A DAG-only tool makes you choose determinism for the whole system. An
agent-framework makes you choose autonomy for the whole system. Both choices are
wrong for most real work, which is deterministic in the parts you understand and
autonomous in the parts you do not.

## Choosing honestly

::::cards
:::card{title="Stay where you are if…" tone="good"}
Your workflows are app-to-app plumbing, the model call is one step among many,
and nothing in the flow needs to wait for a person or be audited afterwards.
:::

:::card{title="Look harder if…" tone="warn"}
You have an agent in production, no eval gate, and no answer to "what stopped it
leaking that?" beyond a line in a prompt.
:::
::::

:::note
Tamtree is pre-launch. Nothing above should be read as a claim that it is
already doing this at your scale — the specification is settled, the
implementation is in progress, and the numbers will be published when they come
from recorded runs rather than from optimism.
:::

:::button
[What automation tools structurally cannot do](/blog/what-automation-tools-cannot-do/)
:::
