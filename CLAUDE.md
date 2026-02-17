# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Status: ARCHIVED

This project is **archived and superseded** by [macos-UnifiedLogs](https://github.com/mandiant/macos-UnifiedLogs) (Rust). Python processing of Unified Logs is slow. Only make changes here if specifically needed.

## Overview

A Python library and CLI tool for parsing macOS/iOS Unified Logging `.tracev3` files. Tested on macOS 10.12.5–10.15 and iOS 12.x. Does not support catalog v2 (macOS 10.12.0).

## Commands

```bash
pip install -r requirements.txt    # Install deps (lz4, biplist, ipaddress)
python run_tests.py                # Run unit tests (unittest discovery)
tox                                # Run tests via tox (py2, py3 envs)
```

**CLI usage:**
```bash
python scripts/UnifiedLogReader.py <uuidtext_path> <timesync_path> <tracev3_path> <output_path> [-f SQLITE|TSV_ALL|LOG_DEFAULT] [-l INFO|DEBUG|WARNING|ERROR]
```

Required input paths:
- `uuidtext_path` — `/var/db/uuidtext` (or `.logarchive` root)
- `timesync_path` — `/var/db/diagnostics/timesync`
- `tracev3_path` — single `.tracev3` file or `/var/db/diagnostics` folder

## Architecture

### Entry Point

`scripts/UnifiedLogReader.py` — CLI script that:
1. Reads timesync files via `UnifiedLogLib.ReadTimesyncFolder()`
2. Reads DSC (shared cache strings) and UUID text files
3. Iterates tracev3 files, creating `TraceV3` parser instances
4. Writes output via `OutputWriter` implementations: `SQLiteDatabaseOutputWriter` or `FileOutputWriter` (TSV_ALL / LOG_DEFAULT modes)

### Core Library (`UnifiedLog/`)

- **`Lib.py`** (`UnifiedLogLib`) — Main library with top-level functions: `ReadTimesyncFolder()`, `ReadDscFiles()`, `ReadAPFSTime()`, `DecompressTraceV3()`. Handles LZ4 decompression of chunk data and timesync boot record parsing.

- **`tracev3_file.py`** (`TraceV3`) — The core parser (73KB). Extends `BinaryDataFormat`. Parses tracev3 binary format including headers, catalog chunks, chunkset chunks, firehose log entries, oversize entries, simpledump, and statedump. Contains the printf-style format string regex pattern and message reconstruction logic. Takes a `VirtualFileSystem`, `VirtualFile`, timesync list, and UUID text path.

- **`dsc_file.py`** (`Dsc`) — Shared-Cache strings parser. Parses `hcsd`-format files containing range entries and UUID entries mapping virtual offsets to library paths/names.

- **`uuidtext_file.py`** (`Uuidtext`) — UUID text file parser. Extracts format strings from per-binary string tables.

- **`data_format.py`** (`BinaryDataFormat`) — Base class providing `_ReadAPFSTime()` and `_ReadCString()` helpers.

- **`resources.py`** — Data classes: `Catalog`, `ChunkMeta`, `ProcInfo`, `ExtraFileReference`, `SubSystemCategory`, `CachedFiles`, `TimeSyncHeader`, `TimeSyncBoot`.

- **`virtual_file_system.py`** / **`virtual_file.py`** — Abstraction layer over filesystem operations. `VirtualFileSystem` wraps `os.path.exists()`, `os.listdir()`, etc. `VirtualFile` wraps file open/read/close. Both can be subclassed for in-memory or remote file access.

- **`logger.py`** — Thin wrapper around Python `logging` module.

### Key Design Patterns

- **Virtual filesystem abstraction** — All file I/O goes through `VirtualFileSystem` and `VirtualFile` classes, allowing the parser to work with non-local file sources by subclassing
- **Log entry format** — Entries are Python lists (not objects) with 23 positional fields: `[SourceFile, SourceFilePos, ContinuousTime, TimeUtc, Thread, Type, ActivityID, ParentActivityID, ProcessID, EUID, TTL, ProcessName, SenderName, Subsystem, Category, SignpostName, SignpostInfo, ImageOffset, SenderUUID, ProcessImageUUID, SenderImagePath, ProcessImagePath, Message]`
- **Oversize data cache** — Large strings that don't fit in normal log entries are cached in a dictionary keyed by `(data_ref_id << 64 | contTime)`, passed between tracev3 file parse calls

### Tests

Tests are in `tests/` using `unittest`. Test data fixtures in `test_data/` (a single tracev3 file and two UUID text files). Run via `python run_tests.py` which discovers `tests/*.py`.
