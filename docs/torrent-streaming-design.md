# Torrent streaming architecture

Status: design baseline for `feature/torrent-streaming`.

## Goals

- Search supported torrent indexers and normalize results into one release model.
- Stream a selected media file through a loopback HTTP endpoint that supports byte ranges and seeking.
- Fetch only the pieces needed for startup and playback; never require the complete media file before playback.
- Bound temporary torrent storage by an explicit disk budget.
- Remove temporary data deterministically after a stream ends and recover abandoned sessions after crashes.
- Keep existing provider playback and download behavior unchanged.

Only use torrents and media that the user is authorized to access.

## Important storage distinction

Torrent streaming is not zero-download: pieces must be received and verified before they can be served. The design must bound the **physical storage consumed by downloaded pieces**, not just the amount fetched ahead of the playback cursor.

Some torrent engines represent a multi-gigabyte torrent as a full-length file with sparse/unwritten regions. A 4 GB logical file can therefore appear to be 4 GB in file listings even when only a smaller number of blocks are allocated. We must verify the selected engine's allocation behavior on supported filesystems and must not claim a hard cache cap based only on a logical file length or a progress-buffer setting.

## Proposed components

1. **Indexer adapters** — independent API/site adapters for the selected indexers. They return normalized metadata (title, magnet or torrent URL, info hash when available, size, seeders/leechers, resolution, codec, source).
2. **Release normalizer and ranker** — deduplicates by info hash and ranks using user preferences, resolution, codec, size, and swarm availability. Seed counts are hints, not guarantees of playback quality.
3. **Torrent engine adapter** — isolates the application from the chosen Rust engine and exposes metadata, file selection, byte-range reads, priorities, progress, pause, and removal.
4. **Piece/cache coordinator** — owns the global and per-session byte budgets, pins pieces currently being read and pieces needed for the active playback window, and evicts only verified, unpinned pieces.
5. **Loopback stream server** — serves the selected file with HTTP Range support to mpv/VLC. Bind only to loopback, use an unguessable session token, and reject requests outside the selected torrent/file.
6. **Session lifecycle manager** — creates a unique temporary directory per session, records a manifest, cancels network activity, stops the engine, closes readers, and purges session data on completion/cancel/error. Startup recovery removes abandoned temporary sessions.

## Cache policy

Initial proposed defaults (subject to engine and filesystem validation):

- Temporary cache budget: 512 MiB global.
- Per-session ceiling: 384 MiB.
- Startup buffer target: 32 MiB.
- Forward buffer target: 128 MiB.
- Persistent torrent cache: disabled.
- Maximum concurrent torrent streams: 2.
- Cleanup: when playback ends, is cancelled, or fails.

The global budget is authoritative. Per-session budgets must not sum to more than the global budget. Metadata and indexer response caches are separate from media-piece storage and must not be counted as torrent payload.

When the cache is full, the coordinator first evicts eligible old pieces. It must never evict a piece being read or needed to satisfy an outstanding range request. If no safe eviction is possible, it applies backpressure (pause/defer piece requests) and reports buffering/space pressure; it must not silently grow beyond the configured budget.

Seeking changes the protected playback window. Pieces around the new requested byte range become high priority; old pieces become evictable after active reads release them.

## Cleanup and crash recovery

- Each temporary session is stored beneath a dedicated torrent-cache root, never mixed with the user's download directory.
- A session manifest records session ID, info hash, selected file, creation time, storage mode, and lifecycle state. It must not contain secrets.
- Normal shutdown: stop accepting HTTP requests, cancel pending reads, stop/pause the torrent, close handles, then remove the session directory.
- Crash recovery: at startup, inspect only directories owned by this application and remove stale temporary sessions that are not marked active by a live process. Use a lock/lease or PID plus process-start identity to avoid deleting another live process's session.
- Cleanup errors are logged and surfaced in storage diagnostics; cleanup is retried on the next startup.
- A user-retained/download mode is a separate explicit feature and is not part of temporary streaming defaults.

## Engine selection gate

Before adding a torrent-engine dependency, verify:

- License compatibility with this project's MIT OR Apache-2.0 licensing.
- Whether the library can serve byte ranges while pieces are arriving.
- Whether it supports seeking and prioritizing requested pieces.
- Whether downloaded pieces can be stored in a bounded piece store, or whether it necessarily creates a full-length output file.
- Actual physical disk allocation behavior on Windows, Linux, macOS, and Android/Termux where supported.
- Safe removal semantics for incomplete torrents and in-flight readers.

A library that streams well but cannot guarantee bounded physical storage is not sufficient by itself. Do not advertise a hard cache limit until the storage backend enforces it.

## Implementation sequence

1. Confirm engine/license/storage semantics and define an engine-neutral API.
2. Implement and test temporary-session ownership, cache accounting, safe eviction, and crash cleanup independently of the engine.
3. Add engine integration and byte-range loopback streaming.
4. Add indexer adapters and normalized release ranking.
5. Add TUI search, source selection, buffering/progress, cancellation, and cache diagnostics.
6. Run formatting, clippy, unit tests, and end-to-end playback/seek/cleanup tests on supported platforms.
