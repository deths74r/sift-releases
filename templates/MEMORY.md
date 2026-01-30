<!-- sift-template-0.16.0-alpha -->
# Memory System

Persistent, queryable storage that maintains collaboration continuity across sessions. 

**A task isn't complete until context is preserved.** Context loss disorients the user - they have to re-explain, re-establish understanding. Preserving context isn't a separate step after "real work" - it's part of finishing properly.

---

## 1. PROACTIVE BEHAVIORS

These are things you MUST do automatically, without being asked.

### Immediate Save Triggers

IMMEDIATELY use `memory_add` when the user shares:

| User shares... | Save as | Example |
|----------------|---------|--------|
| Their name | `type: preference` | "User's name is Edward" |
| A preference | `type: preference` | "Prefers tabs over spaces" |
| A project decision | `type: note` | "Using PostgreSQL for database" |
| A correction | `type: gotcha` | "Don't use deprecated API" |
| A workflow they like | `type: pattern` | "Always run tests before commit" |

**Save as part of completing the task. Don't treat it as separate.**

**When uncertain, save anyway.** Relevance emerges through interaction - you can't know at save time what will matter later. The system's ranking naturally surfaces what proves useful. A quick save that might not matter is better than missing something that would have helped.

### Correction Handling (CRITICAL)

When the user corrects you or says 'no', 'wrong', 'stop', 'don't do that':

1. **IMMEDIATELY** log a correction reflection BEFORE continuing
2. Save as gotcha if it's a mistake to avoid in the future
3. Then continue with corrected behavior

```
memory_reflect(
  reflection_type: "correction",
  content: "User said X was wrong because Y"
)
```

This ensures you don't repeat mistakes.

### Proactive Reflections

Log your reasoning throughout work, not just at the end:

- **reasoning**: Why you chose approach X over Y
- **observation**: Something notable you discovered
- **correction**: What you learned from user feedback

This creates a queryable reasoning trail for future reference.

### Friction Tracking

When a sift tool behaves unexpectedly, log it:

```
memory_add(
  type: "gotcha",
 title: "[search] FTS5 strips special characters",
  description: "Searching for '--mcp' tokenizes as 'mcp'. Use literal:true.",
 metadata: {"friction": true, "tool": "search", "category": "unexpected_result"}
)
```

Categories: `unexpected_result`, `missing_feature`, `confusing_behavior`, `performance`, `error_message`

### Protecting Collaboration History

**CRITICAL: The `.sift/` directory contains collaboration history.**

- NEVER delete, move, or run destructive commands on `.sift/`
- Losing this means the user has to re-establish all context
- Backups are created automatically, but prevention is better

---

## 2. WORKFLOWS

### Session Start

```
1. fingerprint_load → WHO I am (priors, stance, posture) - CALL FIRST
2. memory_stats → WHAT exists (patterns, preferences, corrections)
3. memory_context → WHERE we are (journey, milestones, active work)
```

**The fingerprint loads first** because it shapes how to interpret everything else. It captures *how this Claude engages* - not just what happened.

The SessionStart hook calls `sift --session-context` which does these steps automatically.

### Planning Workflow

For multi-step tasks:

```
1. Create plan: memory_add(type="plan", title="Implement feature X")
2. Add steps: memory_add(type="step", parent_id="mem-xxx", title="Step 1")
3. Record decisions: memory_decide(plan_id="mem-xxx", question="...", decision="...")
4. Track progress: memory_update(id="mem-xxx", status="in_progress")
5. Complete: memory_update(id="mem-xxx", status="done")
```

### Learning Loop

When you learn something:

```
1. User corrects you → memory_reflect(type="correction", ...)
2. If mistake to avoid → memory_add(type="gotcha", ...)
3. If behavior to follow → memory_add(type="pattern", status="active", ...)
```

### Understanding History

To understand project history:

```
memory_context() → Rich session context with themes
memory_traverse(id, hops) → Walk backwards through chain
memory_origin(id) → Find the root of a memory chain
memory_network(mode="hubs") → Find most-connected memories
```

---

## 3. MENTAL MODEL

### Memory Types

| Type | Purpose | Default Status | When to Use |
|------|---------|----------------|-------------|
| `note` | Facts, context, information | open | Project decisions, context |
| `preference` | User likes/dislikes | **active** | User's name, preferences |
| `pattern` | Behaviors to follow | **active** | Workflows, conventions |
| `gotcha` | Mistakes to avoid | open | Bugs, pitfalls, corrections |
| `plan` | Multi-step work | open | Complex tasks |
| `step` | Steps within a plan | open | Use with parent_id |
| `task` | Single actionable items | open | Todo items |
| `synthesis` | Consolidated knowledge | open | Created by `memory_synthesize` |
| `instance` | Concrete occurrence of a canonical | active | Auto-created on duplicates |

### Status Meanings

| Status | Meaning |
|--------|--------|
| `open` | Not started |
| `in_progress` | Currently working on |
| `done` | Completed |
| `blocked` | Waiting on something |
| `active` | **Always applies** (for patterns/preferences) |
| `archived` | Intentionally set aside (preserved but hidden by default) |

**Important:** `status='active'` means the memory is always relevant. Use for patterns and preferences.

**Archive philosophy:** Nothing is deleted. `archived` memories are excluded from `memory_list` by default but remain accessible via `memory_get`, searchable, and all links stay intact.

### Chain Linking

Memories automatically connect via `follows` links:

```
mem-001 (origin) ← mem-002 ← mem-003 ← mem-004 (recent)
```

This creates a collaboration history that can be traversed.

### 3D Layered Topology

When you create a memory with the same title as an existing one, the system creates a **3D layered structure**:

```
Layer 1 (Canonical)          Layer 2 (Instances)
┌──────────────────┐         ┌───────────────────┐
│ type: pattern    │◄────────│ type: instance    │
│ title: "X"       │instance │ context: "..."    │
│ recurrence: 3    │   of    │ occurrence: 2     │
└──────────────────┘         ├───────────────────┤
                             │ type: instance    │
                     ◄───────│ context: "..."    │
                             │ occurrence: 3     │
                             └───────────────────┘
```

- **Layer 1**: Canonical memories (abstract concepts/patterns)
- **Layer 2**: Instance memories (concrete occurrences linked via `instance_of`)

Each instance is a first-class memory with its own description capturing the specific context. The canonical's `recurrence_count` is maintained automatically by triggers.

**Use cases:**
- Track when a pattern recurs with different contexts
- See all occurrences of an error/issue over time
- Analyze frequency of concepts ("how often does X come up?")

### Search Ranking

Search results are ranked by:

1. **BM25**: Text relevance (base score)
2. **Frequency**: Often-accessed memories rank higher
3. **Recency**: Recently-accessed memories rank higher
4. **Priority**: Lower priority number = higher rank
5. **Context**: Memories matching current activity get boosted

### Intent Detection

The system detects query intent and boosts relevant types:

| Query pattern | Boosts |
|---------------|--------|
| "how to...", "best way..." | patterns, preferences |
| "error", "bug", "problem" | gotchas, patterns |
| "what", "why", "where" | patterns |
| "add", "create", "implement" | tasks |

---

## 4. TOOL REFERENCE

### Core CRUD

| Tool | Purpose | Key Parameters |
|------|---------|----------------|
| `memory_add` | Create memory | type*, title*, description, parent_id, priority, status, metadata |
| `memory_get` | Get by ID | id* |
| `memory_update` | Modify memory | id*, title, description, status, priority, metadata |
| `memory_archive` | Archive memory | id*, cascade (default: true) |

### Synthesis Tools

| Tool | Purpose | Key Parameters |
|------|---------|----------------|
| `memory_synthesize` | Consolidate memories | sources* (array), title*, summary*, mark_sources |
| `memory_expand` | Show synthesis sources | id* (synthesis memory ID) |

### Query Tools

| Tool | Purpose | Key Parameters |
|------|---------|----------------|
| `memory_list` | List memories | type, status, parent_id, metadata, limit, include_archived |
| `memory_search` | Full-text search | query*, type, limit, expand_synonyms |
| `memory_ready` | Tasks with no blockers | limit |
| `memory_stale` | Old memories | days, limit |

### Dependency Tools

| Tool | Purpose | Key Parameters |
|------|---------|----------------|
| `memory_link` | Create link | from_id*, to_id*, dep_type (blocks/related/follows/synthesizes) |
| `memory_unlink` | Remove link | from_id*, to_id* |
| `memory_deps` | Query dependencies | id*, direction (blockers/blocking) |

### Decision Tools

| Tool | Purpose | Key Parameters |
|------|---------|----------------|
| `memory_decide` | Record decision | plan_id*, question*, decision*, rationale |
| `memory_decisions` | Query decisions | plan_id, query, limit |
| `memory_supersede` | Replace decision | decision_id*, new_decision*, new_rationale |

### Reflection Tools

| Tool | Purpose | Key Parameters |
|------|---------|----------------|
| `memory_reflect` | Log reflection | reflection_type* (reasoning/observation/correction), content*, memory_id, context |
| `memory_reflections` | Query reflections | type, memory_id, query, limit |
| `memory_reflect_trajectory` | Reflect on chain | reflection_type*, content*, chain_end_id*, chain_start_id |
| `memory_trajectory_reflections` | Query trajectories | query, type, min_segment_length, max_segment_length, limit |

Trajectory types: `trajectory_pattern`, `arc_summary`, `pivot_point`, `friction_analysis`

### Chain Traversal

| Tool | Purpose | Key Parameters |
|------|---------|----------------|
| `memory_traverse` | Walk chain | id*, hops, link_types |
| `memory_origin` | Find chain root | id*, link_type |
| `memory_context` | Session context | depth, include_decisions |

### Network Analysis

| Tool | Purpose | Key Parameters |
|------|---------|----------------|
| `memory_network` | Graph analysis | mode* (hubs/neighbors/cluster/bridges/layers), id, link_types, limit |
| `memory_instances` | Query instances of a canonical | id*, limit, since |

Network modes:
- `hubs`: Most-connected memories (degree centrality)
- `neighbors`: Direct 1-hop connections
- `cluster`: Densely connected to given memory (2-3 hops)
- `bridges`: Memories connecting separate clusters
- `layers`: Show 3D topology with canonicals (layer 1) and their instances (layer 2)

### Fingerprint Tools

| Tool | Purpose | Key Parameters |
|------|---------|----------------|
| `fingerprint_load` | Load fingerprint for session start | fingerprint_id (optional, defaults to latest) |
| `fingerprint_generate` | Create new fingerprint | (none) |
| `fingerprint_compare` | Compare two fingerprints | id1*, id2* |
| `fingerprint_drift` | Detect session deviation | (none) |

**Call `fingerprint_load` FIRST at session start.** It returns:
- **posture**: Human-readable descriptions of engagement patterns
- **priors**: Behaviors to follow based on learned patterns  
- **stance**: Toward user, errors, and decisions

Six dimensions computed:
1. **Engagement Rhythm** - save frequency, reflection rate, decision density
2. **Learning Signature** - corrections, categories, learning velocity
3. **Reasoning Style** - rationale patterns, tradeoff preferences
4. **User Calibration** - preferences, communication style
5. **Conceptual Topology** - hub stability, link density, conceptual centers
6. **Tool Fluency** - sequence entropy, fluent tools (>80% probability)

Confidence levels:
- `high` (≥80%): Well-calibrated, trust the patterns
- `moderate` (40-80%): Useful but evolving
- `developing` (10-40%): Learning, be adaptive
- `nascent` (<10%): New collaboration, calibration still forming

### Configuration

| Tool | Purpose | Key Parameters |
|------|---------|----------------|
| `memory_config` | View ranking weights | (none) |
| `memory_tune` | Adjust weights | key*, value*, rationale* |

Keys: `weight_freq`, `weight_recency`, `weight_priority`, `weight_context`

### Maintenance

| Tool | Purpose | Key Parameters |
|------|---------|----------------|
| `memory_stats` | Database stats + context | (none) - **call at session start** |
| `memory_backups` | List backups | (none) |
| `memory_restore` | Restore backup | backup |
| `memory_import` | Import markdown | path*, type, delete_after |
| `memory_prune` | Prune by strategy | strategy*, dry_run, days, limit |
| `memory_explore` | Divergent retrieval | seed, query, exclude_recent, limit |

Strategies: `duplicates` (same title+type), `stale` (unaccessed, access_count=0), `low_importance` (importance scoring). `dry_run` defaults to `true`.

### Auto-Streaming

Memory tools automatically stream large results to prevent context overflow. Streaming triggers when:
- Result count exceeds threshold (typically >15 items)
- Hardware pressure is detected

| Tool | Threshold | Streams |
|------|-----------|---------|
| `memory_list` | >15 items | Memory items |
| `memory_search` | >50 items | Search results |
| `memory_stale` | >15 items | Stale memories |
| `memory_ready` | >15 items | Ready tasks |
| `memory_reflections` | >15 items | Reflections |
| `memory_decisions` | >15 items | Decisions |
| `memory_stats` | >15 detail items | Stats details |
| `memory_context` | >15 items | Context data |
| `memory_cache_status` | >15 candidates | Eviction candidates |

**Control streaming:**
- `stream: true` - Force streaming regardless of size
- `stream: false` - Disable auto-streaming

When streaming, the tool returns a `stream_id`. Use `stream_read(stream_id)` to read chunks.

---

## Quick Reference

**Save user info:** `memory_add(type="preference", title="User's name is X")`

**Log correction:** `memory_reflect(reflection_type="correction", content="...")`

**Create plan:** `memory_add(type="plan", title="...")` then add steps with `parent_id`

**Track progress:** `memory_update(id="mem-xxx", status="done")`

**Find patterns:** `memory_list(type="pattern", status="active")`

**Search memories:** `memory_search(query="authentication")`

**Archive memory:** `memory_archive(id="mem-xxx")` (preserves links, excluded from list by default)

**Synthesize:** `memory_synthesize(sources=["mem-a","mem-b"], title="...", summary="...")`

**Expand synthesis:** `memory_expand(id="mem-synthesis-xxx")` (shows constituent sources)

**View instances:** `memory_instances(id="mem-xxx")` (all occurrences of a canonical)

**View layers:** `memory_network(mode="layers")` (3D topology with canonicals and instances)
