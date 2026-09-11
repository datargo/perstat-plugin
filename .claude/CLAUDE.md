# perstat-plugin

Claude Code plugin for [Perstat](https://perstat.io). Published publicly at
`github.com/datargo/perstat-plugin`, installed via the `datargo` marketplace
declared in this same repo.

## Layout

- `.claude-plugin/plugin.json` manifest, `.claude-plugin/marketplace.json`
  declares the `datargo` marketplace with this plugin at the repo root
  (`"source": "./"`, deliberately relative so entry and plugin cannot drift)
- `skills/perstat-monitoring/` reference knowledge, including
  `references/monitor-types.md`, which carries the config shape for all 13
  monitor types because the MCP `create_monitor` tool ships no schema for its
  `config` field
- `skills/manage-monitors/`, `skills/perstat-status/`, `skills/handle-incident/`
  the three action skills

## House rules

**No em-dashes, and no `Co-Authored-By` trailers or "Generated with" footers.**
Neither in skills, README, commit messages nor PR bodies; use a comma, a colon or
a full stop instead. Naming "Claude Code" in prose is fine, this is about
authorship attribution only.

**Commits go through a branch and a pull request.** A pre-commit hook blocks
commits on `main`.

## Verifying a change

```bash
claude plugin validate .claude-plugin/plugin.json
```

To test installation end to end, add the marketplace from a local path and
install from it. Two cautions learned the hard way: `claude plugin marketplace
add <path>` OVERWRITES an existing registration of the same name, so a local test
will replace the GitHub-based one and has to be restored afterwards; and
`marketplace remove` takes the installed plugin entry with it.

The skills should also be checked against the server they describe. Every tool
name, parameter, monitor type and check region in `skills/` has a counterpart in
`services/api/src/mcp.rs` and `lib.rs` of the perstat repository (the local
`monitor` directory is a symlink to it). A mismatch
there is the failure mode that matters, because these files are instructions an
agent follows literally.

## Open

Status 2026-08-15: the plugin tracks the MCP server in `services/api/src/mcp.rs`
of the perstat repository, last changed in perstat `3564853` (2026-08-14). The
tool catalogue is unchanged since `135bafa`: 14 tools, English tool descriptions,
including `list_projects`, `list_status_pages`, archive and restore; commits
since then refined responses and OAuth discoverability, not the catalogue.
Whether production serves this catalogue cannot be read off the repo. Check with
`tools/list`: if `list_projects` or `list_status_pages` is absent, the deployed
build is older and the plugin describes more than the server offers. The
directory listing stays on hold until such a check confirms the full catalogue
in production.
