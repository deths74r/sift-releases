# Design: sift_memory_challenge - Adversarial Retrieval Tool

## Problem Statement

The memory system confirms narratives but doesn't challenge them. When I claim "duplicates represent connection strength," the system can find supporting evidence but doesn't automatically surface that I previously called duplicates "the issue" and wanted to fix them.

**Goal:** Build a tool that surfaces counterevidence, not just confirmation.

---

## API Design

### Input
```json
{
  "claim": "duplicates represent connection strength",
  "search_context": true,    // also search context.db (default: true)
  "search_memories": true,   // also search memory.db (default: true)
  "limit": 10,               // max results per category
  "time_window": null        // optional: limit to date range
}
```

### Output
```json
{
  "claim": "duplicates represent connection strength",
  "parsed": {
    "subject": "duplicates",
    "predicate": "represent",
    "object": "connection strength",
    "key_terms": ["duplicates", "connection", "strength"]
  },
  "support": [
    {
      "source": "context",
      "id": "msg-13065",
      "session_id": "a4c0da83...",
      "excerpt": "can't duplicates be thought of as the strength of connection?",
      "timestamp": 1768488629,
      "confidence": "HIGH",
      "reason": "direct quote supporting claim"
    }
  ],
  "counterevidence": [
    {
      "source": "context", 
      "id": "msg-12940",
      "session_id": "a4c0da83...",
      "excerpt": "The issue is that memory_traverse explores all paths in the DAG, causing duplicates",
      "timestamp": 1768487000,
      "type": "negation",
      "confidence": "HIGH",
      "reason": "prior framing: duplicates as 'issue' not 'strength'"
    }
  ],
  "evolution": [
    {
      "phase": "prior",
      "timestamp": 1768487000,
      "excerpt": "duplicates... causing [issues]",
      "framing": "bug"
    },
    {
      "phase": "transition",
      "timestamp": 1768488629,
      "excerpt": "can't duplicates be thought of as strength?",
      "framing": "question/reframe"
    },
    {
      "phase": "current",
      "timestamp": 1768489131,
      "excerpt": "those duplicates represent connection strength",
      "framing": "adopted belief"
    }
  ],
  "assessment": {
    "confidence": "MEDIUM",
    "has_counterevidence": true,
    "evolution_detected": true,
    "warnings": ["narrative_compression"]
  }
}
```

---

## Query Generation Strategy

### 1. Parse Claim into Components
Simple heuristic parsing:
- Extract key nouns/verbs
- Identify subject-predicate-object if possible
- Fall back to keyword extraction

### 2. Generate Adversarial Queries

**Negation queries** (invert the predicate):
- "duplicates represent strength" → 
  - "duplicates bug"
  - "duplicates issue"
  - "duplicates problem"
  - "remove duplicates"
  - "fix duplicates"

**Revision queries** (look for changes):
- "[key_term] + changed / updated / actually / wrong / correction"

**Disconfirmation cue queries**:
- "[key_term] + wait / no / stop / but / however / instead"

### 3. Negation Word Lists

```c
static const char *negation_pairs[][2] = {
  {"strength", "weakness"},
  {"strength", "problem"},
  {"strength", "bug"},
  {"strength", "issue"},
  {"good", "bad"},
  {"feature", "bug"},
  {"signal", "noise"},
  {"useful", "useless"},
  {"important", "unimportant"},
  {"correct", "wrong"},
  {"right", "wrong"},
  {"true", "false"},
  {"always", "never"},
  {"should", "shouldn't"},
  {NULL, NULL}
};

static const char *negation_verbs[] = {
  "remove", "delete", "fix", "eliminate", "avoid", "stop", NULL
};

static const char *revision_cues[] = {
  "actually", "wait", "no", "correction", "wrong", "update",
  "changed", "revised", "instead", "but", NULL
};
```

---

## Implementation Plan

### Phase 1: Core Function
```c
cJSON *memory_challenge(sqlite3 *mem_db, sqlite3 *ctx_db,
                        const char *claim, int search_context,
                        int search_memories, int limit);
```

1. Parse claim into key terms
2. Generate negation queries
3. Generate revision queries  
4. Search memory.db with each query
5. Search context.db with each query
6. Classify results (support/counter/evolution)
7. Build timeline for evolution detection
8. Return structured JSON

### Phase 2: MCP Tool Wrapper
```c
cJSON *mcp_tool_memory_challenge(cJSON *args);
```

Register as `sift_memory_challenge` in tool list.

### Phase 3: Evolution Detection
- Sort all results by timestamp
- Detect framing changes over time
- Flag if earlier messages contradict later beliefs

### Phase 4: Integration Points
- Optional: auto-run on new pattern/preference creation
- Optional: add to `sift_memory_stats` output

---

## Design Decisions (Recorded)

| # | Question | Decision | Rationale |
|---|----------|----------|-----------|
| 11 | Negation strategy | Static word list, expand as gaps found | Predictable behavior critical for self-skepticism tool |
| 12 | Context scope | All sessions, optional time_window | Historical contradictions are the point |
| 13 | Auto-challenge | Yes, advisory warnings on pattern/preference creation | See counterevidence before beliefs solidify |
| 14 | Caching | No result caching, fresh adversarial search | Accuracy over speed for epistemics |
| 15 | RAM loading | SQLite backup API to in-memory DBs | Scale from start - 3,500 memories now, 50,000+ later |

---

## Performance: RAM Loading Strategy

Load both databases into memory at challenge start for fast repeated FTS5 searches:

```c
/* Load disk DB into RAM for fast searching */
static sqlite3 *load_db_to_memory(const char *db_path) {
    sqlite3 *disk_db, *mem_db;
    sqlite3_backup *backup;
    
    /* Open disk database */
    if (sqlite3_open(db_path, &disk_db) != SQLITE_OK)
        return NULL;
    
    /* Create in-memory database */
    if (sqlite3_open(":memory:", &mem_db) != SQLITE_OK) {
        sqlite3_close(disk_db);
        return NULL;
    }
    
    /* Copy disk to memory */
    backup = sqlite3_backup_init(mem_db, "main", disk_db, "main");
    if (backup) {
        sqlite3_backup_step(backup, -1);  /* Copy all pages */
        sqlite3_backup_finish(backup);
    }
    
    sqlite3_close(disk_db);
    return mem_db;
}
```

**Why this scales:**
- Current: 3,500 memories (~5MB), 25,000 messages (~50MB) — loads in milliseconds
- Future: 50,000 memories, 500,000 messages — still fits comfortably in RAM
- FTS5 searches on in-memory DB: microseconds instead of milliseconds

---

## Success Criteria

1. `sift_memory_challenge("duplicates represent connection strength")` returns:
   - Support: Edward's question (msg 13065)
   - Counterevidence: My "issue" framing (msg 12940)
   - Evolution: timeline showing bug→question→belief

2. `sift_memory_challenge("weights barely mattered")` returns:
   - Support: Taguchi results
   - Counterevidence: (none found)
   - No evolution warning

3. Claims with no counterevidence explicitly say so (not silence)
