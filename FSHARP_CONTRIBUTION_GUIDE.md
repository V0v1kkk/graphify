# F#, Razor/Blazor & Cross-Language Improvements for Graphify

This document describes the changes made on branch `fix/fsharp-graph-quality` to add F# and Razor/Blazor language support to graphify and fix several cross-language graph quality issues. Each commit is designed to be cherry-picked independently for separate upstream PRs.

## Upstream Contributions

### Forks

| Upstream | Fork | Local path |
|----------|------|------------|
| [safishamsi/graphify](https://github.com/safishamsi/graphify) | [V0v1kkk/graphify](https://github.com/V0v1kkk/graphify) | `/home/vladimir/GitRoot/external/graphify` |
| [tris203/tree-sitter-razor](https://github.com/tris203/tree-sitter-razor) | [V0v1kkk/tree-sitter-razor](https://github.com/V0v1kkk/tree-sitter-razor) | `/home/vladimir/GitRoot/external/tree-sitter-razor` |
| [ionide/tree-sitter-fsharp](https://github.com/ionide/tree-sitter-fsharp) | [V0v1kkk/tree-sitter-fsharp](https://github.com/V0v1kkk/tree-sitter-fsharp) | `/home/vladimir/GitRoot/external/tree-sitter-fsharp` |

### Submitted Issues & Pull Requests

**graphify** — 3 unblocked PRs submitted:

| Issue | PR | Branch | Description | Status |
|-------|----|--------|-------------|--------|
| [#437](https://github.com/safishamsi/graphify/issues/437) | [#440](https://github.com/safishamsi/graphify/pull/440) | `fix/bcl-method-blocklist` | BCL method blocklist for cross-file inference | Awaiting review |
| [#438](https://github.com/safishamsi/graphify/issues/438) | [#441](https://github.com/safishamsi/graphify/pull/441) | `fix/node-id-collisions` | Disambiguate colliding node IDs from same-name files | Awaiting review |
| [#439](https://github.com/safishamsi/graphify/issues/439) | [#442](https://github.com/safishamsi/graphify/pull/442) | `fix/merge-stub-nodes` | Merge stub nodes with real cross-language definitions | Awaiting review |

**tree-sitter-razor** — 1 bug fix PR:

| Issue | PR | Branch | Description | Status |
|-------|----|--------|-------------|--------|
| [#18](https://github.com/tris203/tree-sitter-razor/issues/18) | [#19](https://github.com/tris203/tree-sitter-razor/pull/19) | `fix/python-scanner` | Include `scanner.c` in Python bindings `setup.py` | Awaiting review |

**tree-sitter-fsharp** — 1 feature request:

| Issue | Description | Status |
|-------|-------------|--------|
| [#176](https://github.com/ionide/tree-sitter-fsharp/issues/176) | Publish Python bindings to PyPI | Awaiting response |

### Blocked PRs (not yet submitted)

| Commits | Description | Blocked on |
|---------|-------------|------------|
| 1 + 2 | F# language support | tree-sitter-fsharp PyPI ([#176](https://github.com/ionide/tree-sitter-fsharp/issues/176)) |
| 6 | Resolve F# `open` statements | F# support PR (commits 1+2) |
| 7 | Razor/Blazor extractor | Can be submitted independently (tree-sitter-razor builds from source) |
| 8 | Deep extraction (F#/Razor parts) | F# and Razor PRs; C# parts could be extracted independently |

---

## Commits (in order)

### 1. `932d6ec` — feat: add F# file extensions (.fs, .fsx) to detection and collection

**Files:** `graphify/detect.py`, `graphify/extract.py`
**PR scope:** Trivial, no dependencies

Registers `.fs` and `.fsx` in `CODE_EXTENSIONS` (detect.py) and `_EXTENSIONS` (extract.py `collect_files`) so graphify discovers F# source files during project scanning.

---

### 2. `0e4207a` — feat: add F# AST extractor with tree-sitter-fsharp

**Files:** `graphify/extract.py`
**PR scope:** Main feature PR, depends on tree-sitter-fsharp PyPI package
**Depends on:** Commit 1

Adds a custom `extract_fsharp()` function (not using `LanguageConfig` due to F#'s nested AST structure for names). Extracts:
- Modules (`module_defn`) with containment edges
- Types: records, unions, classes, interfaces, abbreviations (`type_definition`)
- Union type cases with **qualified labels** (e.g. `EdgeRelation.Contains` instead of `Contains`) to avoid name collisions with common BCL methods
- Functions and values (`function_or_value_defn`) with `function_declaration_left`/`value_declaration_left` navigation
- Members (`member_defn` / `method_or_prop_defn`)
- `open` / import statements (`import_decl`)
- Intra-file call graph via `application_expression`, `dot_expression`, `long_identifier_or_op`
- Unresolved calls saved to `raw_calls` for cross-file resolution

Registers `.fs` and `.fsx` in `_DISPATCH`.

---

### 3. `0b5da50` — fix: add BCL method blocklist to cross-file inference

**Files:** `graphify/extract.py`
**PR scope:** Independent improvement, benefits all .NET languages (C#, F#)
**Submitted:** [PR #440](https://github.com/safishamsi/graphify/pull/440)

Adds a blocklist of ~120 common .NET BCL/framework method names (`Contains`, `Equals`, `ToString`, `Where`, `Select`, `ListAsync`, `CreateDbContext`, etc.) that are skipped during cross-file call resolution.

**Problem:** The cross-file inference resolves unresolved calls by matching callee name to any node label in the graph. Common method names like `Contains` would incorrectly match F# union cases or other types, creating hundreds of false `INFERRED` edges. For example, `string.Contains()` calls in C# would all point to the F# DU case `EdgeRelation.Contains`.

**Impact:** Reduced false INFERRED edges from 538 to 362 in the test project.

---

### 4. `805bc77` — fix: disambiguate colliding node IDs from same-name files

**Files:** `graphify/extract.py`
**PR scope:** Independent improvement, benefits all languages
**Submitted:** [PR #441](https://github.com/safishamsi/graphify/pull/441)

Adds a post-extraction pass that detects when multiple files produce nodes with the same ID (because `_make_id(stem, name)` uses only the file stem). When collisions are found, the parent directory name is prepended to disambiguate.

**Problem:** Two `Program.cs` files in different projects (e.g. `BKD.Api/Program.cs` and `BooksKnowledgeDistillation/Program.cs`) would both produce `program_program` as the node ID, causing their methods to merge into one node.

**After fix:** `bkd_api_program_program` and `booksknowledgedistillation_program_program` are separate nodes.

---

### 5. `1b54f6d` — fix: merge C# stub nodes with real cross-language definitions

**Files:** `graphify/extract.py`
**PR scope:** Cross-language improvement, benefits C#/F# mixed projects
**Submitted:** [PR #442](https://github.com/safishamsi/graphify/pull/442)

Adds a post-extraction pass that merges stub nodes (empty `source_file`) with real definitions by matching labels. Prioritizes definition files (`Interfaces.fs`, `Domain.fs`, `Types.fs`, `Contracts.fs`, `Abstractions.fs`).

**Problem:** When C# code inherits from a type defined in F# (e.g. `class SqliteBookStore : BookStore`), the C# extractor creates a stub node `bookstore` with no source file. The `inherits` edge points to this stub instead of the real F# definition `interfaces_bookstore` from `Interfaces.fs`.

**After fix:** `SqliteBookStore --inherits--> interfaces_bookstore` (the real F# abstract type).

---

### 6. `f31d83e` — fix: resolve F# open statements to actual source files

**Files:** `graphify/extract.py`
**PR scope:** F#-specific, depends on Commit 2

Adds a post-extraction pass that resolves F# `open` import targets to actual file nodes. `open BKD.Core.Domain` now creates an import edge to `src_bkd_core_domain_fs` instead of a generic stub node `domain`.

---

### 7. `337f088` — feat: add Razor/Blazor extractor with tree-sitter-razor

**Files:** `graphify/detect.py`, `graphify/extract.py`
**PR scope:** Main feature PR, depends on tree-sitter-razor (can be built from source)
**Depends on:** Commit 1 (extension registration pattern)

Adds `extract_razor()` function that parses `.razor` files using
[tris203/tree-sitter-razor](https://github.com/tris203/tree-sitter-razor) and extracts:
- `@inject` directives as service dependency edges
- `@using` directives as import edges
- `@implements` directives as inherits edges
- Blazor component references (`<StatusBadge>`, `<SubmitBookDialog>`) excluding FluentUI and framework-internal components
- Method declarations from `@code { }` blocks
- Static method calls (`ClassName.Method()`) from `@code` blocks

**Problem:** In Blazor projects, many C# classes (e.g. `DashboardFilterHelper`) are only
referenced from `.razor` files. Without Razor support, these classes appear as completely
disconnected graph components despite being core to the application.

**Impact:** Main graph component grew from 915 to 995 nodes. `DashboardFilterHelper` and
its 27-node cluster joined the main component.

**Note:** `tree-sitter-razor` has a `setup.py` but the upstream version is missing `scanner.c`
in the sources list. Fix submitted: [tree-sitter-razor PR #19](https://github.com/tris203/tree-sitter-razor/pull/19).

---

### 8. `478bdcb` — fix: deep extraction improvements for C# and F#

**Files:** `graphify/extract.py`
**PR scope:** Independent improvement, benefits all .NET languages
**Depends on:** Commits 2, 7

Multiple fixes to the C#, F#, and Razor extractors:

**C# improvements:**
- Add `object_creation_expression` to `call_types` (`new Type()` → edge to the created type)
- Extract generic type arguments from `invocation_expression` (`AddDbContext<BookDbContext>()` → edge to `BookDbContext`)
- Walk `constructor_declaration` bodies for call graph extraction via `_csharp_extra_walk`
- Walk `global_statement` (top-level C# code in `Program.cs`) for calls (`app.UseMiddleware<ApiKeyMiddleware>()` → edge to `ApiKeyMiddleware`)

**F# improvements:**
- Fix `member_defn` body extraction: find body after `=` token instead of `child_by_field_name("body")` which returns `None` for F# members. This was causing all member method bodies to be invisible to call-graph analysis.
- Extract root identifier from `dot_expression` chains (`PipelineMetrics.counter.Add()` → edge to `PipelineMetrics` module)

**Razor improvements:**
- Extract generic type arguments from `@code` blocks (`ShowDialogAsync<SubmitBookDialog>()` → edge to `SubmitBookDialog`)
- Extract `new Type()` patterns from `@code` blocks

**Impact:** Previously disconnected clusters (BookDbContext, ChapterStore, PageGistStore, BookProcessingHub, BkdWebApplicationFactory, PipelineMetrics, ApiKeyMiddleware, SubmitBookDialog) all joined the main component. Main component grew from 995 to 1058 nodes.

---

## Local Build Instructions

### tree-sitter-fsharp Python Bindings

Until tree-sitter-fsharp publishes to PyPI ([#176](https://github.com/ionide/tree-sitter-fsharp/issues/176)), build locally:

```bash
git clone https://github.com/ionide/tree-sitter-fsharp /path/to/tree-sitter-fsharp
cd /path/to/tree-sitter-fsharp

# Create setup.py (not included in upstream)
cat > setup.py << 'SETUP'
from setuptools import setup, Extension
setup(
    name="tree-sitter-fsharp",
    version="0.3.0",
    package_dir={"": "bindings/python"},
    packages=["tree_sitter_fsharp"],
    ext_modules=[Extension(
        "tree_sitter_fsharp._binding",
        sources=["bindings/python/tree_sitter_fsharp/binding.c",
                 "fsharp/src/parser.c", "fsharp/src/scanner.c"],
        include_dirs=["fsharp/src"],
        extra_compile_args=["-std=c11"],
    )],
    python_requires=">=3.10",
)
SETUP

# Create Python binding files
mkdir -p bindings/python/tree_sitter_fsharp
cat > bindings/python/tree_sitter_fsharp/__init__.py << 'INIT'
from ._binding import language
__all__ = ["language"]
INIT

# binding.c should already exist; if not, create it (see tree-sitter docs)

pip install -e .
```

### tree-sitter-razor Python Bindings

```bash
git clone https://github.com/tris203/tree-sitter-razor /path/to/tree-sitter-razor
cd /path/to/tree-sitter-razor

# Fix setup.py: add scanner.c to sources (PR #19 pending)
# In setup.py, change:
#   sources=["bindings/python/tree_sitter_razor/binding.c", "src/parser.c"]
# To:
#   sources=["bindings/python/tree_sitter_razor/binding.c", "src/parser.c", "src/scanner.c"]

pip install -e .
```

---

## Test Results

Tested on a mixed F#/C#/Blazor project (BooksKnowledgeDistillation, 141 code files):

| Metric | Before | After all fixes |
|--------|--------|-----------------|
| Nodes | 1055 | 1127 |
| Edges | 1699 | 2177 |
| INFERRED edges | 538 | 362 |
| Communities | 57 | 44 (all labeled) |
| Isolated (degree=0) | 119 | 3 |
| Main component | 915 | 1058 |
| Disconnected components | 24 | 20 |
| Top hub | `Contains` (false!) | `BookService` (correct) |
| F# types extracted | 0 | 44 (from Domain.fs alone) |
| Razor pages extracted | 0 | 18 |
| C# → F# inherits | broken (stubs) | correct (real definitions) |
| DashboardFilterHelper | disconnected | in main component |
| BookDbContext | disconnected | in main component |
| PipelineMetrics | disconnected | in main component |
| ApiKeyMiddleware | disconnected | in main component |

The remaining 20 disconnected components are genuinely isolated code: Python utility scripts, JavaScript files, test classes without external references, unused Blazor components, and legacy code.
