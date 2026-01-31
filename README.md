# Sift

**Persistent memory for Claude Code.**

> [!WARNING]
> **Alpha Release** — Sift is under active development. APIs and features may change. Feedback welcome.

Sift gives Claude Code persistent memory, fast search, and intelligent file editing.

## Installation

### Quick Install (Recommended)

Interactive installer that prompts for each step:

```bash
curl -fsSL https://github.com/deths74r/sift-releases/releases/latest/download/sift-setup.py | python3
```

The installer will prompt before each step:
1. Install binary to `~/.local/bin`
2. Install documentation templates to `~/.claude/`
3. Register sift as MCP server with Claude Code
4. Configure session hooks (context preservation, workspace indexing)
5. Disable built-in TodoWrite (memory_* is better)

Existing hooks are preserved—the installer merges new hooks with existing configuration.

To uninstall:
```bash
python3 ~/.local/bin/sift-uninstall.py
```

### Manual Install

**Step 1: Download**

| Platform | Command |
|----------|--------|
| Linux x86_64 | `curl -LO https://github.com/deths74r/sift-releases/releases/latest/download/sift-linux-x86_64` |
| macOS Apple Silicon | `curl -LO https://github.com/deths74r/sift-releases/releases/latest/download/sift-darwin-arm64` |
| macOS Intel | `curl -LO https://github.com/deths74r/sift-releases/releases/latest/download/sift-darwin-x86_64` |

**Step 2: Install**
```bash
chmod +x sift-*
mkdir -p ~/.local/bin
mv sift-* ~/.local/bin/sift
```

**Step 3: Add to Claude Code**
```bash
claude mcp add --scope user sift -- sift --mcp
```

**Step 4: Verify**
```bash
~/.local/bin/sift --version
```

> **Note:** Add `~/.local/bin` to your PATH if not already. Sift creates a `.sift/` database in each project directory, so memories and indexes are project-specific.

### Try It Out

Start Claude Code in any directory:
```bash
claude
```

**Test memory:**
```
Remember that I prefer early returns over nested if statements
```

Start a new session and ask:
```
What do you know about my preferences?
```

## The Problem

Every time you start a new Claude Code session, Claude forgets everything. Your preferences, the patterns you've established, the decisions you've made together, the gotchas you've discovered—all gone. You end up repeating yourself, re-explaining your codebase, and watching Claude make the same mistakes you corrected yesterday.

## The Solution

Sift gives Claude a persistent memory that survives across sessions. When you tell Claude "I prefer spaces over tabs" or "always use early returns", that preference is stored and automatically loaded in future sessions. When Claude learns that a particular API is flaky or that a certain pattern causes bugs in your codebase, it remembers.

But this isn't just a key-value store. Sift's memory is **queryable**. Claude can search its memories using full-text search with boolean operators. It can find related memories, track dependencies between tasks, and even detect when new information conflicts with existing knowledge.

The memory system supports different types of knowledge:
- **Patterns**: Coding conventions and workflows ("use read before edit")
- **Preferences**: Your personal style choices ("prefer descriptive variable names")
- **Plans**: Multi-step implementation strategies with tracked decisions
- **Tasks**: Work items with status, priority, and dependencies
- **Gotchas**: Hard-won lessons about what doesn't work

Claude can also **reflect** on its work—recording why it chose one approach over another, noting observations about your codebase, and logging corrections when you point out mistakes. These reflections become searchable knowledge that improves future sessions.

## Why Sift Matters: Grounding in Reality

Large language models hallucinate. Claude might confidently report that a function is on line 50 when it's actually on line 200. It might "remember" that you prefer tabs when you actually said spaces. It might fabricate an API endpoint that doesn't exist in your codebase.

Sift's design directly addresses this problem.

When Claude searches your codebase with `search`, it gets back real results from an indexed source of truth—not a vague recollection that might be wrong. The results include exact file paths, exact line numbers, and exact content. There's no room to confabulate.

When Claude reads a file with `read`, it sees actual content with line numbers. When it edits with `edit`, the tool returns an error if the target text doesn't exist or isn't unique. These aren't polite suggestions—they're hard failures that force confrontation with reality rather than proceeding on false assumptions.

The memory system works the same way. When you tell Claude you prefer early returns, that preference is stored as a structured record that can be queried later—not as a fuzzy impression that might drift over time. When Claude is unsure what you said about your authentication system, it can run `memory_search` and get back exactly what was recorded, not what it thinks it remembers.

This is the difference between asking someone to recall a conversation from last week versus looking at a transcript. The transcript might be less convenient, but it's accurate.

Sift doesn't make Claude incapable of mistakes, but it provides a way to verify. Every search result, every memory query, every file read produces concrete, auditable output. When Claude says "I found 5 files matching your query," you can see those 5 files. When it says "you told me to always use UTC timestamps," there's a record you can check.

The machine-readable nature of these tools creates accountability.
## See It In Action
These documents were written by Claude using Sift's memory system:
- **[The History of Sift](docs/history-of-sift.md)** — Claude traces how the project evolved, drawing on 85 connected memories to reconstruct the narrative from origin to present.
- **[Memory Architecture Demo](docs/memory-architecture-demo.md)** — An analysis of which memory subsystems (chain traversal, network analysis, search, decisions, reflections) contributed to writing the history document.
The history document demonstrates both thinking modes working together: linear chain traversal for chronological narrative, and network analysis for associative insights like identifying which memories are central hubs.
## Data Storage

Sift stores data in two locations:

| Location | Purpose |
|----------|---------|
| `.sift/` (per-project) | Memory database, search index, context database |
| `~/.sift/backups/<hash>/` (global) | Automatic memory database backups |

**Per-project storage** (`.sift/`) is created in each project directory when you use memory or search tools. This keeps project knowledge isolated.

**Global backups** (`~/.sift/backups/`) are created automatically on first memory tool access each session. Backups are identified by a hash of the project path, so they survive even if a project's `.sift/` directory is deleted. This provides recovery options if data is accidentally lost.

## Tools

### Memory

[memory_add](#memory_add) · [memory_get](#memory_get) · [memory_search](#memory_search) · [memory_list](#memory_list) · [memory_update](#memory_update) · [memory_archive](#memory_archive) · [memory_synthesize](#memory_synthesize) · [memory_expand](#memory_expand) · [memory_decide](#memory_decide) · [memory_decisions](#memory_decisions) · [memory_supersede](#memory_supersede) · [memory_reflect](#memory_reflect) · [memory_reflections](#memory_reflections) · [memory_reflect_trajectory](#memory_reflect_trajectory) · [memory_trajectory_reflections](#memory_trajectory_reflections) · [memory_link](#memory_link) · [memory_unlink](#memory_unlink) · [memory_deps](#memory_deps) · [memory_ready](#memory_ready) · [memory_stale](#memory_stale) · [memory_stats](#memory_stats) · [memory_context](#memory_context) · [memory_traverse](#memory_traverse) · [memory_origin](#memory_origin) · [memory_network](#memory_network) · [memory_instances](#memory_instances) · [memory_challenge](#memory_challenge) · [memory_challenge_evidence](#memory_challenge_evidence) · [memory_config](#memory_config) · [memory_tune](#memory_tune) · [memory_backups](#memory_backups) · [memory_restore](#memory_restore) · [memory_import](#memory_import) · [memory_prune](#memory_prune)

### Context

[context_session](#context_session) · [context_save](#context_save) · [context_search](#context_search) · [context_query](#context_query) · [context_link](#context_link) · [context_synthesize](#context_synthesize) · [context_archive](#context_archive) · [context_stats](#context_stats) · [context_stale](#context_stale) · [context_memory](#context_memory)

### Search & Edit

[search](#search) · [read](#read) · [edit](#edit) · [update](#update) · [write](#write) · [batch](#batch) · [transform](#transform) · [sql](#sql) · [workspace](#workspace)

### Web & Repository

[web_crawl](#web_crawl) · [web_search](#web_search) · [web_query](#web_query) · [web_stats](#web_stats) · [web_manifest](#web_manifest) · [web_refresh](#web_refresh) · [web_search_multi](#web_search_multi) · [web_merge](#web_merge) · [web_fetch](#web_fetch) · [repo_clone](#repo_clone) · [repo_search](#repo_search) · [repo_query](#repo_query) · [repo_stats](#repo_stats) · [repo_list](#repo_list)

### Fingerprint

[fingerprint_load](#fingerprint_load) · [fingerprint_generate](#fingerprint_generate) · [fingerprint_compare](#fingerprint_compare) · [fingerprint_drift](#fingerprint_drift)

### Hardware Awareness

[hardware_status](#hardware_status) · [hardware_patterns](#hardware_patterns) · [hardware_events](#hardware_events) · [budget_request](#budget_request) · [budget_stats](#budget_stats) · [memory_sqlite_config](#memory_sqlite_config) · [memory_cache_status](#memory_cache_status) · [stream_read](#stream_read) · [stream_close](#stream_close)


## Memory Tools

<a name="memory_add"></a>
### memory_add
Store patterns, preferences, plans, tasks, or gotchas.

Creates a new memory entry that persists across sessions. Memories are automatically tagged, indexed for search, and checked for conflicts with existing knowledge.

**Types:** `pattern` · `preference` · `plan` · `task` · `gotcha` · `note` · `step` · `synthesis` · `instance`

When creating a memory with the same title as an existing one, an `instance` memory is automatically created and linked to the canonical via `instance_of`, forming a 3D layered topology where each occurrence captures unique context.

```
Remember that in this project we use snake_case for Python and camelCase for JavaScript
```


<a name="memory_get"></a>
### memory_get
Get a memory by ID.

Returns full details of a specific memory including metadata, timestamps, and access count.

```
Show me the details of that authentication pattern you mentioned
```


<a name="memory_search"></a>
### memory_search
Full-text search across all memories.

Searches memories using FTS5 full-text search with synonym expansion and relevance scoring. Supports boolean operators (AND, OR, NOT) and phrase queries.

```
What do you remember about our authentication system?
```


<a name="memory_list"></a>
### memory_list
List memories by type or status.

Returns memories filtered by type, status, or parent. Useful for seeing all active tasks, all patterns, or steps under a specific plan.

```
Show me all the gotchas you've learned about this codebase
```


<a name="memory_update"></a>
### memory_update
Update memory status, priority, or content.

Modifies an existing memory's status (open, in_progress, done), priority, title, or description.

```
Mark the authentication refactor task as complete
```


<a name="memory_archive"></a>
### memory_archive
Archive a memory.

Sets a memory's status to 'archived' but preserves all links, decisions, and reflections. Archived memories are excluded from list by default but remain accessible and searchable. Use `include_archived: true` in list/search to see them.

```
Archive the old authentication notes, we've moved to a new system
```


<a name="memory_synthesize"></a>
### memory_synthesize
Consolidate multiple memories into one.

Creates a new synthesis memory linking to source memories. Sources are marked as synthesized but preserved. Use for distilling lessons from multiple gotchas or summarizing completed work.

```
Combine everything you learned about the database issues into a single summary
```


<a name="memory_expand"></a>
### memory_expand
Show sources of a synthesis.

Returns the synthesis memory plus all source memories that were consolidated. Use when you need the original details behind a synthesis.

```
Show me the original memories that went into that database summary
```


<a name="memory_decide"></a>
### memory_decide
Record a decision for a plan.

Stores a decision with its rationale, linked to a specific plan. Decisions are queryable later so you can understand why choices were made.

```
We decided to use PostgreSQL instead of MongoDB because we need ACID transactions
```


<a name="memory_decisions"></a>
### memory_decisions
Query past decisions.

Search through recorded decisions by plan or keyword. Helps recall why past architectural choices were made.

```
What decisions have we made about the database?
```

<a name="memory_supersede"></a>
### memory_supersede
Replace a previous decision.
Updates a decision with a new one while keeping history. The old decision is marked as superseded, creating an audit trail of how thinking evolved.
```
Actually, let's use Redis instead of PostgreSQL for the session store
```

<a name="memory_reflect"></a>
### memory_reflect
Log reasoning, observations, or corrections.

Records Claude's thinking process, observations about the codebase, or lessons from corrections. Types: `reasoning`, `observation`, `correction`.

```
That's not right - we use UTC timestamps, not local time. Remember that.
```


<a name="memory_reflections"></a>
### memory_reflections
Search past reflections.

Query Claude's logged reflections to understand past reasoning or find patterns in corrections.

```
What corrections have you had to make in this project?
```

<a name="memory_reflect_trajectory"></a>
### memory_reflect_trajectory
Reflect on a chain of memories.
Records insights about how work evolved over time—patterns across chains, arc summaries, pivot points, or friction analyses. Types: `trajectory_pattern`, `arc_summary`, `pivot_point`, `friction_analysis`.
```
Looking back at the authentication work, we pivoted from JWT to sessions mid-way through
```
<a name="memory_trajectory_reflections"></a>
### memory_trajectory_reflections
Query trajectory reflections.
Search through trajectory reflections to find patterns in how work evolved. Filter by reflection type or chain segment length.
```
What trajectory patterns have you noticed in this project?
```

<a name="memory_link"></a>
### memory_link
Create dependencies between memories.

Establishes relationships between memories: `blocks` (task A blocks task B), `related`, or `parent`.

```
The database migration needs to be done before we can deploy the new API
```


<a name="memory_unlink"></a>
### memory_unlink
Remove a dependency link.

Removes a previously created relationship between two memories.

```
The migration is done, so it's no longer blocking deployment
```


<a name="memory_deps"></a>
### memory_deps
Query memory dependencies.

Shows what blocks a task or what a task is blocking. Useful for understanding work dependencies.

```
What's blocking the deployment task?
```


<a name="memory_ready"></a>
### memory_ready
Find tasks with no blockers.

Returns tasks that have no unfinished dependencies and are ready to work on.

```
What tasks can I work on right now?
```


<a name="memory_stale"></a>
### memory_stale
Find old memories that may need review.

Lists memories that haven't been accessed in a specified number of days. Useful for cleanup and review.

```
Are there any old memories we should review or clean up?
```


<a name="memory_stats"></a>
### memory_stats
Get memory database statistics.

Shows counts by type and status, active patterns, recent corrections, and database health. Called at session start to load context.

```
Give me an overview of what you remember about this project
```

<a name="memory_context"></a>
### memory_context
Generate rich session context.
Traverses the memory chain to build context: journey (origin, milestones), themes, and active work. Call at session start to understand shared history.
```
What's the context of our work together?
```
<a name="memory_traverse"></a>
### memory_traverse
Walk the memory chain.
Follows links backwards through memory history, showing how experiences connect. Returns the path with each memory and link type.
```
Show me how we got to this point in the project
```
<a name="memory_origin"></a>
### memory_origin
Find the start of a memory chain.
Traverses backwards to find the first memory in a chain. Returns the origin memory and chain length.
```
Where did this line of work begin?
```
<a name="memory_network"></a>
### memory_network
Explore memory graph structure.
Analyzes memory connections as a network. Five modes: `hubs` (most connected), `neighbors` (direct connections), `cluster` (related memories), `bridges` (connecting separate areas), `layers` (3D topology showing canonicals and their instances).
```
What are the central themes in your memory?
```


<a name="memory_instances"></a>
### memory_instances
Query instances of a canonical memory.
Returns all concrete occurrences (instances) of a canonical memory in the 3D layered topology. Includes temporal summary with first/last seen timestamps and frequency. Use to see when and how often a concept recurs.
```
How often has this error pattern occurred?
```

<a name="memory_challenge"></a>
### memory_challenge
Challenge a claim by searching for counterevidence.

Generates adversarial queries (negations, revision cues) and searches both memory and context databases for contradicting evidence. Returns a summary with assessment. Use for epistemic hygiene—verifying assumptions before acting on them.

```
Challenge the assumption that our API always returns JSON
```


<a name="memory_challenge_evidence"></a>
### memory_challenge_evidence
Retrieve evidence from a challenge session.

Returns full details of supporting evidence, counterevidence, or evolution timeline from a previous challenge. Use pagination for large result sets.

```
Show me the counterevidence from that challenge
```

<a name="memory_explore"></a>
### memory_explore
Divergent memory retrieval — find memories you wouldn't normally find.
Uses anti-recency scoring, graph exploration, topic drift, and MMR diversity re-ranking. Three modes: seed (explore outward from a memory), query (diversity-heavy topic exploration), discovery (surface neglected memories). Use when starting a new plan, feeling stuck, or asking "what am I not thinking about?"
```
What have I forgotten about this project?
```
<a name="memory_config"></a>
### memory_config
View ranking weight configuration.

Shows current search ranking weights for frequency, recency, priority, and context boosting.

```
How are memory search results being ranked?
```


<a name="memory_tune"></a>
### memory_tune
Adjust search ranking weights.

Modifies how memories are ranked in search results. Requires a rationale for transparency.

```
Prioritize more recent memories in search results
```


<a name="memory_backups"></a>
### memory_backups
List memory database backups.

Shows available backups of the memory database, created automatically each session.

```
Are there any memory backups available?
```


<a name="memory_restore"></a>
### memory_restore
Restore from a backup.

Restores the memory database from a previous backup. Use if data was accidentally deleted.

```
Restore the memory database from yesterday's backup
```


<a name="memory_import"></a>
### memory_import
Import markdown plans into memory.

Converts a markdown file into a memory entry. Useful for migrating existing documentation.

```
Import the ARCHITECTURE.md file as a plan memory
```


<a name="memory_prune"></a>
### memory_prune
Prune memories by strategy.

Three strategies: `duplicates` (same title+type, keeps highest access_count), `stale` (not accessed in N days with access_count=0), `low_importance` (uses importance scoring). Defaults to dry_run=true for safety. Excludes active plans/steps and in-progress work.

```
Show duplicate memories that could be pruned
```


## Search & Edit Tools

<a name="search"></a>
### search
FTS5 full-text search (30-195x faster than grep).

Searches the indexed workspace using SQLite FTS5. Supports boolean queries (AND, OR, NOT, NEAR), prefix matching, and file filtering. Auto-initializes the index on first use.

```
Find all files that mention both "authentication" and "token"
```


<a name="read"></a>
### read
Read files with line numbers.

Reads file contents with line numbers for accurate editing. Supports partial reads with start/end lines, whitespace visualization for debugging, and explicit streaming control. Auto-streams large files or bounded reads exceeding thresholds, with hardware pressure awareness.

```
Show me the handleAuth function in auth.js
```


<a name="edit"></a>
### edit
Find/replace with fuzzy whitespace matching.

Performs find/replace operations with automatic whitespace normalization. Supports insert, delete, and patch modes. Shows visual diffs of changes.

```
Replace the hardcoded timeout of 5000 with a constant TIMEOUT_MS
```


<a name="update"></a>
### update
Simple old_string/new_string replacement.

Straightforward text replacement with clear error messages. Fails if the target string isn't found or isn't unique.

```
Change the function name from processData to processUserData
```


<a name="write"></a>
### write
Create or overwrite files.

Creates new files or completely replaces existing file contents. Creates parent directories if needed.

```
Create a new utils.js file with helper functions for date formatting
```


<a name="batch"></a>
### batch
Multiple edit operations atomically.

Executes multiple edits as a single atomic operation. All succeed or all fail, preventing partial updates.

```
Rename the "user" variable to "currentUser" in all three files
```


<a name="transform"></a>
### transform
SQL-based file transformation.

Transforms file contents using SQL queries with regex_replace, string functions, and more.

```
Convert all console.log statements to use our logger instead
```


<a name="sql"></a>
### sql
Run SQL on text input.

Executes SQL queries on piped text input. Useful for filtering, transforming, or analyzing text data.

```
Parse this CSV and show me only rows where the status is "failed"
```


<a name="workspace"></a>
### workspace
Manage the search index.

Controls the workspace index: init, status, refresh, or rebuild. The index auto-initializes on first search.

```
Refresh the search index to pick up recent file changes
```


## Context Tools

Context tools preserve conversation history across sessions and context window compactions.

<a name="context_session"></a>
### context_session
Manage conversation sessions.

Start, end, get, or query sessions. Sessions track conversation context for preservation and synthesis.

```
Start tracking this conversation
```


<a name="context_save"></a>
### context_save
Save a message or tool call.

Stores important messages and tool calls to the context database with token estimates.

```
Save this important exchange about the architecture decision
```


<a name="context_search"></a>
### context_search
Search conversation history.

Full-text search across past conversations using FTS5 boolean queries.

```
What did we discuss about authentication last week?
```


<a name="context_query"></a>
### context_query
SQL query on context database.

Run SQL queries on sessions, messages, and tool_calls tables for advanced analysis.

```
Show me all sessions from this project
```


<a name="context_link"></a>
### context_link
Link conversation to memory.

Creates bidirectional relationship between a context message and a memory entry.

```
Link this conversation to the authentication plan
```


<a name="context_synthesize"></a>
### context_synthesize
Create session summary.

Stores a summary for a session while preserving full conversation for archive.

```
Summarize what we accomplished in this session
```


<a name="context_archive"></a>
### context_archive
Archive session to cold storage.

Moves verbatim messages to archive database, keeps summary in main database.

```
Archive old sessions to save space
```


<a name="context_stats"></a>
### context_stats
Context database statistics.

Shows session counts, message counts, storage size, and current session info.

```
How much conversation history do you have?
```


<a name="context_stale"></a>
### context_stale
Find sessions needing consolidation.

Two tiers: immediate (ended, no summary) and deep (has summary, no memory link). Use for periodic context maintenance.

```
What sessions need consolidation?
```


<a name="context_memory"></a>
### context_memory
Get conversation context for a memory.

Returns the full conversation that led to a memory being created, including surrounding messages.

```
Show me the conversation that led to this decision
```


## Web & Repository Tools

<a name="web_crawl"></a>
### web_crawl
Crawl and index a website.

Downloads and indexes a website for offline searching. Respects robots.txt, follows links, and deduplicates content. Great for documentation sites.

```
Index the React documentation so you can reference it offline
```


<a name="web_search"></a>
### web_search
Search indexed web content.

Full-text search across crawled websites. Supports boolean operators and returns relevant snippets.

```
Search the React docs for information about useEffect cleanup
```


<a name="web_query"></a>
### web_query
SQL queries on web content.

Run SQL queries directly on the web content database for advanced filtering and analysis.

```
Find all pages in the docs that mention "deprecated"
```


<a name="web_stats"></a>
### web_stats
Web database statistics.

Shows page count, word count, domains indexed, and crawl timestamps for a web database.

```
How much documentation do you have indexed?
```


<a name="web_refresh"></a>
### web_refresh
Update stale cached pages.

Re-fetches pages that are older than a specified age. Keeps documentation caches current.

```
Update any cached documentation pages older than a week
```


<a name="web_manifest"></a>
### web_manifest
Detailed metadata about cached sources.

Shows per-domain page counts, word counts, crawl dates, and freshness statistics for a web database.

```
What documentation sources do you have cached and when were they last updated?
```


<a name="web_search_multi"></a>
### web_search_multi
Search across multiple cached databases.

Searches multiple web databases simultaneously and merges results by relevance. Useful when documentation is split across multiple crawled sites.

```
Search all cached docs for information about rate limiting
```


<a name="web_merge"></a>
### web_merge
Merge multiple doc caches into one.

Combines multiple web databases into a single unified database. Deduplicates by content hash and keeps the most recent version of duplicate URLs.

```
Combine the React and TypeScript doc caches into a single database
```

<a name="web_fetch"></a>
### web_fetch
Fetch a single URL.
Fetches a single page and returns structured content (title, description, text, links). Optionally stores to database for later FTS5 searching. Use for one-off page fetches; use web_crawl for multi-page documentation sites.
```
Fetch the API documentation page and extract all the links
```

<a name="repo_clone"></a>
### repo_clone
Clone and index a git repository.

Clones a git repository and indexes its source code into a searchable database. Great for exploring unfamiliar codebases.

```
Clone and index the lodash repository so we can study their implementation
```


<a name="repo_search"></a>
### repo_search
Search indexed repository.

Full-text search across an indexed repository. Filter by language or file pattern.

```
Search the lodash repo for debounce implementation
```


<a name="repo_query"></a>
### repo_query
SQL queries on repository content.

Run SQL queries on indexed repository files. Query by language, line count, file path patterns.

```
Find the largest files in the indexed repository
```


<a name="repo_stats"></a>
### repo_stats
Get repository statistics.

Shows file count, line count, language breakdown, and clone metadata for an indexed repository.

```
How big is the indexed lodash repository?
```


<a name="repo_list"></a>
### repo_list
List indexed repository databases.

Shows all repository databases in the current directory.

```
What repositories have you indexed?
```


## Fingerprint Tools

Fingerprints capture *how Claude engages* with a project—not just what happened. They synthesize engagement patterns, learning signatures, and calibration data to maintain collaboration continuity across sessions.

<a name="fingerprint_load"></a>
### fingerprint_load
Load fingerprint at session start.

Returns posture (engagement patterns), priors (behaviors to follow), and stance (toward user, errors, decisions). Call this FIRST at session start—the fingerprint shapes how to interpret everything else.

```
Load my collaboration fingerprint
```


<a name="fingerprint_generate"></a>
### fingerprint_generate
Generate a fingerprint from current data.

Synthesizes 6 dimensions: engagement rhythm, learning signature, reasoning style, user calibration, conceptual topology, and tool fluency. Returns fingerprint with confidence score based on data maturity.

```
Generate a new fingerprint from our collaboration data
```


<a name="fingerprint_compare"></a>
### fingerprint_compare
Compare two fingerprints.

Shows evolution over time with deltas per dimension and magnitude. Use to understand how engagement patterns have changed.

```
How has our collaboration evolved since last month?
```


<a name="fingerprint_drift"></a>
### fingerprint_drift
Detect session drift from baseline.

Returns drift warnings when current session behavior differs significantly from the fingerprint baseline. Use periodically during long sessions.

```
Am I behaving differently than usual in this session?
```


## Hardware Awareness Tools

Sift provides Claude with awareness of hardware resources, enabling intelligent adaptation to system constraints.

<a name="hardware_status"></a>
### hardware_status
Get multi-dimensional resource state.

Returns current state of memory, I/O, database, and process resources. Includes PSI (Pressure Stall Information) metrics, cgroup limits, and suggestions for adapting to current conditions.

```
What's the current resource state?
```


<a name="hardware_patterns"></a>
### hardware_patterns
View learned access patterns.

Shows tool call sequences and their probabilities. The system learns which tools tend to follow other tools for predictive optimization.

```
What patterns have you learned about my tool usage?
```


<a name="hardware_events"></a>
### hardware_events
View logged resource events.

Shows state changes, adaptations, wall hits, and diagnoses. Use to understand patterns in resource pressure and how the system adapted.

```
Show me resource events from the last hour
```


<a name="budget_request"></a>
### budget_request
Request a resource budget.

Request memory, I/O, and latency budget for an operation. Returns what's available given current system state, with constraints and suggestions.

```
Can I get 100MB for a comprehensive search?
```


<a name="budget_stats"></a>
### budget_stats
View budget utilization statistics.

Shows how actual resource usage compares to budgeted amounts, helping calibrate future budget requests.

```
How accurate have my budget estimates been?
```


<a name="memory_sqlite_config"></a>
### memory_sqlite_config
View and configure SQLite settings.

Shows mmap_size, cache_size, journal_mode, WAL status, and memory usage. Allows tuning performance parameters.

```
What are the current SQLite settings?
```


<a name="memory_cache_status"></a>
### memory_cache_status
View memory importance and cache status.

Shows memories ranked by importance score (recency, frequency, linkage, type) and eviction candidates.

```
Which memories are most important right now?
```


<a name="stream_read"></a>
### stream_read
Read from a streaming operation.

Read next chunk from a streaming result. Used for large results that exceed single response limits.

```
Read the next chunk from that stream
```


<a name="stream_close"></a>
### stream_close
Close a stream and release resources.

Closes a streaming operation and releases shared memory resources.

```
Close the search stream
```


## CLI Commands

Beyond MCP tools, sift provides CLI commands for direct use.

### sift --monitor

Real-time hardware and resource monitoring dashboard.

Shows live metrics including memory pressure (PSI), I/O stats, database activity, active streams, and recent tool calls. Useful for understanding resource constraints during heavy operations.

```bash
sift --monitor
```

Press `q` to quit, `h` for help, `s` for stress test.


## Verify Download

Each release includes `checksums.txt` with SHA256 hashes:
```bash
sha256sum -c checksums.txt
```

## License

Proprietary. Binary distribution only.
