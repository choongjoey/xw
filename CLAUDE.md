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

- **Read:** `ls`, `cat`, `head`, `tail`, `range`, `outline`, `wc`
- **Search:** `search`, `find`, `grep`
- **Write:** `put <ref>` (page), `obj set|ls|get <ref>` (AWM/XObject records)
- **Cache:** `cache add|ls|rm|clear|path`
- **Misc:** `man`

See `SKILLS.md` for full agent-facing usage and `README.md` for humans.

## Repo conventions

- **One Bash script, no build/tests.** Changes go in `./xw` plus the three docs
  (`README.md`, `SKILLS.md`, `CLAUDE.md`). Run `bash -n xw` after editing.
- **Keep it macOS-compatible.** BSD `awk`/`bash` differ from GNU — see commit
  `8152c40` ("fix awk issues on macOS"). Avoid GNU-only flags.
- **Reuse existing helpers** (`_AUTH`, `REST_BASE`, `_urlencode`, `_ref_to_url`,
  `_cache_path`, `_rest`, `_restq`, `_restwrite`, `_mcp`) rather than adding
  dependencies. The only external tools are `curl`, `jq`, `python3`.

## Safety

Write commands (`put`, `obj set`) mutate a **live wiki**. `put` is idempotent;
`obj set` requires the page to exist first (create it with `put`). Confirm the
target ref/space before running any write.
