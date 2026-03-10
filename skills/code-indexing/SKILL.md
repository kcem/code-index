---
name: code-indexing
version: 0.3.1
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
  module", "what does this package export"), spotting duplicate
  definitions ("are there two classes named Config", "same function name
  in multiple files"), or searching across all your projects ("where did
  I implement retry logic", "find UserService across projects"). Index
  once with ctags, then search and read precisely. Supports local
  (in-project) and global (~/.local/share/code-index/) storage modes.
  Requires universal-ctags.
---

# Code Indexing — Fast, Precise Codebase Exploration

Use this skill when you need to explore or navigate a codebase without reading
whole files. Index once with ctags, then search symbols precisely. **Default to
this over reading whole files.**

**Core rule: never read a full file when you only need part of it.** Use the
index to find the symbol's start and end lines, then read only that range.

**Allowed tools:** Use Bash to run `ctags` commands (indexing, version checks).
Use Grep to search tags files. Use Read with offset/limit for symbol retrieval.

## Prerequisites

Before indexing, verify universal-ctags is installed:

```bash
command -v ctags && ctags --version
```

The output **must** contain "Universal Ctags". If it shows "Exuberant Ctags" or
the command is not found, the user needs to install universal-ctags:

- **macOS**: `brew install universal-ctags`
- **Ubuntu/Debian**: `apt install universal-ctags`
- **Alpine**: `apk add ctags`
- **Fallback** — if no package is available (e.g., Wolfi, minimal containers),
  use a static binary (no dependencies):
  ```bash
  curl -LO https://github.com/universal-ctags/ctags-nightly-build/releases/latest/download/uctags-$(date +%Y.%m.%d)-linux-x86_64.release.tar.gz
  tar xzf uctags-*.tar.gz && cp uctags-*/bin/ctags /usr/local/bin/
  ```
  For aarch64, replace `x86_64` with `aarch64`. Browse all builds at
  https://github.com/universal-ctags/ctags-nightly-build/releases

If a shell alias interferes with `ctags`, use the full binary path instead (e.g.,
`/opt/homebrew/bin/ctags` on macOS with Homebrew, `/usr/local/bin/ctags` for
static binary installs). Find it with `which -a ctags`.

If universal-ctags is not available or the wrong version is detected, tell the
user and stop. Do not attempt to index with Exuberant Ctags — the output format
is incompatible.

## Storage Modes

The skill supports two storage modes for config and tags files:

### Local mode (in-project)

Config and tags live in the project root:

```
<project-root>/
├── .ctags.d/code-index.ctags    # config (excludes, required options)
└── .ctags                       # tags file (relative paths)
```

- `.ctags.d/` is committed to git (project configuration)
- `.ctags` is added to `.gitignore` (generated artifact)
- Tags contain relative paths
- Cross-project search is not available

### Global mode (stealth)

Config and tags live in `~/.local/share/code-index/`:

```
~/.local/share/code-index/
├── config.ctags          # shared required options (all projects)
├── a3f2b1e9.tags         # project index (absolute paths)
├── a3f2b1e9.ctags        # project config (excludes)
├── 7c9e4d82.tags         # another project
└── 7c9e4d82.ctags
```

- Zero files in the project — nothing to gitignore, nothing committed
- Tags contain absolute paths (self-describing)
- Cross-project search available across all global indexes

### Project key

Each project is identified by a hash of its absolute root path:

```bash
project_root="$(git rev-parse --show-toplevel 2>/dev/null || pwd)"
hash="$(echo -n "$project_root" | shasum -a 256 | cut -c1-8)"
```

## Index Resolution

On first use in a session, determine which index to use:

1. Compute the project key (see above).
2. Check `~/.local/share/code-index/<hash>.tags` — if it exists, use global mode.
3. Check `<project-root>/.ctags` — if it exists, use local mode.
4. Neither exists — ask the user: "Store index locally (in project, tracked by
   git) or globally (~/.local/share/code-index/, no project files touched)?"

The existence of the tags file IS the preference — no separate config needed.
Once resolved, use that mode for the rest of the session.

## Ctags Configuration

### Required options

These options are **non-negotiable** — the skill cannot work without them:

- `--fields=+neKS` — end line (`e`), line number (`n`), full kind name (`K`),
  signature (`S`). The `+` adds to existing fields.
- `--extras=+q` — qualified tags (class-qualified entries).
- `--output-format=u-ctags` — structured field output format.

### First-time setup — local mode

Run this on the **first index of the session** — whether the config exists or not.

The project root is the directory containing `.git` — find it with
`git rev-parse --show-toplevel` if unsure. If there's no git, use the working
directory and skip all git-related steps (`.gitignore`, commit suggestions).

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

### First-time setup — global mode

1. **Create the global directory** if it doesn't exist:
   ```bash
   mkdir -p ~/.local/share/code-index
   ```

2. **Create or verify `config.ctags`** — the shared required options file at
   `~/.local/share/code-index/config.ctags`:
   ```
   --fields=+neKS
   --extras=+q
   --output-format=u-ctags
   ```

3. **Create `<hash>.ctags`** — the per-project config at
   `~/.local/share/code-index/<hash>.ctags`. This contains only `--recurse` and
   excludes (required options are in `config.ctags`):
   ```
   --recurse
   --exclude=.git
   --exclude=node_modules
   --exclude=__pycache__
   ...
   ```
   Use the same autodetect excludes logic as local mode.

4. **Inform the user** — briefly summarize what was configured.

### Config examples

**Local mode — no existing configs** (typical case — Python + TypeScript project):

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

**Local mode — project already has configs** with `--recurse`, `--languages`, and excludes:

```
--fields=+neKS
--extras=+q
--output-format=u-ctags
```

**Global mode — shared config** (`~/.local/share/code-index/config.ctags`):

```
--fields=+neKS
--extras=+q
--output-format=u-ctags
```

**Global mode — per-project config** (`~/.local/share/code-index/<hash>.ctags`):

```
--recurse
--exclude=.git
--exclude=node_modules
--exclude=__pycache__
--exclude=.venv
--exclude=dist
--exclude=build
--exclude=coverage
```

### Updating configuration

**Local mode:** Only update `.ctags.d/code-index.ctags` when the user explicitly
asks — e.g., "exclude the migrations directory", "only index Python files", "add
more excludes". Do not re-run autodetection on every index.

**Global mode:** Only update `~/.local/share/code-index/<hash>.ctags` when the
user explicitly asks. Same rules apply.

## Index Generation

### Local mode

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

### Global mode

1. On the **first index of the session**, run the **first-time setup** (see
   Ctags Configuration above) to ensure the config and directory exist.
2. Compute the project key:
   ```bash
   project_root="$(git rev-parse --show-toplevel 2>/dev/null || pwd)"
   hash="$(echo -n "$project_root" | shasum -a 256 | cut -c1-8)"
   ```
3. Run the index command **from the project root**:
   ```bash
   cd "$project_root" && ctags \
     --options=~/.local/share/code-index/config.ctags \
     --options=~/.local/share/code-index/$hash.ctags \
     --tag-relative=never \
     -f ~/.local/share/code-index/$hash.tags
   ```
   `--tag-relative=never` writes absolute paths so tags are self-describing.

## When to Index

- **First symbol lookup** — if the tags file doesn't exist, index now.
- **Symbol not found but source file exists** — stale index, re-index.
- **Line numbers don't match actual code** — index is outdated, re-index.
- **User explicitly asks** to re-index or refresh — re-index.
- **When in doubt, re-index** — ctags runs in seconds. Don't overthink
  staleness, just re-index.
- **Index on first read or search, not before** — don't index proactively
  on session start. Index the moment you need to read, search, or edit code.

## Do

- **Search the tags file before reading any file** — a symbol lookup is faster
  and more focused than reading an entire module
- **Read symbols by line range** (`offset` + `limit`), not whole files — use
  `line:N` and `end:M` from the ctags entry to read only the exact definition
- **Prefer outlines over full reads** — when the user asks "what's in this
  file", produce a file outline from the index instead of reading the file
- **Use `scope:` fields to navigate hierarchies** — to find all methods of a
  class, grep for `scope:ClassName` instead of reading the whole class
- **Batch related lookups** — if you need multiple symbols, run parallel Grep
  calls against the tags file rather than sequential file reads
- **Use cross-project search** when a symbol isn't found in the current project
  or when the user asks about symbols across projects
- **Re-index when results look wrong or stale** — ctags runs in seconds, don't
  overthink staleness

## Don't

- **Read a whole file to find one function or class** — use the index
- **Use Glob or Grep on source files to find definitions** — search the tags
  file instead
- **Run ctags with inline flags** — use config files (`.ctags.d/` for local
  mode, `--options=` for global mode)
- **Override existing project ctags configs without asking the user**
- **Mix storage modes for the same project** — use whichever mode the index
  resolution determined, don't create both local and global indexes
- **Update config on every index** — only update when the user explicitly asks

## Switching Modes

Index resolution picks global over local. To switch modes, delete the old
mode's tags file — the index will regenerate automatically on next use.

- **Local → global:** delete `<project-root>/.ctags`
- **Global → local:** delete `~/.local/share/code-index/<hash>.tags`
- **Project moved/renamed:** old hash is orphaned, delete it from
  `~/.local/share/code-index/`

## Example: Reading a Symbol

Instead of reading a whole file, use the index:

1. Search the index for the symbol (use the resolved tags file path — see
   Index Resolution):
   `Grep pattern="^get_user\t" path="<tags-file>" output_mode="content"`
2. Parse the result — find `line:N` and `end:M` in the ctags entry.
3. Read only the symbol's lines:
   `Read file_path="/absolute/path/to/src/services/user.py" offset=N limit=(M-N+1)`

Always use absolute paths for Read — source files may not be in your current
working directory. In global mode, the tags file already contains absolute paths.
In local mode, prepend the project root.

## Symbol Search

Search the tags file using Grep. The ctags format is tab-delimited:
`symbol<TAB>file<TAB>pattern<TAB>fields...`

In local mode, file paths in the tags are relative. In global mode, they are
absolute. Use the appropriate path form when searching by file.

### By name

Find a symbol by exact name, prefix, or substring:

```
# Exact match (local mode — path to project's .ctags)
Grep pattern="^UserService\t" path="/absolute/path/to/project/.ctags" output_mode="content"

# Exact match (global mode — path to global tags file)
Grep pattern="^UserService\t" path="~/.local/share/code-index/<hash>.tags" output_mode="content"

# Prefix match (e.g., all User* symbols)
Grep pattern="^User[^\t]*\t" path="<tags-file>" output_mode="content"

# Substring match (broad search)
Grep pattern="User" path="<tags-file>" output_mode="content"
```

### By kind

The kind is a tab-separated column (e.g., `\tclass\t`, `\tfunction\t`):

```
Grep pattern="\tclass\t" path="<tags-file>" output_mode="content"
```

### By file

Find all symbols defined in a specific file:

```
# Local mode (relative path)
Grep pattern="\tsrc/services/user.py\t" path="<tags-file>" output_mode="content"

# Global mode (absolute path)
Grep pattern="\t/absolute/path/to/project/src/services/user.py\t" path="<tags-file>" output_mode="content"
```

### Combined

Chain name and kind for precise lookup — search by name first, then filter
results by kind:

```
Grep pattern="^SymbolName\t" path="<tags-file>" output_mode="content"
```

Then visually confirm the `kind:` field in the results matches what you need
(e.g., `kind:class` vs `kind:function`).

### Cross-project search

When the user asks about a symbol across all their projects, or when a symbol
is not found in the current project index, search all global indexes:

```
Grep pattern="^UserService\t" path="~/.local/share/code-index" glob="*.tags" output_mode="content"
```

This searches every `*.tags` file in the global directory. Results contain
absolute paths, so you can immediately see which project each symbol belongs to.

Use cross-project search when:
- The user explicitly asks to search across projects
- A symbol is not found in the current project but might exist elsewhere
- The user asks "where did I implement X" without specifying a project

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

In global mode, the file path in the ctags entry is already absolute — use it
directly. In local mode, prepend the project root to get the absolute path.

If `end:` is not present (some symbol kinds omit it), read a reasonable chunk
(e.g., 30 lines) and adjust if the definition continues.

## File Outline

To get an overview of a single file, grep all its symbols from the index and
present them as a structured list:

1. Search:
   ```
   Grep pattern="\tpath/to/file.py\t" path="<tags-file>" output_mode="content"
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
Grep pattern="\t(class|function|module)\t" path="<tags-file>" output_mode="content"
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
