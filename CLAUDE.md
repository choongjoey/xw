# CLAUDE.md — guidance for agents working in this repo

`xw` is a single Bash script (`./xw`) that wraps XWiki's MCP + REST APIs as a
Unix-style CLI. It started read-only (navigate/read/search) and now also
**writes**: create/update pages and create/update AppWithinMinutes (AWM)
records (XObjects of an app's data class attached to a page).

## Required environment

| Var | Required | Purpose |
|---|---|---|
| `XWIKI_HOST` | yes | wiki hostname (no default) |
| `XWIKI_USER` + `XWIKI_PASS` | one of these | basic auth |
| `XWIKI_TOKEN` | or this | pre-encoded `Base64(user:pass)`, overrides user/pass |
| `XWIKI_WIKI` | no | wiki name (default `xwiki`) |
| `XWIKI_OBJECT_CLASS` | no | default XObject class for `grep --field` and `obj` |
| `XW_CACHE_DIR` | no | default `~/.cache/xw/<host>/<wiki>/` |

## Command map

- **Read:** `ls`, `cat` (`--version`), `head`, `tail`, `range`, `outline`, `wc`
- **Search:** `search`, `find`, `grep`
- **Write:** `put <ref>` (page, `--parent`), `append`, `section set`, `edit`
  (`--replace/--with`), `meta set`, `rm`, `mv`, `tag {ls|add|rm|set}`,
  `obj set|ls|get|rm <ref>` (AWM/XObject records)
- **History:** `log`, `cat --version`, `diff`
- **Links:** `backlinks`, `links`
- **Attach:** `attach ls|put|get|rm`
- **Cache:** `cache add|ls|rm|clear|path`
- **Misc:** `man`

Opt-in write guards on `put`/`append`/`section set`/`edit`/`meta set`:
`--if-version N` (abort unless the page is at version N) and `--create-only`
(abort if the page exists). Default is unchanged: last-write-wins, idempotent.

See `SKILLS.md` for full agent-facing usage and `README.md` for humans.

## Repo conventions

- **One Bash script, no build/tests.** Changes go in `./xw` plus the three docs
  (`README.md`, `SKILLS.md`, `CLAUDE.md`). Run `bash -n xw` after editing.
- **Keep it macOS-compatible.** BSD `awk`/`bash` differ from GNU — see commit
  `8152c40` ("fix awk issues on macOS"). Avoid GNU-only flags.
- **Reuse existing helpers** (`_AUTH`, `REST_BASE`, `_urlencode`, `_ref_to_url`,
  `_normalize_ref`, `_rest`, `_restq`, `_restwrite`, `_mcp`) and the write-side
  helpers (`_page_json`, `_precond`, `_write_page`, `_edit_page`, `_restdel`,
  `_restwrite_bin`, `_restroot`, `_mime`, `_docref_xml`) rather than adding
  dependencies. The only external tools are `curl`, `jq`, `python3`.

## Discovering & navigating the XWiki REST API

When adding a feature, don't guess endpoints — discover them. The techniques
below are what actually locked the shapes in the next section (env: `set -a; .
./.env; set +a`, then `B="https://$XWIKI_HOST/rest/wikis/$XWIKI_WIKI"`).

1. **What a resource allows** — `curl -s -u "$XWIKI_USER:$XWIKI_PASS" -X OPTIONS
   -D - -o /dev/null "$B/spaces/Sandbox/pages/WebHome/tags"` and read the
   `Allow:` header. `-I` (HEAD) also surfaces the `XWiki-Form-Token:` header
   used for CSRF (see `_form_token`).
2. **Response JSON shape** — GET with `-H "Accept: application/json"` and pipe to
   `jq 'keys'` / `jq -c '.foo[0]'` to see field names before coding a parser.
   Everything is also available as XML (`Accept: application/xml`), which is
   sometimes more revealing (see #5).
3. **The resource tree** — XWiki's REST resources are wiki-scoped under
   `/rest/wikis/<wiki>/…` (that's `REST_BASE`); a few (the **jobs** API) live at
   `/rest/…` (root — use `_restroot`). Standard subpaths off a page:
   `/objects`, `/objects/<class>/<n>`, `/attachments/<name>`, `/tags`,
   `/history`, `/history/<version>`, `/children`.
4. **The wire model (authoritative)** — every JAXB type is defined in
   `xwiki.rest.model.xsd`. Read it to get exact element names (e.g. `JobRequest`
   → `JobId.element`, `MapEntry{key,value}`) instead of trial-and-error:
   `gh api "/repos/xwiki/xwiki-platform/contents/xwiki-platform-core/xwiki-platform-rest/xwiki-platform-rest-model/src/main/resources/xwiki.rest.model.xsd" -H "Accept: application/vnd.github.raw+json"`.
5. **When (de)serialization is opaque, read the server source** via `gh api`
   (`Accept: application/vnd.github.raw+json`) — use `gh api "/search/code?q=<sym>+repo:xwiki/xwiki-platform"`
   to locate files. Load-bearing ones found this way: `JAXBConverter.java`
   (anyType values are **XStream**-unmarshaled → the jobs API needs XML, not
   JSON), `refactoring/job/EntityRequest.java` + `MoveRequest.java` (property
   keys `entityReferences`/`destination`/`autoRedirect`), and job-api
   `AbstractCheckRightsRequest.java` (`checkrights`, `user.reference`).
6. **Empirical iteration against a throwaway `Sandbox.*` page** — submit with
   `-w $'\n%{http_code}'`, read the error, adjust, repeat. XWiki's 400/500 bodies
   are informative (Jackson "Unrecognized field …", `ClassCastException: String
   cannot be cast to EntityReference`) and walk you straight to the right shape.
   Always clean up (`xw rm --purge`).
7. **Solr fields** — `GET $B/query?q=<field>:<val>&type=solr[&fq=wiki:<wiki>]`.
   To reverse-engineer a stored token, probe with wildcards (`field:*Foo*`) then
   confirm the exact value with a quoted phrase query. Caveat: the REST
   `SearchResult` model exposes only a fixed field set, and its `.links` array is
   **HATEOAS**, not the Solr `links` field — so read `SolrLinkSerializer.java` /
   `FieldUtils.java` (via #5) rather than trying to dump raw field values.
8. **MCP side** — `xw man` lists the MCP tool catalog; `xw man <tool>` shows one
   tool's schema. The read commands (`cat`/`head`/`tail`/`range`/`outline`) go
   through MCP `get_document`; everything structural goes through REST.

## Notable endpoint shapes (live-verified on `aster`)

- **`mv`** uses the async **jobs API** at `https://$XWIKI_HOST/rest/jobs`
  (outside `REST_BASE`, via `_restroot`). Body is **XML** (not JSON): the JAXB
  `JobRequest` wraps property values that XWiki XStream-unmarshals, so
  `entityReferences`/`destination` must be XStream-serialized `DocumentReference`
  elements whose parents carry `class="…SpaceReference"`/`class="…WikiReference"`
  (a plain `EntityReference` parent fails an internal `WikiReference` cast). See
  `_docref_xml`. `jobType=refactoring/rename` handles same-space rename **and**
  cross-space move; `checkrights=false` + `autoRedirect=false` are set so the
  source is actually removed. Then poll `GET /rest/jobstatus/refactoring/<id>`.
- **`backlinks`** queries Solr `links:"entity:document:<wiki>:<Space.Page>"`
  (the `links` field stores `entity:` + a `withtype/withparameters` reference).
- **Tags:** `PUT /…/pages/<page>/tags` with `{"tags":[{"name":"x"},…]}`.
- **History:** `GET /…/history` (`.historySummaries[]`) and `/history/<v>`
  (has `.content`). **Attachments:** `GET/PUT/DELETE /…/attachments/<name>`.

## Safety

Write commands mutate a **live wiki**. `put` is idempotent; `append`/`section
set`/`edit`/`meta set`/`obj set`/`tag`/`attach put` require the page to exist
first (create it with `put`). `rm`/`obj rm`/`attach rm` delete; `rm --purge`
skips the recycle bin. `mv` removes the source (no redirect left behind). Guards
(`--if-version`, `--create-only`) are best-effort (GET-then-compare, small
TOCTOU window). Confirm the target ref/space before running any write.
