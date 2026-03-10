# code-index

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![npm](https://img.shields.io/npm/v/pi-code-index)](https://www.npmjs.com/package/pi-code-index)
[![Buy me a coffee](https://img.shields.io/badge/Buy%20me%20a%20coffee-buycoffee.to-orange.svg)](https://buycoffee.to/kcem)

Token-efficient codebase exploration via [universal-ctags](https://ctags.io/). A pure skill plugin that teaches AI coding agents to search symbols, browse outlines, and retrieve precise code ranges — saving 80-98% of context window tokens compared to reading whole files. No MCP server, no runtime dependencies beyond the `ctags` binary.

## How it works

1. **Index** — runs `ctags` to generate a standard `.ctags` tags file
2. **Search** — grep the tags file by symbol name, kind, or file
3. **Retrieve** — read only the exact lines of a symbol using line ranges from the index
4. **Outline** — get file or project-level symbol overviews from the index

The agent decides autonomously when to index, re-index, or search — no manual commands needed.

## Prerequisites

Install [universal-ctags](https://ctags.io/) (must say "Universal Ctags" in `ctags --version`):

```bash
# macOS
brew install universal-ctags

# Ubuntu / Debian
apt install universal-ctags
```

## Installation

### Claude Code

```bash
claude plugin marketplace add https://github.com/kcem/code-index.git
claude plugin install code-index@code-index
```

### GitHub Copilot CLI

```bash
copilot plugin install kcem/code-index
```

### pi.dev

```bash
pi install git:github.com/kcem/code-index
```

Or via npm: `pi install npm:pi-code-index`

### Manual (any agent)

Copy `skills/code-indexing/SKILL.md` into your agent's skills directory. Works with any agent supporting the Agent Skills format.

## Supported languages

Default: **Python, TypeScript, JavaScript** — the agent can extend to any of the 100+ languages universal-ctags supports.

## License

MIT

## Support

100% free, 100% open source, 100% powered by coffee. Saved your tokens? A coffee keeps the next plugin brewing.

[☕ Buy me a coffee](https://buycoffee.to/kcem)
