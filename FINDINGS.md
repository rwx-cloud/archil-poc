# Archil on RWX — Performance Findings

Analysis of RWX run [`234ae5ace1bf4a85831389b744e6ab3b`](https://cloud.rwx.com/mint/rwx/runs/234ae5ace1bf4a85831389b744e6ab3b)
(status: **succeeded**), exercising an [Archil](https://archil.com) shared disk
mounted inside RWX task containers.

## Environment

| | |
|---|---|
| Archil client | `v0.8.18-1782513184` (proto Version2) |
| Disk | `rwx/dan-testing` (`dsk-0000000000016dc4`) |
| Region | `aws-us-east-1` |
| Mount mode | `--shared` (read-only by default; writes via dynamic ownership) |
| Runner OS | Ubuntu 24.04.4 LTS, kernel `6.17.0-1019-aws`, x86_64 |
| Base image | `ubuntu:24.04` + `rwx/base 1.0.2` |

## Headline throughput (100 MiB payload)

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
   even with fsync.
2. **Cold reads are network-bound (~24 MB/s)** — for read-heavy workloads that
   re-fetch cold data, factor this in or pre-warm the cache (`archil mount
   --pre-fetch`).
3. **Mounts are cheap after the first** (~0.3 s), so per-task mounting is viable.
4. **Prefer per-run directories + dynamic ownership** over `checkout`/`chown` in
   shared mode — simpler, collision-free, and avoids the read-only-filesystem
   errors seen in the earlier failing run.
