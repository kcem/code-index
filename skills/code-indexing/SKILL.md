---
name: code-indexing
version: 0.1.0
description: >-
  This skill should be used INSTEAD OF reading whole files or guessing file
  paths when exploring unfamiliar or large codebases. Use it when: searching
  for symbol definitions ("find class UserService", "where is get_user
  defined"), understanding project structure ("what classes exist", "show me
  an outline"), or navigating code you haven't seen before. Index once with
  ctags, then search symbols precisely — faster and more accurate than
  grepping source files. Do NOT use for small projects or when you already
  know which file to read. Requires universal-ctags.
---

# Code Indexing — Fast, Precise Codebase Exploration

Use this skill when you need to explore or navigate a codebase without reading
whole files. Index once with ctags, then search symbols precisely. **Default to
this over Glob+Read for any project with 10+ source files.**

**Allowed tools:** Use Bash to run `ctags` commands (indexing, version checks).
Use Grep to search `.ctags` files. Use Read with offset/limit for symbol retrieval.

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

## Index Generation

Use a config file to keep the ctags invocation reproducible.

**Config file:** `.ctags.d/code-index.ctags`

```
--recurse
--fields=+neKS
--extras=+q
--output-format=u-ctags
--languages=Python,TypeScript,JavaScript
--exclude=.git
--exclude=node_modules
--exclude=__pycache__
--exclude=.venv
--exclude=venv
--exclude=dist
--exclude=build
--exclude=.tox
--exclude=*.min.js
--exclude=package-lock.json
--exclude=yarn.lock
```

**Steps:**

1. If `.ctags.d/code-index.ctags` does not exist, create it with the contents
   above (create the `.ctags.d/` directory first if needed).
2. Run the index command:
   ```bash
   ctags --options=NONE --options=.ctags.d/code-index.ctags -f .ctags
   ```
3. If `.ctags` is not in `.gitignore`, append it:
   ```bash
   grep -qxF '.ctags' .gitignore 2>/dev/null || echo '.ctags' >> .gitignore
   ```

## When to Index

Make autonomous decisions about when to generate or regenerate the index:

- **`.ctags` file does not exist** — index now.
- **Symbol not found but source file exists** — stale index, re-index.
- **`.ctags` mtime is older than the last git commit** — re-index. Check with:
  ```bash
  # macOS
  [ "$(stat -f %m .ctags)" -lt "$(git log -1 --format=%ct)" ] && echo "stale"
  # Linux
  [ "$(stat -c %Y .ctags)" -lt "$(git log -1 --format=%ct)" ] && echo "stale"
  ```
- **User explicitly asks** to re-index or refresh — re-index.
- **Never index on session start unprompted** — only index when a symbol search
  or exploration task actually requires it.

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

Find all symbols of a given kind (class, function, method, variable, etc.):

```
Grep pattern="kind:class" path=".ctags" output_mode="content"
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
   kind:class    ClassName         line:10
   kind:method     method_one      line:15
   kind:method     method_two      line:30
   kind:function standalone_func   line:55
   ```

Indent methods and members under their parent class using the `scope:` field
from the ctags output to determine hierarchy.

## Project Outline

To understand the overall project structure, grep for top-level symbols only:

```
Grep pattern="kind:(class|function|module)" path=".ctags" output_mode="content"
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

## Extending Languages

To index additional languages, add them to the `--languages` line in
`.ctags.d/code-index.ctags`:

```
--languages=Python,TypeScript,JavaScript,Go,Rust,Java
```

Add language-specific excludes as needed:

```
--exclude=target          # Rust build output
--exclude=vendor          # Go vendor directory
--exclude=*.class         # Java compiled classes
```

After modifying the config, re-index:

```bash
ctags --options=NONE --options=.ctags.d/code-index.ctags -f .ctags
```
