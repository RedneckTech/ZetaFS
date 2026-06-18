# ZetaFS

A high-performance, space-efficient tape library filesystem for LTO drives. Written in Zig.

ZetaFS is a POSIX-mountable filesystem designed for large LTO tape libraries. It replaces LTFS with a binary metadata format, native multi-drive scheduling, compaction-based garbage collection, and configurable metadata placement (SSD-primary for speed or tape-only for portability).

## Design Goals

- **Any LTO gen, any drive count** — auto-detect drive capabilities, scale from 1 to N drives
- **Fast mount** — milliseconds (metadata on SSD) or seconds (binary tape snapshot), never minutes
- **Maximum space** — compaction GC recovers deleted space; small files packed into shared blocks
- **Self-describing** — tapes readable on any ZetaFS system without external metadata databases
- **Configurable metadata** — SSD-primary mode (speed) or tape-only mode (portability), chosen at format time

## Architecture

```
Application (POSIX I/O)
       │ FUSE
┌──────▼──────────────────────┐
│      ZetaFS Daemon           │
│  ┌──────────┐ ┌──────────┐  │
│  │ FUSE ops │ │ I/O Pipe │  │
│  └────┬─────┘ └────┬─────┘  │
│  ┌────▼────┐ ┌─────▼─────┐  │
│  │ RocksDB │ │ Scheduler  │  │
│  │ (meta)  │ │(drive pool)│  │
│  └─────────┘ └─────┬─────┘  │
└────────────────────┼────────┘
                     │ SCSI
┌────────────────────┼────────┐
│   Tape Library      │        │
│ ┌──────┐ ┌──────┐  │ Robot  │
│ │Drive1│ │DriveN│  │ (mtx)  │
│ └──┬───┘ └──┬───┘  │        │
│ Cartridge Pool      │        │
└─────────────────────┴────────┘
```

## On-Tape Format

Binary layout, not XML. Dual-partition (LTO-5+):

- **Partition 0 (Index)**: Superblock + binary index snapshots
- **Partition 1 (Data)**: File data segments with dense small-file packing

All metadata is stored as compact binary structs. Index snapshots use fixed-size headers and entries for O(1) parsing. See [SDD.md](SDD.md) for the full format specification.

## Status

Pre-alpha. Nothing works yet.

## Building

_Coming soon._

## License

GNU General Public License v3.0 — see [LICENSE](LICENSE).
