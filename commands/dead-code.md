# Dead Code Analysis

Analyze the codebase for dead code using jCodeMunch and jDocMunch MCP tools. This skill performs a comprehensive, tool-driven dead code analysis across ALL indexed file types without relying on grep or manual tracing.

## Usage

```
/dead-code                  # Analyze entire codebase (all languages + docs)
/dead-code src/             # Scope to a specific directory
/dead-code src/utils        # Scope to a subdirectory
/dead-code server.py        # Analyze a single file for dead functions
/dead-code *.ts             # Scope to TypeScript files only
/dead-code docs             # Find orphaned documentation only
```

You can pass any directory path, file path, file pattern, or the keyword `docs` (for documentation-only analysis). The skill auto-detects the project structure — no project-specific knowledge needed.

### What it does

1. Inventories all files in scope (code + docs)
2. Checks every file for importers using jCodeMunch `find_importers`
3. Runs blast radius analysis on entangled code using `get_blast_radius`
4. Maps dependency graphs for complex cases using `get_dependency_graph`
5. Finds dead functions inside live files using `find_references`
6. Finds orphaned docs referencing dead code using jDocMunch `search_sections`
7. Traces CLI flags / API routes to identify dead code paths
8. Presents tiered findings (dead files → dead functions → dead docs → needs verification)

### What you get

A tiered report with confidence levels:
- **Tier 1:** Dead whole files (0 importers — safe to delete)
- **Tier 2:** Dead functions inside live files (0 references)
- **Tier 3:** Dead/orphaned documentation
- **Tier 4:** Dead CLI flags or API routes
- **Tier 5:** Items needing manual verification

All findings are verified by jCodeMunch/jDocMunch before being reported. Nothing is deleted automatically — you review and decide.

## Arguments

- `$ARGUMENTS` — Optional: scope to analyze (e.g., "pipeline", "backend", "tests", "frontend", "docs", or a specific file/directory). Defaults to full codebase (all indexed file types).

## Supported File Types

This skill analyzes ALL file types indexed by jCodeMunch (44 languages) and jDocMunch (10+ formats):

### Code files (via jCodeMunch — 44 languages, tree-sitter parsed)

**Full symbol extraction:**
Python (`.py`), JavaScript (`.js`/`.jsx`), TypeScript (`.ts`), TSX (`.tsx`), Go (`.go`), Rust (`.rs`), Java (`.java`), PHP (`.php`), Dart (`.dart`), C# (`.cs`), C (`.c`), C++ (`.cpp`/`.cc`/`.cxx`/`.h`), Swift (`.swift`), Elixir (`.ex`/`.exs`), Ruby (`.rb`/`.rake`), Perl (`.pl`/`.pm`), Kotlin (`.kt`/`.kts`), Gleam (`.gleam`), Bash (`.sh`), GDScript (`.gd`), Scala (`.scala`), Lua (`.lua`), Erlang (`.erl`), Fortran (`.f90`+), SQL (`.sql`), Verse (`.verse`), Objective-C (`.m`/`.mm`), Protocol Buffers (`.proto`), HCL/Terraform (`.tf`/`.hcl`), GraphQL (`.graphql`/`.gql`), Groovy (`.groovy`/`.gradle`), Nix (`.nix`), Vue (`.vue`), Blade (`.blade.php`), EJS (`.ejs`), Assembly (`.asm`/`.s`), AutoHotkey (`.ahk`), XML (`.xml`), AL (`.al`)

**Text search indexing:**
Haskell (`.hs`), Julia (`.jl`), R (`.r`), CSS (`.css`), TOML (`.toml`)

### Documentation & data files (via jDocMunch)
- **Markdown** (`.md`, `.markdown`, `.mdx`) — heading-based section splitting, frontmatter support
- **reStructuredText** (`.rst`) — adornment-based heading detection
- **AsciiDoc** (`.adoc`)
- **HTML** (`.html`)
- **Plain text** (`.txt`) — paragraph-block splitting
- **OpenAPI/Swagger** (`.yaml`/`.yml`/`.json`) — operations grouped by tag
- **JSON/JSONC** (`.json`/`.jsonc`) — top-level keys as sections
- **XML/SVG/XHTML** (`.xml`/`.svg`/`.xhtml`)
- **Jupyter Notebooks** (`.ipynb`) — markdown cells as sections, code cells as content

## Prerequisites

- jCodeMunch MCP must be available and indexed (`mcp__jcodemunch__*`)
- jDocMunch MCP must be available and indexed (`mcp__jdocmunch__*`)
- Run `mcp__jcodemunch__index_folder(path=".", incremental=true, use_ai_summaries=false)` and `mcp__jdocmunch__index_local(path=".", use_ai_summaries=false)` if not already done this session.

## Analysis Steps

Execute these steps in order. Use ONLY jCodeMunch and jDocMunch tools — no grep, no Read on code files, no manual tracing.

### Step 1: Inventory All Files

Use `mcp__jcodemunch__get_file_tree(repo, path_prefix="")` to get the full file tree of indexed code files.
Use `mcp__jcodemunch__get_repo_outline(repo)` to see the language breakdown and file counts.

Filter to the scope requested by the user (or all files if no scope given).
Count total files per language and note the scope.

### Step 2: Find Entry Points

**Python:**
Use `mcp__jcodemunch__search_text(repo, query="__name__.*__main__", is_regex=true, file_pattern="*.py")` to find all CLI entry points.

**TypeScript/TSX:**
Use `mcp__jcodemunch__search_text(repo, query="export default", file_pattern="*.{ts,tsx}")` to find default exports.
Use `mcp__jcodemunch__search_text(repo, query="app\\.listen|createServer|NextResponse", is_regex=true, file_pattern="*.{ts,tsx}")` to find server entry points.

For each entry point, note:
- File path
- Language
- Purpose (from symbol summary or docstring)

### Step 3: Check Each File for Importers

For EVERY code file in scope, run:
```
mcp__jcodemunch__find_importers(repo, file_path)
```

Classify each file:
- **0 importers + no entry point** → DEAD candidate (high confidence)
- **0 importers + is entry point** → Standalone script/page (review with user)
- **Importers only from test files** → TEST-ONLY (may be testing dead code)
- **Has live importers** → ALIVE

### Step 4: Blast Radius on Dead Candidates

For any file classified as DEAD that has symbols imported elsewhere, run:
```
mcp__jcodemunch__get_blast_radius(repo, symbol, depth=2)
```

This distinguishes:
- **confirmed** — file is imported AND symbol name is referenced (true dependency)
- **potential** — file is imported but symbol name not found (may be wildcard/namespace import)

A file with only "potential" blast radius and 0 "confirmed" references to its symbols is truly dead.

### Step 5: Dependency Graph for Entangled Code

For files that ARE imported by live code but may contain dead code paths, run:
```
mcp__jcodemunch__get_dependency_graph(repo, file, direction="both", depth=2)
```

This reveals:
- Whether the import is from a live file or another dead file
- The full web of dependencies (dead files importing live modules is safe to remove)
- Circular dependency chains

### Step 6: Find Dead Functions/Components Inside Live Files

For key live files (identified by the user or by size), run:
```
mcp__jcodemunch__get_file_outline(repo, file_path)
```

Then for each function/class/component, check if it's referenced:
```
mcp__jcodemunch__find_references(repo, identifier="function_name")
```

Items with 0 references outside their own file are DEAD candidates:
- **Python:** dead functions, classes, constants
- **TypeScript/TSX:** dead exported functions, components, types, interfaces
- **Any language:** dead classes, methods

### Step 7: Find Dead Documentation

Two-part analysis using jDocMunch:

**Part A — Docs referencing dead code:**
```
mcp__jdocmunch__search_sections(repo, query="dead_module_name1 dead_module_name2 ...")
```
Any docs referencing dead modules should be flagged for cleanup.

**Part B — Orphaned documentation files:**
Use `mcp__jdocmunch__get_toc_tree(repo)` to see all indexed docs.
For each doc file, check if it documents code that still exists:
- READMEs for deleted modules → DEAD
- API docs for removed endpoints → DEAD
- Session summaries / historical docs → keep (historical record)
- Architectural docs that reference only dead patterns → flag for review

### Step 8: CLI Flag / Route Tracing (if applicable)

**Python CLI (argparse/click):**
```
mcp__jcodemunch__search_text(repo, query="add_argument", file_pattern="*.py")
```

**TypeScript API routes (Next.js/Express):**
```
mcp__jcodemunch__search_text(repo, query="export.*GET|export.*POST|router\\.", is_regex=true, file_pattern="*.{ts,tsx}")
```

Cross-reference flags/routes with the user's actual usage to identify dead flags/endpoints and their code paths.

## Output Format

Present findings in tiered format:

### Tier 1: Dead Whole Files (by language)
| File | Language | Importers | Evidence | Confidence |
|------|----------|-----------|----------|------------|

### Tier 2: Dead Functions/Components Inside Live Files
| Symbol | File | Language | References | Evidence | Confidence |
|--------|------|----------|-----------|----------|------------|

### Tier 3: Dead/Orphaned Documentation
| Doc File | References Dead Code | Action |
|----------|---------------------|--------|

### Tier 4: Dead CLI Flags / Routes
| Flag/Route | File | Used? | Evidence |
|-----------|------|-------|----------|

### Tier 5: Needs Verification
| Item | Question | How to Verify |
|------|----------|---------------|

### Summary Statistics
| Category | Count |
|----------|-------|
| Dead code files | N |
| Dead doc files | N |
| Dead functions in live files | N |
| Dead CLI flags/routes | N |
| Needs verification | N |

## Rules

1. **ONLY use jCodeMunch and jDocMunch tools** — no Grep, no Read on code files, no manual tracing
2. **`find_importers` is the source of truth** for file-level dead code — if it says 0 importers, the file is dead
3. **`get_blast_radius` is the source of truth** for symbol-level impact — use it before recommending removal of entangled code
4. **Never recommend removing a file without checking `find_importers` first**
5. **Never recommend removing a function without checking `find_references` first**
6. **Flag but don't auto-classify** standalone scripts/pages (files with entry points but 0 importers) — the user decides if they're still useful
7. **Present findings for user review** — don't assume removal is desired. The user decides what to act on.
8. **Analyze ALL indexed file types** — not just one language. Check every language and doc format the project uses.
9. **Save findings** when the user confirms — suggest a reasonable path like `DEAD-CODE-REPORT.md` or ask the user where to save.

## Tool Reference

| Analysis Need | jCodeMunch Tool | Works On |
|--------------|----------------|----------|
| List all files | `get_file_tree` | All indexed code |
| Language breakdown | `get_repo_outline` | All indexed code |
| Who imports a file | `find_importers` | All indexed code |
| Who uses a specific symbol | `get_blast_radius` | All indexed code |
| Full dependency web | `get_dependency_graph` | All indexed code |
| Find function/class definitions | `search_symbols` | All indexed code |
| Is a symbol used anywhere | `find_references` | All indexed code |
| Get file structure | `get_file_outline` | All indexed code |
| Search for patterns | `search_text` | All indexed code |
| Class inheritance tree | `get_class_hierarchy` | Python, TS, Java, C# |

| Analysis Need | jDocMunch Tool | Works On |
|--------------|---------------|----------|
| Find docs referencing dead code | `search_sections` | .md, .mdx, .rst, .adoc, .txt, .html, .yaml, .json, .ipynb |
| Full doc index | `get_toc_tree` | All indexed docs |
| Check doc structure | `get_toc` | Single doc file |
| Single doc outline | `get_document_outline` | Single doc file |
| Read specific section | `get_section` | Single doc section |
| Batch read sections | `get_sections` | Multiple sections |
| Section with context | `get_section_context` | Section + ancestors + children |
