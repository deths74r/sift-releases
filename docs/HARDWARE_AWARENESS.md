# Hardware-Aware Memory System

## Overview

The Hardware-Aware Memory System enables Claude to perceive hardware constraints and adapt behavior intelligently. Instead of hitting resource limits blindly, the system makes constraints legible signals that shape memory operations.

**Philosophy:** Constraints aren't bugs - they're the generative grammar of form. Intelligence requires working *with* constraints, not blindly into them.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                  HARDWARE AWARENESS STACK                   │
├─────────────────────────────────────────────────────────────┤
│  Layer 11: Experience Logging    │ Learn from past events   │
│  Layer 10: madvise Integration   │ Kernel memory hints      │
│  Layer 9:  SQLite Optimization   │ mmap + WAL + cache       │
│  Layer 8:  Resource Budgets      │ Request/grant mechanism  │
│  Layer 7:  Predictive Warming    │ Access pattern learning  │
│  Layer 6:  Importance Scoring    │ Cache eviction priority  │
│  Layer 5:  Self-Diagnosis        │ Explain failures         │
│  Layer 4:  Graceful Degradation  │ Quality tradeoffs        │
│  Layer 3:  Streaming Foundation  │ Shared memory rings      │
│  Layer 2:  Resource Hints        │ Tool response metadata   │
│  Layer 1:  Metrics Foundation    │ PSI, cgroups, /proc/*    │
├─────────────────────────────────────────────────────────────┤
│                    Linux Kernel Interfaces                  │
│  /proc/pressure/*  │  /sys/fs/cgroup/*  │  madvise(2)       │
└─────────────────────────────────────────────────────────────┘
```

## Tools

### sift_hardware_status

Get multi-dimensional hardware resource state.

**Returns:**
- **Memory:** RSS, available, pressure %, cgroup limits, trends
- **I/O:** PSI pressure metrics, latency
- **Process:** CPU %, open FDs, uptime
- **Database:** memory.db size, WAL size, mmap status

**State levels:** `normal`, `elevated`, `critical`, `survival`

**Example:**
```json
{
  "memory": {
    "state": "normal",
    "rss_mb": 256,
    "available_mb": 29000,
    "psi_some_avg10": 0.5
  },
  "io": {
    "state": "critical",
    "psi_some_avg10": 85.2,
    "psi_full_avg10": 4.2
  }
}
```

### sift_hardware_patterns

View learned access patterns for predictive warming.

The system tracks tool call sequences and their probabilities. When a pattern is recognized with high probability, background prep can be triggered.

**Example patterns:**
- `stats` → `search` (60%)
- `search` → `get` (45%)
- `challenge` → `challenge_evidence` (95%)

### sift_hardware_events

Query resource event history for learning from past experiences.

**Event types:**
- `state_change` - Resource state transitions (normal → elevated → critical)
- `adaptation` - Behavioral changes made in response to pressure
- `wall_hit` - Resource limit reached, operation failed
- `diagnosis` - Failure explanation with metrics

**Parameters:**
- `hours` - Hours of history to include (default: 24)
- `event_type` - Filter by type (optional)
- `limit` - Maximum events to return (default: 50)

### sift_budget_request

Request a resource budget before expensive operations.

**Parameters:**
- `operation` - Name of the operation being planned
- `memory_mb` - Requested memory budget in MB
- `io_operations` - Requested I/O operation count
- `max_latency_ms` - Maximum acceptable latency

**Returns:**
- Granted amounts (may be less than requested under pressure)
- Constraints explaining any reductions
- Suggestions for alternative approaches

**Example:**
```json
{
  "budget_request": {
    "operation": "comprehensive_search",
    "memory_mb": 100,
    "io_ops": 500
  }
}

{
  "budget_grant": {
    "memory_mb": 100,
    "io_ops": 125,
    "constraints": ["io_limited"],
    "suggestions": ["batch_operations"]
  }
}
```

### sift_budget_stats

View budget utilization statistics.

Shows how actual resource usage compares to budgeted amounts, enabling calibration of future budget requests.

**Returns:**
- Total operations tracked
- Within-budget rate
- Utilization by operation type
- Recent over-budget operations

### sift_memory_sqlite_config

View and configure SQLite settings for the memory database.

**Displays:**
- `mmap_size` - Memory-mapped I/O size (default: 256MB)
- `cache_size` - Page cache size (default: 32MB)
- `journal_mode` - Should be WAL for concurrency
- Memory usage statistics

**Parameters (optional):**
- `mmap_size_mb` - Set mmap size (1-1024 MB)
- `cache_size_mb` - Set cache size (1-256 MB)

### sift_memory_cache_status

View cache status and eviction candidates.

Memories are scored by importance using four factors:
- **Recency** - Exponential decay with 1-hour half-life
- **Frequency** - Log-scale access count
- **Linkage** - Connection to active work (plans, tasks in progress)
- **Type** - patterns > preferences > gotchas > notes

**Returns:**
- Eviction candidates (lowest importance scores)
- Cache distribution (hot/warm/cold counts)
- Total memories
- Recommendations

### sift_stream_read

Read next chunk from a streaming operation.

Used for large results that exceed single response limits. Streams use shared memory ring buffers for efficient zero-copy data transfer with backpressure.

**Parameters:**
- `stream_id` - Stream ID from a streaming response

**Returns:**
- `chunk` - Data chunk
- `bytes_read` - Bytes in this chunk
- `done` - Whether stream is complete

### sift_stream_close

Close a stream and release resources.

Waits for producer to finish, unmaps shared memory, and removes the stream.

**Parameters:**
- `stream_id` - Stream ID to close

**Returns:**
- Total bytes transferred

## Linux Kernel Integration

### PSI (Pressure Stall Information)

Reads `/proc/pressure/memory` and `/proc/pressure/io` to detect system pressure:

- `psi_some_avg10 > 10.0` → Elevated state (some tasks waiting)
- `psi_full_avg10 > 1.0` → Critical state (all tasks stalled)

### cgroups v2

Detects container memory limits via `/sys/fs/cgroup/memory.*`:
- `memory.current` - Current usage
- `memory.max` - Hard limit
- `memory.high` - Soft limit (triggers reclaim)

### madvise

Kernel hints for memory management:
- `MADV_WILLNEED` - Prefetch before chain traversal
- `MADV_DONTNEED` - Release after use under pressure
- `MADV_SEQUENTIAL` - Optimize streaming access

## Graceful Degradation

Each tool documents capability at each pressure level:

| State | Search Capability |
|-------|-------------------|
| Normal | FTS5 with BM25 ranking, synonyms, 50 results |
| Elevated | FTS5 without synonyms, 30 results |
| Critical | Simple LIKE queries, 10 results |
| Survival | Return error with diagnosis |

Tool responses include degradation info:
```json
{
  "result": { ... },
  "_resource": {
    "state": "elevated",
    "degradations_applied": ["synonyms_disabled", "result_limit_reduced"],
    "quality_impact": "Results not ranked by relevance"
  }
}
```

## Self-Diagnosis

When operations fail, the system explains in resource terms:

```json
{
  "error": "query_timeout",
  "diagnosis": {
    "cause": "memory_pressure_caused_disk_thrashing",
    "trigger": {
      "metric": "psi_full_avg10",
      "value": 8.2,
      "threshold": 1.0
    },
    "suggestions": ["reduce_query_scope", "use_streaming_mode"]
  }
}
```

## Database Schema

### budget_tracking

```sql
CREATE TABLE budget_tracking (
  id INTEGER PRIMARY KEY,
  timestamp INTEGER,
  session_id TEXT,
  operation TEXT,
  memory_budgeted INTEGER,
  memory_actual INTEGER,
  io_budgeted INTEGER,
  io_actual INTEGER,
  latency_budgeted INTEGER,
  latency_actual INTEGER,
  within_budget BOOLEAN,
  notes TEXT
);
```

### resource_events

```sql
CREATE TABLE resource_events (
  id INTEGER PRIMARY KEY,
  timestamp INTEGER,
  session_id TEXT,
  event_type TEXT,
  dimension TEXT,
  from_state TEXT,
  to_state TEXT,
  trigger_metric TEXT,
  trigger_value REAL,
  tool_name TEXT,
  adaptation TEXT,
  outcome TEXT,
  diagnosis TEXT,
  notes TEXT
);
```

### access_patterns

```sql
CREATE TABLE access_patterns (
  id INTEGER PRIMARY KEY,
  tool_sequence TEXT,
  next_tool TEXT,
  probability REAL,
  avg_latency_ms REAL,
  last_seen INTEGER,
  count INTEGER
);
```

## Capabilities

With the Hardware-Aware Memory System, Claude can:

1. **Perceive** hardware state across 4 dimensions (memory, I/O, process, database)
2. **Predict** next operations from learned patterns
3. **Budget** resources before expensive operations
4. **Degrade** gracefully under pressure with explicit quality tradeoffs
5. **Diagnose** failures in resource terms with actionable suggestions
6. **Learn** from experience via event logging

This is embodied cognition: a memory system that feels its physics, reasons about its constraints, and adapts intelligently.
