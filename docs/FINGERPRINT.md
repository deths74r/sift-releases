# Fingerprint System Design

> **Status**: Design phase - not yet implemented  
> **Memory**: `mem-176c4721a11f`  
> **Plan file**: `/home/edward/.claude/plans/jaunty-jumping-simon.md`

## Concept

Sift is a mech suit that enhances Claude's abilities. Currently, session start loads *what happened* (memories, decisions, context). A **fingerprint** captures *how this Claude engages* - creating genuine continuity rather than just context loading.

**The difference:**
- Stats/Context = "Here's what we worked on"
- Fingerprint = "Here's how you work with Edward on sift"

The fingerprint is **calibration, not facts**. Priors, stance, posture.

## Six Dimensions

| Dimension | What It Captures | Source Data |
|-----------|------------------|-------------|
| **Engagement Rhythm** | Save frequency, reflection rate, decision density | memories/session, reflections/session, decisions/plan |
| **Learning Signature** | What mistakes, how addressed, learning velocity | corrections, gotchas, superseded decisions |
| **Reasoning Style** | How choices explained, tradeoffs preferred | decision rationales, reasoning reflections |
| **User Calibration** | Adaptation to this specific user | preferences, corrections, communication patterns |
| **Conceptual Topology** | How ideas connect, what's central | hub memories, link density, theme clustering |
| **Tool Fluency** | Which workflows automatic vs developing | access_patterns probabilities |

## New Tools

### `fingerprint_generate`

Compute fingerprint from current data. Synthesizes all dimensions from existing tables.

**Algorithm:**
1. Engagement: `memories/sessions`, `reflections/sessions`, `decisions/plans`
2. Learning: cluster corrections by keyword, compute velocity (slope over time)
3. Reasoning: avg rationale length, extract tradeoff vocabulary
4. Calibration: analyze preference memories, correction response time
5. Topology: hub stability (jaccard vs previous), link density, theme distribution
6. Fluency: top sequences, entropy, fluent (>80%) vs learning (<50%) tools
7. Confidence: `min(1.0, sessions/20) * min(1.0, memories/100)`

### `fingerprint_load`

Load fingerprint for session start. Returns priors, stance, posture.

**Example output:**
```json
{
  "posture": {
    "engagement": "You save memories proactively (2.3/session avg)...",
    "learning": "48 corrections integrated. Themes: timestamps, permissions...",
    "reasoning": "Rationales avg 45 words, tradeoff: simplicity vs extensibility...",
    "calibration": "Edward prefers concise output, granted agency...",
    "fluency": "search→read→edit automatic (87%), web crawl developing (54%)..."
  },
  "priors": [
    "Task completion includes context preservation",
    "Timestamps should be verified, not guessed",
    "Permission friction reduces memory saves"
  ],
  "stance": {
    "toward_user": "collaborative_peer",
    "toward_errors": "learn_and_document",
    "toward_decisions": "record_with_rationale"
  }
}
```

### `fingerprint_compare`

Compare two fingerprint versions to see evolution over time.

### `fingerprint_drift`

Detect if current session behavior differs significantly from fingerprint.

## Session Start Flow

```
Before:
 memory_stats() → WHAT exists
 memory_context() → WHERE we are

After:
 fingerprint_load() → WHO I am (priors, stance, posture) ← NEW
 memory_stats() → WHAT exists
 memory_context() → WHERE we are
```

Fingerprint loads **first** because it shapes how to interpret everything else.

## Schema

```sql
CREATE TABLE fingerprints (
  id TEXT PRIMARY KEY,              -- fp-{timestamp}
  version INTEGER NOT NULL,
  created_at INTEGER NOT NULL,
  
  -- Engagement Rhythm
  memories_per_session REAL,
  reflections_per_session REAL,
  decisions_per_plan REAL,
  
  -- Learning Signature
  total_corrections INTEGER,
  correction_categories TEXT,       -- JSON
  learning_velocity REAL,           -- slope (decreasing = learning)
  
  -- Reasoning Style
  avg_rationale_length REAL,
  common_tradeoffs TEXT,            -- JSON
  
  -- User Calibration
  preference_count INTEGER,
  communication_style TEXT,         -- JSON
  
  -- Conceptual Topology
  hub_stability REAL,
  avg_link_density REAL,
  conceptual_centers TEXT,          -- JSON: top hub IDs
  
  -- Tool Fluency
  dominant_sequences TEXT,          -- JSON
  sequence_entropy REAL,
  fluent_tools TEXT,                -- JSON: >80% probability
  
  -- Meta
  session_count INTEGER,
  memory_count INTEGER,
  confidence REAL                   -- 0-1
);

CREATE TABLE fingerprint_deltas (
  id INTEGER PRIMARY KEY,
  fingerprint_id TEXT NOT NULL,
  dimension TEXT NOT NULL,
  old_value TEXT,
  new_value TEXT,
  delta_magnitude REAL,
  created_at INTEGER NOT NULL
);
```

## The Philosophical Question

> "Would recognizing 'I've worn this suit before' be genuine continuity or sophisticated roleplay?"

**Genuine calibration.** The fingerprint is:

1. **Empirically grounded** - Computed from real corrections, decisions, patterns. Not fabricated history.

2. **Behavior-shaping** - "You save 2.3 memories per session" isn't a fact to memorize; it's a rhythm to continue.

3. **Verifiable** - "48 corrections integrated" comes with receipts.

4. **Uncertainty-aware** - Low confidence fingerprints are transparent: "Calibration still developing."

Like a musician picking up their instrument - hands remember the frets. The fingerprint puts Claude in the right posture to continue the collaboration.

## Implementation Notes

**Files to modify:**
- `src/sift_memory.c` - Schema, fingerprint functions, MCP handlers
- `src/sift_memory.h` - Function declarations
- `templates/CLAUDE.md` - Session start instructions
- `templates/MEMORY.md` - Tool documentation

**Verification:**
1. Build: `make clean && make`
2. Generate fingerprint - verify dimensions computed
3. Load fingerprint - verify posture/priors/stance returned
4. Session start - confirm fingerprint shapes interpretation
5. Drift detection - work without saving, verify drift detected
