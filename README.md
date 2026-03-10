# code-index

Token-efficient codebase exploration via [universal-ctags](https://ctags.io/). Search symbols, browse outlines, and retrieve precise code ranges — without reading whole files.

A pure skill plugin. No MCP server, no runtime dependencies beyond the `ctags` binary. Works with any AI coding agent that supports Agent Skills.

## How it works

1. **Index** — runs `ctags` to generate a standard `.ctags` tags file
2. **Search** — grep the tags file by symbol name, kind, or file
3. **Retrieve** — read only the exact lines of a symbol using line ranges from the index
4. **Outline** — get file or project-level symbol overviews from the index

The AI agent decides autonomously when to index, re-index, or search — no manual commands needed.

## Prerequisites

Install [universal-ctags](https://ctags.io/):

```bash
# macOS
brew install universal-ctags

# Ubuntu / Debian
apt install universal-ctags
```

Verify it's the right version (must say "Universal Ctags"):

```bash
ctags --version
```

## Installation

### Claude Code

**From a marketplace:**

```bash
claude plugin marketplace add https://github.com/YOURUSER/code-index.git
claude plugin install code-index@code-index
```

**Or clone and add locally:**

```bash
git clone https://github.com/YOURUSER/code-index.git
claude plugin marketplace add /path/to/code-index
claude plugin install code-index@code-index
```

### GitHub Copilot CLI

**Copy the skill into your project:**

```bash
mkdir -p .github/skills/code-indexing
cp skills/code-indexing/SKILL.md .github/skills/code-indexing/SKILL.md
```

Copilot auto-discovers skills from `.github/skills/`.

**Or install as a plugin:**

```bash
copilot plugin install YOURUSER/code-index
```

### pi.dev

**From Git:**

```bash
pi install git:github.com/YOURUSER/code-index
```

**From npm** (after publishing):

```bash
pi install npm:pi-code-index
```

### Manual (any agent)

Copy `skills/code-indexing/SKILL.md` into your agent's skills directory. The skill is a standard markdown file that works with any agent supporting the Agent Skills format.

## What's in the box

```
code-index/
├── .claude-plugin/
│   ├── plugin.json           # Claude Code plugin manifest
│   └── marketplace.json      # Standalone marketplace for Claude Code
├── package.json              # pi.dev package definition
├── skills/
│   └── code-indexing/
│       └── SKILL.md          # The skill — works across all platforms
├── LICENSE                   # MIT
└── README.md
```

## Supported languages

Default: **Python, TypeScript, JavaScript**

universal-ctags supports 100+ languages. To add more, the agent extends the `.ctags.d/code-index.ctags` config file automatically.

## License

[MIT](LICENSE)

## Support

If you find this useful, consider [buying me a coffee](https://buycoffee.to/kcem).
