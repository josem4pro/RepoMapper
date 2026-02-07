# CLAUDE.md - RepoMapper

## Description

Python CLI tool and MCP server that generates navigational "maps" of repositories for LLM consumption. Parses source code with Tree-sitter, extracts definitions/references, builds a directed graph, applies PageRank to rank code elements by importance. Based on Aider's RepoMap design. Fork of pdavis68/RepoMapper.

## Commands

```bash
# Install dependencies
pip install -r requirements.txt
# Or with uv:
uv sync

# CLI usage
python repomap.py .                              # Map current directory
python repomap.py . --map-tokens 2048            # Custom token limit
python repomap.py --chat-files main.py --other-files src/  # Prioritized files
python repomap.py . --verbose --force-refresh    # Debug with cache refresh

# MCP server (STDIO-based FastMCP)
python repomap_server.py
```

No test suite exists.

## Key Metrics

| Metric | Value |
|--------|-------|
| Python LOC | 1,299 (6 modules) |
| Total LOC | ~5,000 (including .scm queries) |
| Python version | >= 3.13 |
| Tree-sitter queries | 49 .scm files (2 sets) |
| Commits | 26, 7 contributors |
| License | MIT |

## Architecture

Six Python modules at root level (no packages):

| Module | LOC | Responsibility |
|--------|-----|----------------|
| `repomap.py` | 229 | [ENTRY] CLI: args, file discovery, output |
| `repomap_class.py` | 616 | [CORE] RepoMap class: parsing, graph, PageRank, rendering |
| `repomap_server.py` | 279 | FastMCP server: `repo_map` + `search_identifiers` tools |
| `utils.py` | 58 | Token counting (tiktoken), file reading, Tag namedtuple |
| `scm.py` | 59 | Language-to-.scm query file mapping |
| `importance.py` | 58 | Important file detection (README, Dockerfile, etc.) |

### Data Flow

```
Files → Tree-sitter parsing → Tag namedtuples (def/ref, file, line, name)
  → networkx.MultiDiGraph (file→file edges via shared symbols)
  → PageRank (chat_files get 100x weight)
  → to_tree() renders via grep_ast.TreeContext
  → Binary search fits output within max_map_tokens
```

### Ranking Weights

- Chat files: 20x personalization
- Mentioned identifiers: 10x boost
- Mentioned files: 5x boost
- Important files (README, etc.): additional boost via importance.py

### Caching

Tags cached in `.repomap.tags.cache.v1/` via `diskcache`, keyed by filename with mtime invalidation.

### Tree-sitter Queries

Two query sets in `queries/`:
- `tree-sitter-language-pack/` (27 .scm files)
- `tree-sitter-languages/` (21 .scm files)

## Dependencies

networkx, tree-sitter, tree-sitter-language-pack, tree-sitter-languages, grep-ast, tiktoken, diskcache, fastmcp

## Known Issues

| Severity | Issue |
|----------|-------|
| Medium | Tag namedtuple duplicated in utils.py and repomap_class.py |
| Medium | No test suite |
| Low | Python >= 3.13 is a strict requirement (limits compatibility) |
| Low | No CI/CD or pre-commit hooks |

## Relevance to Workstation IA

This is the upstream project for Jose's `repo-mapper` agent (`~/.claude/agents/repo-mapper.md`). The agent uses RepoMapper's analysis methodology but is adapted to work within the Claude Code agent framework.
