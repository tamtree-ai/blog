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
:::note / :::warn / :::tip   a callout
:::aside                     a digression
:::steps                     wraps ONE ordered/unordered Markdown list
:::ledger                    wraps ONE Markdown table
:::run{src="…"} / :::compare wraps ONE Markdown table
::dag{src="…"}               a diagram, from a recorded run fixture
::stat{value="…" label="…"}  a single figure
::terminal{src="…"}          a recorded session
```

Every directive must also read correctly as plain text — the `.md` is the
artifact and the site is a rendering of it, so a block that only makes sense
once styled is a block written wrong.

**No figure describing a Tamtree run may be hand-typed.** Costs, durations and
token counts come from a recorded run fixture, or they are bracketed as
placeholders.
