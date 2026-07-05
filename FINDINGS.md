# Archil on RWX — Performance Findings

Analysis of RWX runs exercising an [Archil](https://archil.com) shared disk
mounted inside RWX task containers:

- **Bulk throughput** — one 100 MiB file:
  [`234ae5ace1bf4a85831389b744e6ab3b`](https://cloud.rwx.com/mint/rwx/runs/234ae5ace1bf4a85831389b744e6ab3b)
- **Small files** — 12,000 × 1 KiB files:
  [`6a11cbe188d44e76bb569dc84ff1b88b`](https://cloud.rwx.com/mint/rwx/runs/6a11cbe188d44e76bb569dc84ff1b88b)

## Environment

| | |
|---|---|
| Archil client | `v0.8.18-1782513184` (proto Version2) |
| Disk | `rwx/dan-testing` (`dsk-0000000000016dc4`) |
| Region | `aws-us-east-1` |
| Mount mode | `--shared` (read-only by default; writes via dynamic ownership) |
| Runner OS | Ubuntu 24.04.4 LTS, kernel `6.17.0-1019-aws`, x86_64 |
| Base image | `ubuntu:24.04` + `rwx/base 1.0.2` |

## Bulk throughput — single 100 MiB file (`benchmark`)

Measured with `dd` on a 100 MiB random-data file in the `benchmark` task.

| Operation | Time | Throughput | Notes |
|---|---|---|---|
| **Write** (mount #1) | 0.499 s | **~210 MB/s** | includes `conv=fsync`; absorbed by local write cache |
| **Read** (mount #2, cold cache) | 4.413 s | **~23.8 MB/s** | fresh mount after unmount dropped the cache — true backend fetch |
| Integrity | — | ✅ SHA256 match | `df6f1b33…f9c66` identical for source and read-back |

The write/read asymmetry is expected and instructive:

- **Writes** land in Archil's local cache and return quickly; even with `fsync`
  the 100 MiB flush completed in ~0.5 s (210 MB/s).
- **Cold reads** are the honest number for this workload. The benchmark
  deliberately unmounts (flushing writes and dropping the local cache) and
  remounts before reading, so the 23.8 MB/s reflects pulling every byte from
  the backing store over the network with no cache assistance. A warm re-read
  (data already in cache) would be materially faster; that case was not
  measured in this run.

## Small-file throughput — many 1 KiB files (`benchmark-small-files`)

The `benchmark-small-files` task generates N × 1 KiB files locally (one blob
`split` into 1 KiB pieces — no per-file forks), copies them to Archil, unmounts
to flush + drop the cache, remounts, and reads everything back cold. A sha256
manifest doubles as the integrity check, so destination content is read exactly
once. Run with **12,000 files (≈ 11.7 MiB)**:

| Operation | Time | **MB/s** | files/s | Per-file |
|---|---|---|---|---|
| **Write** (create 12k files, `cp`) | 2.61 s | **4.49 MB/s** | 4,596 | ~0.22 ms |
| Flush (unmount) | 0.80 s | 14.69 MB/s | 15,039 | — |
| **Read — metadata walk** (`find`, cold) | 5.61 s | 2.09 MB/s\* | 2,141 | ~0.47 ms |
| **Read — full content** (cold, sequential) | 257.6 s | **0.05 MB/s** | 47 | ~21.5 ms |
| Integrity | — | — | — | ✅ hash + count match |

\* The metadata walk doesn't actually transfer file bytes; its MB/s is only the
nominal payload ÷ elapsed. `files/s` is the meaningful metric for that row.

The story here is **per-file latency, not bandwidth** — and the MB/s numbers show
just how steep the small-file penalty is versus the bulk 100 MiB benchmark:

| | Bulk (single 100 MiB file) | Small files (12k × 1 KiB) | Ratio |
|---|---|---|---|
| **Write** | ~210 MB/s | **4.49 MB/s** | ~47× slower |
| **Cold read** | ~23.8 MB/s | **0.05 MB/s** | **~475× slower** |

- **Writes are cheap** (~0.22 ms/file, 4.49 MB/s) because file creation lands in
  the local write cache — 12k files created in 2.6 s.
- **Cold sequential reads are dominated by round-trip latency** (~21.5 ms/file).
  Each small file is a separate backend fetch, so throughput collapses to
  **0.05 MB/s** — about 475× slower per byte than the ~23.8 MB/s bulk read.
- **Metadata is much cheaper than content** — a cold `find` walk stats ~2,100
  files/s (~0.47 ms/file), ~45× faster than reading their contents.

### Why not the full 100 MiB (102,400 files)?

An initial attempt at the full 100 MiB / 1 KiB workload (102,400 files) **timed
out at the default 10-minute task limit** — at ~21.5 ms/file the cold sequential
read alone projects to ~30 minutes. The task now sets `timeout: 30m`, and the
file count is tuned (`NUM_FILES`) to keep a single run within ~5 minutes; 12,000
files lands at ~4.6 min of task execution. To benchmark the full 100 MiB
practically, the cold read would need concurrency (e.g. `xargs -P`) or Archil
prefetch (below) to hide per-file latency — sequential access does not scale to
six-figure file counts here.

## Archil read-ahead / prefetch options

Archil has **no explicit sequential "read-ahead" flag** (the strings
"read-ahead"/"readahead" do not appear in its docs or CLI). The mechanism it
offers for hiding read latency is **prefetch** — proactively warming the local
cache — plus cache-sizing controls. These are the relevant knobs for the
small-file cold-read problem above:

| Option | Where | What it does |
|---|---|---|
| `--pre-fetch <paths>` | `archil mount` flag | Comma-separated paths warmed into cache right after mount (e.g. `--pre-fetch models,data/2024`). |
| `archil prefetch <mount> <path>…` | command | Warms specific paths post-mount. **Recurses into directories**, **returns immediately**, and continues fetching in the **background**. |
| `--max-cache-mb <MiB>` | `archil mount` flag | Cache ceiling. Default: min(¼ of RAM, 2048 MiB). Prefetched data counts against this; oversized sets evict earlier entries. |
| `--target-cache-mb <MiB>` | `archil mount` flag | Steady-state cache target. Default: 75% of max. |
| `archil set-cache-expiry` | command | Tunes how long `readdir` results are cached. |
| `archil invalidate-cache` | command | Forces fresh server reads. |

**How this applies to small files.** The 21.5 ms/file cost is per-file backend
round-trip latency incurred serially at read time. Because `archil prefetch`
recurses over a directory in the background (and warms the cache concurrently
rather than one blocking `open()` at a time), kicking it off immediately after
mount lets fetching overlap with other task work, so by the time the reader
reaches each file it is already local. In effect prefetch is Archil's substitute
for OS read-ahead — bulk/directory-level rather than per-stream.

Suggested usage in the benchmark / real jobs:

```bash
mount_disk
# Warm the whole tree in the background right after mounting.
sudo archil prefetch /mnt/archil "small-files/${RWX_RUN_ID}"
# ...do other setup while the cache fills...
# Reads now hit warm cache instead of paying 21.5 ms/file cold.
```

Caveats: prefetch is asynchronous, so a read that races ahead of the warm-up
still blocks until its data lands; and prefetched bytes consume cache, so for
datasets larger than the default cache raise `--max-cache-mb`. **This has not yet
been measured** — a natural next experiment is a `benchmark-small-files` variant
that prefetches after mount #2 and compares the cold-read MB/s.

## Mount / unmount latency

Derived from log timestamps.

| Event | Duration | Context |
|---|---|---|
| First mount (cold, `archil-demo`) | ~3.4 s | first attach of the disk in the run |
| Subsequent mount (`benchmark` #1) | ~0.28 s | warm |
| Subsequent mount (`benchmark` #2) | ~0.26 s | warm, after unmount |
| Unmount (flush + drop cache) | ~0.5 s | between benchmark phases |

The first mount pays a one-time cold cost (~3.4 s). Repeat mounts against the
same disk from the same runner settle to **~0.25–0.3 s**, cheap enough to mount
per-task.

## Metadata / small-file operations (`archil-demo` task)

All sub-second against the live mount:

- Directory listing + recursive `find` (maxdepth 3) over existing contents: fast.
- Create per-run directory, write two small files (`greeting.txt`, `host.txt`),
  read them back: fast.
- `archil delegations` confirmed dynamic ownership was granted for the newly
  created `rwx-runs/<run-id>` directory (plus the client's control dir).

## Task wall-clock (command execution)

| Task | Exec time | Notes |
|---|---|---|
| `install-archil` | ~12 s | `apt-get` + install script + version check (one-time, cached as a layer) |
| `archil-demo` | ~4.8 s | mount → write/read small files → delegations → unmount |
| `benchmark` | ~9.5 s | gen data → mount → 100 MiB write → remount → 100 MiB read → verify |
| `benchmark-small-files` | ~4.6 min | gen 12k files → write → remount → cold metadata + content read → verify (read-bound) |

RWX layer caching means `install-archil` (and the base layers) are reused across
runs — the first failed run and this successful run both hit cached layers for
setup, so steady-state runs only pay the `archil-demo` / `benchmark` execution
cost.

## Shared-mode behavior notes

- Shared mode mounts **read-only by default**. Write access comes from either an
  explicit `archil checkout` or **dynamic ownership** — creating a brand-new
  file or directory automatically grants the client write access to it.
- The config writes exclusively into freshly-created, per-run directories
  (`rwx-runs/<run-id>`, `bench/<run-id>`), so it relies on dynamic ownership and
  never needs `checkout`. This also avoids cross-run collisions when multiple
  runs share the disk concurrently.
- Data survived the unmount/remount cycle intact (integrity check passed),
  confirming writes are durably flushed on `archil unmount`.

## Takeaways

1. **Write-heavy CI steps are well-served** — cache-backed writes hit ~210 MB/s
   for bulk data and ~4,600 file-creates/s for tiny files.
2. **Cold reads are network-bound** — ~24 MB/s for one large file, but they
   collapse to ~0.05 MB/s for many tiny files (~475× slower per byte) because
   each is a separate backend round-trip (~21.5 ms/file). Archil has no
   read-ahead flag; hide the latency with **prefetch** (`archil mount
   --pre-fetch` or the recursive, background `archil prefetch` command — see
   [Archil read-ahead / prefetch options](#archil-read-ahead--prefetch-options))
   and/or read concurrently.
3. **Small files are latency-bound, not bandwidth-bound.** Archil (like most
   networked/object-backed filesystems) strongly favors fewer, larger files.
   For CI caches of many small files, pack them into an archive (tar/zip) and
   store/read the single blob rather than the loose tree.
4. **Mounts are cheap after the first** (~0.3 s), so per-task mounting is viable.
5. **Prefer per-run directories + dynamic ownership** over `checkout`/`chown` in
   shared mode — simpler, collision-free, and avoids the read-only-filesystem
   errors seen in the earlier failing run.
