# Sift

**SQL-powered CLI for text processing and agent memory.**

Sift combines SQLite FTS5 full-text search with powerful text transformation capabilities, designed for AI agent workflows.

## Installation

Download the latest release for your platform:

### Linux (x86_64)
```bash
curl -LO https://github.com/edwardedmonds/sift-releases/releases/latest/download/sift-linux-x86_64
chmod +x sift-linux-x86_64
sudo mv sift-linux-x86_64 /usr/local/bin/sift
```

### macOS (Apple Silicon)
```bash
curl -LO https://github.com/edwardedmonds/sift-releases/releases/latest/download/sift-darwin-arm64
chmod +x sift-darwin-arm64
sudo mv sift-darwin-arm64 /usr/local/bin/sift
```

### macOS (Intel)
```bash
curl -LO https://github.com/edwardedmonds/sift-releases/releases/latest/download/sift-darwin-x86_64
chmod +x sift-darwin-x86_64
sudo mv sift-darwin-x86_64 /usr/local/bin/sift
```

## Claude Code Integration

Add sift as an MCP server to Claude Code:

```bash
claude mcp add sift -- sift --mcp
```

This enables Claude Code to use sift's powerful tools:
- `sift_search` - FTS5 full-text search (30-195x faster than grep)
- `sift_read` - Read files with line numbers
- `sift_edit` - Find/replace with fuzzy whitespace matching
- `sift_memory_*` - Persistent agent memory across sessions

## Features

- **FTS5 Full-Text Search**: Boolean queries (AND, OR, NOT, NEAR), 30-195x faster than grep
- **SQL Text Processing**: Transform text with regex_replace, csv_field, base64 functions
- **Workspace Indexing**: Persistent file index with incremental updates
- **MCP Server**: Model Context Protocol support for AI agent integration
- **Agent Memory**: Persistent queryable memory for AI workflows

## Quick Start

```bash
# Search indexed workspace
sift --quarry "malloc AND free" --files "*.c"

# SQL text transformation
echo "hello world" | sift --for "SELECT upper(content) FROM lines"

# MCP server mode (for AI agents)
sift --mcp
```

## Verify Download

Each release includes `checksums.txt` with SHA256 hashes:
```bash
sha256sum -c checksums.txt
```

## License

Proprietary. Binary distribution only.
