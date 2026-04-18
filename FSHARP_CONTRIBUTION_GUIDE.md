# F# Support for Graphify — Contribution Guide

This document describes the changes made on branch `fix/fsharp-graph-quality` to add F# language support to graphify and fix several cross-language graph quality issues. Each commit is designed to be cherry-picked independently for separate upstream PRs.

## Prerequisites

Before these changes can be upstreamed, the [tree-sitter-fsharp](https://github.com/ionide/tree-sitter-fsharp) project needs to publish Python bindings to PyPI. Currently, `tree-sitter-fsharp` has `"python": false` in its `tree-sitter.json` and no PyPI package exists. The grammar itself works (v0.3.0), and Python bindings can be built locally from source using `setup.py` (see below).

Once `tree-sitter-fsharp` is on PyPI, it can be added to graphify's optional dependencies just like other language grammars.

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

Adds a blocklist of ~120 common .NET BCL/framework method names (`Contains`, `Equals`, `ToString`, `Where`, `Select`, `ListAsync`, `CreateDbContext`, etc.) that are skipped during cross-file call resolution.

**Problem:** The cross-file inference resolves unresolved calls by matching callee name to any node label in the graph. Common method names like `Contains` would incorrectly match F# union cases or other types, creating hundreds of false `INFERRED` edges. For example, `string.Contains()` calls in C# would all point to the F# DU case `EdgeRelation.Contains`.

**Impact:** Reduced false INFERRED edges from 538 to 362 in the test project.

---

### 4. `805bc77` — fix: disambiguate colliding node IDs from same-name files

**Files:** `graphify/extract.py`
**PR scope:** Independent improvement, benefits all languages

Adds a post-extraction pass that detects when multiple files produce nodes with the same ID (because `_make_id(stem, name)` uses only the file stem). When collisions are found, the parent directory name is prepended to disambiguate.

**Problem:** Two `Program.cs` files in different projects (e.g. `BKD.Api/Program.cs` and `BooksKnowledgeDistillation/Program.cs`) would both produce `program_program` as the node ID, causing their methods to merge into one node.

**After fix:** `bkd_api_program_program` and `booksknowledgedistillation_program_program` are separate nodes.

---

### 5. `1b54f6d` — fix: merge C# stub nodes with real cross-language definitions

**Files:** `graphify/extract.py`
**PR scope:** Cross-language improvement, benefits C#/F# mixed projects

Adds a post-extraction pass that merges stub nodes (empty `source_file`) with real definitions by matching labels. Prioritizes definition files (`Interfaces.fs`, `Domain.fs`, `Types.fs`, `Contracts.fs`, `Abstractions.fs`).

**Problem:** When C# code inherits from a type defined in F# (e.g. `class SqliteBookStore : BookStore`), the C# extractor creates a stub node `bookstore` with no source file. The `inherits` edge points to this stub instead of the real F# definition `interfaces_bookstore` from `Interfaces.fs`.

**After fix:** `SqliteBookStore --inherits--> interfaces_bookstore` (the real F# abstract type).

---

### 6. `f31d83e` — fix: resolve F# open statements to actual source files

**Files:** `graphify/extract.py`
**PR scope:** F#-specific, depends on Commit 2

Adds a post-extraction pass that resolves F# `open` import targets to actual file nodes. `open BKD.Core.Domain` now creates an import edge to `src_bkd_core_domain_fs` instead of a generic stub node `domain`.

---

## Suggested PR Strategy

1. **PR 1 (Commits 1 + 2):** "Add F# language support" — the main feature. Blocked on tree-sitter-fsharp PyPI availability.
2. **PR 2 (Commit 3):** "Add BCL method blocklist to cross-file inference" — independent quality fix, can be submitted immediately.
3. **PR 3 (Commit 4):** "Fix node ID collisions across same-name files" — independent quality fix, can be submitted immediately.
4. **PR 4 (Commit 5):** "Merge stub nodes with real cross-language definitions" — general improvement, can be submitted immediately.
5. **PR 5 (Commit 6):** "Resolve F# open statements to source files" — depends on PR 1.

PRs 2, 3, and 4 are independent improvements that benefit all languages and can be submitted right away without waiting for tree-sitter-fsharp.

## Local Build: tree-sitter-fsharp Python Bindings

Until tree-sitter-fsharp publishes to PyPI, build locally:

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
in the sources list. Fix: add `"src/scanner.c"` to the `ext_modules` sources in `setup.py`.

---

## Updated PR Strategy

1. **PR 1 (Commits 1 + 2):** "Add F# language support" — blocked on tree-sitter-fsharp PyPI.
2. **PR 2 (Commit 3):** "Add BCL method blocklist" — **submit now**.
3. **PR 3 (Commit 4):** "Fix node ID collisions" — **submit now**.
4. **PR 4 (Commit 5):** "Merge stub nodes" — **submit now**.
5. **PR 5 (Commit 6):** "Resolve F# open statements" — depends on PR 1.
6. **PR 6 (Commit 7):** "Add Razor/Blazor support" — can be submitted independently (tree-sitter-razor builds from source).

---

## Local Build: tree-sitter-razor Python Bindings

```bash
git clone https://github.com/tris203/tree-sitter-razor /path/to/tree-sitter-razor
cd /path/to/tree-sitter-razor

# Fix setup.py: add scanner.c to sources
# In setup.py, change:
#   sources=["bindings/python/tree_sitter_razor/binding.c", "src/parser.c"]
# To:
#   sources=["bindings/python/tree_sitter_razor/binding.c", "src/parser.c", "src/scanner.c"]

pip install -e .
```

---

## Test Results

Tested on a mixed F#/C#/Blazor project (BooksKnowledgeDistillation):

| Metric | Before | After all fixes |
|--------|--------|-----------------|
| Nodes | 1055 | 1127 |
| Edges | 1699 | 1714 |
| INFERRED edges | 538 | 362 |
| Communities | 57 | 56 |
| Isolated (degree=0) | 119 | 3 |
| Main component | 915 | 995 |
| Disconnected components | 24 | 32 |
| Top hub | `Contains` (false!) | `BookService` (correct) |
| F# types extracted | 0 | 44 (from Domain.fs alone) |
| Razor pages extracted | 0 | 18 |
| C# → F# inherits | broken (stubs) | correct (real definitions) |
| DashboardFilterHelper | disconnected | in main component |
