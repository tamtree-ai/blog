# tamtree-ai/blog

Markdown only. No Node, no build step, nothing to install.

- `posts/<slug>.md` — one file per post. Frontmatter fields are listed below.
- `authors/<id>.md` — one file per author; `author:` in a post references the filename.

## Publishing

**Push to `main`.** That is the whole process. A GitHub Action calls the site's
deploy hook, the site rebuilds against this repo at HEAD, and the post is live
in about a minute. Nothing needs committing in the site repo — it holds no
pointer to this one, deliberately.

To take a post down, delete it or set `draft: true`, and push.

The filename is the URL: `posts/why-runs-fail.md` publishes at
`/blog/why-runs-fail/`, with its source at `/blog/why-runs-fail.md`. Renaming a
file breaks that URL for anyone who linked to it, so treat the slug as
permanent once it is out.

## Post frontmatter

| Field | Required | Notes |
|---|---|---|
| `title` | yes | ≤ 120 characters |
| `description` | yes | 40–300 characters; used in `llms.txt` and the OG card |
| `publishDate` | yes | `YYYY-MM-DD` |
| `updatedDate` | no | `YYYY-MM-DD` |
| `category` | yes | exactly one of `engineering`, `architecture`, `product` — see below |
| `tags` | no | list of strings |
| `author` | yes | an id from `authors/` |
| `draft` | no | `true` hides it from production, keeps it visible in `pnpm dev` |
| `featured` | no | `true` puts it in the lead slot on `/blog/`. If nothing is flagged the newest post leads and is badged "Latest" instead |

## Categories

The list is **closed**, and each one is a page (`/blog/category/engineering/`):

| `category:` | Covers |
|---|---|
| `engineering` | how the runtime is built — the executor protocol, the ledger, the failures behind both |
| `architecture` | the decisions underneath the code: what was chosen, rejected, and what it cost to find out |
| `product` | what an agent-first workflow tool owes the people who operate it |

Anything else **fails the build**, by design. A category is a URL, and if any
string were allowed then `engineering`, `Engineering` and `eng` would be three
pages for one idea, and a typo would silently publish a category nobody meant to
create. Adding a real one is a one-line change in the site repo
(`src/lib/categories.ts`) — ask, don't work around it.

`tags` are free-form: use whatever describes the post. They are not pages yet.

## The directive vocabulary

Blocks beyond plain Markdown are written as directives. The vocabulary is
**closed** — an unknown name fails the build with a line number rather than
vanishing silently from the page.

```
:::note / :::tip             a callout — context, and a suggestion
:::warn / :::error           a callout — "this will bite you", "this is broken"
:::aside                     a digression
:::steps                     wraps ONE ordered/unordered Markdown list
:::ledger                    wraps ONE Markdown table
:::compare                   wraps ONE Markdown table, laid out for comparison
:::button                    wraps ONE Markdown link, rendered as a button
:::figure                    ONE Markdown image, plus an optional caption
:::card{title="…"}           a card — title, optional href and tone
::::cards                    a grid of two or more :::card blocks
:::run{src="…"}              wraps ONE Markdown table, from a run fixture
::dag{src="…"}               a diagram, from a recorded run fixture
::stat{value="…" label="…"}  a single figure
::terminal{src="…"}          a recorded session
```

### Cards need a longer outer fence

Directive fences nest by **length**, so a grid opens with **four** colons:

```
::::cards
:::card{title="Crash-safe runs" tone="brand"}
A worker dying mid-run is a scheduling event, not data loss.
:::

:::card{title="Read the spec" href="/blog/" tone="compose"}
With `href`, the title becomes the link.
:::
::::
```

Write `:::cards` with three and the first card's closing `:::` closes the whole
grid — the rest of the cards fall outside it and a stray `:::` renders as a
paragraph. The build catches this and says so, because a grid must hold at least
two cards. For one card on its own, use `:::card` with no grid around it.

`tone` is one of `brand`, `compose`, `good`, `warn`, `danger` and colours the
card's top edge only.

### Images

Images live in `images/` and are referenced from a post with an ordinary
relative Markdown link:

```
![The ledger after a retry](../images/ledger-attempts.png)
```

The site resizes them, converts them to WebP, emits a `srcset`, and writes real
`width`/`height` attributes so the page does not jump while they load. None of
that needs anything from you beyond the relative path — but the path has to be
relative (`../images/…`). An absolute `/images/…` is served untouched, at full
weight, with no dimensions.

**Alt text is required and the build enforces it.** `![](x.png)` fails, and it
fails on purpose: an empty `alt` is valid HTML meaning *decorative, skip me*, so
nothing downstream can tell a picture you forgot to describe from one you meant
to hide. Describe what the picture shows, not that it is a picture.

For a captioned image, use `:::figure` — it produces a real `<figure>` and
`<figcaption>`, which is the association between a picture and its words that
Markdown on its own cannot express:

```
:::figure
![Five ledger rows, one of them a retry](../images/ledger-attempts.png)

One run, five rows. The failed attempt is not overwritten by the retry.
:::
```

One image, and at most one paragraph of caption. A caption that wants two
paragraphs is prose, and prose belongs under the figure where it reads as prose.

### Hero images

`hero:` in the frontmatter puts a picture at the top of the post and on its card
in the index, and it becomes the post's social share image:

```yaml
hero: ../images/hero-durable-runs.png
heroAlt: A three-step run with the third step failing once and succeeding on retry.
```

`heroAlt` is **required whenever `hero` is set** — the build rejects the pair
otherwise. Draw heroes at 1200×630: that is the size every social card wants,
and the page scales down from it. Anything under about 1000px wide will look
soft on the post page.

The picture is announced with its `heroAlt` on the post, and marked decorative
on the index card — the card repeats the title and description right beneath it,
and reading the same post three times over is not an accessibility win.

### Buttons are links

```
:::button
[Read the workflow builder piece](/blog/tamtree-workflow-builder/)
:::
```

Exactly one Markdown link and nothing else. It renders as the site's primary
button and stays an ordinary link in the `.md`, which is the artifact — a
`::button{href=… label=…}` form would put the words and the destination into
attributes, where a reader of the source sees machinery instead of a link.

**Internal links must point at pages that exist**, or `gate:links` fails the
build. Today that means `/`, `/blog/`, a category page, another post, or
`/#waitlist`.

Every directive must also read correctly as plain text — the `.md` is the
artifact and the site is a rendering of it, so a block that only makes sense
once styled is a block written wrong.

**No figure describing a Tamtree run may be hand-typed.** Costs, durations and
token counts come from a recorded run fixture, or they are bracketed as
placeholders.
