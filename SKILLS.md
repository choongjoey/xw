# xw — Claude Code Skill

`xw` is a shell CLI for reading and searching an XWiki instance. Use it whenever
a task requires navigating wiki content, looking up knowledge articles, or
searching for pages by title, content, or metadata.

## Setup (required before any command)

```bash
export XWIKI_HOST=wiki.example.com   # required — no default
export XWIKI_USER=yourname
export XWIKI_PASS=yourpassword
# optional
export XWIKI_WIKI=xwiki              # default: xwiki
export XWIKI_OBJECT_CLASS=MySpace.Code.MyClass  # only needed for grep --field
```

---

## Decision tree — pick the right command

```
Do you know the exact page reference?
  yes → xw cat / head / tail / range / outline
  no  →
    Searching by title only (fast)?        → xw find -name "*glob*"
    Searching by content / relevance?      → xw search "terms"
    Filtering by space / author / date?    → xw find / xw search with flags
    Matching a specific XObject property?  → xw grep --field <prop>
    Browsing what spaces exist?            → xw ls [space]
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
```

`--field` requires `XWIKI_OBJECT_CLASS` to be set. Use it when you need to
match structured metadata (tags, descriptions, custom fields) rather than page body.
`-l` works anywhere in the flag list.

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
| `man` | MCP `man` |

---

## Token efficiency tips

- Prefer `head`/`tail`/`range` over `cat` for pages larger than ~1500 tokens.
- Use `outline` to get a heading map before deciding which section to read.
- Cache pages you'll re-read — a cached `cat` costs zero tokens on the API.
- Use `find -name` for title lookup — it's a REST call with no Solr overhead.
- `grep --field -l` returns refs only — cheapest way to get a hit list before reading.
