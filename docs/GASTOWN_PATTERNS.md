# Gastown-Inspired Enhancements for Sift

Analysis of patterns from [steveyegge/gastown](https://github.com/steveyegge/gastown) 
that could enhance sift's mech suit architecture.

## Background

Gastown is a multi-agent orchestration system for Claude (624 files, 205k lines of Go).
It solves **multi-agent coordination** while sift solves **single-agent augmentation**.
Some patterns transfer well.

## Summary

| Pattern | Value | Complexity | Status |
|---------|-------|------------|--------|
| **Keepalive** | Clear - smarter resource mgmt | Low | Implement |
| **Enhanced Patterns** | Clear - executable workflows | Medium | Implement |
| **Token Tracking** | Medium - visibility | Medium | Consider |
| **Ephemeral** | Unclear - archive may suffice | Low | Defer |

---

## Pattern 1: Keepalive / Activity Tracking

### Problem
Hardware layer doesn't know if Claude is actively working or idle.
Resources stay allocated even when unused.

### Solution
Track last tool call timestamp. Hardware layer uses this to:
- Release resources during idle (>5 min)
- Trigger maintenance during stale (>30 min)
- Inform `sift --monitor` display

### Implementation
```c
// sift_hardware.c
static struct {
    char last_tool[64];
    time_t last_activity;
    int tool_count;
} activity_state;

void hardware_signal_activity(const char *tool_name) {
    strncpy(activity_state.last_tool, tool_name, 63);
    activity_state.last_activity = time(NULL);
    activity_state.tool_count++;
}

int hardware_activity_age_seconds(void) {
    if (activity_state.last_activity == 0) return INT_MAX;
    return (int)(time(NULL) - activity_state.last_activity);
}

const char *hardware_last_tool(void) {
    return activity_state.last_tool;
}
```

### Integration Points
1. Call `hardware_signal_activity(tool_name)` at start of every MCP handler
2. `sift_hardware_status` includes activity age in response
3. `sift --monitor` shows "ACTIVE (2s ago)" vs "IDLE (5m)"
4. Resource decisions consider activity state

### Files to Modify
- `src/sift_hardware.c` - Add activity tracking functions
- `src/sift_hardware.h` - Declare functions  
- `src/sift.c` - Call from MCP tool handlers (50+ tools)
- `src/sift_memory.c` - Call from memory tool handlers
- `src/sift_monitor.c` - Display activity state

### Mech Suit Benefit
Resource management becomes predictive, not just reactive. The suit powers down 
when the pilot rests.

---

## Pattern 2: Enhanced Patterns (Executable)

### Problem
Patterns are documentation. Claude reads them and manually creates plans.
No automation, no variable substitution.

### Solution
Allow patterns to include structured steps that can be "poured" into plans.

### Current Pattern (unchanged)
```
sift_memory_add(
    type: "pattern",
    title: "Debug workflow", 
    description: "1. Reproduce 2. Analyze 3. Fix 4. Test"
)
```
This continues to work as documentation.

### Executable Pattern (new capability)
```
sift_memory_add(
    type: "pattern",
    title: "Debug workflow",
    description: "Standard debugging process",
    metadata: {
        executable: true,
        category: "debug",
        variables: [
            {name: "component", description: "What to debug", required: true},
            {name: "symptom", description: "What's wrong"}
        ],
        steps: [
            {id: "reproduce", title: "Reproduce {{symptom}} in {{component}}"},
            {id: "analyze", title: "Analyze root cause", needs: ["reproduce"]},
            {id: "fix", title: "Implement fix", needs: ["analyze"]},
            {id: "test", title: "Verify fix", needs: ["fix"]}
        ]
    }
)
```

### New Tool: sift_memory_pattern_pour
```
sift_memory_pattern_pour(
    pattern_id: "mem-xyz",           // Find by ID
    // OR pattern_name: "debug-workflow",  // Find by title
    vars: {component: "stream handling", symptom: "memory leak"}
)

→ Returns:
{
    plan_id: "mem-abc",
    pattern_used: "debug-workflow",
    steps_created: [
        {id: "mem-s1", title: "Reproduce memory leak in stream handling"},
        {id: "mem-s2", title: "Analyze root cause", needs: ["mem-s1"]},
        {id: "mem-s3", title: "Implement fix", needs: ["mem-s2"]},
        {id: "mem-s4", title: "Verify fix", needs: ["mem-s3"]}
    ]
}
```

### Helper Tool: sift_memory_pattern_list
```
sift_memory_pattern_list(executable: true, category: "debug")

→ Returns:
{
    patterns: [
        {id: "mem-xyz", title: "Debug workflow", category: "debug", 
         variables: ["component", "symptom"], step_count: 4},
        ...
    ]
}
```

### Claude-User Interaction

**Option A: Claude-Initiated (invisible)**
User: "Debug the memory leak in streams"
Claude recognizes task type, pours template automatically. User just sees 
structured plan appear.

**Option B: User-Initiated (power user)**
User: "Use the debug-workflow pattern for streams"
Claude pours explicitly requested template.

**Option C: Discovery (interactive)**
Claude: "I found some debug templates. Use 'debug-memory' (thorough) or 
'debug-issue' (general)?"

### Files to Modify
- `src/sift_memory.c` - Add `mcp_tool_memory_pattern_pour`, `mcp_tool_memory_pattern_list`
- `src/sift.c` - Register tools in MCP manifest

### Mech Suit Benefit
Pre-programmed combat maneuvers. One command creates structured plan with 
dependencies.

---

## Pattern 3: Token Tracking (Consider Later)

### Problem
No visibility into token usage per plan/task.

### Gastown Approach
Ledger entries track cost per session with attribution:
```go
type CostEntry struct {
    SessionID string
    Role      string
    CostUSD   float64
}
```

### Challenge for Sift
How does sift know token counts?
- **A) Claude reports** - Friction (Claude must self-report)
- **B) Estimate** - `strlen(text) / 4` (inaccurate but automatic)
- **C) Claude Code integration** - Needs investigation

### Potential Schema
```sql
CREATE TABLE token_usage (
    memory_id TEXT,
    session_id TEXT,
    tool_name TEXT,
    input_estimate INTEGER,
    output_estimate INTEGER,
    timestamp INTEGER
);
```

### Mech Suit Benefit
Fuel gauge. "This plan cost 30k tokens."

### Status
Defer until keepalive and patterns are working.

---

## Pattern 4: Ephemeral Memories (Defer)

### Problem
Scratch notes may pollute long-term memory and backups.

### Gastown Approach
"Wisps" are ephemeral issues with `ephemeral: true`:
- Not exported to backup
- Auto-cleaned by garbage collection
- Used for scratch work, debugging

### Counter-argument
Sift already has archive. "Don't save" is always an option.
The value of a separate ephemeral tier vs. archive is unclear.

### If Implemented
```
sift_memory_add(type: "note", title: "...", ephemeral: true)
```
- Not in backups
- Auto-archive after N days
- Lower search ranking

### Status
Defer. Revisit if real need emerges.

---

## Implementation Plan

### Phase 1: Keepalive
1. Add activity tracking to sift_hardware.c
2. Call from all MCP handlers
3. Integrate with monitor
4. Test idle/active detection

### Phase 2: Enhanced Patterns
1. Add pattern_pour tool
2. Add pattern_list tool
3. Implement variable substitution
4. Test with sample patterns

### Phase 3: Future
- Token tracking if estimation proves useful
- Ephemeral if real need emerges

---

## Verification Checklist

### Keepalive
- [ ] `sift_hardware_status` shows last_tool and activity_age
- [ ] `sift --monitor` shows ACTIVE/IDLE state
- [ ] 5 min idle → suggestions include resource release

### Enhanced Patterns
- [ ] Create executable pattern with variables and steps
- [ ] `sift_memory_pattern_list(executable: true)` finds it
- [ ] `sift_memory_pattern_pour(pattern_id, vars)` creates plan + steps
- [ ] Variables substituted in step titles
- [ ] Step dependencies linked correctly

---

## Gastown Patterns NOT Applicable to Sift

| Pattern | Why Not |
|---------|---------|
| Multi-agent routing | Sift is single-agent |
| Tmux session management | Not relevant to MCP |
| Git worktree isolation | Not sift's domain |
| Mail protocol | Single agent, no recipients |
| Watchdog chain | No external supervisor |
| Role taxonomy (Mayor/Deacon/Witness/etc.) | Multi-agent concept |
