# xw — XWiki Shell Helper

`xw` is a Unix-style CLI wrapper around XWiki's MCP and REST APIs.
It lets you navigate, read, search, and create/update wiki content from the
terminal or from an AI agent's tool calls, and maintains a local page cache to
eliminate repeated API hits.

## Use with Claude Code (skill)

`SKILLS.md` is the agent-facing skill guide for `xw` — point Claude Code (or any
agent) at it so the model knows which command to reach for. To make `xw`
available to an agent:

1. Put the `xw` script on your `PATH` (e.g. `ln -s "$PWD/xw" ~/bin/xw`).
2. Export the `XWIKI_*` env vars (see [Auth](#auth)) in the agent's environment.
3. Tell the agent to read `SKILLS.md` and to **prefer `xw` over ad-hoc `curl`**
   for any XWiki task — reading, searching, and now writing.

`CLAUDE.md` in the repo root holds guidance for agents editing `xw` itself.

## Prerequisites

- `bash` 4+, `curl`, `jq`, `python3` (all standard on macOS/Linux)
- A running XWiki instance with the [WAISE / ai-llm](https://extensions.xwiki.org/xwiki/bin/view/Extension/LLM%20Application/) extension installed

## Auth

```bash
export XWIKI_HOST=wiki.example.com      # required — no default
export XWIKI_USER=yourname
export XWIKI_PASS=yourpassword
# or pre-encoded:
export XWIKI_TOKEN=<Base64(user:pass)>  # overrides user/pass

# Optional
export XWIKI_WIKI=xwiki                 # default: xwiki
export XWIKI_OBJECT_CLASS=MySpace.Code.MyClass   # needed for grep --field
export XW_CACHE_DIR=~/.cache/xw/$XWIKI_HOST/$XWIKI_WIKI
```

The script exits immediately with an error if `XWIKI_HOST` is not set or no auth is configured.

---

## Read / Navigate

### `xw ls [space]`
List spaces. No arg → top-level wiki spaces. With a space name → direct sub-spaces.

```bash
xw ls
xw ls MySpace
```

---

### `xw cat <ref>`
Full page content: metadata header + line-numbered body (via MCP). Serves from
cache if available.

```bash
xw cat "MySpace.SubSpace.WebHome"
```

**Reference format**: dot-separated XWiki hierarchy — `Space.SubSpace.PageName`.

---

### `xw head [-n N] <ref>`
First N lines of a page (default 20).

```bash
xw head "MySpace.SubSpace.WebHome"
xw head -n 5 "MySpace.SubSpace.WebHome"
```

---

### `xw tail [-n N] <ref>`
Last N lines (default 20). Two-step: MCP `outline=true` to get the line count,
then MCP `get_document` with the tail range. Cache hit serves locally.

```bash
xw tail -n 2 "MySpace.SubSpace.WebHome"
```

---

### `xw range <ref> <start> <end>`
Read a specific line range (1-based, inclusive).

```bash
xw range "MySpace.SubSpace.WebHome" 1 2
```

---

### `xw outline <ref>`
Heading map with line numbers. Also shows the `Size:` summary — useful for planning a slice.

```bash
xw outline "MySpace.SubSpace.WebHome"
```

---

### `xw wc [-l] <space|ref>`
Without `-l`: count direct sub-spaces of a space.
With `-l`: line count of a page.

```bash
xw wc MySpace
xw wc -l "MySpace.SubSpace.WebHome"
```

---

## Search

### `xw search <terms> [--space S] [--author A] [--since N] [--sort S] [--limit N]`
Full-text + relevance search via MCP `query_documents` (edismax).

| Flag | Values |
|------|--------|
| `--space S` | Restrict to space |
| `--author A` | Filter by last author |
| `--since N` | `day` · `week` · `month` · `year` |
| `--sort S` | `relevance` (default) · `newest` · `oldest` · `title` |
| `--limit N` | Results per page (default 10, max 25) |

```bash
xw search "install guide"
xw search "onboarding" --space HR --limit 5
xw search "" --space Docs --sort newest
```

---

### `xw find [-name glob] [-space S] [-author A] [-since N] [-n N]`
Find pages by title or metadata.

- `-name` alone → REST title search (`scope=name`), fast, glob-like.
- Any other flag (or combined with `-name`) → MCP `query_documents`.

```bash
xw find -name "*install*"
xw find -space Docs -since week
```

---

### `xw grep [-l] <pattern> [--space S] [--field F]`
Content and property search.

- No `--field`: MCP `query_documents` full-text search.
- `--field F`: REST Solr query on a specific XObject property field
  (requires `XWIKI_OBJECT_CLASS`). Output: `score\tref\ttitle`. With `-l`: refs only.

```bash
xw grep "install" --space Docs
xw grep "onboarding" --field keywords
xw grep "onboarding" --field keywords -l
```

**When to use `--field`**: when you need to match a specific AWM XObject
property (e.g., `keywords`, `description`) rather than the page body.

---

## Cache

The cache stores verbatim MCP `get_document` output under
`$XW_CACHE_DIR/<percent-encoded-ref>.txt`.

All read commands (`cat`, `head`, `tail`, `range`, `outline`, `wc -l`) check
the cache first — no API call if a cache file is present.

### `xw cache add <ref>`
Fetch the full page and write to cache.

### `xw cache ls`
List all cached refs.

### `xw cache rm <ref>` / `xw cache clear`
Remove one entry or all entries for the current host/wiki.

### `xw cache path <ref>`
Print the cache file path. Use this to pipe cached content to arbitrary local tools:

```bash
xw cache add "MySpace.LongDocument.WebHome"
grep -n "keyword" "$(xw cache path "MySpace.LongDocument.WebHome")"
```

---

## Write / Edit

> ⚠️ These commands **mutate the live wiki.** `put` is idempotent (a PUT that
> creates the page if absent, overwrites it if present). `obj set` requires the
> page to already exist. Confirm the target ref/space before running.

### `xw put <ref> [--title T] [--syntax S] (--content C | --file F | -)`
Create or update a page. Content comes from `--content`, a `--file`, or stdin
(pass `-` or just pipe). Syntax defaults to `xwiki/2.1`; `--title` is optional.
The page's cache entry is evicted on success so the next read is fresh.

```bash
xw put Sandbox.XwTest --title "xw test" --content "= Hello =

first line"
xw put Sandbox.XwTest --file page.xwiki
cat page.xwiki | xw put Sandbox.XwTest -
```

### `xw obj set <ref> [--class C] [--number N] key=value ...`
Create or update an AppWithinMinutes (AWM) record — an XObject of an app's data
class attached to a page. Class defaults to `$XWIKI_OBJECT_CLASS`. Without
`--number`, `xw` updates the first existing object of that class or creates a new
one (number 0); pass `--number` to target a specific object.

```bash
xw obj set Sandbox.XwTest --class MyApp.Code.MyAppClass name="Widget" status=Open
xw obj set Sandbox.XwTest --class MyApp.Code.MyAppClass status=Closed   # updates same object
```

The page must exist first — otherwise `xw` errors and points you to `xw put`.

### `xw obj ls <ref>`
List the XObjects attached to a page as `<className>[<number>]`.

```bash
xw obj ls Sandbox.XwTest
```

### `xw obj get <ref> [--class C] [--number N]`
Print an object's properties as `key=value` lines. Class defaults to
`$XWIKI_OBJECT_CLASS`, number to 0.

```bash
xw obj get Sandbox.XwTest --class MyApp.Code.MyAppClass
```

---

## Misc

### `xw man [tool]`
Without args: list all MCP tools. With a tool name: show its full documentation.

```bash
xw man
xw man get_document
xw man query_documents
```

---

## MCP vs REST — when each is used

| `xw` command | Backend |
|---|---|
| `ls`, `wc` (no `-l`), `find -name` | REST `/spaces`, `/search` |
| `grep --field` | REST `/query?type=solr` |
| `cat`, `head`, `tail`, `range`, `outline`, `wc -l` | MCP `get_document` |
| `search`, `find` (with filters), `grep` (no field) | MCP `query_documents` |
| `put` | REST `PUT /pages/<Page>` (JSON page) |
| `obj ls`, `obj get` | REST `GET .../objects[/<class>/<n>]` |
| `obj set` | REST `POST .../objects` or `PUT .../objects/<class>/<n>` (form) |
| `man` | MCP `man` |

---

## Caching workflow pattern

```bash
# 1. Pre-fetch a large document
xw cache add "Space.LongDocument.WebHome"

# 2. Check what's in it without re-fetching
xw outline "Space.LongDocument.WebHome"
xw wc -l "Space.LongDocument.WebHome"

# 3. Read a section
xw range "Space.LongDocument.WebHome" 40 80

# 4. Arbitrary local tools — no API calls
grep -n "keyword" "$(xw cache path "Space.LongDocument.WebHome")"
wc -l "$(xw cache path "Space.LongDocument.WebHome")"
```
