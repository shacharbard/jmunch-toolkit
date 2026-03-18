# jmunch-toolkit

Command toolkit powered by [jCodeMunch](https://github.com/jgravelle/jcodemunch-mcp) and [jDocMunch](https://github.com/jgravelle/jdocmunch-mcp) MCP servers.

Provides reusable Claude Code commands for codebase analysis — dead code detection, and more to come.

## Prerequisites

This plugin requires jCodeMunch and jDocMunch MCP servers to be installed and configured. Install them via [jmunch-claude-code-setup](https://github.com/shacharbard/jmunch-claude-code-setup) or manually:

```bash
claude mcp add jcodemunch -- npx jcodemunch-mcp
claude mcp add jdocmunch -- npx jdocmunch-mcp
```

## Installation

### From the marketplace (recommended)

First, add the [jmunch-marketplace](https://github.com/shacharbard/jmunch-marketplace):

```
/plugin marketplace add shacharbard/jmunch-marketplace
```

Then install the plugin:

```
/plugin install jmunch-toolkit@jmunch-marketplace
```

### From a local clone

```bash
git clone https://github.com/shacharbard/jmunch-toolkit.git
claude plugin add ./jmunch-toolkit
```

## Available Commands

### `/dead-code`

Comprehensive dead code analysis across all 44 languages supported by jCodeMunch and 10+ doc formats supported by jDocMunch.

```
/dead-code                  # Analyze entire codebase
/dead-code src/             # Scope to a directory
/dead-code server.py        # Analyze a single file
/dead-code *.ts             # Scope by file type
/dead-code docs             # Find orphaned docs only
```

Uses `find_importers`, `get_blast_radius`, `get_dependency_graph`, and `search_sections` to produce a tiered report of dead files, dead functions, orphaned docs, and items needing verification.

Nothing is deleted automatically — you review and decide.

## Adding New Commands

Create a new `.md` file in the `commands/` directory. The filename becomes the slash command name (e.g., `commands/foo.md` → `/foo`).

## Structure

```
jmunch-toolkit/
├── .claude-plugin/
│   └── plugin.json   # Plugin manifest
├── commands/         # Slash commands (user-invocable)
│   └── dead-code.md  # /dead-code
├── skills/           # Skills (auto-triggered by context)
├── agents/           # Subagent definitions
└── README.md
```

## Credits

Built on [jCodeMunch](https://github.com/jgravelle/jcodemunch-mcp) and [jDocMunch](https://github.com/jgravelle/jdocmunch-mcp) by [J. Gravelle](https://github.com/jgravelle). Licensing for those tools is per their respective repos.

## License

MIT (this plugin only). [jCodeMunch](https://github.com/jgravelle/jcodemunch-mcp) and [jDocMunch](https://github.com/jgravelle/jdocmunch-mcp) MCP servers are licensed under their creator's terms — see their repos for details.
