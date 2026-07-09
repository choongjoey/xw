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
export XWIKI_OBJECT_CLASS=MySpace.Code.MyClass   # default class for grep --field and obj (override with --class)
export XW_CACHE_DIR=~/.cache/xw/$XWIKI_HOST/$XWIKI_WIKI
```

The script exits immediately with an error if `XWIKI_HOST` is not set or no auth is configured.

---

## Read / Navigate

### `xw ls [space]`
List spaces. No arg → top-level wiki spaces. With a space name → direct sub-spaces.
Add `-l` for a detail listing (modified date, version, author, ref).

```bash
xw ls
xw ls Docs
xw ls -l Docs
```

```text
$ xw ls
Help
Main
Sandbox
Docs

$ xw ls Docs
Docs.Install.WebHome
Docs.API.WebHome

$ xw ls -l Docs        # columns are tab-separated
2026-07-09   v3.2   jsmith   Docs.Install.WebHome
2026-06-28   v1.1   akhan    Docs.API.WebHome
```

---

### `xw cat <ref>`
Full page content: metadata header + line-numbered body (via MCP). Serves from
cache if available.

```bash
xw cat "Docs.Install.WebHome"
```

```text
Reference: xwiki:Docs.Install.WebHome
URL: https://wiki.example.com/wiki/xwiki/view/Docs/Install
Title: Install Guide
Syntax: xwiki/2.1
Version: 3.2
Size: 9 lines · 93 chars · ~23 tokens

     1	= Install =
     2	
     3	Download the package.
     4	Run the installer.
     ...
```

**Reference format**: dot-separated XWiki hierarchy — `Space.SubSpace.PageName`.

---

### `xw head [-n N] <ref>`
First N lines of a page (default 20).

```bash
xw head -n 3 "Docs.Install.WebHome"
```

```text
Reference: xwiki:Docs.Install.WebHome
URL: https://wiki.example.com/wiki/xwiki/view/Docs/Install
Title: Install Guide
Syntax: xwiki/2.1
Version: 3.2
Size: 9 lines · 93 chars · ~23 tokens

     1	= Install =
     2	
     3	Download the package.
Showing lines 1-3 of 9.
```

---

### `xw tail [-n N] <ref>`
Last N lines (default 20). Two-step: MCP `outline=true` to get the line count,
then MCP `get_document` with the tail range. Cache hit serves locally.

```bash
xw tail -n 2 "Docs.Install.WebHome"
```

```text
... (metadata header) ...

     8	Some detail text.
     9	Final line.
Showing lines 8-9 of 9.
```

---

### `xw range <ref> <start> <end>`
Read a specific line range (1-based, inclusive).

```bash
xw range "Docs.Install.WebHome" 1 3
```

```text
... (metadata header) ...

     1	= Install =
     2	
     3	Download the package.
Showing lines 1-3 of 9.
```

---

### `xw outline <ref>`
Heading map with line numbers. Also shows the `Size:` summary — useful for planning a slice.

```bash
xw outline "Docs.Install.WebHome"
```

```text
... (metadata header, incl. Size line) ...

L1: Install
L6: Troubleshooting
```

---

### `xw wc [-l] <space|ref>`
Without `-l`: count direct sub-spaces of a space.
With `-l`: line count of a page.

```bash
xw wc Docs
xw wc -l "Docs.Install.WebHome"
```

```text
$ xw wc Docs
2

$ xw wc -l "Docs.Install.WebHome"
9
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

```text
$ xw search "install" --space Docs --limit 2
Title: Install Guide
Reference: xwiki:Docs.Install.WebHome
URL: https://wiki.example.com/wiki/xwiki/view/Docs/Install
Modified: 2026-07-09T13:06:24Z by xwiki:XWiki.jsmith
Score: 20.82
Snippet: Install  Download the package. Run the installer. ...

Found 1 matching document.
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

```text
$ xw find -name "*install*"
Docs.Install.WebHome
Docs.Install.Windows.WebHome
```

---

### `xw grep [-l] <pattern> [--space S] [--field F] [--class C]`
Content and property search.

- No `--field`: MCP `query_documents` full-text search (same block format as `search`).
- `--field F`: REST Solr query on a specific XObject property field. The class
  defaults to `$XWIKI_OBJECT_CLASS`; `--class C` overrides it for that one call
  (one of the two must be set). Output: `score\tref\ttitle`. With `-l`: refs only.

```bash
xw grep "install" --space Docs
xw grep "onboarding" --field keywords                             # env default class
xw grep "onboarding" --field keywords --class Other.Code.OtherClass  # override
xw grep "onboarding" --field keywords -l
```

```text
$ xw grep "onboarding" --field keywords     # score / ref / title, tab-separated
6.41    HR.Onboarding.WebHome    Employee Onboarding
3.09    HR.Checklists.WebHome    New-Hire Checklist

$ xw grep "onboarding" --field keywords -l   # -l: refs only
HR.Onboarding.WebHome
HR.Checklists.WebHome
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
xw cache add "Docs.LongDocument.WebHome"
grep -n "keyword" "$(xw cache path "Docs.LongDocument.WebHome")"
```

```text
$ xw cache add "Docs.LongDocument.WebHome"
cached: /Users/you/.cache/xw/wiki.example.com/xwiki/Docs.LongDocument.WebHome.txt

$ xw cache ls
Docs.LongDocument.WebHome
Docs.Install.WebHome

$ xw cache path "Docs.LongDocument.WebHome"
/Users/you/.cache/xw/wiki.example.com/xwiki/Docs.LongDocument.WebHome.txt
```

---

## Write / Edit

> ⚠️ These commands **mutate the live wiki.** `put` is idempotent (a PUT that
> creates the page if absent, overwrites it if present). `append`, `section
> set`, `edit`, `meta set`, `obj set`, `tag`, and `attach put` require the page
> to already exist. `rm`, `obj rm`, `attach rm`, and `mv` delete or move content.
> Confirm the target ref/space before running.

### `xw put <ref> [--title T] [--syntax S] [--parent P] (--content C | --file F | -)`
Create or update a page (whole-page overwrite). Content comes from `--content`,
a `--file`, or stdin (pass `-` or just pipe). Syntax defaults to `xwiki/2.1`;
`--title` and `--parent` are optional. The page's cache entry is evicted on
success so the next read is fresh.

```bash
xw put Sandbox.XwTest --title "xw test" --content "= Hello =

first line"
xw put Sandbox.XwTest --file page.xwiki
cat page.xwiki | xw put Sandbox.XwTest -
xw put Sandbox.XwChild --parent Sandbox.WebHome --content "child page"
```

Confirmations print to stderr:

```text
$ xw put Sandbox.XwTest --title "xw test" --content "= Hello ="
updated: Sandbox.XwTest
```

### In-place editing: `append` / `section set` / `edit`
Edit an existing page without resending the whole body. Each fetches the current
content, mutates it, and re-PUTs preserving title/syntax/parent.

```bash
xw append Sandbox.XwTest --content "trailing line"      # or --file F, or stdin via -
xw section set Sandbox.XwTest "Overview" --content "…"   # replace body under a heading
xw edit Sandbox.XwTest --replace OLD --with NEW [--first]  # literal substitution; errors if no match
```

```text
$ xw append Sandbox.XwTest --content "trailing line"
appended: Sandbox.XwTest

$ xw section set Sandbox.XwTest "Overview" --content "new overview body"
section set: Sandbox.XwTest [Overview]

$ xw edit Sandbox.XwTest --replace TODO --with DONE
xw edit: replaced 2 occurrence(s)
edited: Sandbox.XwTest

$ xw edit Sandbox.XwTest --replace nope --with x    # aborts, page untouched
xw edit: no match for 'nope' (aborting)
```

- `section set` matches an xwiki/2.1 heading (`= H =` … `====== H ======`) and
  replaces its body up to the next heading of the same or higher level.
- `edit` is a literal (non-regex) replace — all occurrences by default, one with
  `--first` — and reports the match count on stderr.

### `xw meta set <ref> [--parent P] [--title T]`
Change a page's `parent` and/or `title` without touching content (resends the
current content unchanged).

```bash
xw meta set Sandbox.XwTest --parent Sandbox.WebHome --title "Renamed"
```

```text
$ xw meta set Sandbox.XwTest --parent Sandbox.WebHome --title "Renamed"
meta set: Sandbox.XwTest parent=Sandbox.WebHome
```

### `xw rm <ref> [--purge]`
Delete a single document. Default sends it to the recycle bin; `--purge` skips
the recycle bin (permanent). Deletes the doc's own objects/attachments too, but
does **not** recurse into child pages.

```bash
xw rm Sandbox.XwTest
xw rm Sandbox.XwTest --purge
```

```text
$ xw rm Sandbox.XwTest --purge
removed: Sandbox.XwTest (purged)
```

### `xw mv <src> <dst>`
Rename or move a page. Same space + new name renames; a different space moves
(both handled by an async refactoring job that `xw` submits and polls to
completion). The source is removed (no redirect placeholder is left behind).

```bash
xw mv Sandbox.Old Sandbox.New
xw mv Sandbox.Old Archive.Space.New
```

```text
$ xw mv Sandbox.Old Sandbox.New
moved: Sandbox.Old -> Sandbox.New
```

### `xw tag {ls|add|rm|set} <ref> [tag ...]`
List, add, remove, or replace a page's tags.

```bash
xw tag ls  Sandbox.XwTest
xw tag add Sandbox.XwTest api draft
xw tag rm  Sandbox.XwTest draft
xw tag set Sandbox.XwTest api stable
```

```text
$ xw tag add Sandbox.XwTest api draft
tags add: Sandbox.XwTest -> api, draft

$ xw tag ls Sandbox.XwTest
api
draft
```

### Concurrency guards (opt-in)
`put`, `append`, `section set`, `edit`, and `meta set` accept `--if-version N`
(abort unless the page is currently at version N) and `--create-only` (abort if
the page already exists). Default behaviour is unchanged (last-write-wins). These
are best-effort GET-then-compare checks — XWiki has no server-side ETag, so a
small TOCTOU window remains.

```text
$ xw put Sandbox.XwTest --create-only --content x
xw: --create-only: page Sandbox.XwTest already exists (aborting)

$ xw append Sandbox.XwTest --content y --if-version 99.9
xw: --if-version 99.9: page Sandbox.XwTest is at version 3.1 (aborting)
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

### `xw obj rm <ref> [--class C] [--number N]`
Delete an XObject record. Without `--number`, removes the first object of the
class (same resolution as `obj set`).

```bash
xw obj rm Sandbox.XwTest --class MyApp.Code.MyAppClass
```

```text
$ xw obj set Sandbox.XwTest --class MyApp.Code.MyAppClass name="Widget" status=Open
set: Sandbox.XwTest MyApp.Code.MyAppClass[0]

$ xw obj ls Sandbox.XwTest
MyApp.Code.MyAppClass[0]

$ xw obj get Sandbox.XwTest --class MyApp.Code.MyAppClass
name=Widget
status=Open

$ xw obj rm Sandbox.XwTest --class MyApp.Code.MyAppClass
removed obj: Sandbox.XwTest MyApp.Code.MyAppClass[0]
```

### `xw attach {ls|put|get|rm} <ref> ...`
Manage page attachments.

```bash
xw attach ls  Sandbox.XwTest                       # name <tab> mimeType <tab> size
xw attach put Sandbox.XwTest ./diagram.png          # optional --name to rename
xw attach get Sandbox.XwTest diagram.png -o out.png # omit -o to stream to stdout
xw attach rm  Sandbox.XwTest diagram.png
```

```text
$ xw attach put Sandbox.XwTest ./diagram.png
attached: Sandbox.XwTest -> diagram.png

$ xw attach ls Sandbox.XwTest        # name / mimeType / size, tab-separated
diagram.png    image/png    48213

$ xw attach get Sandbox.XwTest diagram.png -o out.png
saved: out.png
```

---

## History & links

### `xw log <ref> [-n N]`
Revision history, newest first: `version <tab> date <tab> author <tab> comment`.

```bash
xw log Sandbox.XwTest
xw log Sandbox.XwTest -n 5
```

```text
$ xw log Sandbox.XwTest -n 3       # version / date / author / comment, tab-separated
3.1    2026-07-09    jsmith    Update property hidden
2.1    2026-07-09    jsmith
1.1    2026-07-08    akhan     Imported from draft
```

### `xw cat <ref> --version V`
Print the content of a specific revision (fetched via REST, bypassing cache/MCP).

```bash
xw cat Sandbox.XwTest --version 2.1
```

```text
$ xw cat Sandbox.XwTest --version 1.1
= Overview =

initial overview text
```

### `xw diff <ref> <v1> <v2>`
Unified `diff -u` between two revisions' content.

```bash
xw diff Sandbox.XwTest 1.1 3.1
```

```text
$ xw diff Sandbox.XwTest 1.1 3.1
--- Sandbox.XwTest@1.1
+++ Sandbox.XwTest@3.1
@@ -1,3 +1,3 @@
 = Overview =
 
-initial overview text
+revised overview text
```

### `xw backlinks <ref> [-l]`
Pages that link to `<ref>`, via Solr (`links:` field). Output
`score <tab> ref <tab> title`; `-l` prints refs only. Depends on the Solr index,
so a just-created link may take a moment to appear.

```bash
xw backlinks Sandbox.XwTest
xw backlinks Sandbox.XwTest -l
```

```text
$ xw backlinks Sandbox.XwTest        # score / ref / title, tab-separated
2.97    Docs.Overview.WebHome    Overview

$ xw backlinks Sandbox.XwTest -l     # -l: refs only
Docs.Overview.WebHome
```

### `xw links <ref>`
Outbound `[[...]]` document links parsed from the page's raw content
(best-effort; external URLs and attachments are skipped).

```bash
xw links Sandbox.XwTest
```

```text
$ xw links Docs.Overview.WebHome
Sandbox.XwTest
Docs.Install.WebHome
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

```text
$ xw man
Available tools and reference pages — use `man <name>` for full documentation:

Help
  man - Show documentation for the available MCP tools.
  get_document - Retrieve a wiki document's content ...
  query_documents - Search documents by content/metadata ...
```

---

## MCP vs REST — when each is used

| `xw` command | Backend |
|---|---|
| `ls`, `wc` (no `-l`), `find -name` | REST `/spaces`, `/search` |
| `grep --field` | REST `/query?type=solr` |
| `cat`, `head`, `tail`, `range`, `outline`, `wc -l` | MCP `get_document` |
| `search`, `find` (with filters), `grep` (no field) | MCP `query_documents` |
| `put`, `append`, `section set`, `edit`, `meta set` | REST `PUT /pages/<Page>` (JSON page) |
| `rm` | REST `DELETE /pages/<Page>` |
| `mv` | REST jobs API `PUT /rest/jobs` (XML) + poll `/rest/jobstatus/<id>` |
| `tag ls/add/rm/set` | REST `GET`/`PUT .../pages/<Page>/tags` |
| `log`, `cat --version`, `diff` | REST `GET .../history[/<version>]` |
| `backlinks` | REST `/query?type=solr` (`links:` field) |
| `links` | REST page GET (`[[...]]` parsed client-side) |
| `attach ls/put/get/rm` | REST `GET/PUT/DELETE .../attachments/<name>` |
| `obj ls`, `obj get` | REST `GET .../objects[/<class>/<n>]` |
| `obj set` | REST `POST .../objects` or `PUT .../objects/<class>/<n>` (form) |
| `obj rm` | REST `DELETE .../objects/<class>/<n>` |
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
