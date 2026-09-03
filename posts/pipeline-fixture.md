---
title: Pipeline fixture
description: The fixture post that proves the Markdown pipeline end to end — directives, semantics and syntax highlighting all in one file.
publishDate: 2026-09-02
category: engineering
tags: [build, markdown]
author: dilhan
draft: true
---

This post exists to exercise every mechanism the pipeline is supposed to have.
It is a draft, so it never reaches production; `gate:directives` reads it in
dev to prove each directive still resolves to its element.

:::note
A callout wraps an `aside`, so an extractor that strips unknown elements still
finds a semantic block rather than a bare paragraph.
:::

:::steps
1. Parse the Markdown with Sätteri, directives on.
2. Resolve each directive to its custom element.
3. Fail the build on any name the vocabulary does not contain.
:::

:::ledger
| Step | Status | Duration |
|---|---|---|
| plan | ok | [pending] |
| draft | ok | [pending] |
| review | paused | [pending] |

:::

::stat{value="677 ms" label="500 posts, markdown pipeline only"}

A fenced block, to prove Expressive Code actually ran under Sätteri:

```js
export function ttDirectives() {
  return { name: 'tt-directives', options: { position: true } };
}
```

:::warn
Figures describing a run are bracketed until a recorded fixture exists. That is
contract C5 working, not a gap in the copy.
:::

:::error
An `error` callout is the fourth severity: `warn` says this will bite you,
`error` says this is already broken.
:::

## Blocks that generate their own headings

A card's title becomes an `h3`, so the fixture needs a real `h2` above the grid:
a page whose outline jumps `h1` → `h3` fails `gate:semantic`, and the fixture is
the one post where every such structure has to be exercised on purpose.

::::cards
:::card{title="A card" tone="brand"}
The title becomes a real `h3` inside an `article`, not a styled first line.
:::

:::card{title="A linked card" href="/blog/" tone="compose"}
When `href` is set the heading carries the link.
:::
::::

## Images

:::figure
![A ledger with five rows: two steps succeeded, one failed, the same step succeeded on retry, and a final step succeeded.](../images/ledger-attempts.png)

A `figure` wraps a real `<figure>` and turns the paragraph under the image into a
`<figcaption>` — the association Markdown alone cannot express. A bare
`![alt](…)` still works and is still the right thing for an image with nothing
to say about itself.
:::

:::compare
| Question | Tamtree | Something else |
|---|---|---|
| Where does state live? | The ledger | [pending] |

:::

:::button
[Read the writing](/blog/)
:::
