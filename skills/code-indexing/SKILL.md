---
name: code-indexing
version: 0.2.0
description: >-
  This skill should be used INSTEAD OF reading whole files or grepping
  source code for definitions. Use it when: reading code ("read the
  UserService class", "show me the get_user function" — use the index to
  read only the exact lines, not the whole file), before editing code
  ("change the get_user method" — find the exact symbol range first),
  finding where symbols are defined ("where is UserService", "which file
  has get_user" — use the index instead of Glob or Grep on source),
  understanding project structure ("what classes exist", "show me an
  outline"), navigating between related symbols ("find all classes in this
  module", "what does this package export"), or spotting duplicate
  definitions ("are there two classes named Config", "same function name
  in multiple files"). Index once with ctags, then search and read
  precisely. Requires universal-ctags.
---

# Code Indexing — Fast, Precise Codebase Exploration

Use this skill when you need to explore or navigate a codebase without reading
whole files. Index once with ctags, then search symbols precisely. **Default to
this over reading whole files.**

**Core rule: never read a full file when you only need part of it.** Use the
index to find the symbol's start and end lines, then read only that range.

**Allowed tools:** Use Bash to run `ctags` commands (indexing, version checks).
Use Grep to search `.ctags` files. Use Read with offset/limit for symbol retrieval.

## Do

- Search `.ctags` before reading any file
- Read symbols by line range (`offset` + `limit`), not whole files
- Use outlines from the index to understand file/project structure
- Re-index when results look wrong or stale

## Don't

- Read a whole file to find one function or class
- Use Glob or Grep on source files to find where a symbol is defined
- Run ctags with inline flags — use `.ctags.d/` config
- Override existing project ctags configs without asking the user

## Example: Reading a Symbol

Instead of reading a whole file, use the index:

1. Search the index for the symbol:
   `Grep pattern="^get_user\t" path="/absolute/path/to/project/.ctags" output_mode="content"`
2. Parse the result — find `line:N` and `end:M` in the ctags entry.
3. Read only the symbol's lines:
   `Read file_path="/absolute/path/to/project/src/services/user.py" offset=N limit=(M-N+1)`

Always use absolute paths — the `.ctags` file and source files live in the
project root, but you may not be working from there.

## Prerequisites

Before indexing, verify universal-ctags is installed:

```bash
command -v ctags && ctags --version
```

The output **must** contain "Universal Ctags". If it shows "Exuberant Ctags" or
the command is not found, the user needs to install universal-ctags:

- **macOS**: `brew install universal-ctags`
- **Ubuntu/Debian**: `apt install universal-ctags`

If a shell alias interferes with `ctags`, use the full binary path instead (e.g.,
`/opt/homebrew/bin/ctags` on macOS with Homebrew). Find it with
`brew list universal-ctags | grep bin` or `which -a ctags`.

If universal-ctags is not available or the wrong version is detected, tell the
user and stop. Do not attempt to index with Exuberant Ctags — the output format
is incompatible.

## Ctags Configuration

Both `.ctags.d/` and `.ctags` **must** live in the project root. If the project
is a git repo, that's the directory containing `.git` — find it with
`git rev-parse --show-toplevel` if unsure. If there's no git, use the top-level
project directory and skip all git-related steps (`.gitignore`, commit
suggestions).

Universal-ctags auto-discovers all `*.ctags` files from `.ctags.d/` in the
current directory. This skill adds its own config file there. Run the
**first-time setup** on the first index of every session. After the first index,
only update the config if the user explicitly asks.

### Required options

These options are **non-negotiable** — the skill cannot work without them:

- `--fields=+neKS` — end line (`e`), line number (`n`), full kind name (`K`),
  signature (`S`). The `+` adds to existing fields.
- `--extras=+q` — qualified tags (class-qualified entries).
- `--output-format=u-ctags` — structured field output format.

### First-time setup

Run this on the **first index of the session** — whether the config exists or not:

1. **Check for existing configs** — read all files in `.ctags.d/` (if it exists).

2. **Check compatibility** — look for conflicts with required options:
   - `--output-format=e-ctags` — **incompatible.** Inform the user: "This skill
     requires `u-ctags` output format for structured field parsing (`line:`,
     `end:`, `kind:`). The existing config sets `e-ctags`. How would you like
     to proceed?"
   - `--fields=-n` or `--fields=-e` — **incompatible.** Inform the user: "This
     skill needs `line` and `end` fields. The existing config removes them."
   - `--languages`, `--exclude`, `--recurse` — **compatible.** Note what's
     already covered so you don't duplicate it.

3. **Languages** — do **not** set `--languages`. Universal-ctags supports 100+
   languages out of the box. Let it index everything it recognizes. Only add
   `--languages` if the user explicitly asks to restrict indexing.

4. **Autodetect excludes** — scan the project and add excludes for all known
   build output, vendored dependencies, and generated files. Excludes match by
   name anywhere in the tree (e.g., `--exclude=.venv` catches `.venv/` at any
   depth). Skip any already covered by other configs.
   - **Always:** `.git`
   - **Python:** `__pycache__`, `.venv`, `venv`, `.tox`, `*.pyc`
   - **Node/JS/TS:** `node_modules`, `dist`, `build`, `*.min.js`, `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`
   - **Go:** `vendor`
   - **Rust:** `target`
   - **Java/Kotlin:** `*.class`, `.gradle`, `build`
   - **Ruby:** `vendor/bundle`
   - **C/C++:** `*.o`, `*.so`, `*.a`
   - **General:** `coverage`, `.cache`, `.tmp`
   - Add any other large generated/vendored directories visible in the project.

5. **Create or update `.ctags.d/code-index.ctags`** — include required options
   plus only what's not already covered by existing configs.

6. **Inform the user** — briefly summarize what was detected and configured.
   If any changes were made to `.ctags.d/` or `.gitignore`, offer to commit
   them (it's project configuration, like `.editorconfig`).

### Config examples

**No existing configs** (typical case — Python + TypeScript project):

```
--recurse
--fields=+neKS
--extras=+q
--output-format=u-ctags
--exclude=.git
--exclude=node_modules
--exclude=__pycache__
--exclude=.venv
--exclude=venv
--exclude=.tox
--exclude=*.pyc
--exclude=dist
--exclude=build
--exclude=*.min.js
--exclude=package-lock.json
--exclude=yarn.lock
--exclude=coverage
```

**Project already has configs** with `--recurse`, `--languages`, and excludes:

```
--fields=+neKS
--extras=+q
--output-format=u-ctags
```

### Updating configuration

Only update `.ctags.d/code-index.ctags` when the user explicitly asks — e.g.,
"exclude the migrations directory", "only index Python files", "add more
excludes". Do not re-run autodetection on every index.

## Index Generation

1. On the **first index of the session**, run the **first-time setup** (see
   Ctags Configuration above) to ensure the config is current.
2. Run the index command **from the project root** (where `.git` lives):
   ```bash
   ctags -f .ctags
   ```
   This auto-discovers all configs from `.ctags.d/`.
3. If `.ctags` is not in `.gitignore`, append it:
   ```bash
   grep -qxF '.ctags' .gitignore 2>/dev/null || echo '.ctags' >> .gitignore
   ```

## When to Index

- **First symbol lookup** — if `.ctags` doesn't exist, index now.
- **Symbol not found but source file exists** — stale index, re-index.
- **Line numbers don't match actual code** — index is outdated, re-index.
- **User explicitly asks** to re-index or refresh — re-index.
- **When in doubt, re-index** — ctags runs in seconds. Don't overthink
  staleness, just re-index.
- **Index on first read or search, not before** — don't index proactively
  on session start. Index the moment you need to read, search, or edit code.

## Symbol Search

Search the `.ctags` file using Grep. The ctags format is tab-delimited:
`symbol<TAB>file<TAB>pattern<TAB>fields...`

### By name

Find a symbol by exact name, prefix, or substring:

```
# Exact match
Grep pattern="^UserService\t" path=".ctags" output_mode="content"

# Prefix match (e.g., all User* symbols)
Grep pattern="^User[^\t]*\t" path=".ctags" output_mode="content"

# Substring match (broad search)
Grep pattern="User" path=".ctags" output_mode="content"
```

### By kind

The kind is a tab-separated column (e.g., `\tclass\t`, `\tfunction\t`):

```
Grep pattern="\tclass\t" path=".ctags" output_mode="content"
```

### By file

Find all symbols defined in a specific file:

```
Grep pattern="\tpath/to/file.py\t" path=".ctags" output_mode="content"
```

### Combined

Chain name and kind for precise lookup — search by name first, then filter
results by kind:

```
Grep pattern="^SymbolName\t" path=".ctags" output_mode="content"
```

Then visually confirm the `kind:` field in the results matches what you need
(e.g., `kind:class` vs `kind:function`).

## Symbol Retrieval

Once you find a symbol in the ctags output, extract its location to read just
that symbol — not the entire file.

1. Parse `line:N` from the ctags entry to get the start line.
2. Parse `end:M` from the ctags entry to get the end line.
3. Compute limit: `L = end - line + 1`.
4. Read the symbol:
   ```
   Read file_path="path/to/file.py" offset=N limit=L
   ```

If `end:` is not present (some symbol kinds omit it), read a reasonable chunk
(e.g., 30 lines) and adjust if the definition continues.

## File Outline

To get an overview of a single file, grep all its symbols from the index and
present them as a structured list:

1. Search:
   ```
   Grep pattern="\tpath/to/file.py\t" path=".ctags" output_mode="content"
   ```
2. Present results as a structured outline:
   ```
   class       ClassName         line:10
     method      method_one      line:15
     method      method_two      line:30
   function    standalone_func   line:55
   ```

Indent methods and members under their parent class using the `scope:` field
from the ctags output to determine hierarchy.

## Project Outline

To understand the overall project structure, grep for top-level symbols only:

```
Grep pattern="\t(class|function|module)\t" path=".ctags" output_mode="content"
```

Group results by file path and present as a tree:

```
src/auth/service.py
  class AuthService          line:12
  function create_token      line:85

src/users/repository.py
  class UserRepository       line:8
  class UserNotFound         line:95
```

This gives a high-level map of the codebase without reading any source files.

## Best Practices

1. **Always search `.ctags` before reading whole files.** A symbol lookup is
   faster and more focused than reading an entire module.
2. **Use line ranges from ctags entries.** Read only the lines that define the
   symbol you need (`offset` + `limit`), not the surrounding file.
3. **Prefer outlines over full reads.** When the user asks "what's in this
   file", produce a file outline from the index — don't read the file.
4. **Use `scope:` fields to navigate hierarchies.** To find all methods of a
   class, grep for `scope:ClassName` instead of reading the whole class.
5. **Batch related lookups.** If you need multiple symbols, run parallel Grep
   calls against `.ctags` rather than sequential file reads.

