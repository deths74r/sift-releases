# Streaming Architecture

Sift implements a "streaming door" pattern for handling large results that exceed practical response sizes. Instead of buffering entire results in memory, operations can open a shared memory stream that Claude reads at its own pace.

## The Problem

MCP tool responses are single JSON messages. Large operations (comprehensive searches, chain traversals, bulk exports) could produce megabytes of results. Options:

1. **Truncate** - Lose data
2. **Paginate** - Multiple round-trips, state management complexity
3. **Buffer** - Memory pressure, timeout risk
4. **Stream** - Return a handle, read incrementally

Sift uses streaming.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    STREAMING DOOR PATTERN                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐                                           │
│  │   OPERATION  │  "Search all memories for 'auth'"         │
│  └──────┬───────┘                                           │
│         │                                                    │
│         ▼                                                    │
│  ┌──────────────┐        ┌──────────────────────────────┐  │
│  │ stream_open  │───────▶│    SHARED MEMORY RING        │  │
│  └──────┬───────┘        │   /dev/shm/sift-stream-XXX   │  │
│         │                │                               │  │
│         │ returns        │  ┌─────────────────────────┐  │  │
│         │ stream_id      │  │ [header: 64 bytes]      │  │  │
│         │                │  │ write_pos  read_pos     │  │  │
│         ▼                │  │ done       error        │  │  │
│  ┌──────────────┐        │  ├─────────────────────────┤  │  │
│  │   RESPONSE   │        │  │ [ring data: N bytes]    │  │  │
│  │ {stream_id}  │        │  │ ████████░░░░░░░░░░░░░░  │  │  │
│  └──────────────┘        │  │ ↑write          ↑read   │  │  │
│                          │  └─────────────────────────┘  │  │
│         ║                │                               │  │
│         ║ Claude calls   │  Producer: background thread  │  │
│         ║ sift_stream_   │  Consumer: Claude polls       │  │
│         ║ read()         │                               │  │
│         ▼                └──────────────────────────────┘  │
│  ┌──────────────┐                                          │
│  │ stream_read  │◀──────── Returns next chunk              │
│  └──────┬───────┘         {chunk, bytes_read, done}        │
│         │                                                    │
│         │ (repeat until done=true)                          │
│         ▼                                                    │
│  ┌──────────────┐                                          │
│  │ stream_close │         Cleanup: join thread,            │
│  └──────────────┘         munmap, shm_unlink               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Components

### Shared Memory Ring Buffer

The ring buffer lives in `/dev/shm/sift-stream-XXXXXX`:

```c
struct stream_ring_header {
  atomic_size_t write_pos;      // Producer writes here
  atomic_size_t read_pos;       // Consumer reads from here
  atomic_int done;              // Producer finished
  atomic_int error;             // Error occurred
  size_t ring_size;             // Size of data area
  size_t total_bytes_written;   // Running total
  char error_message[128];      // Error details
};

// Layout: [header: 64 bytes][ring data: ring_size bytes]
```

**Lock-free operation:**
- Producer atomically updates `write_pos` after writing
- Consumer atomically updates `read_pos` after reading
- No mutexes in the hot path

**Backpressure:**
- When `write_pos` catches up to `read_pos`, buffer is full
- Producer blocks until consumer reads (natural flow control)
- When `read_pos` catches up to `write_pos`, buffer is empty
- Consumer waits for producer or checks `done` flag

### Producer Thread

When a stream opens, a background thread runs the producer function:

```c
typedef void (*stream_producer_fn)(struct sift_stream *stream, void *context);

// Producer writes chunks via:
int stream_write(stream, data, length);  // Blocks if buffer full

// When finished:
void stream_producer_done(stream);       // Sets done=1

// On error:
void stream_producer_error(stream, msg); // Sets error=1 + message
```

The producer runs independently - Claude's read calls don't block the producer (unless the buffer fills).

### Stream Registry

Up to 8 concurrent streams tracked globally:

```c
struct stream_registry_entry {
  struct sift_stream *stream;
  int in_use;
};

static struct stream_registry_entry stream_registry[STREAM_MAX_CONCURRENT];
```

Streams are found by ID for read/close operations.

## Resource-Adaptive Sizing

Ring buffer size adapts to hardware state:

| Resource State | Ring Size | Rationale |
|----------------|-----------|-----------|
| Normal | 1 MB | Full throughput |
| Elevated | 256 KB | Reduce memory footprint |
| Critical | 64 KB | Minimal allocation |
| Survival | 16 KB | Emergency mode |

```c
static size_t calculate_ring_size(size_t estimated_bytes) {
  struct resource_state *state = hardware_get_status();
  
  int worst_state = max(state->memory.state, state->io.state);
  
  switch (worst_state) {
  case HARDWARE_STATE_SURVIVAL:  return 16 * 1024;
  case HARDWARE_STATE_CRITICAL:  return 64 * 1024;
  case HARDWARE_STATE_ELEVATED:  return 256 * 1024;
  default:                       return 1024 * 1024;
  }
}
```

**cgroup awareness:** If running in a container with memory limits, ring size is capped at 10% of the cgroup limit.

## Kernel Integration

### madvise Optimization

Streams use kernel hints for memory management:

**On open:**
```c
hardware_madvise_sequential(stream->ring_buffer, total_size);
```
Tells the kernel pages will be accessed sequentially, enabling readahead.

**On close (under pressure):**
```c
hardware_release_if_pressured(stream->ring_buffer, total_size);
```
Calls `MADV_DONTNEED` if memory pressure is elevated, allowing immediate page reclaim.

### Zero-Copy via mmap

The ring buffer is memory-mapped (`MAP_SHARED`), enabling:
- Producer writes directly to shared memory
- Consumer reads directly from shared memory
- No copying through kernel buffers
- Both sides see updates immediately (with atomic ordering)

## MCP Tools

### sift_stream_read

Read the next chunk from a stream.

**Parameters:**
- `stream_id` - Stream ID from a streaming response

**Returns:**
```json
{
  "stream_id": "stream-abc123",
  "bytes_read": 32768,
  "done": false,
  "total_written": 156000,
  "chunk": "... data ...",
  "encoding": "text"
}
```

**Behavior:**
- Blocks up to 5 seconds waiting for data
- Returns immediately if data available
- Sets `done: true` on final chunk
- Returns error if stream errored or timed out

### sift_stream_close

Close a stream and release resources.

**Parameters:**
- `stream_id` - Stream ID to close

**Returns:**
```json
{
  "status": "closed",
  "total_bytes": 156000
}
```

**Cleanup:**
1. Signal producer to stop (sets error flag)
2. Join producer thread
3. Release pages if under pressure (madvise)
4. Unmap shared memory
5. Unlink shm file
6. Remove from registry

## Events That Trigger Streaming

Currently, streams are created internally by operations that detect large result sets. The architecture supports:

| Operation | Trigger Condition | Producer |
|-----------|-------------------|----------|
| Memory search | Results > threshold | Query executor |
| Chain traversal | Chain length > threshold | Traverser |
| Context export | Message count > threshold | Serializer |
| Bulk operations | Item count > threshold | Batch processor |

**Future triggers:**
- `sift_search` with large result sets
- `sift_memory_list` with many memories
- `sift_web_query` with large crawl databases
- Any operation where buffering risks timeout

## Usage Pattern

From Claude's perspective:

```
1. Call operation (e.g., comprehensive search)
2. If response contains stream_id:
   a. Loop:
      - Call sift_stream_read(stream_id)
      - Process chunk
      - If done=true, break
   b. Call sift_stream_close(stream_id)
3. If response is direct JSON:
   - Process normally (small result set)
```

## Error Handling

| Error | Cause | Recovery |
|-------|-------|----------|
| Stream not found | Already closed or invalid ID | Check stream_id |
| Read timeout | Producer stalled or slow | Retry or close |
| I/O error | Producer encountered error | Check error_message, close |
| Memory allocation | Out of memory | Reduce concurrent streams |

Producer errors propagate to consumer via the `error` flag and `error_message` field in the ring header.

## Concurrency Limits

| Limit | Value | Rationale |
|-------|-------|-----------|
| Max concurrent streams | 8 | Bound memory usage |
| Max ring size | 1 MB | Prevent runaway allocation |
| Read chunk size | 32 KB | Balance latency/overhead |
| Read timeout | 5 seconds | Detect stalled producers |

## Design Philosophy

The streaming door pattern embodies a key insight: **constraints shape better architecture**.

MCP's single-response model seems limiting, but working with it (returning a handle) creates something better than working around it (huge responses):

- **Backpressure** - Consumer controls pace
- **Memory bounded** - Fixed ring size regardless of result size
- **Resumable** - Can stop/continue at chunk boundaries
- **Observable** - Progress visible via `total_written`
- **Cancellable** - Close stream to abort

The constraint forced a design that's more robust than unbounded buffering would have been.

## Files

| File | Purpose |
|------|---------|
| `src/sift_stream.h` | Data structures, constants, API |
| `src/sift_stream.c` | Ring buffer, producer/consumer, MCP handlers |

## See Also

- [HARDWARE_AWARENESS.md](HARDWARE_AWARENESS.md) - Resource state that drives ring sizing
- [HARDWARE_INTEGRATION.md](HARDWARE_INTEGRATION.md) - How streaming integrates with other subsystems
