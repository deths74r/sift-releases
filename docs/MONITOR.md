# sift --monitor

Real-time terminal dashboard showing sift's hardware state, database activity, and resource events.

## Usage

```bash
sift --monitor
```

Requires an interactive terminal. Press `q` to quit.

## Display Sections

### Header
```
sift monitor  15:42:38  [1] Activity
```
Shows current time and active view mode.

### Hardware State
```
Memory:   NORMAL    PSI some:  0.0%  full:  0.0%  avail: 28.2 GB  rss: 11 MB
I/O:      NORMAL    PSI some:  0.1%  full:  0.0%  read: 1.2 MB  write: 892 KB
```
- **State**: NORMAL (green), ELEVATED (yellow), CRITICAL (red), SURVIVAL (magenta+blink)
- **PSI**: Pressure Stall Information from `/proc/pressure/memory` and `/proc/pressure/io`
- **avail/rss**: Available memory and process resident set size
- **read/write**: I/O bytes from `/proc/self/io`

### Database Stats
```
Databases
  memory.db    4.2 MB  mem:3951  ref:48  pat:194  dec:16
  context.db   1.8 MB  sess:88   msg:27K links:3596
  workspace.db 12.4 MB files:307 lines:142K
```
Shows size and row counts for each database. Color coded:
- Cyan: memory.db
- Yellow: context.db  
- Green: workspace.db

### Activity Feed (View 1)
```
Activity Feed (32 entries)
  15:42:37  M  search     "sift_monitor"           12ms
  15:42:35  M  add        mem-a1b2c3 (note)        8ms
  15:42:34  C  user       Design the...            
  15:42:32  M  get        mem-b4c5d6               3ms
```
Real-time streaming of database operations:
- `M` (cyan): Memory DB operations from `activity_log`
- `C` (yellow): Context DB messages
- Latency shown in red if >100ms

### Tool Rates
```
Tool Rates (60s)  avg: 14ms  budget: 98%
 read:12 search:8 memory_*:15
```
Tool call counts and average latency over the last 60 seconds.

### Streams
```
Streams  2 active
  abc123: 45% written=1.2MB DONE
  def456: 12% written=256KB
```
Active ring buffer streams in `/dev/shm/sift-stream-*`.

### Patterns
```
Patterns: 194 total  [stats,search] -> get (60%)
```
Tool sequence predictions from the access pattern learner.

### Events (View 4)
```
Events (10 recent)
  15:42:30  state_change  memory   normal → elevated
  15:42:28  adaptation    io       reduced batch size
```
Resource events from `resource_events` table.

## Keyboard Controls

| Key | Action |
|-----|--------|
| `q` | Quit |
| `1` | Activity feed view (default) |
| `2` | Memory DB detail view |
| `3` | Context DB detail view |
| `4` | Events view |
| `j` | Scroll activity feed down |
| `k` | Scroll activity feed up |
| `c` | Clear activity feed and events |
| `s` | Start stress test (then press `m` or `i`) |
| `S` | Stop stress test |

## Stress Testing

The monitor includes built-in stress testing to observe how sift responds to resource pressure.

### Starting a Stress Test

1. Press `s` to enter stress mode
2. Press `m` for **memory stress** or `i` for **I/O stress**

### Memory Stress (`s` then `m`)
Runs `stress-ng --vm 2 --vm-bytes 50% --timeout 30s`:
- Spawns 2 VM stressor workers
- Each allocates/deallocates 50% of available memory
- Runs for 30 seconds
- Watch PSI `some` and `full` metrics rise
- Watch state transition: NORMAL → ELEVATED → CRITICAL

### I/O Stress (`s` then `i`)
Runs `stress-ng --iomix 4 --timeout 30s`:
- Spawns 4 mixed I/O workers (read/write/sync)
- Generates heavy disk activity
- Runs for 30 seconds
- Watch I/O PSI metrics and read/write bytes

### Stopping a Stress Test
Press `S` (capital S) to stop immediately. The indicator `*stress-m` or `*stress-i` in the help bar shows when stress is active.

### Requirements
Stress testing requires `stress-ng` to be installed:
```bash
# Ubuntu/Debian
sudo apt install stress-ng

# Fedora
sudo dnf install stress-ng

# macOS
brew install stress-ng
```

## Refresh Rates

| Data | Interval |
|------|----------|
| Hardware (PSI, memory, I/O) | 250ms |
| Activity feed | 500ms |
| Events | 1s |
| Tool rates | 2s |
| DB stats, Patterns | 5s |

## Data Sources

- `/proc/pressure/memory` - Memory PSI metrics
- `/proc/pressure/io` - I/O PSI metrics
- `/proc/meminfo` - Available memory
- `/proc/self/status` - Process RSS
- `/proc/self/io` - Process I/O counters
- `/dev/shm/sift-stream-*` - Ring buffer streams
- `.sift/memory.db` - Events, patterns, activity, tool calls
- `.sift/context.db` - Sessions, messages, links
- `.sift/workspace.db` - Indexed files and lines
