# tamtree-blog-content

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
