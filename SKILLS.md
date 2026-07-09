# xw — Claude Code Skill

`xw` is a shell CLI for reading, searching, and creating/updating an XWiki
instance. Use it whenever a task requires navigating wiki content, looking up
knowledge articles, searching for pages by title, content, or metadata, or
creating/updating pages and AppWithinMinutes (AWM) records.

## Setup (required before any command)

```bash
export XWIKI_HOST=wiki.example.com   # required — no default
export XWIKI_USER=yourname
export XWIKI_PASS=yourpassword
# optional
export XWIKI_WIKI=xwiki              # default: xwiki
export XWIKI_OBJECT_CLASS=MySpace.Code.MyClass  # default class for grep --field and obj
```

### Config resolution — env default, flag override

The XObject class follows one rule everywhere it's used (`grep --field`, `obj
get/set/rm`): **`$XWIKI_OBJECT_CLASS` is the default; a `--class` flag overrides
it for that one call.** Set the env var for the app you work in most, and reach
for `--class` only when you cross into another AWM app. Either must be present —
if neither is set, the command errors. (Auth resolves the same way: `XWIKI_TOKEN`
overrides `XWIKI_USER`/`XWIKI_PASS`.)

```bash
export XWIKI_OBJECT_CLASS=ServiceDesk.Code.ServiceDeskClass  # the everyday default
xw grep lolita --field keywords                              # uses the env default
xw grep lolita --field keywords --class Other.Code.OtherClass  # override, this call only
```

---

## Decision tree — pick the right command

```
Reading, or writing?
  reading →
    Do you know the exact page reference?
      yes → xw cat / head / tail / range / outline
      no  →
        Searching by title only (fast)?        → xw find -name "*glob*"
        Searching by content / relevance?      → xw search "terms"
        Filtering by space / author / date?    → xw find / xw search with flags
        Matching a specific XObject property?  → xw grep --field <prop>
        Browsing what spaces exist?            → xw ls [space]
  writing →
    Overwrite a whole page?                  → xw put <ref> ...
    Add to the end of a page?                → xw append <ref> ...
    Replace one section (under a heading)?   → xw section set <ref> <heading> ...
    Find-and-replace a literal string?       → xw edit <ref> --replace OLD --with NEW
    Change parent/title, keep content?       → xw meta set <ref> [--parent P] [--title T]
    Delete a page?                           → xw rm <ref> [--purge]
    Rename or move a page?                   → xw mv <src> <dst>
    Manage a page's tags?                    → xw tag {ls|add|rm|set} <ref> ...
    Attach / fetch a file?                   → xw attach {ls|put|get|rm} <ref> ...
    Set/inspect an AWM / XObject field?      → xw obj {set|ls|get|rm} <ref> ...  (page must exist)
    Guard against a concurrent edit?         → add --if-version N or --create-only
  history / links →
    Revision history of a page?              → xw log <ref>
    Read / diff an old revision?             → xw cat <ref> --version V ; xw diff <ref> v1 v2
    What links to / from a page?             → xw backlinks <ref> / xw links <ref>
```

---

## Reading pages

All read commands serve from local cache when available — no API hit.

### Reference format
Dot-separated XWiki hierarchy: `Space.SubSpace.PageName`.
Page names with spaces and special characters work as-is — quote the whole ref in the shell.

```bash
xw cat "MySpace.SubPage.WebHome"
xw cat "HR.Onboarding Guide.WebHome"   # spaces in name are fine
```

### Slicing large pages — always outline first

```bash
xw outline "Space.Page.WebHome"   # shows Size + heading map with line numbers
# then read just what you need:
xw range "Space.Page.WebHome" 40 80
xw head -n 30 "Space.Page.WebHome"
xw tail -n 20 "Space.Page.WebHome"
```

`outline` also returns `Size: N lines · C chars · ~T tokens` — use this to
decide whether to read the whole page or slice it. If tokens > ~1500, slice.

### Full read

```bash
xw cat "Space.Page.WebHome"    # full metadata header + numbered body
```

Output format:
```
Reference: wiki:Space.Page.WebHome
URL: https://...
Title: Page Title
Syntax: xwiki/2.1
Version: 3.2
Size: 42 lines · 1823 chars · ~456 tokens

     1  = Heading
     2  Content here
     ...
```

---

## Searching

### `xw search` — relevance-ranked full-text

```bash
xw search "install guide"
xw search "onboarding" --space HR --since week --limit 5
xw search "" --space Docs --sort newest    # browse latest in a space
```

Output: one result block per match with Title, Reference, URL, Modified, Score, Snippet.

### `xw find` — title lookup or filtered browse

```bash
xw find -name "*onboard*"              # REST title search — fast, no index hit
xw find -space HR -since month         # MCP query with filters
xw find -name "*guide*" -space Docs    # combined: MCP query
```

`-name` alone is the fastest path (REST, no Solr). Add any other flag and it
switches to MCP `query_documents`.

### `xw grep` — pattern match

```bash
xw grep "GDPR" --space Legal           # full-text via MCP
xw grep "onboarding" --field keywords  # specific XObject property via Solr
xw grep "onboarding" --field keywords -l  # refs only (no titles/scores)
xw grep "onboarding" --field keywords --class Other.Code.OtherClass  # override class
```

`--field` needs an XObject class: it uses `$XWIKI_OBJECT_CLASS` by default, or
`--class C` to override per call (see *Config resolution* above). Use it when you
need to match structured metadata (tags, descriptions, custom fields) rather than
page body. `-l` works anywhere in the flag list.

---

## Writing

> These commands mutate the live wiki. Confirm the target ref/space first.

### `xw put` — create or update a page

Idempotent: creates the page if it doesn't exist, overwrites it if it does.
Content comes from `--content`, `--file`, or stdin. Syntax defaults to
`xwiki/2.1`. The cached copy is evicted on success.

```bash
xw put Sandbox.XwTest --title "xw test" --content "= Hello =

body line"
xw put Sandbox.XwTest --file page.xwiki
some-generator | xw put Sandbox.XwTest -
```

### `xw obj set` — create or update an AWM / XObject record

An AWM record is an XObject of an app's data class attached to a page. Set
fields as `key=value` args. Class comes from `--class` or `$XWIKI_OBJECT_CLASS`.
Without `--number`, `xw` updates the first existing object of that class (or
creates number 0); pass `--number` to target a specific one.

```bash
xw obj set Sandbox.XwTest --class MyApp.Code.MyAppClass name="Widget" status=Open
xw obj set Sandbox.XwTest --class MyApp.Code.MyAppClass status=Closed   # same object
```

**Rule: the page must already exist before `obj set`.** If it doesn't, `xw`
errors and tells you to run `xw put` first. So the create-a-record flow is:

```bash
xw put Sandbox.XwTest --content "record page"          # 1. ensure the page
xw obj set Sandbox.XwTest --class MyApp.Code.MyAppClass name="Widget"  # 2. set fields
```

### `xw obj ls` / `xw obj get` / `xw obj rm` — inspect / delete objects

```bash
xw obj ls Sandbox.XwTest                                 # <className>[<number>]
xw obj get Sandbox.XwTest --class MyApp.Code.MyAppClass   # key=value per property
xw obj rm  Sandbox.XwTest --class MyApp.Code.MyAppClass    # delete (first of class, or --number N)
```

### In-place editing — the everyday "edit a file" operations

`put` overwrites the whole page. These three edit **in place** — they fetch the
current content, mutate it, and re-PUT preserving title/syntax (cache evicted).
All require the page to exist.

```bash
xw append Sandbox.XwTest --content "a new trailing line"   # or --file F / stdin
xw section set Sandbox.XwTest "Overview" --content "..."     # replace body under the = Overview = heading
xw edit Sandbox.XwTest --replace "TODO" --with "DONE"        # literal replace-all (--first for one); errors if no match
```

`section set` matches an xwiki/2.1 heading (`= H =` … `====== H ======`) and
replaces everything under it up to the next heading of the same or higher level.
`edit` is a literal (non-regex) substitution and reports the match count.

### `xw meta set` — change parent/title without touching content

```bash
xw meta set Sandbox.XwTest --parent Sandbox.WebHome --title "New Title"
```

The frontmatter-parity path: resends the current content unchanged and only
moves `parent`/`title`.

### `xw rm` / `xw mv` — delete, rename, move

```bash
xw rm Sandbox.XwTest              # → recycle bin
xw rm Sandbox.XwTest --purge      # skip recycle bin (permanent)
xw mv Sandbox.Old Sandbox.New     # rename in place
xw mv Sandbox.Old Other.Space.New # move across spaces (async refactoring job)
```

`rm` deletes a **single** document (its own objects + attachments go with it; it
does **not** recurse into child pages). `mv` submits an async job, polls to
completion, and removes the source (no redirect placeholder left behind).

### `xw tag` — manage page tags

```bash
xw tag ls  Sandbox.XwTest
xw tag add Sandbox.XwTest api draft
xw tag rm  Sandbox.XwTest draft
xw tag set Sandbox.XwTest api stable   # replace the whole set
```

### `xw attach` — attachments

```bash
xw attach ls  Sandbox.XwTest
xw attach put Sandbox.XwTest ./diagram.png [--name diagram.png]
xw attach get Sandbox.XwTest diagram.png -o ./out.png   # omit -o to stream to stdout
xw attach rm  Sandbox.XwTest diagram.png
```

### Concurrency guards (opt-in)

Default writes are last-write-wins. Pass a guard to `put`/`append`/`section
set`/`edit`/`meta set` to make the write conditional:

```bash
xw put Sandbox.XwTest --content "..." --create-only          # abort if it already exists
xw append Sandbox.XwTest --content "..." --if-version 3.1     # abort unless current version is 3.1
```

Best-effort (GET-then-compare — XWiki has no server-side ETag), so a small
race window remains. Use `xw log <ref>` to read the current version first.

---

## History & diff

```bash
xw log Sandbox.XwTest              # version <tab> date <tab> author <tab> comment (newest first)
xw log Sandbox.XwTest -n 5         # last 5 revisions
xw cat Sandbox.XwTest --version 2.1   # content of an old revision
xw diff Sandbox.XwTest 1.1 3.1     # unified diff between two revisions
```

## Link graph

```bash
xw backlinks Sandbox.XwTest        # pages that link TO this one (Solr): score <tab> ref <tab> title
xw backlinks Sandbox.XwTest -l     # refs only
xw links Sandbox.XwTest            # outbound [[...]] doc links FROM this page (best-effort)
```

`backlinks` depends on the Solr index (a freshly created link may take a moment
to appear). `links` parses `[[...]]` targets from the raw content and is
xwiki-syntax-dependent (external URLs / attachments are skipped).

---

## Cache

Pre-fetch pages you will read more than once in a session. The cache persists
across invocations so subsequent calls are free.

```bash
xw cache add "Space.Page.WebHome"     # fetch and store
xw cache ls                           # see what's cached
xw cache path "Space.Page.WebHome"    # get file path for local tools
xw cache rm "Space.Page.WebHome"      # evict one entry
xw cache clear                        # clear all
```

### Pattern: large document workflow

```bash
xw cache add "Space.BigDoc.WebHome"          # one API call
xw outline "Space.BigDoc.WebHome"            # free — served from cache
xw range "Space.BigDoc.WebHome" 40 80        # free
grep -n "term" "$(xw cache path "Space.BigDoc.WebHome")"  # local grep, no quota
```

---

## Listing spaces

```bash
xw ls              # top-level spaces
xw ls MySpace      # direct sub-spaces (e.g., pages under MySpace)
xw wc MySpace      # count of direct sub-spaces
xw wc -l "Space.Page.WebHome"   # line count of a specific page
```

---

## MCP tool docs

```bash
xw man                    # list available MCP tools
xw man get_document       # full docs for a tool
xw man query_documents
```

---

## MCP vs REST at a glance

| Command | Backend |
|---|---|
| `ls`, `wc`, `find -name` | REST |
| `grep --field` | REST (Solr) |
| `cat`, `head`, `tail`, `range`, `outline`, `wc -l` | MCP `get_document` |
| `search`, `find` (with filters), `grep` (no field) | MCP `query_documents` |
| `put`, `append`, `section set`, `edit`, `meta set` | REST `PUT /pages/<Page>` (JSON) |
| `rm` | REST `DELETE /pages/<Page>` |
| `mv` | REST jobs API `PUT /rest/jobs` (XML) + poll `/rest/jobstatus/<id>` |
| `tag ls/add/rm/set` | REST `GET`/`PUT .../pages/<Page>/tags` |
| `log`, `cat --version`, `diff` | REST `GET .../history[/<v>]` |
| `backlinks` | REST `/query?type=solr` (`links:` field) |
| `links` | REST page GET, `[[...]]` parsed client-side |
| `attach ls/put/get/rm` | REST `GET/PUT/DELETE .../attachments/<name>` |
| `obj ls`, `obj get` | REST `GET .../objects[/<class>/<n>]` |
| `obj set` | REST `POST .../objects` or `PUT .../objects/<class>/<n>` (form) |
| `obj rm` | REST `DELETE .../objects/<class>/<n>` |
| `man` | MCP `man` |

---

## Token efficiency tips

- Prefer `head`/`tail`/`range` over `cat` for pages larger than ~1500 tokens.
- Use `outline` to get a heading map before deciding which section to read.
- Cache pages you'll re-read — a cached `cat` costs zero tokens on the API.
- Use `find -name` for title lookup — it's a REST call with no Solr overhead.
- `grep --field -l` returns refs only — cheapest way to get a hit list before reading.
