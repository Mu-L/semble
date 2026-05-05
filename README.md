
<h2 align="center">
  <img width="30%" alt="semble logo" src="https://raw.githubusercontent.com/MinishLab/semble/main/assets/images/semble_logo.png"><br/>
  Fast and Accurate Code Search for Agents<br/>
  <sub>Uses ~98% fewer tokens than grep+read</sub>
</h2>

<div align="center">
  <h2>
    <a href="https://pypi.org/project/semble/"><img src="https://img.shields.io/pypi/v/semble?color=%23007ec6&label=pypi%20package" alt="Package version"></a>
    <a href="https://app.codecov.io/gh/MinishLab/semble">
      <img src="https://codecov.io/gh/MinishLab/semble/graph/badge.svg?token=SZKRFKPPCG" alt="Codecov">
    </a>
    <a href="https://github.com/MinishLab/semble/blob/main/LICENSE">
      <img src="https://img.shields.io/badge/license-MIT-green" alt="License - MIT">
    </a>
  </h2>

[Quickstart](#quickstart) •
[When to use Semble](#when-to-use-semble) •
[MCP Server](#mcp-server) •
[Bash / AGENTS.md](#bash-integration) •
[CLI](#cli) •
[Python API](#python-api) •
[Benchmarks](#benchmarks)

</div>

Semble is a local code search engine built for coding agents. Instead of grep + read loops, agents can ask natural-language, symbol, or identifier queries and get back the most relevant code chunks with file paths and line numbers. This keeps context small, reduces latency, and avoids sending large irrelevant files to the model. Semble runs on CPU with no API keys, no GPU, and no external service. It can be used as an [MCP server](#mcp-server), via [Bash / AGENTS.md](#bash-integration), or as a [Python library](#python-api).

**At a glance**

| Capability | Result |
|---|---:|
| Token usage vs. grep + read | ~98% fewer tokens |
| Index time | ~263 ms |
| Query latency, p50 | ~1.5 ms |
| Retrieval quality | 0.854 NDCG@10 |
| Deployment | Local CPU, no API keys |

See [benchmarks](#benchmarks) for methodology and comparisons.

## Quickstart

Your agent will automatically use Semble whenever it needs to find code. Instead of grepping with a keyword and reading full files, it queries in natural language (e.g. `"How is authentication handled?"`) and gets back only the relevant context. Semble can be set up as an MCP server or as a bash tool:

### MCP

Add Semble to Claude Code (requires [uv](https://docs.astral.sh/uv/getting-started/installation/)):

```bash
claude mcp add semble -s user -- uvx --from "semble[mcp]" semble
```

Using another agent harness? See [MCP Server](#mcp-server) for setup instructions for Codex, OpenCode, Cursor, and other MCP clients.

### Bash / AGENTS.md

Install Semble first, then add the [code search snippet](#bash-integration) to your `AGENTS.md` or `CLAUDE.md`:

```bash
pip install semble  # Install with pip
uv add semble       # Or install with uv
```

> Note: for Claude Code or Codex CLI sub-agents, use the [Bash integration](#bash-integration) instead of, or alongside, MCP.

To update Semble, see [Updating](#updating).

## When to use Semble

Use Semble when:
- an agent needs to understand a codebase without repeatedly grepping and reading files;
- you want semantic code search without a hosted embedding service;
- you need both natural-language search and exact symbol or identifier matching;
- you want an MCP-compatible code search tool for Claude Code, Cursor, Codex, OpenCode, or another agent;
- you want fast local search over a repo that can be cloned and indexed on demand.

Use grep or ripgrep when you need exhaustive literal matching or want to confirm that an exact string appears somewhere.

## Main Features

- **Agent-first search**: built for coding agents that need relevant code chunks, not whole files.
- **Hybrid retrieval**: combines semantic search with lexical matching for symbols, identifiers, and API names.
- **Fast local indexing**: indexes benchmark repositories in ~263 ms and answers queries in ~1.5 ms p50, all on CPU.
- **Token-efficient context**: returns focused chunks and uses ~98% fewer tokens than grep + read.
- **No external service**: runs locally on CPU with no API keys, GPU, or hosted vector database.
- **Flexible input**: search a local path or a git URL; remote repositories are cloned and indexed on demand.
- **Multiple interfaces**: use Semble through MCP, the CLI, or the Python API.

## MCP Server

Semble can run as an MCP server so coding agents can search repositories directly. Pass a local path or git URL; remote repositories are cloned and indexed on demand, and indexes are cached for the lifetime of the session. Local paths are watched for file changes and re-indexed automatically.

### Setup

> Requires [uv](https://docs.astral.sh/uv/getting-started/installation/) to be installed.

#### Claude Code
```bash
claude mcp add semble -s user -- uvx --from "semble[mcp]" semble
```

#### Codex
Add to `~/.codex/config.toml`:
```toml
[mcp_servers.semble]
command = "uvx"
args = ["--from", "semble[mcp]", "semble"]
```

#### OpenCode
Add to `~/.opencode/config.json`:
```json
{
  "mcp": {
    "semble": {
      "type": "local",
      "command": ["uvx", "--from", "semble[mcp]", "semble"]
    }
  }
}
```

#### Cursor
Add to `~/.cursor/mcp.json` (or `.cursor/mcp.json` in your project):
```json
{
  "mcpServers": {
    "semble": {
      "command": "uvx",
      "args": ["--from", "semble[mcp]", "semble"]
    }
  }
}
```

### Tools

| Tool | Description |
|---|---|
| `search` | Search a repository with a natural-language, symbol, identifier, or code query. Pass `repo` as a git URL or local path. |
| `find_related` | Given a file path and line number from a previous result, return semantically similar chunks from the same repository. |


## Bash integration

An alternative to MCP is to invoke Semble via Bash. For Claude Code and Codex CLI, this is the only option for sub-agents, which cannot call MCP tools directly (both lazy-load MCP schemas at the top-level agent only).

To add Bash support, append the following to your `AGENTS.md` or `CLAUDE.md`:

```markdown
## Code Search

Use `semble search` to find code by describing what it does or naming a symbol/identifier, instead of grep:

​```bash
semble search "authentication flow" ./my-project
semble search "save_pretrained" ./my-project
semble search "save model to disk" ./my-project --top-k 10
​```

Use `semble find-related` to discover code similar to a known location (pass `file_path` and `line` from a prior search result):

​```bash
semble find-related src/auth.py 42 ./my-project
​```

`path` defaults to the current directory when omitted; git URLs are accepted.

If `semble` is not on `$PATH`, use `uvx --from "semble[mcp]" semble` in its place.

## Workflow

1. Start with `semble search` to find relevant chunks.
2. Inspect full files only when the returned chunk is not enough context.
3. Optionally use `semble find-related` with a promising result's `file_path` and `line` to discover related implementations.
4. Use grep only when you need exhaustive literal matches or quick confirmation of an exact string.
```

**Claude Code sub-agent**: Claude Code also supports a dedicated sub-agent. Run this once in your project root:

```bash
semble init
# or, if semble is not on $PATH:
uvx --from "semble[mcp]" semble init
```

This writes [`.claude/agents/semble-search.md`](src/semble/agents/semble-search.md).

## CLI

Semble also ships as a standalone CLI for scripts, sub-agents, and local terminal use.

```bash
# Search the current directory
semble search "authentication flow"

# Search a local repository
semble search "save_pretrained" ./my-project

# Search a remote repository, cloned on demand
semble search "save model to disk" https://github.com/MinishLab/model2vec

# Return more results
semble search "save model to disk" ./my-project --top-k 10

# Find code similar to a known location
semble find-related src/auth.py 42 ./my-project
```

`path` defaults to the current directory when omitted. Git URLs are accepted.

If `semble` is not on `$PATH`, use:

```bash
uvx --from "semble[mcp]" semble search "authentication flow"
```

### Updating

To update/upgrade Semble to the latest version:

```bash
pip install --upgrade semble   # with pip
uv add semble --upgrade        # with uv
uv cache clean semble          # for MCP users (restart your MCP client after)
```

## Python API

Semble can also be used as a Python library for programmatic access, useful when building custom tooling or integrating search directly into your own code.

```python
from semble import SembleIndex

# Index a local directory
index = SembleIndex.from_path("./my-project")

# Index a remote git repository
index = SembleIndex.from_git("https://github.com/MinishLab/model2vec")

# Search with a natural-language, symbol, or code query
results = index.search("save model to disk", top_k=3)

# Find code similar to a specific result
related = index.find_related(results[0], top_k=3)

# Each result exposes the matched chunk
result = results[0]
result.chunk.file_path   # "model2vec/model.py"
result.chunk.start_line  # 127
result.chunk.end_line    # 150
result.chunk.content     # "def save_pretrained(self, path: PathLike, ..."
```

## How it works

Semble uses a hybrid retrieval pipeline designed for code:

1. **Chunk code structurally.** Files are split into code-aware chunks using [Chonkie](https://github.com/chonkie-inc/chonkie).
2. **Score chunks semantically.** Static [Model2Vec](https://github.com/MinishLab/model2vec) embeddings from [potion-code-16M](https://huggingface.co/minishlab/potion-code-16M) capture behavior-level and natural-language similarity.
3. **Score chunks lexically.** [BM25](https://github.com/xhluca/bm25s) captures exact symbols, identifiers, API names, and other literal matches.
4. **Fuse rankings.** Semantic and lexical rankings are combined with Reciprocal Rank Fusion (RRF).
5. **Rerank for code.** Results are adjusted with code-aware signals that push canonical implementations above incidental matches.

<details>
<summary><b>Ranking signals</b></summary>

- **Adaptive weighting**: symbol-like queries (`Foo::bar`, `_private`, `getUserById`) get more lexical weight, while natural-language queries stay balanced between semantic and lexical retrieval.
- **Definition boosts**: chunks defining the queried symbol (a `class`, `def`, `func`, etc.) rank above chunks that only reference it.
- **Identifier stems**: query tokens are matched against identifier stems, so `parse config` can boost chunks containing `parseConfig`, `ConfigParser`, or `config_parser`.
- **File coherence**: multiple matching chunks from the same file boost that file, helping broad file-level relevance beat isolated matches.
- **Noise penalties**: tests, examples, compatibility shims, legacy code, and `.d.ts` declaration stubs are down-ranked so canonical implementations surface first.

</details>

Because the embedding model is static, Semble does not run a transformer forward pass at query time. Queries stay fast on CPU.

## Benchmarks

We benchmark quality and speed on ~1,250 queries across 63 repositories in 19 languages. The plot compares total cold-start latency, including indexing and the first query, against NDCG@10 retrieval quality. Marker size reflects model parameter count.

![Speed vs quality](https://raw.githubusercontent.com/MinishLab/semble/main/assets/images/speed_vs_ndcg_cold.png)

| Method | NDCG@10 | Index time | Query p50 |
|--------|--------:|-----------:|----------:|
| CodeRankEmbed Hybrid | 0.862 | 57 s | 16 ms |
| **semble** | **0.854** | **263 ms** | **1.5 ms** |
| CodeRankEmbed | 0.765 | 57 s | 16 ms |
| ColGREP | 0.693 | 5.8 s | 124 ms |
| BM25 | 0.673 | 263 ms | 0.02 ms |
| grepai | 0.561 | 35 s | 48 ms |
| probe | 0.387 | — | 207 ms |
| ripgrep | 0.126 | — | 12 ms |

Semble reaches 0.854 NDCG@10, close to the 0.862 NDCG@10 of the 137M-parameter [CodeRankEmbed](https://huggingface.co/nomic-ai/CodeRankEmbed) Hybrid, while indexing 218x faster and answering queries 11x faster. See [benchmarks](benchmarks/README.md) for per-language results, ablations, and methodology.

### Token efficiency

Agents using grep + read often spend most of their context budget on irrelevant code. Semble returns focused chunks instead of whole files, keeping token usage low while preserving recall.

![Token efficiency: recall vs. retrieved tokens](https://raw.githubusercontent.com/MinishLab/semble/main/assets/images/token_efficiency.png)

In our benchmark, Semble uses **~98% fewer tokens** on average. It reaches 94% recall with a 2k-token budget, while grep + read needs a 100k-token context window to reach 85% recall. See [benchmarks](benchmarks/README.md#token-efficiency) for details.

## License

MIT

## Citing

If you use Semble in your research, please cite the following:

```bibtex
@software{minishlab2026semble,
  author       = {{van Dongen}, Thomas and Stephan Tulkens},
  title        = {Semble: Fast and Accurate Code Search for Agents},
  year         = {2026},
  publisher    = {Zenodo},
  doi          = {10.5281/zenodo.19785932},
  url          = {https://github.com/MinishLab/semble},
  license      = {MIT}
}
```
