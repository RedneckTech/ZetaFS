# ZetaFS — Software Design Document

## 1. Overview

**ZetaFS** is a POSIX-mountable tape library filesystem optimized for multi-drive LTO libraries, providing both high performance and maximum space utilization. Written in Zig.

### Design Goals

1. **Any LTO gen, any drive count** — auto-detect drive capabilities, scale from 1 to N drives
2. **Fast mount** — milliseconds (metadata on SSD) or seconds (binary tape snapshot), never minutes
3. **Maximum space** — compaction GC recovers deleted space; small files packed into shared blocks
4. **Self-describing** — tapes readable on any ZetaFS system without external metadata databases
5. **Configurable metadata** — SSD-primary mode (speed) or tape-only mode (portability), chosen at format time

### Architecture Diagram

```
┌────────────────────────────────────────────────────┐
│                   Application                        │
│                   (POSIX I/O)                        │
└────────────────────┬───────────────────────────────┘
                     │ FUSE
┌────────────────────▼───────────────────────────────┐
│              ZetaFS Daemon (userspace)               │
│                                                       │
│  ┌──────────────┐  ┌──────────────────────────┐     │
│  │ FUSE ops     │  │  I/O Pipeline             │     │
│  │ (getattr,    │  │                           │     │
│  │  read, write,│  │  Write: app → NVMe buffer │     │
│  │  readdir...) │  │  → batch → tape segments  │     │
│  └──────┬───────┘  │                           │     │
│         │          │  Read: app → cache → tape │     │
│  ┌──────▼───────┐  │  → stage to warm tier     │     │
│  │ RocksDB       │  └───────────┬───────────────┘     │
│  │ (metadata)    │              │                      │
│  │               │  ┌──────────▼───────────────┐     │
│  │ /→dir→file    │  │  Scheduler                │     │
│  │ extents       │  │  (drive pool + queues)   │     │
│  │ dead bytes    │  └──────────┬───────────────┘     │
│  └───────────────┘             │                      │
└────────────────────────────────┼──────────────────────┘
                                 │ SCSI (sg driver)
┌────────────────────────────────┼──────────────────────┐
│        Tape Library             │                      │
│  ┌──────┐ ┌──────┐  ┌──────┐  │  ┌────────────────┐  │
│  │Drive1│ │Drive2│  │DriveN│  │  │ Robot (mtx/SMC)│  │
│  └──┬───┘ └──┬───┘  └──┬───┘  │  └────────────────┘  │
│     │        │         │       │                      │
│  Cartridge Pool: [0001][0002]...[NNNN]                │
└───────────────────────────────────────────────────────┘
```

---

## 2. On-Tape Format Specification

The on-tape format is the **same regardless of metadata mode**. This ensures tape interchangeability.

### 2.1 Tape Layout

Dual partition (LTO-5+ hardware partitions):

```
Partition 0 (Index Partition, ~0.1% of capacity):
  BOT → [Superblock][Snap 0][Snap 1]...[Snap N] → EOP

Partition 1 (Data Partition, ~99.9% of capacity):
  BOT → [Segment 0][Segment 1]...[Segment N] → EOP
```

Index snapshots are written sequentially on the Index Partition. When the Index Partition fills, the oldest snapshots are garbage collected (only the last N must be retained). Data segments are written sequentially on the Data Partition.

### 2.2 Superblock (Partition 0, Block 0)

Binary layout (big-endian):

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0 | 8 | `magic` | `0x5A455441465302` ("ZETAFSv1") |
| 8 | 4 | `format_version` | Currently 1 |
| 12 | 4 | `blocksize` | In bytes, default 1048576 (1 MiB) |
| 16 | 16 | `volume_uuid` | Random UUID v4 |
| 32 | 8 | `format_time_unix_ns` | Nanoseconds since Unix epoch |
| 40 | 1 | `compression` | 0 = raw, 1 = LTO hardware compression enabled |
| 41 | 1 | `metadata_mode` | 0 = SSD-primary, 1 = tape-only |
| 42 | 1 | `partition_count` | 2 |
| 43 | 5 | _reserved_ | Zeroed |
| 48 | 8 | `ip_size_blocks` | Index partition size in blocks |
| 56 | 8 | `dp_size_blocks` | Data partition size in blocks |
| 64 | 8 | `max_file_size` | 0 = unlimited |
| 72 | 184 | _reserved_ | Zeroed for future use |
| 256 | 32 | `checksum` | SHA-256 of bytes 0..256 |

Total: 288 bytes (padded to blocksize at write time).

### 2.3 Index Snapshot

An Index Snapshot is a contiguous sequence of blocks on the Index Partition. It contains the complete state of the filesystem at a point in time.

**Snapshot Header** (first 256 bytes of first block):

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0 | 4 | `magic` | `0x534E4150` ("SNAP") |
| 4 | 4 | `header_version` | 1 |
| 8 | 8 | `generation` | Monotonic counter, incremented per snapshot |
| 16 | 8 | `timestamp_unix_ns` | When this snapshot was written |
| 24 | 8 | `prev_gen_block` | Block of previous snapshot on same partition (0 = first) |
| 32 | 8 | `prev_gen_partition` | 0 or 1 |
| 40 | 4 | `num_dir_entries` | Number of directories |
| 44 | 4 | `num_file_entries` | Number of files |
| 48 | 4 | `num_symlink_entries` | Number of symlinks |
| 52 | 4 | `num_deleted_entries` | Number of tombstone entries |
| 56 | 4 | `num_xattr_entries` | Number of extended attribute entries |
| 60 | 4 | `total_blocks` | Total blocks in this snapshot (header + entries) |
| 64 | 2 | `volume_name_len` | Length of volume name (0-255) |
| 66 | 1 | `volume_lock_state` | 0 = unlocked, 1 = locked |
| 67 | 61 | _reserved_ | Zeroed |
| 128 | 128 | `volume_name` | UTF-8 volume name (padded with zeros) |

**Directory Entries** (immediately after header, packed sequentially):

```
struct DirEntry {
    dentry_id: u64,              // unique ID
    parent_id: u64,              // parent directory ID (0 = root)
    name_len: u16,               // bytes in name
    name: [name_len]u8,          // UTF-8 name (no separator, no null)
    mode: u16,                   // Unix permissions
    uid: u32,
    gid: u32,
    atime_ns: i64,
    mtime_ns: i64,
    ctime_ns: i64,
    btime_ns: i64,               // birth/create time
}
```

**File Entries**:

```
struct FileEntry {
    dentry_id: u64,
    parent_id: u64,
    name_len: u16,
    name: [name_len]u8,
    mode: u16,
    uid: u32,
    gid: u32,
    size: u64,                   // logical file size
    atime_ns: i64,
    mtime_ns: i64,
    ctime_ns: i64,
    btime_ns: i64,
    num_extents: u32,            // number of data extents
    extents: [num_extents]Extent,
}
```

**Symlink Entries**:

```
struct SymlinkEntry {
    dentry_id: u64,
    parent_id: u64,
    name_len: u16,
    name: [name_len]u8,
    mode: u16,
    uid: u32,
    gid: u32,
    target_len: u32,
    target: [target_len]u8,      // symlink target path
}
```

**Deleted/Tombstone Entries** (records which files/dirs were deleted since previous snapshot):

```
struct DeletedEntry {
    dentry_id: u64,              // the removed entry
}
```

**Extended Attribute Entry**:

```
struct XAttrEntry {
    dentry_id: u64,              // the entry this xattr belongs to
    key_len: u16,
    key: [key_len]u8,            // xattr key (e.g., "user.myattr")
    value_len: u32,
    value: [value_len]u8,        // xattr value (raw bytes)
}
```

**Extent**:

```
struct Extent {
    cartridge_id: u32,           // which cartridge (0 = this cartridge)
    partition: u8,               // 0 (Index) or 1 (Data)
    start_block: u64,            // first block of data
    byte_offset: u32,            // offset within start_block
    byte_count: u64,             // contiguous bytes
}
```

### 2.4 Data Segment

File data is stored in **segments** on the Data Partition. A segment may span multiple blocks.

**Segment Header** (first 128 bytes of first block of segment):

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0 | 4 | `magic` | `0x53454701` ("SEG" + version) |
| 4 | 4 | `segment_version` | 1 |
| 8 | 8 | `dentry_id` | File this data belongs to |
| 16 | 8 | `file_offset` | Offset in file this segment starts at |
| 24 | 8 | `byte_count` | Number of data bytes in this segment |
| 32 | 32 | `data_checksum` | SHA-256 of data bytes |
| 64 | 8 | `write_time_ns` | When this segment was written |
| 72 | 2 | `flags` | Bit 0: first segment of file, Bit 1: last segment of file |
| 74 | 2 | _reserved_ | |
| 76 | 8 | `next_extent_block` | Block of next segment for this file (0 = none) |
| 84 | 44 | _reserved_ | |

Data fills the remainder of the first block and continues into subsequent blocks. If `byte_count` is less than remaining space in the first block, the rest of that block is padded with zeros.

**Small file packing**: If a file's data fits in the remaining space of a block after another file's segment header, it goes there. The `file_offset` determines where in the logical file the data belongs.

### 2.5 Cartridge Lifecycle

```
[Format] → Active (read/write) → Full (read-only) → Compacting → Recycled
                ↓                                    ↑
             Deleted data tracked              Compaction rewrites
             as "dead bytes"                   live data to new cart
```

- **Active**: Accepting writes at append position
- **Full**: Cartridge has no more free space (EOD reached). Read-only for user data; compaction may still read from it
- **Compacting**: Cartridge being drained by compaction daemon
- **Recycled**: All data migrated off; cartridge can be reformatted

---

## 3. Metadata Architecture

### 3.1 RocksDB Schema (SSD Mode)

```
# Key → Value

# Directory tree
d:{dentry_id} → {parent_id, name, mode, uid, gid, timestamps}
f:{dentry_id} → {parent_id, name, mode, uid, gid, size, timestamps}
l:{dentry_id} → {parent_id, name, mode, uid, gid, target}

# Extents
e:{dentry_id}:{extent_idx} → {cartridge_id, partition, start_block, byte_offset, byte_count}

# Lookup by parent for readdir
c:{parent_id}:{name} → {dentry_id, type}  (type: 0=dir, 1=file, 2=symlink)

# Cartridge catalog
cart:{cartridge_id} → {uuid, slot, state, dead_bytes, format_time, total_capacity}

# Reverse: cartridge → dentries (for compaction planning)
rev:{cartridge_id}:{dentry_id} → {total_bytes}

# Global
next_dentry_id → u64
format_version → u32
```

### 3.2 Mode Comparison

| Aspect | SSD-primary (Mode 0) | Tape-only (Mode 1) |
|--------|---------------------|-------------------|
| **Mount time** | 5-50ms (RocksDB open) | 1-5s (seek + read latest snapshot) |
| **Dir listing** | 0.1ms (DB query) | 0.1ms + 0-3s (may need tape read) |
| **File create** | 0.1ms (DB write) | 0.1ms + deferred snapshot flush |
| **Delete** | 0.1ms (tombstone in DB) | marks in-memory, snapshot on flush |
| **Snapshot writes** | Periodic (configurable) | Every sync/unmount |
| **Portability** | Requires SSD + tape | Tape-only, truly self-describing |
| **Crash recovery** | RocksDB WAL + replay | Re-scan from latest on-tape snapshot |

---

## 4. Write Path

### 4.1 Write Buffer (NVMe)

```
Application write(fd, buf, count)
  → FUSE → ZetaFS daemon
    → Allocate in NVMe buffer (hot tier)
    → Copy data from buf
    → Return to application (10-50μs)
```

Buffer is a **journaled ring buffer** on NVMe:
- Segmented into 256 MiB chunks, each with a journal entry
- On flush: read buffer chunk → create tape segment(s) → write to tape → commit journal entry → mark buffer free
- Crash recovery: replay uncommitted buffer entries on restart

### 4.2 Flush Pipeline

```
Flush trigger: buffer ≥ 256 MiB OR timeout (5s) OR fsync()
  → Scheduler picks least-loaded drive
  → If cartridge not loaded → instruct robot (async)
  → Seek to append position (once per batch)
  → Write all buffered data as contiguous segment(s)
  → Update RocksDB with new extents
  → Commit NVMe journal → free buffer
  → If interval since last snapshot exceeds threshold → write Index Snapshot
```

### 4.3 Segment Allocation Logic

- **File < (blocksize - 128 bytes)**: Pack into existing block gap if available; else allocate new segment
- **File < 64 MiB**: Single segment, contiguous blocks
- **File ≥ 64 MiB**: Multiple segments, chained via `next_extent_block`
- **Append to existing file**: New segment after current append position; update file's extent list

---

## 5. Read Path

### 5.1 Cache Hierarchy

```
Application read(fd, buf, count)
  → Hot tier (NVMe write buffer): check if data still resident
    → Hit: copy from NVMe buffer (5-10μs)
  → Warm tier (SSD/HDD): check if staged from prior read
    → Hit: copy from warm tier (100μs - 5ms)
  → Tape: miss
    → RocksDB lookup → cartridge, partition, blocks
    → Submit to read scheduler
    → Wait for cartridge load + seek + read
    → Data written to warm tier + returned to app
```

### 5.2 Read Scheduler

```
Read Queue (priority-ordered):
  - Priority 0: Application blocking reads
  - Priority 1: Prefetch (application-hinted via fadvise)
  - Priority 2: Read-ahead (speculative, sequential detection)

Per-drive state machine:
  IDLE → LOADING → SEEKING → READING → RETURNING → IDLE
```

### 5.3 Prefetch Heuristics

- **Sequential detection**: If a file is read sequentially for >4 blocks, prefetch the next 64 blocks into warm tier
- **Adjacent file prefetch**: If directory listing shows a pattern, prefetch nearby files
- **Application hint**: `posix_fadvise(fd, offset, len, POSIX_FADV_WILLNEED)` triggers staged read

---

## 6. Multi-Drive Scheduler

### 6.1 Drive Pool

```
struct DrivePool {
    drives: []Drive,
    robot: Robot,

    fn allocate(cartridge_id: u32, op: Operation) !Drive {
        // 1. Find a drive with this cartridge already loaded
        // 2. If none, find idle drive → issue robot load
        // 3. If all busy, queue and wait
    }
}
```

Drive states: `Offline`, `Empty`, `Loaded(uuid, append_pos)`, `Reading`, `Writing`, `Loading`, `Unloading`, `Cleaning`

### 6.2 Scheduler Queues

```
Three priority levels of I/O:
  1. User reads (foreground, low latency needed)
  2. User writes (background, batched)
  3. Compaction (lowest priority, yields to 1 and 2)

Policy: drives prioritize reads → writes → compaction
         idle drives pick up compaction work
```

### 6.3 Robot Control

Interface via `mtx` CLI or direct SMC (SCSI Medium Changer) commands:

```
const Robot = struct {
    fn load(slot: u32, drive: u8) !void;
    fn unload(drive: u8) !void;
    fn inventory() !Inventory;
};
```

---

## 7. Space Management

### 7.1 Dead Space Tracking

```
RocksDB tracks per-cartridge:
  - total_bytes_written (sum of all segments ever written)
  - dead_bytes (sum of segments belonging to deleted/overwritten files)
  - live_ratio = (total - dead) / total
```

### 7.2 Compaction

Triggered when a cartridge crosses `dead_ratio_threshold` (default 0.40):

```
1. Query RocksDB for all LIVE extents on cartridge C
2. Allocate a fresh cartridge D from pool
3. Read all live extents sequentially from C
4. Write to D sequentially (new segments, new extents)
5. Update RocksDB: all dentries now point to cartridge D
6. Mark cartridge C as "recycled" → return to pool for re-format
```

Compaction runs as a background daemon with configurable I/O bandwidth limits and time windows. It yields to user reads/writes.

### 7.3 Small File Packing Efficiency

LTFS wastes an entire block per small file. ZetaFS packs segments densely:

```
Block N:
  [SegmentHeader for file A (128B)][data for file A (10KB)]
  [SegmentHeader for file B (128B)][data for file B (4KB)]
  [zero padding to end of block]

Block N+1:
  [SegmentHeader for file C (128B)][data for file C (500KB)]
  ...
```

A segment header's `byte_offset` within the file allows a segment to start mid-block after a previous segment's data ends. This eliminates the per-file block overhead.

---

## 8. POSIX Interface (FUSE)

### 8.1 Supported Operations

| Operation | Implementation |
|-----------|----------------|
| `getattr` | RocksDB lookup → fill `struct stat` |
| `readdir` | Query `c:{parent_id}:*` → list children |
| `lookup` | `c:{parent_id}:{name}` → dentry |
| `open` | No-op (metadata already cached) |
| `create` | Allocate dentry_id → insert in DB → write buffer (NVMe) |
| `read` | Extent list → read scheduler → copy to user |
| `write` | Write buffer (NVMe) → deferred tape flush |
| `release`/`flush` | If `O_SYNC` or last handle → trigger flush |
| `fsync` | Force flush of file's buffered data + write index snapshot |
| `truncate` | Extend only (sparse). Shrink returns `ENOTSUP` |
| `unlink` | Tombstone in DB. Data on tape marked dead |
| `rename` | Metadata-only (update parent + name in DB) |
| `mkdir`/`rmdir` | DB insert/delete (rmdir requires empty dir) |
| `symlink`/`readlink` | DB insert + read |
| `chmod`/`chown`/`utimens` | DB update |

### 8.2 Tape-Specific Semantics (Documented)

- **`write()` always appends** new extents; existing data on tape is never overwritten
- **`truncate()` can only extend** a file; shrinking returns `ENOTSUP` if data exists on tape
- **No `mmap()`** — tape data is not page-addressable
- **`unlink()` does not reclaim space** until compaction runs
- **`rename()` is instant** (metadata only, no data movement)

### 8.3 FUSE Configuration

- `direct_io`: Enabled — bypass kernel page cache
- `max_read = blocksize` (1 MiB): Minimize FUSE round trips
- `async_read`: Enabled — prefetch while app processes current buffer
- `atomic_o_trunc`: Not supported — tape data cannot be truncated to zero

---

## 9. Implementation Plan

### 9.1 Module Tree

```
src/
├── main.zig              # Daemon entry point, CLI arg parsing
├── tape/
│   ├── scsi.zig           # SCSI command layer (sg_io ioctl)
│   ├── drive.zig          # Drive state machine
│   ├── partition.zig      # Partition layout, LUN management
│   └── robot.zig          # Library robotics (mtx/SMC interface)
├── layout/
│   ├── superblock.zig     # Superblock read/write/verify
│   ├── snapshot.zig       # Index snapshot encode/decode
│   ├── segment.zig        # Data segment read/write
│   └── extent.zig         # Extent manipulation
├── metadata/
│   ├── db.zig             # RocksDB wrapper (C FFI)
│   ├── tree.zig           # Directory tree operations
│   ├── objects.zig        # File/dir/symlink CRUD
│   └── cartridges.zig     # Cartridge catalog + dead tracking
├── scheduler/
│   ├── pool.zig           # Drive pool + allocation
│   ├── read_queue.zig     # Read scheduling + prefetch
│   ├── write_queue.zig    # Write coalescing + flush pipeline
│   └── compaction.zig     # GC orchestration
├── cache/
│   ├── hot.zig            # NVMe write buffer management
│   └── warm.zig           # SSD/HDD read cache tier
├── fuse/
│   ├── ops.zig            # FUSE operation dispatch
│   └── fs.zig             # ZetaFS VFS adapter
├── proto/
│   └── zetafs.proto       # gRPC service (for admin/monitoring)
└── utils/
    ├── mkzetafs.zig       # Format a tape (like mkfs)
    ├── zetafsck.zig       # FS check and repair
    └── zetafs_compact.zig # Manual compaction trigger
```

### 9.2 Phase Roadmap

| Phase | Scope | Key Modules |
|-------|-------|-------------|
| **P0: Core Format** | Superblock + segment R/W + file backend emulator + SCSI layer | `tape/scsi.zig`, `layout/` |
| **P1: FUSE Metadata** | FUSE ops + RocksDB metadata + single-drive read/write | `fuse/`, `metadata/` |
| **P2: Indexing** | Index snapshot encode/decode + periodic write + mount recovery | `layout/snapshot.zig` |
| **P3: GC** | Dead space tracking + compaction daemon | `scheduler/compaction.zig` |
| **P4: Multi-Drive** | Drive pool + read/write queues + robot control | `scheduler/pool.zig`, `tape/robot.zig` |
| **P5: Polish** | Error recovery, fsck, config, man pages, CI, packaging | `utils/` |

### 9.3 Testing Strategy

- **Unit tests**: Each module in isolation
- **File backend**: Emulate tape as files on disk (like LTFS `file` backend) for CI
- **Integration**: Full stack test with file backend + FUSE mount
- **Hardware CI**: Periodic real drive tests
- **Fuzz testing**: Random I/O patterns, crash/power-loss simulation

### 9.4 Zig Bindings Required

| Interface | Approach |
|-----------|----------|
| SCSI `sg_io` ioctl | `@cImport` from `<linux/bsg.h>` / `<scsi/sg.h>` |
| FUSE (`libfuse3`) | `@cImport` from `<fuse3/fuse.h>` |
| RocksDB | `@cImport` from `<rocksdb/c.h>` |
| `mtx` CLI | `std.ChildProcess.exec` (fallback for SMC) |

---

## 10. Open Design Decisions

1. **Partition layout**: **Dual** — Index Partition for snapshots, Data Partition for segments. Matches LTO hardware partition support (LTO-5+).

2. **Snapshot encoding**: **Custom binary** — compact, simple to parse, no serialization dependency.

3. **Write buffer crash safety**: **Journaled NVMe buffer** — fast with crash recovery. Config option for synchronous mode (no buffer).

4. **Maximum file size**: **Unlimited** by default. Large files get multiple chained segments.

5. **Directory depth limit**: **None** — tree stored in DB, depth limited only by path length.
