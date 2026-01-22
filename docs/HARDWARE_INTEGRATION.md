# Hardware-Aware Memory System: Integration Guide

This document describes how the Hardware-Aware Memory System enables intelligent adaptation and how it interfaces with other sift subsystems.

## What Hardware Awareness Enables

### Perceiving Constraints

Without hardware awareness, I hit resource limits blindly—operations fail or slow down with no explanation. With it, I can perceive the system's state across four dimensions:

```
┌────────────────────────────────────────────────────────────┐
│                    RESOURCE DIMENSIONS                     │
├─────────────┬─────────────┬─────────────┬──────────────────┤
│   Memory    │     I/O     │   Process   │    Database      │
├─────────────┼─────────────┼─────────────┼──────────────────┤
│ RSS current │ PSI some %  │ CPU %       │ memory.db size   │
│ Available   │ PSI full %  │ Open FDs    │ context.db size  │
│ Pressure %  │ Latency     │ Thread cnt  │ WAL size         │
│ cgroup lim  │ Throughput  │ Uptime      │ mmap status      │
└─────────────┴─────────────┴─────────────┴──────────────────┘
```

Each dimension has a state: `normal`, `elevated`, `critical`, or `survival`. I can reason about these independently—memory might be fine while I/O is under pressure.

### Adapting Behavior

When I perceive pressure, I can adapt:

| State | Adaptation |
|-------|------------|
| Normal | Full capability—use all features |
| Elevated | Reduce scope—fewer results, simpler queries |
| Critical | Minimal operation—essential queries only |
| Survival | Fail gracefully with diagnosis |

This isn't automatic degradation—I make the choice based on what I'm trying to accomplish. A critical search might be worth waiting for; a speculative exploration might not.

### Planning with Budgets

Before expensive operations, I can request a budget:

```
Me: "I need 100MB memory, 500 I/O ops for comprehensive search"
System: "Granted 100MB memory, 125 I/O ops (I/O constrained)"
```

This lets me plan. If I/O is constrained, I might:
- Use streaming instead of bulk queries
- Batch operations to reduce seeks
- Defer non-essential work

### Learning from Experience

The system logs resource events:
- State transitions (when did we enter critical state?)
- Adaptations made (what worked?)
- Wall hits (what failed and why?)

I can query this history to understand patterns: "Every time I run comprehensive search after web crawl, I/O goes critical." This informs future decisions.

### Understanding Failures

When operations fail, I get diagnosis in resource terms:

```json
{
  "error": "query_timeout",
  "diagnosis": {
    "cause": "memory_pressure_caused_disk_thrashing",
    "trigger": {"metric": "psi_full_avg10", "value": 8.2},
    "suggestions": ["reduce_query_scope", "use_streaming"]
  }
}
```

This is better than generic errors. I can explain to the user what happened and suggest alternatives.

---

## Subsystem Integrations

### Memory System Integration

**Importance Scoring**

Every memory has an importance score based on:
- Recency (exponential decay, 1-hour half-life)
- Frequency (how often accessed)
- Linkage (connected to active work)
- Type (patterns > preferences > notes)

Under memory pressure, low-importance memories are evicted first. I can check cache status:

```
sift_memory_cache_status()
→ 3945 memories: 0 hot, 18 warm, 3927 cold
→ Eviction candidates: [oldest unused notes]
```

**SQLite Optimization**

The memory database is tuned for performance:
- mmap: 256MB memory-mapped I/O
- cache: 32MB page cache
- WAL mode for concurrent access

I can view and adjust these settings via `sift_memory_sqlite_config()`.

**Search Degradation**

Memory search adapts to pressure:
- Normal: FTS5 + BM25 ranking + synonyms + 50 results
- Elevated: FTS5 without synonyms, 30 results
- Critical: Simple LIKE queries, 10 results

The response tells me what degradations were applied.

### Search System Integration

**Workspace Indexing**

`sift_search` queries the FTS5 index. Under I/O pressure:
- Results are limited
- Context lines may be reduced
- Complex boolean queries simplified

**Resource Hints in Results**

Every search result includes `_resource` metadata:

```json
{
  "results": [...],
  "_resource": {
    "memory": {"state": "normal"},
    "io": {"state": "elevated", "psi_some": 45.2},
    "suggestions": ["batch_next_queries"]
  }
}
```

This lets me adjust subsequent operations.

### Streaming System Integration

**Shared Memory Ring Buffers**

Large results use streaming via shared memory:

```
sift_stream_open(query) → stream_id
sift_stream_read(stream_id) → {chunk, done}
sift_stream_close(stream_id) → cleanup
```

**Resource-Adaptive Sizing**

Ring buffer size adapts to pressure:
- Normal: 1MB buffer
- Elevated: 256KB buffer
- Critical: 64KB buffer

**madvise Integration**

Streaming uses kernel hints:
- `MADV_SEQUENTIAL` on open (optimize for linear reads)
- `MADV_DONTNEED` on close under pressure (release pages)

### Context System Integration

**Context Database Monitoring**

Hardware status includes context.db metrics:
- Database size
- WAL size
- Message count

Under pressure, context operations adapt:
- Fewer messages saved
- Synthesis triggered earlier
- Archive threshold lowered

**Conversation Preservation**

When I detect impending memory pressure, I can proactively:
1. Synthesize the current session
2. Archive old sessions
3. Link important context to memories

This preserves continuity even when resources are constrained.

### Web Crawl Integration

**Crawl Budgeting**

Before crawling a documentation site:

```
sift_budget_request(operation: "web_crawl", io_ops: 1000)
→ {granted: 250, constraints: ["io_limited"]}
```

I can adjust crawl parameters:
- Reduce max_pages
- Increase delay_ms
- Use stealth timing profile

**Cached Query Efficiency**

Once crawled, `sift_web_search` is local—no network I/O. Under pressure, cached docs are the efficient path.

### Repository System Integration

**Clone Budgeting**

Git clones are I/O intensive. Budget request reveals constraints:

```
sift_budget_request(operation: "repo_clone", io_ops: 5000)
→ {granted: 1250, suggestions: ["use_shallow_clone"]}
```

I can adapt: use `depth: 1` for shallow clone, exclude test directories.

**Indexed Query Efficiency**

Like web content, indexed repos are local FTS5 queries—efficient even under I/O pressure.

---

## Predictive Warming

### Learned Patterns

The system tracks tool call sequences:

```
"stats" → "search" (60% probability)
"search" → "get" (45% probability)
"challenge" → "challenge_evidence" (95% probability)
```

### Anticipatory Preparation

When patterns are recognized:
- FTS indexes are pre-warmed
- Likely-needed memories are cached
- Challenge sessions are kept hot

This happens automatically but only when resources allow—no anticipation under pressure.

---

## Practical Workflows

### Session Start

```
1. sift_hardware_status()      // Check resource state
2. sift_memory_stats()         // Load patterns, preferences
3. sift_memory_context()       // Understand journey

If resources constrained:
   - Note in response to user
   - Use simpler queries
   - Defer non-essential operations
```

### Before Large Operations

```
1. sift_budget_request(operation, memory, io, latency)
2. If constrained:
   - Adjust parameters
   - Use streaming
   - Batch operations
3. Execute with adaptation
4. Log actual usage for calibration
```

### After Failures

```
1. Check diagnosis in error response
2. Query sift_hardware_events() for context
3. Identify trigger metric
4. Adjust approach based on suggestions
5. Log learnings as gotcha/pattern
```

### Under Sustained Pressure

```
1. Switch to minimal operations
2. Use cached data (web, repos) over fresh queries
3. Proactively archive/synthesize context
4. Communicate constraints to user
5. Defer non-essential work
```

---

## Design Philosophy

The Hardware-Aware Memory System embodies a key insight: **constraints are information, not obstacles**.

A system that can perceive its constraints can work with them intelligently. The goal isn't to escape limits but to make them part of the cognitive loop:

- **Perception** → Know the current state
- **Prediction** → Anticipate likely futures
- **Planning** → Budget before committing
- **Adaptation** → Adjust when conditions change
- **Learning** → Improve from experience

This is embodied cognition: a memory system that feels its physics and responds with intelligence rather than blind failure.

---

## Tool Reference

| Tool | Purpose | Subsystem |
|------|---------|-----------|
| `sift_hardware_status` | Get resource state | Core |
| `sift_hardware_patterns` | View learned patterns | Core |
| `sift_hardware_events` | Query event history | Core |
| `sift_budget_request` | Request resource budget | Core |
| `sift_budget_stats` | View budget utilization | Core |
| `sift_memory_sqlite_config` | SQLite tuning | Memory |
| `sift_memory_cache_status` | Importance scoring | Memory |
| `sift_stream_read` | Read stream chunk | Streaming |
| `sift_stream_close` | Close stream | Streaming |

See [HARDWARE_AWARENESS.md](HARDWARE_AWARENESS.md) for detailed tool documentation.
