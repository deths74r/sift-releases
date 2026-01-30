# Memory Forget Model - Human Memory Semantics

## Goal
Replace hard deletion with "forget" - memories fade but aren't destroyed, preserving network connections and enabling recall.

---

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Status name | `forgotten` | Matches human memory metaphor |
| Icon | `○` (empty circle) | "Faded out" but still there |
| Tool name | Rename to `memory_forget` | Name reflects new semantics |
| Cascade default | Yes | Forgetting a plan forgets its steps |
| Recall default cascade | No | Intentional asymmetry - recall is selective |

---

## Changes

### 1. Add Status Constant
**File:** `src/sift_memory.h` (line 41)
```c
#define MEM_STATUS_FORGOTTEN "forgotten"
```

### 2. Update Validation
**File:** `src/sift_memory.c` (line 1562)
- Add `MEM_STATUS_FORGOTTEN` to `VALID_STATUSES[]`

### 3. Add Icon
**File:** `src/sift_memory.c` (in `status_icon()`)
```c
if (strcmp(status, "forgotten") == 0)
  return "○";
```

### 4. Replace `memory_delete()` with `memory_forget()`
**File:** `src/sift_memory.c` (line 2870)
- Instead of DELETE, UPDATE status to `forgotten`
- Preserve all links, decisions, reflections
- Cascade to children by default (configurable)

### 5. Add `memory_recall()` Function
**File:** `src/sift_memory.c` (new function)
- Restore forgotten memory to specified status (default: `open`)
- Optional cascade to recall forgotten children

### 6. Update `memory_list()` Filtering
**File:** `src/sift_memory.c` (line 2927)
- Add `include_forgotten` parameter (default: false)
- Exclude forgotten memories from normal queries

### 7. Update `memory_search()` Filtering
**File:** `src/sift_memory.c` (line 3196)
- Add `include_forgotten` parameter (default: false)
- Exclude forgotten memories from search results

### 8. Update MCP Handlers
**File:** `src/sift_memory.c`
- Rename `mcp_tool_memory_delete` → `mcp_tool_memory_forget`
- New `mcp_tool_memory_recall`: Add recall capability
- `mcp_tool_memory_list`: Add `include_forgotten` param
- `mcp_tool_memory_search`: Add `include_forgotten` param

### 9. Update Tool Definitions
**File:** `src/sift.c`
- Rename tool from `memory_delete` to `memory_forget`
- Add `cascade` parameter
- Add new `memory_recall` tool
- Add `include_forgotten` to list/search tools
- Update dispatch routing for forget/recall

### 10. Header Declarations
**File:** `src/sift_memory.h`
```c
int memory_forget(sqlite3 *db, const char *id, int cascade);
int memory_recall(sqlite3 *db, const char *id, const char *new_status, int cascade);
cJSON *mcp_tool_memory_forget(cJSON *args);
cJSON *mcp_tool_memory_recall(cJSON *args);
```

---

## Behavioral Changes

| Operation | Before | After |
|-----------|--------|-------|
| `memory_forget(id)` | (was `delete`: hard delete) | Status → `forgotten`, children forgotten, links/decisions preserved |
| `memory_list()` | Shows all statuses | Excludes `forgotten` by default |
| `memory_search()` | Searches all | Excludes `forgotten` by default |
| Network traversal | N/A (deleted = gone) | Links to forgotten still work |

---

## New Tool: `memory_recall`

```
memory_recall(
  id: "mem-abc123",      // Required
  status: "open",        // Optional, default: "open"
  cascade: false         // Optional, default: false
)
```

---

## Verification

1. Build: `make` (ask Edward to run)
2. Test forget:
   - Create a memory with children
 - `memory_forget(id)` - verify status is `forgotten`
 - `memory_list()` - verify memory not shown
 - `memory_list(include_forgotten: true)` - verify memory shown
 - `memory_get(id)` - verify memory still accessible
 - Verify links still work via `memory_deps`
3. Test recall:
 - `memory_recall(id)` - verify status restored
 - `memory_list()` - verify memory now shown
4. Test search exclusion:
 - `memory_search(query)` - verify forgotten excluded
 - `memory_search(query, include_forgotten: true)` - verify found
