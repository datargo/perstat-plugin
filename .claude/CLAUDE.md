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

**No em-dashes anywhere.** Not in skills, README, commit messages or PR bodies.
Use a comma, a colon or a full stop. This is a standing rule across all of
Andreas' repositories.

**No `Co-Authored-By` trailers on commits, and no "Generated with" footer on pull
requests.** Commits carry their author and nothing else. The product name
"Claude Code" in prose is fine, this is about authorship attribution only.

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
`services/api/src/mcp.rs` and `lib.rs` of the monitor repository. A mismatch
there is the failure mode that matters, because these files are instructions an
agent follows literally.

## Open

The plugin tracks the MCP server as of monitor `135bafa` (14 tools, English tool
descriptions, `list_projects`, archive and restore). Production may still serve
an older catalogue. Check with `tools/list`: if `list_projects` is absent, the
deployed build predates it and the plugin describes more than the server offers.
That gap is the reason the directory listing is on hold.
