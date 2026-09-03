---
title: Seven things an automation tool structurally cannot do
description: Durable conversations, enforced guardrails, evals as CI, memory beyond the thread — the capabilities that need to be in the engine, and why bolting them onto a task runner does not work.
publishDate: 2026-09-03
category: product
tags: [positioning, architecture, agents]
author: dilhan
featured: true
---

Every automation tool on the market can call a language model. Most of them
added a node for it, wired it to the same executor that runs their HTTP nodes,
and shipped. That is not a criticism — it is the correct move for a product
whose engine was designed to move JSON between APIs.

It is also the reason those tools hit a wall the moment the workflow stops being
a pipeline and starts being an *agent*. The wall is structural, not a missing
feature. Seven of them are below.

:::note
Everything claimed about Tamtree here comes from its architecture
specification, section by section. Where a figure would be needed and no
recorded measurement exists, it is bracketed rather than estimated — see the
last section on why.
:::

## 1. A conversation that survives the process that started it

An agent run is not a request. It thinks, calls a tool, waits, thinks again, and
may need a human to answer something before it can continue. In a task-runner
architecture that whole span lives inside one execution: if the worker restarts,
the run is gone, and "waiting for a human" means holding a process open.

Tamtree runs on a durable execution engine, so the run's state lives outside any
process that happens to be executing it.

::::cards
:::card{title="Crash-safe runs" tone="brand"}
A worker dying mid-run is a scheduling event, not data loss. The run resumes at
the step it reached.
:::

:::card{title="Multi-day waits at zero compute" tone="compose"}
A run waiting on a human approval holds no process and no memory. Waiting is
free, so a workflow can wait as long as the business actually takes.
:::

:::card{title="Resume-where-you-left-off" tone="good"}
A conversation thread picks up with its history intact, days later, without
replaying every tool call to rebuild the state.
:::
::::

This is the one that cannot be retrofitted. Durability is a property of how the
engine stores and resumes state, and an executor that keeps run state in process
memory cannot acquire it by adding a node.

## 2. Guardrails that run, rather than guardrails you ask for

The common approach to safety in a prompt-driven system is to write the rule
into the prompt and hope. That is not a control; it is a request, and the model
is free to decline it.

Tamtree filters every model call at the runtime level — PII, prompt injection,
moderation and groundedness — and each filter has an enforced action rather than
a log line.

:::ledger
| Action | What happens | When you want it |
|---|---|---|
| `block` | The call does not proceed | Hard policy — data that must never leave |
| `redact` | The offending span is removed, the call continues | PII in an otherwise valid request |
| `regenerate` | The model is asked again with the failure as context | Groundedness misses, format violations |
| `escalate` | A human is brought into the run | Judgement calls the policy cannot make |

:::

:::warn
`escalate` only works if the run can wait for a person without burning compute —
which brings you back to point 1. The features are not independent; the
durability substrate is what makes the guardrail actions affordable.
:::

## 3. Evals as CI, not as a dashboard

Everyone measures agents after the fact. Very few gate on the measurement.

The distinction matters because an agent regression does not look like a crash.
It looks like slightly worse answers, which nobody notices until a customer
does. The only reliable defence is the same one the software industry already
settled on for code: a suite that runs before the change ships, and a gate that
refuses to promote when the suite regresses.

:::steps
1. Write a suite — cases with expected behaviour, not expected strings.
2. Score with LLM-as-judge where the output is prose, deterministic checks where
   it is structure.
3. Gate promotion on the score. A version that regresses does not reach
   production.
4. Keep scoring online after release, with feedback and drift alerts, because
   the world changes even when the prompt does not.
:::

:::tip
The step most teams skip is the third one. A dashboard tells you what happened;
a gate decides what ships. Only one of them changes the outcome.
:::

## 4. Memory that outlives the thread

A chat session remembers itself. That is table stakes and it is not memory.

Memory is the system knowing that *this* end user asked about their invoice last
month, across a different conversation, in a different channel. That needs a
per-end-user identity and long-term semantic storage attached to it — a data
model decision, made early or not at all.

## 5. Cost you can govern, not just observe

::stat{value="per step" label="the granularity at which token spend has to be attributed before any of the controls above become possible"}

Token cost per step feeds a ledger; the ledger feeds budgets; budgets feed a
kill-switch and model routing. Each stage depends on the one before it, and the
chain starts at attribution. A tool that reports spend per workflow run cannot
build the rest of it, because by the time the run has finished, the decision it
needed to make has passed.

## 6. Two primitives, one engine

Most tools give you a DAG. Some give you an agent. Tamtree's position is that
the interesting work is in composing both, which requires them to be primitives
on the same engine rather than two products sharing a logo.

:::compare
| | Pipeline | Crew |
|---|---|---|
| Control flow | Fixed, author-defined | The lead agent decides each turn |
| Best for | Auditable repeatable procedures — sync, ETL, billing | Open-ended, adaptive work — research, triage, support |
| Cost and latency | Low, predictable | Higher, variable |
| Composition | A pipeline node can *be* a crew | A crew can call a pipeline as a tool |

:::

That last row is the point. **Crew-as-node** and **pipeline-as-tool** mean you
put determinism exactly where you can afford it and autonomy exactly where you
need it, in the same workflow, rather than choosing once for the whole system.

:::aside
This is also why the two are not separate products. The moment a crew calls a
pipeline as a tool, they share a run, a ledger, a guardrail policy and a set of
credentials. Splitting them would mean splitting all four.
:::

## 7. Being a client of the connector ecosystem instead of rebuilding it

Connector count is a race a new entrant loses by definition. So Tamtree does not
enter it: it is MCP-native, consuming any MCP server as tools and exposing its
own workflows as MCP tools. Existing automation platforms are registered *as
tools* rather than reimplemented.

:::tip
This is a strategic choice disguised as an integration detail. Rebuilding four
hundred connectors is years of work that produces parity, not advantage. Calling
them is a week of work that produces the same reach.
:::

## What is not claimed here

:::error
No latency, cost or throughput figure for Tamtree appears in this post, and that
is deliberate. The product is pre-launch. Any number quoted now would be an
estimate wearing the costume of a measurement, and the one thing worse than no
benchmark is a benchmark that turns out to be aspirational.
:::

Figures will appear here when they come from recorded runs, and they will say
which run they came from. Until then, brackets.

:::button
[Read the workflow builder piece](/blog/tamtree-workflow-builder/)
:::
