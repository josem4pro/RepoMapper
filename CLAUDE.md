# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

RepoMap is a Python tool (CLI + MCP server) that generates code maps of repositories for LLMs. It uses Tree-sitter to parse source code, extracts definitions/references, builds a graph, and applies PageRank to rank code elements by importance. Based on Aider's RepoMap design.

## Commands

```bash
# Install dependencies
pip install -r requirements.txt
# Or with uv (pyproject.toml is configured):
uv sync

# CLI usage
python repomap.py .                              # Map current directory
python repomap.py . --map-tokens 2048            # Custom token limit
python repomap.py --chat-files main.py --other-files src/  # Prioritized files
python repomap.py . --verbose --force-refresh    # Debug with cache refresh

# MCP server
python repomap_server.py                         # Starts STDIO-based FastMCP server
```

No test suite exists in this repository.

## Architecture

Six Python modules, all at the root level (no packages):

- **`repomap.py`** - CLI entry point. Parses args, discovers files via `find_src_files()`, instantiates `RepoMap`, calls `get_repo_map()`.
- **`repomap_class.py`** - Core `RepoMap` class. The pipeline: `get_repo_map()` → `get_ranked_tags_map()` → `get_ranked_tags_map_uncached()` → `get_ranked_tags()` → `to_tree()`. Uses binary search to fit output within token limits.
- **`repomap_server.py`** - FastMCP server exposing two tools: `repo_map` (generates maps) and `search_identifiers` (searches code symbols). Wraps `RepoMap` with async via `asyncio.to_thread`.
- **`utils.py`** - `count_tokens()` (tiktoken), `read_text()`, and the `Tag` namedtuple.
- **`scm.py`** - Maps language names to Tree-sitter `.scm` query files in `queries/`.
- **`importance.py`** - Identifies "important" files (README, package.json, Dockerfile, etc.) for ranking boosts.

**Key data flow in `RepoMap`:**
1. Files are parsed with Tree-sitter → `Tag` namedtuples (def/ref, file, line, name)
2. Tags build a `networkx.MultiDiGraph` (file→file edges via shared symbols)
3. PageRank scores files; chat_files get 100x personalization weight
4. Mentioned idents get 10x boost, mentioned files 5x, chat files 20x
5. `to_tree()` renders output using `grep_ast.TreeContext` for code snippets
6. Binary search finds max tags fitting within `max_map_tokens`

**Caching:** Tags are cached in `.repomap.tags.cache.v1/` via `diskcache`, keyed by filename with mtime invalidation.

**Tree-sitter queries** live in `queries/tree-sitter-language-pack/` and `queries/tree-sitter-languages/` — two sets of `.scm` files for different grammar packages.

## Key Types

- `Tag = namedtuple("Tag", "rel_fname fname line name kind")` — defined in both `utils.py` and `repomap_class.py` (duplicated)
- `FileReport` dataclass in `repomap_class.py` — tracks excluded files, definition/reference counts
- `RepoMap.__init__` accepts dependency-injected functions: `token_counter_func`, `file_reader_func`, `output_handler_funcs`

## Python Version

Requires Python >= 3.13 (per pyproject.toml).
