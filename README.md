# Sift

**SQL-powered MCP server for Claude Code.**

Sift gives Claude Code persistent memory, fast search, and intelligent file editing.

## The Problem

Every time you start a new Claude Code session, Claude forgets everything. Your preferences, the patterns you've established, the decisions you've made together, the gotchas you've discovered—all gone. You end up repeating yourself, re-explaining your codebase, and watching Claude make the same mistakes you corrected yesterday.

## The Solution

Sift gives Claude a persistent memory that survives across sessions. When you tell Claude "I prefer spaces over tabs" or "always use early returns", that preference is stored and automatically loaded in future sessions. When Claude learns that a particular API is flaky or that a certain pattern causes bugs in your codebase, it remembers.

But this isn't just a key-value store. Sift's memory is **queryable**. Claude can search its memories using full-text search with boolean operators. It can find related memories, track dependencies between tasks, and even detect when new information conflicts with existing knowledge.

The memory system supports different types of knowledge:
- **Patterns**: Coding conventions and workflows ("use sift_read before sift_edit")
- **Preferences**: Your personal style choices ("prefer descriptive variable names")
- **Plans**: Multi-step implementation strategies with tracked decisions
- **Tasks**: Work items with status, priority, and dependencies
- **Gotchas**: Hard-won lessons about what doesn't work

Claude can also **reflect** on its work—recording why it chose one approach over another, noting observations about your codebase, and logging corrections when you point out mistakes. These reflections become searchable knowledge that improves future sessions.

## Quick Start

### 1. Download

**Linux (x86_64)**
```bash
curl -LO https://github.com/edwardedmonds/sift-releases/releases/latest/download/sift-linux-x86_64
chmod +x sift-linux-x86_64
sudo mv sift-linux-x86_64 /usr/local/bin/sift
```

**macOS (Apple Silicon)**
```bash
curl -LO https://github.com/edwardedmonds/sift-releases/releases/latest/download/sift-darwin-arm64
chmod +x sift-darwin-arm64
sudo mv sift-darwin-arm64 /usr/local/bin/sift
```

**macOS (Intel)**
```bash
curl -LO https://github.com/edwardedmonds/sift-releases/releases/latest/download/sift-darwin-x86_64
chmod +x sift-darwin-x86_64
sudo mv sift-darwin-x86_64 /usr/local/bin/sift
```

### 2. Add to Claude Code

Add sift globally (recommended):
```bash
claude mcp add --scope user sift -- sift --mcp
```

Or add to a specific project only:
```bash
cd your-project
claude mcp add --scope project sift -- sift --mcp
```

**Note:** Sift automatically creates a separate `.sift/` database in each project directory, so your memories and search indexes are always project-specific even when sift is installed globally.

### 3. Try It Out

Create a new directory and start Claude Code:

```bash
mkdir test-sift && cd test-sift
claude
```

**Test the memory system:**
```
Remember that I prefer early returns over nested if statements
```

```
Remember that our API rate limits to 100 requests per minute
```

Start a new session and ask:
```
What do you know about my preferences and our API?
```

**Test search and editing:**
```
Create a few Python files with different functions, then search for all function definitions
```

```
Create a config.json file with some settings, then update the timeout value to 30
```

## Tools

### Memory

<details>
<summary><b>sift_memory_add</b> — Store patterns, preferences, plans, tasks, or gotchas</summary>

Creates a new memory entry that persists across sessions. Memories are automatically tagged, indexed for search, and checked for conflicts with existing knowledge.

**Types:**
- `pattern` — Coding conventions and workflows
- `preference` — Personal style choices
- `plan` — Multi-step implementation strategies
- `task` — Work items with status tracking
- `gotcha` — Lessons learned, things to avoid

**Example prompt:**
```
Remember that in this project we use snake_case for Python and camelCase for JavaScript
```
</details>

<details>
<summary><b>sift_memory_search</b> — Full-text search across all memories</summary>

Searches memories using FTS5 full-text search with synonym expansion and relevance scoring. Supports boolean operators (AND, OR, NOT) and phrase queries.

**Example prompt:**
```
What do you remember about our authentication system?
```
</details>

<details>
<summary><b>sift_memory_list</b> — List memories by type or status</summary>

Returns memories filtered by type, status, or parent. Useful for seeing all active tasks, all patterns, or steps under a specific plan.

**Example prompt:**
```
Show me all the gotchas you've learned about this codebase
```
</details>

<details>
<summary><b>sift_memory_update</b> — Update memory status, priority, or content</summary>

Modifies an existing memory's status (open, in_progress, done), priority, title, or description.

**Example prompt:**
```
Mark the authentication refactor task as complete
```
</details>

<details>
<summary><b>sift_memory_delete</b> — Remove a memory</summary>

Permanently deletes a memory and its associated decisions and links. Use for outdated or incorrect information.

**Example prompt:**
```
Forget what you know about the old API endpoint format, we've changed it
```
</details>

<details>
<summary><b>sift_memory_decide</b> — Record a decision for a plan</summary>

Stores a decision with its rationale, linked to a specific plan. Decisions are queryable later so you can understand why choices were made.

**Example prompt:**
```
We decided to use PostgreSQL instead of MongoDB because we need ACID transactions
```
</details>

<details>
<summary><b>sift_memory_decisions</b> — Query past decisions</summary>

Search through recorded decisions by plan or keyword. Helps recall why past architectural choices were made.

**Example prompt:**
```
What decisions have we made about the database?
```
</details>

<details>
<summary><b>sift_memory_reflect</b> — Log reasoning, observations, or corrections</summary>

Records Claude's thinking process, observations about the codebase, or lessons from corrections. Types: `reasoning`, `observation`, `correction`.

**Example prompt:**
```
That's not right - we use UTC timestamps, not local time. Remember that.
```
</details>

<details>
<summary><b>sift_memory_reflections</b> — Search past reflections</summary>

Query Claude's logged reflections to understand past reasoning or find patterns in corrections.

**Example prompt:**
```
What corrections have you had to make in this project?
```
</details>

<details>
<summary><b>sift_memory_link</b> — Create dependencies between memories</summary>

Establishes relationships between memories: `blocks` (task A blocks task B), `related`, or `parent`.

**Example prompt:**
```
The database migration needs to be done before we can deploy the new API
```
</details>

<details>
<summary><b>sift_memory_deps</b> — Query memory dependencies</summary>

Shows what blocks a task or what a task is blocking. Useful for understanding work dependencies.

**Example prompt:**
```
What's blocking the deployment task?
```
</details>

<details>
<summary><b>sift_memory_ready</b> — Find tasks with no blockers</summary>

Returns tasks that have no unfinished dependencies and are ready to work on.

**Example prompt:**
```
What tasks can I work on right now?
```
</details>

<details>
<summary><b>sift_memory_stale</b> — Find old memories that may need review</summary>

Lists memories that haven't been accessed in a specified number of days. Useful for cleanup and review.

**Example prompt:**
```
Are there any old memories we should review or clean up?
```
</details>

<details>
<summary><b>sift_memory_stats</b> — Get memory database statistics</summary>

Shows counts by type and status, active patterns, recent corrections, and database health. Called at session start to load context.

**Example prompt:**
```
Give me an overview of what you remember about this project
```
</details>

<details>
<summary><b>sift_memory_config</b> — View ranking weight configuration</summary>

Shows current search ranking weights for frequency, recency, priority, and context boosting.

**Example prompt:**
```
How are memory search results being ranked?
```
</details>

<details>
<summary><b>sift_memory_tune</b> — Adjust search ranking weights</summary>

Modifies how memories are ranked in search results. Requires a rationale for transparency.

**Example prompt:**
```
Prioritize more recent memories in search results
```
</details>

<details>
<summary><b>sift_memory_backups</b> — List memory database backups</summary>

Shows available backups of the memory database, created automatically each session.

**Example prompt:**
```
Are there any memory backups available?
```
</details>

<details>
<summary><b>sift_memory_restore</b> — Restore from a backup</summary>

Restores the memory database from a previous backup. Use if data was accidentally deleted.

**Example prompt:**
```
Restore the memory database from yesterday's backup
```
</details>

<details>
<summary><b>sift_memory_import</b> — Import markdown plans into memory</summary>

Converts a markdown file into a memory entry. Useful for migrating existing documentation.

**Example prompt:**
```
Import the ARCHITECTURE.md file as a plan memory
```
</details>

### Search & Edit

<details>
<summary><b>sift_search</b> — FTS5 full-text search (30-195x faster than grep)</summary>

Searches the indexed workspace using SQLite FTS5. Supports boolean queries (AND, OR, NOT, NEAR), prefix matching, and file filtering. Auto-initializes the index on first use.

**Example prompt:**
```
Find all files that mention both "authentication" and "token"
```
</details>

<details>
<summary><b>sift_read</b> — Read files with line numbers</summary>

Reads file contents with line numbers for accurate editing. Supports partial reads with start/end lines and whitespace visualization for debugging.

**Example prompt:**
```
Show me the handleAuth function in auth.js
```
</details>

<details>
<summary><b>sift_edit</b> — Find/replace with fuzzy whitespace matching</summary>

Performs find/replace operations with automatic whitespace normalization. Supports insert, delete, and patch modes. Shows visual diffs of changes.

**Example prompt:**
```
Replace the hardcoded timeout of 5000 with a constant TIMEOUT_MS
```
</details>

<details>
<summary><b>sift_update</b> — Simple old_string/new_string replacement</summary>

Straightforward text replacement with clear error messages. Fails if the target string isn't found or isn't unique.

**Example prompt:**
```
Change the function name from processData to processUserData
```
</details>

<details>
<summary><b>sift_write</b> — Create or overwrite files</summary>

Creates new files or completely replaces existing file contents. Creates parent directories if needed.

**Example prompt:**
```
Create a new utils.js file with helper functions for date formatting
```
</details>

<details>
<summary><b>sift_batch</b> — Multiple edit operations atomically</summary>

Executes multiple edits as a single atomic operation. All succeed or all fail, preventing partial updates.

**Example prompt:**
```
Rename the "user" variable to "currentUser" in all three files
```
</details>

<details>
<summary><b>sift_transform</b> — SQL-based file transformation</summary>

Transforms file contents using SQL queries with regex_replace, string functions, and more.

**Example prompt:**
```
Convert all console.log statements to use our logger instead
```
</details>

<details>
<summary><b>sift_sql</b> — Run SQL on text input</summary>

Executes SQL queries on piped text input. Useful for filtering, transforming, or analyzing text data.

**Example prompt:**
```
Parse this CSV and show me only rows where the status is "failed"
```
</details>

<details>
<summary><b>sift_workspace</b> — Manage the search index</summary>

Controls the workspace index: init, status, refresh, or rebuild. The index auto-initializes on first search.

**Example prompt:**
```
Refresh the search index to pick up recent file changes
```
</details>

### Web & Repository

<details>
<summary><b>sift_web_crawl</b> — Crawl and index a website</summary>

Downloads and indexes a website for offline searching. Respects robots.txt, follows links, and deduplicates content. Great for documentation sites.

**Example prompt:**
```
Index the React documentation so you can reference it offline
```
</details>

<details>
<summary><b>sift_web_search</b> — Search indexed web content</summary>

Full-text search across crawled websites. Supports boolean operators and returns relevant snippets.

**Example prompt:**
```
Search the React docs for information about useEffect cleanup
```
</details>

<details>
<summary><b>sift_web_query</b> — SQL queries on web content</summary>

Run SQL queries directly on the web content database for advanced filtering and analysis.

**Example prompt:**
```
Find all pages in the docs that mention "deprecated"
```
</details>

<details>
<summary><b>sift_web_stats</b> — Web database statistics</summary>

Shows page count, word count, domains indexed, and crawl timestamps for a web database.

**Example prompt:**
```
How much documentation do you have indexed?
```
</details>

<details>
<summary><b>sift_web_refresh</b> — Update stale cached pages</summary>

Re-fetches pages that are older than a specified age. Keeps documentation caches current.

**Example prompt:**
```
Update any cached documentation pages older than a week
```
</details>

<details>
<summary><b>sift_repo_clone</b> — Clone and index a git repository</summary>

Clones a git repository and indexes its source code into a searchable database. Great for exploring unfamiliar codebases.

**Example prompt:**
```
Clone and index the lodash repository so we can study their implementation
```
</details>

<details>
<summary><b>sift_repo_search</b> — Search indexed repository</summary>

Full-text search across an indexed repository. Filter by language or file pattern.

**Example prompt:**
```
Search the lodash repo for debounce implementation
```
</details>

<details>
<summary><b>sift_repo_query</b> — SQL queries on repository content</summary>

Run SQL queries on indexed repository files. Query by language, line count, file path patterns.

**Example prompt:**
```
Find the largest files in the indexed repository
```
</details>

## Verify Download

Each release includes `checksums.txt` with SHA256 hashes:
```bash
sha256sum -c checksums.txt
```

## License

Proprietary. Binary distribution only.
