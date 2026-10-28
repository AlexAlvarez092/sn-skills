---
name: servicenow-docs
description: Search and read the official, public ServiceNow AI Platform product documentation (github.com/ServiceNow/ServiceNowDocs) as markdown, per release family. Use when the user asks how an out-of-the-box ServiceNow application, API, or concept works (e.g. "how does the Table API work?", "what does CSM Workforce Optimization do?", "GlideRecord methods", "Flow Designer actions"). No credentials needed.
license: MIT
metadata:
  author: Alejandro Alvarez
  version: "1.1"
compatibility: Requires only curl. Optional for better search - gh (authenticated) or git. No credentials, API keys or environment variables.
---

# ServiceNow Docs

[ServiceNow/ServiceNowDocs](https://github.com/ServiceNow/ServiceNowDocs) is
ServiceNow's own public repository with the full AI Platform product
documentation as markdown, optimized for LLM consumption. One git branch per
release family (e.g. `australia`, `zurich`, `yokohama`, `xanadu`), and one
folder per product area ("publication") under `markdown/`.

**Never fetch `servicenow.com/docs`** — it is a JavaScript SPA that returns
nothing readable. Use the raw markdown instead.

Raw URL pattern:

```text
https://raw.githubusercontent.com/ServiceNow/ServiceNowDocs/{branch}/markdown/{publication}/{file}.md
```

## Workflow

1. **Pick the release family.** Ask the user which family their instance runs
   (Settings / `stats.do`) if it matters. Otherwise use the latest from
   `llms.txt` (see step 2), which states which family is the latest. Only the
   3–4 most recent families exist as branches.
2. **List publications** (one line per product area, with slugs in the URLs):

   ```bash
   curl -fsSL https://raw.githubusercontent.com/ServiceNow/ServiceNowDocs/<branch>/llms.txt \
     | grep -iE '<keyword>'
   ```

   Drop the `grep` to see the family list and all publications.
3. **Browse a publication's files** (title, path and one-line description):

   ```bash
   curl -fsSL https://raw.githubusercontent.com/ServiceNow/ServiceNowDocs/<branch>/markdown/<publication>/index.md \
     | grep -iE '<keyword>'
   ```

4. **Read a file** (use the URL from the index; frontmatter has `title`,
   `product_area`, `last_updated`, `canonical_url`):

   ```bash
   curl -fsSL https://raw.githubusercontent.com/ServiceNow/ServiceNowDocs/<branch>/markdown/<publication>/<path>.md
   ```

   Files can be long: pipe through `head -n 200` first, then read more as
   needed.

Common publication slugs: `api-reference` (server/client scripting APIs, REST
APIs), `application-development` (building apps),
`platform-administration`. Always confirm slugs via `llms.txt` rather than
guessing.

## Searching

Pick the first option that works; both are optional upgrades over browsing
with `llms.txt` + `index.md`.

### Option A (preferred): `gh` code search

If `gh` is installed and authenticated (`gh auth status`), this is the
fastest option: no download, one call, searches all publications:

```bash
gh search code '<keywords>' --repo ServiceNow/ServiceNowDocs --limit 10 --json path,url
```

Result paths look like `markdown/<publication>/<path>.md`. Read the file
with the raw URL pattern above. Limits: it only searches the repo's default
branch (the latest family), so for another family use Option B; results are
ranked by GitHub, not exhaustive; and it is rate-limited.

### Option B (fallback, no `gh` needed): `git` sparse clone + `grep`

There is no unauthenticated search API. Use a **partial + sparse clone** so
only the chosen publication is downloaded (a few MB up to ~50 MB; never
clone the whole ~435 MB repo unless truly needed):

```bash
CACHE="${TMPDIR:-/tmp}/servicenow-docs-cache"
git clone --depth 1 --filter=blob:none --sparse --branch <branch> \
  https://github.com/ServiceNow/ServiceNowDocs.git "$CACHE"
git -C "$CACHE" sparse-checkout set markdown/<publication>
grep -rIl --include='*.md' -i '<keywords>' "$CACHE/markdown/<publication>"
```

To search another publication later:
`git -C "$CACHE" sparse-checkout add markdown/<other-publication>`.
To refresh: `git -C "$CACHE" pull`. The cache is disposable — delete it any
time. Ask the user before cloning if a publication looks large.

If neither `gh` nor `git` is available, fall back to browsing: `llms.txt` →
`index.md` (with `grep`) → file.

Search is plain text/regex matching, not semantic: pick keywords likely to
appear verbatim (class names, table names, field names).

## Example

Question: "What methods does GlideAggregate have?"

```bash
gh search code 'GlideAggregate' --repo ServiceNow/ServiceNowDocs --limit 5 --json path
# -> markdown/api-reference/server-api-reference/c_GlideAggregateAPI.md
curl -fsSL https://raw.githubusercontent.com/ServiceNow/ServiceNowDocs/<branch>/markdown/api-reference/server-api-reference/c_GlideAggregateAPI.md | head -n 200
```

Without `gh`: `curl` the `api-reference` index, `grep -i glideaggregate`,
then fetch the matching file.

## Limitations

- Read-only; covers generic out-of-the-box ServiceNow only, not a customer's
  own customizations (use a live-instance query tool for those).
- No images or diagrams — the upstream repo omits media.
- Older families disappear as new ones ship. If a branch 404s, re-read
  `llms.txt` from the latest branch.
- Content is updated at least monthly; check `last_updated` in frontmatter
  when accuracy matters, and cite the `canonical_url` to the user.
