---
title: The workflow builder, and the five decisions inside it
description: A visual canvas with no ceiling — expressions, variable scoping that survives retries, test runs against real payloads, and error messages written for the person who built the flow rather than for a Python programmer.
publishDate: 2026-09-02
category: engineering
tags: [builder, expressions, dx]
author: dilhan
---

Most workflow builders are judged on how quickly you can get the first thing
working. That is the wrong benchmark, because the first thing always works. The
interesting question is what happens on day forty, when the flow has eleven
nodes, a loop, a retry that fires occasionally, and someone else is on call.

Five decisions in Tamtree's builder are aimed squarely at day forty.

:::note
This piece describes the builder as specified. Where behaviour is normative in
the architecture document, it is normative here too — these are contracts the
implementation has to satisfy, not preferences it may drift from.
:::

## 1. A ladder, not a wall

The usual shape of these products is a visual editor with a cliff at the edge of
it: everything is easy until you need one thing the canvas cannot express, and
then you are writing a separate service.

The builder is designed as a ladder instead, where each rung is a smaller step
than the cliff it replaces.

:::steps
1. **Visual editing** — the canvas, trigger-first, configure each node against
   real input data.
2. **`{{ }}` expressions** — reach into prior nodes' data without leaving the
   field you are typing in.
3. **Code nodes** — sandboxed Python or JavaScript, for the transformation the
   expression language deliberately will not do.
4. **The Python SDK** — author your own nodes and tools as first-class citizens,
   not as HTTP calls to somewhere else.
5. **Git sync and the CLI** — the flow is text, reviewable in a pull request.
6. **The full REST API** — everything the UI does, available to a program.
:::

:::tip
Notice that no rung requires abandoning the one below it. A flow that reaches
for the SDK on one node keeps the visual canvas for the other ten — which is the
entire point, and the thing a "just write code instead" escape hatch fails to
deliver.
:::

## 2. An expression language that says no on purpose

`{{ }}` expressions resolve against prior node data — `$json` for the current
item, `$node("Name")` for another node's output, `$vars` for the run's
variables, `$now`, `$env`, and a small set of others.

The notable part is the list of things it refuses to evaluate.

:::ledger
| Allowed | Refused |
|---|---|
| Constants, whitelisted names, subscripts | Any dunder access |
| Arithmetic, comparisons, boolean operators | `import`, lambdas, comprehensions |
| Conditional expressions (`a if c else b`) | Assignment and the walrus operator |
| Calls to whitelisted functions and methods | Attribute access reaching Python internals |

:::

The implementation parses with `ast.parse(mode="eval")` and rejects unknown node
types **by allowlist, never by blocklist** — the distinction that decides
whether a sandbox holds up under someone actively trying to get out of it.

```python
# Whitelisted functions, in full. The shortness is the feature.
len min max sum abs round sorted str int float bool list dict

# Whitelisted string methods.
upper lower strip split join replace startswith endswith title
```

:::warn
`$vars` access is subscript-only: `{{ $vars["draft_ready"] }}`, never
`{{ $vars.draft_ready }}`. The dotted form raises, because attribute access on a
wrapped mapping reaches only its whitelisted methods. This trips people up
exactly once, and the error message says so.
:::

## 3. Variable scoping that survives a retry

This is the subtle one, and it is where most home-grown orchestrators quietly
break.

A retried step must see what the first attempt saw. If it does not,
`{{ $vars['n'] + 1 }}` increments twice on a retry and your counter is wrong in
a way that only shows up under load.

::::cards
:::card{title="Snapshot at dispatch" tone="brand"}
The `$vars` snapshot an activity resolves against is fixed when the activity is
dispatched — so a retried attempt sees exactly what the first attempt saw, and
the expression is idempotent.
:::

:::card{title="Writes land forward" tone="compose"}
A write becomes visible to the *next* node. Never to the node that wrote it, and
never to a node that already ran. A step that fails writes nothing.
:::

:::card{title="Children get their own scope" tone="good"}
Sub-workflows, crews and every pass of a loop body get a fresh scope from their
own declarations. They neither inherit nor write back.
:::
::::

:::error
The consequence of that third card is easy to miss and expensive to discover: a
loop's stop signal **cannot** travel in a variable, because the loop body's scope
does not write back to the parent. It has to travel on the loop-carried item.
A stop condition written as a variable will loop forever, and it will do it
silently.
:::

## 4. Test runs that test the thing you are editing

There is a distinction here that sounds pedantic and is not.

:::compare
| | Editor test run | Triggered or API run |
|---|---|---|
| What executes | The **draft** definition | The **published** version |
| Payload | An optional trigger payload you supply | Whatever the trigger delivered |
| Where the payload lives | Run state — staged per run, gone when the editor closes | The trigger's own data |
| Replayable afterwards | Yes — the draft is snapshotted into the run | Yes |

:::

The draft is snapshotted into the run's input, so a test run stays replayable
even after you have edited the flow underneath it. And the trigger payload takes
the same one-object shape the public API takes, so `{{ $json[...] }}` resolves in
the editor exactly as it will in production.

:::aside
Sample data that belongs to the flow rather than to one test is `pinned_data`
instead: saved with the draft, stripped at publish. The two are separate on
purpose — test payloads are throwaway, pinned data is part of how the flow is
authored.
:::

## 5. Errors addressed to the person who built the flow

Missing keys resolve to `None` rather than raising, because that keeps authoring
forgiving. It also means the failure surfaces one step *later* in the path, as an
operation on nothing — and the default message for that is useless:

> `'NoneType' object is not subscriptable`

That names no parameter, no expression, and no next move. So the evaluator never
lets an interpreter message through. It names the sub-expression that was
**empty** — not the one that raised — rendered back in the author's own syntax:

```text
$json['data'] is empty (null), so ['items'] cannot be read from it
  field: fields.oops
  hint:  the "Fetch order" node returned no items for this input
```

::stat{value="workflow.invalid_expression" label="the error class an expression failure is given — never platform.unexpected, because an unclassified fault gets sanitized as ours and tells the author their workflow is not the cause"}

It is also **non-retryable**. Re-evaluating the same inputs is deterministic, so
retrying only delays the failure and buries it under a retry-exhausted message
that hides the real one.

:::tip
This is the kind of decision that never appears in a feature comparison and
determines whether people can actually operate the thing. An error message is
part of the product surface, not part of the exception handling.
:::

## What this adds up to

Composition is the payoff. A pipeline node can *be* a crew; a crew can call a
pipeline as a tool. Both share one run, one ledger, one guardrail policy and one
set of credentials — which is only possible because they are two primitives on
one engine rather than two products.

:::button
[See how that compares to the alternatives](/blog/tamtree-vs-n8n-zapier-make/)
:::
