# Spec — ROM Directory Sync

**Status:** Draft for review
**Owner:** klampatech fork (`github.com/klampatech/retroassembly`)
**Target install:** `watch` — RetroAssembly at `http://watch:8000/`, Docker image `klampatech/retroassembly:local`

---

## 1. Problem

RetroAssembly's only ingestion path is the browser. A user drags files onto
`/library/platform/<platform>` and each one is POSTed as a multipart `File` to
`POST /api/v1/roms` (`src/api/routes/library/roms.ts:32`), handled by
`createRom()` (`src/controllers/roms/create-roms.ts:61`).

That works for a dozen games. It does not work for a self-hoster who already
owns a 1,285-game collection sitting on a disk. The upstream author says as much
in issue #36:

> The reason we only support uploading is that RetroAssembly is not initially
> released as a self-hostable service, so some technical design decisions are not
> optimized for that scenario and it's a bit hard to change.
> — [arianrhodsandlot, 2025-11-21](https://github.com/arianrhodsandlot/retroassembly/issues/36)

He promised "something like _mount mode_" and in March 2026 said it "might be
released before early May". As of **2026-11-24 it is still open and unimplemented** —
13 months after the request, 6 months after the estimate.

The upstream tree is **completely greenfield** for this. Verified against
`upstream/main` (`fb062eff`):

```bash
git grep -inE 'readdir|readDir|opendir|glob\(' upstream/main -- src scripts
# → no matches
```

There is no `readdir`, no `opendir`, no glob helper anywhere in `src/` or
`scripts/`. Upstream never touches the filesystem to enumerate files — it only
ever writes the single file it was handed. The API app
(`src/api/app.ts`) mounts just `auth`, `users` and `library/*` (`favorites`,
`files`, `launch_records`, `preference`, `roms`, `states`); there is no
platform-list endpoint and no import endpoint. The author has **22 public
repos** and none of them is a CLI, a rom-scanner, or a bulk-import tool
(`package.json` has no `bin` field).

The one adjacent project that _does_ matter is
[**msleuth**](https://github.com/arianrhodsandlot/msleuth) — the metadata
service that matches ROM files against the LaunchBox and Libretro databases by
filename and MD5. Any importer should reuse it rather than inventing its own
matching.

There is one prior-art implementation: **PR #63** by `howeasy` ("feat: runtime
subpath hosting + host-mounted shared ROM library", 1,342 additions across 56 files,
opened 2026-06-12, **closed unmerged 2026-06-14** with no maintainer comment).
It is a good implementation and is worth reading, but it is not adoptable as-is:

- It bundles two unrelated features (subpath hosting + shared library).
- It is based on `beab0da2` (June 2026), five months behind current `main`.
- It threads a sentinel-user ownership filter through **every** ROM query path
  (10 call sites) — invasive, and every missed site is a silent data leak.
- It is gated behind a "shared library" mode that also makes uploads read-only,
  conflating two different ideas.

**It is still readable.** The author's fork branch survives at
`https://github.com/howeasy/retroassembly/tree/feat/runtime-subpath-and-shared-rom-library`
(`b1d838d6`, fetched locally as the `pr63` branch), and it is the most complete
existing description of the problem. Worth reading before writing any of
Phase 2–5.

Two other closed upstream PRs land squarely in the areas this work touches:

| PR                                                               | Title                                                                 | Relevance                                                                                                          |
| ---------------------------------------------------------------- | --------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| [#79](https://github.com/arianrhodsandlot/retroassembly/pull/79) | "fix: stream ROM uploads to disk instead of buffering them in memory" | The exact fix Phase 1 needs, already proposed upstream and closed unmerged.                                        |
| [#83](https://github.com/arianrhodsandlot/retroassembly/pull/83) | "fix: keep stored file paths inside the storage directory"            | A path-containment fix in `createStorage()`. Phase 7's `external/` id scheme must not repeat the hole this closes. |

Two upstream discussions cover adjacent ground and are worth a read:
[#46](https://github.com/arianrhodsandlot/retroassembly/discussions/46) and
[#59](https://github.com/arianrhodsandlot/retroassembly/discussions/59) on
sharing a library between accounts (this is the per-user `libraryMode` feature
already merged — _not_ a disk-directory feature), and
[#65](https://github.com/arianrhodsandlot/retroassembly/discussions/65) on
fully self-contained deployment.

**One independent copy** of the project exists —
[`daemonless/retroassembly`](https://github.com/daemonless/retroassembly), a
non-fork mirror last pushed 2026-11-15. It contains no scan, shared, import or
sync files either, so it is not prior art for this feature.

### 1.1 Our specific situation

`watch` runs RetroAssembly from the official image with the data directory at
`/home/kyle/retroassembly/data`. As of 2026-11-24 that library was **empty**:
the compose file had been repointed at a new data path at some point and the
previous install's data was left behind at `/opt/retroassembly/data`
(1,285 ROMs, 263 save states, 130 launch records, 1.3 GB of ROM blobs). That
library was restored during this work — see §8.

So we now have a working library again, **but the underlying problem is
unchanged**: there is still no supported way to say "here is a directory, serve
these." Any future collection — a No-Intro set, a second platform, a folder
that grows — still has to go through the browser, one file at a time.

## 2. Goals

| #      | Goal                                                                                                                                                                                                                                                            |
| ------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **G1** | Point the instance at a host directory of ROMs and have them appear in the library, with their original filenames and looked-up metadata.                                                                                                                       |
| **G2** | The directory stays authoritative. A re-sync picks up additions and content changes.                                                                                                                                                                            |
| **G3** | The app **never** mutates or deletes anything in the source directory.                                                                                                                                                                                          |
| **G4** | Support exactly the platforms RetroAssembly already supports — `platformMap`'s **32 platform keys**, covering the **28 consoles** the readme lists (the extra keys are aliases of the same hardware, §3.2) — detected from directory names and file extensions. |
| **G5** | Zero regression risk to the existing upload path, the multi-user model, favorites, save states, or launch history.                                                                                                                                              |

### Non-goals

- Not a scraper. Nothing is downloaded from the internet.
- Not a media server. No transcoding, no artwork hosting, no scraping of metadata
  beyond what `msleuth` already provides.
- Not a replacement for browser upload. Upload stays the path for the handful of
  games you just dumped on your desktop.

## 3. User-facing behavior

### 3.1 Configuration (server-side)

```
RETROASSEMBLY_RUN_TIME_ROM_SOURCE_ROOTS=/roms,/media/games
RETROASSEMBLY_RUN_TIME_ROM_SYNC_ENABLED=true
```

Roots are absolute paths **inside the container**. The client never supplies an
arbitrary path — it picks a root by id and the server resolves it against the
allowlist. A root must be mounted into the container, e.g.

```yaml
services:
  retroassembly:
    image: klampatech/retroassembly:local
    volumes:
      - /home/kyle/retroassembly/data:/app/data
      - /srv/media/games:/roms:ro # source collection, read-only
```

### 3.2 Layout detection

Two layouts are supported, detected automatically:

```
# A) platform folders (preferred — unambiguous)
/roms/snes/Super Mario World (U).smc
/roms/gba/Advance Wars v1.1 (2001)(Nintendo)(US).zip
/roms/nes/Super Mario Bros.nes

# B) flat directory (platform inferred from file extension)
/roms/Super Mario World (U).smc
/roms/Advance Wars v1.1 (2001)(Nintendo)(US).zip
```

In layout B the platform is inferred from the extension, which is only possible
when the extension is **unambiguous**. The extension→platform map derived from
`platformMap` is lopsided: **31 of the 32 platforms accept `.zip`** (only
PlayStation is `.chd`-only), so a flat `.zip` file is **ambiguous** and the sync
reports it rather than guessing. Extensions that map to exactly one platform
include `.gba`, `.a26`, `.lnx`, `.chd`, `.pce`, `.fds`, `.n64` and `.32x`.

Four pairs of platforms are **duplicate entries for the same hardware** —

| Platform key | Its twin    | Shared extensions               |
| ------------ | ----------- | ------------------------------- |
| `snes`       | `sfc`       | `.smc`, `.sfc`, `.zip`          |
| `nes`        | `famicom`   | `.nes`, `.unif`, `.unf`, `.zip` |
| `genesis`    | `megadrive` | `.md`, `.gen`, `.zip`           |
| `gb`         | `gbc`       | `.gb`, `.zip`                   |

— so those extensions are ambiguous _by platform key_ even though the hardware
is not. The alias table (§4.2) resolves each pair to one canonical key, and
the UI lets you pin a platform for a root to resolve the rest.

Folder names are matched case-insensitively against the platform key, the
libretro name, the display name, and a curated alias list — so `SNES`,
`Super Nintendo Entertainment System`, `Super Nintendo`, and `Nintendo - Super
Nintendo Entertainment System` all resolve to `snes`.

### 3.3 The sync

1. **Preview** (`dryRun`) — scan and report what _would_ happen, without
   writing. Shows counts: new, changed, unchanged, pruned, skipped, errors.
2. **Apply** — performs the import.
3. **Resync** — re-run against the same root. Idempotent: unchanged files are
   skipped, new files are added, files whose content changed are re-imported
   in place, files that vanished from the source are pruned (configurable).

Progress is streamed over a background job — a 1,285-game import takes on the
order of a minute, mostly spent in `msleuth` lookups, and must not be a
single blocking HTTP request.

### 3.4 Two storage modes

| Mode                    | What happens                                                                                                                                                               | Disk cost                          | Source dir                          |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------- | ----------------------------------- |
| **`link`** (v1 default) | The file is materialized into RetroAssembly's own storage (`roms/<platform>/<digest><ext>`) via **hardlink → copy → symlink** fallback. The app owns the bytes afterwards. | ~0 with hardlink (same filesystem) | Read-only is fine                   |
| **`reference`** (v2)    | The app stores a pointer and streams the bytes straight from the source directory. Nothing is copied.                                                                      | 0                                  | **Must** be readable; never written |

Both modes satisfy G3. In `link` mode the source file is only ever read; the
app's hardlink is deleted independently (removing a link does not touch the
source inode). In `reference` mode `storage.delete()` is a hard no-op for
external files.

`link` mode is v1 because it requires **no change to any ROM query path** — the
imported ROMs are ordinary rows and the app's existing ownership, favorites,
states and launch records all work unchanged (see §5.2). `reference` mode is
needed for read-only mounts that live on a different filesystem, where hardlink
is impossible and copying gigabytes is unacceptable.

## 4. Architecture

### 4.1 New components

```
src/constants/rom-source.ts          env knobs + defaults
src/constants/platform-aliases.ts    folder-name → PlatformName alias table
src/databases/schema.ts              + romSourceRootsTable, romSourcesTable
src/utils/server/file.ts             + getFilePartialDigestFromPath()
src/utils/server/storage.ts          + put(id, path) streaming from disk
src/utils/server/rom-source/
  scan.ts                            walk + classify a root into candidates
  detect-platform.ts                 folder/extension → PlatformName
  import.ts                          materialize a candidate into a ROM row
  sync.ts                            the sync job: plan → apply → report
  jobs.ts                            in-process job registry + progress
src/controllers/rom-sources/*.ts     list roots / preview / sync / job status
src/api/routes/library/rom-sources.ts
src/pages/library/.../rom-source-*.tsx   the Sync dialog
```

### 4.2 Platform detection

`platformMap` (`src/constants/platform.ts`) is the single source of truth for
which platforms exist and which extensions each accepts. Two new tables are
derived from it at module load:

- `extension → PlatformName[]` — every extension across all platforms, mapped
  to the platforms that accept it. 31 of 32 platforms accept `.zip`, so `.zip`
  is ambiguous; `.gba`, `.a26`, `.lnx`, `.chd`, `.pce`, `.fds`, `.n64` and
  `.32x` are unambiguous. The four duplicate platform pairs (`snes`/`sfc`,
  `nes`/`famicom`, `genesis`/`megadrive`, `gb`/`gbc`) are collapsed to one
  canonical key by the alias table, so their shared extensions are unambiguous
  in practice.
- `alias → PlatformName` — platform key, `libretroName`, display name, and a
  curated alias list, lowercased.

Detection order for a candidate file:

1. If any **path segment** below the root matches an alias → that platform.
2. Else if the extension is unambiguous → that platform.
3. Else → **ambiguous**: skipped and reported, unless the root has a pinned
   platform.

### 4.3 Import pipeline

For each candidate file the pipeline computes:

- **`digest`** — `getFilePartialDigestFromPath(path, size)`: SHA-256 over
  first 1 KiB + last 1 KiB + size, truncated to 16 hex chars. This is
  **byte-for-byte the same function** the upload path uses
  (`src/utils/server/file.ts:3`), so a ROM imported from a directory and the
  same ROM uploaded through the browser land on the same `fileId` and dedupe
  against each other.
- **`md5`** — computed server-side only for files ≤ 64 MiB, and used as the
  `msleuth` match key. Larger files are matched by name only (same policy PR
  #63 used). This avoids hashing a 700 MiB PSX CHD on every resync.
- **metadata** — `msleuth.identify({ files: [{ md5, name }], platform })`, the
  same call `createRom()` makes, batched 100 at a time. The service is
  [`arianrhodsandlot/msleuth`](https://github.com/arianrhodsandlot/msleuth),
  reachable at `RETROASSEMBLY_RUN_TIME_MSLEUTH_HOST` with
  `RETROASSEMBLY_RUN_TIME_MSLEUTH_FALLBACK_HOST` as a second endpoint; it
  matches against the LaunchBox and Libretro databases and returns the
  `launchbox` / `libretro` objects `createRom()` already knows how to map onto
  the `roms` row.

The result is a normal `roms` row: same columns, same `fileId` layout, same
metadata fields.

### 4.4 Refactor: `createRom()` accepts a path

`createRom({ file, md5, platform })` currently takes a web `File`. The import
path has a path on disk. Rather than duplicating the logic, `createRom` is
refactored to accept a discriminated source:

```ts
type RomSource = { kind: 'file'; file: File } | { kind: 'path'; path: string; size: number; name: string }
```

and `storage.put()` gains a path variant that **streams** from disk
(`createReadStream` → `pipeline` → `createWriteStream`) instead of buffering
the whole file. This is not incidental: it also fixes the memory profile
complained about in upstream issue #68 ("Memory not released after ROM upload"),
where the current code path forces an entire ROM through memory.

The upload path is unaffected — a `File` source keeps its existing behavior.

### 4.5 Manifest: the `rom_sources` table

A sidecar table records what was imported from where. It is the basis for
resync, and it means the sync diffs against **the manifest**, not against the
whole library.

```sql
CREATE TABLE rom_source_roots (
  id          TEXT PRIMARY KEY,
  path        TEXT NOT NULL UNIQUE,
  platform    TEXT,              -- NULL = auto-detect
  mode        TEXT NOT NULL,     -- 'link' | 'reference'
  strategy    TEXT NOT NULL,     -- 'hardlink' | 'copy' | 'symlink'
  prune       INTEGER NOT NULL,  -- 1 = remove library entries when the source file vanishes
  enabled     INTEGER NOT NULL,
  lastSyncAt  INTEGER,
  createdAt, updatedAt, status
);

CREATE TABLE rom_sources (
  id            TEXT PRIMARY KEY,
  rootId        TEXT NOT NULL,
  platform      TEXT NOT NULL,
  relativePath  TEXT NOT NULL,
  fileName      TEXT NOT NULL,
  digest        TEXT NOT NULL,
  size          INTEGER NOT NULL,
  mtimeMs       INTEGER NOT NULL,
  romId         TEXT NOT NULL,    -- → roms.id
  strategy      TEXT NOT NULL,
  lastSeenAt    INTEGER NOT NULL,
  createdAt, updatedAt, status
);
```

`UNIQUE(rootId, relativePath)` makes the manifest self-deduplicating and makes
re-import idempotent.

### 4.6 Resync semantics

| Source state                        | Action                                                                                                                                                                                                                                |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| File present, digest unchanged      | No-op. `lastSeenAt` bumped.                                                                                                                                                                                                           |
| File present, digest changed        | Re-materialize (new blob at the new digest's `fileId`), update the row's `fileId` in place. The `roms.id` is preserved so favorites, states and launch history survive. The superseded blob is deleted if nothing else references it. |
| File present, not in manifest       | New import.                                                                                                                                                                                                                           |
| File absent, manifest entry present | If `prune = 1`: soft-delete the ROM row and remove the app's copy/link. If `prune = 0`: leave the library entry alone and report it as "orphaned". **The source directory is never touched either way.**                              |

### 4.7 API

```
GET    /api/v1/rom-sources                     list configured roots + last sync
POST   /api/v1/rom-sources/preview             dry-run plan for a root
POST   /api/v1/rom-sources/sync                start a sync job
GET    /api/v1/rom-sources/sync/:jobId         job progress + summary
PATCH  /api/v1/rom-sources/:rootId             update root settings
```

All of these are authenticated (the library router already sits behind
`authMiddleware`) and additionally restricted to supervisor users via the
existing `RETROASSEMBLY_RUN_TIME_SUPERVISER_USER_IDS` knob — a sync mutates the
library, so it should not be available to every account on a multi-user instance.

The client passes a **root id**, never a path. The server resolves the id
against `RETROASSEMBLY_RUN_TIME_ROM_SOURCE_ROOTS`.

### 4.8 Job model

An in-process `Map` of jobs, each with a status, counters, and a log of
per-file outcomes. The job survives in memory only; a restart mid-import leaves
the manifest partially updated, which is safe because the next sync recomputes
from the manifest and re-imports anything incomplete (the manifest row is only
written after the blob is materialized).

### 4.9 UI

A **Sync** entry in the library settings menu → dialog listing configured roots
with their last sync time and game count, a "Preview" button that shows the
plan, and a "Sync now" button that shows live progress. Errors are listed
per-file with the reason (unsupported extension, ambiguous platform, unreadable,
missing BIOS, …).

## 5. Security model

| Threat                                     | Mitigation                                                                                                                                                                                                              |
| ------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Path traversal via client-supplied path    | Client supplies a **root id** only. Paths are resolved server-side from the configured allowlist.                                                                                                                       |
| Symlink escape from the source dir         | Every resolved path is `realpath`'d and must remain under the root's `realpath`. Symlinks pointing outside are skipped.                                                                                                 |
| Write into the source dir                  | The app never opens a source file for writing. In `reference` mode `storage.delete()` and `storage.put()` are hard no-ops for external ids. `link` mode only creates new entries under the app's own storage directory. |
| Resource exhaustion from a huge directory  | Configurable max file count and max depth per root; the scan is streaming, never accumulates the whole tree.                                                                                                            |
| Memory exhaustion from a large ROM         | Streaming from disk end-to-end; never buffer a ROM.                                                                                                                                                                     |
| md5/CPU burn on huge files                 | md5 only for files ≤ 64 MiB; the partial digest reads 2 KiB per file.                                                                                                                                                   |
| A sync wiping the library                  | Pruning is off by default and is an explicit, separately-confirmed action.                                                                                                                                              |
| Arbitrary file read via a crafted `fileId` | `reference`-mode ids are resolved through the same realpath containment check; `fileId` values are never taken from the client for content reads (the server looks the id up by `roms.id`).                             |

## 6. Verification plan

**Unit**

- `detect-platform`: every alias resolves; ambiguous extensions are rejected;
  ambiguous folder names do not match.
- `getFilePartialDigestFromPath` produces **the same digest** as
  `getFilePartialDigest` for the same bytes (this is the dedupe guarantee).
- Scanner: skips unsupported extensions, hidden files, out-of-root symlinks;
  respects depth and count limits.
- Storage strategies: hardlink succeeds on the same filesystem, falls back to
  copy across filesystems, and the source file is byte-identical afterwards.
- Manifest diffing: new / changed / unchanged / removed classification.

**Integration**

- Import a fixture tree → assert row count, `fileId` layout, metadata fields,
  and that the blobs exist in storage.
- Re-import → assert zero changes (idempotent).
- Mutate a fixture file → assert the row updates and keeps its `roms.id`.
- Delete a fixture file → assert prune behavior with `prune` on and off.
- Assert the source directory is byte-identical (hash the tree) after every
  operation.

**End-to-end (Playwright)**

- The Sync dialog renders configured roots; preview shows counts; sync
  progresses; the library shows the imported games; a game launches.

**Live on `watch`**

- Import the real 1,288-file collection into a scratch instance and confirm
  1,285 games, correct titles, and no source mutation.

## 7. Risks

| Risk                                                                            | Severity | Mitigation                                                                                                                                                                                                                                      |
| ------------------------------------------------------------------------------- | -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `msleuth` rate-limits or falls over during a bulk import                        | Medium   | Batch 100, cap concurrency at 4, exponential backoff, per-batch error capture that does not abort the run. The metadata migration path at `src/utils/server/migration/raw-metadata.ts` already demonstrates this pattern working on 1,279 ROMs. |
| Import silently produces an empty library because the platform was mis-detected | Medium   | Preview is mandatory before apply; the report lists every skipped file with a reason.                                                                                                                                                           |
| `link` mode with `copy` strategy doubles the collection's disk usage            | Medium   | Hardlink is the default and the strategy is reported per root; the preview shows the estimated bytes to be materialized.                                                                                                                        |
| A long import outlives the HTTP connection                                      | Low      | Background job with a polled status endpoint.                                                                                                                                                                                                   |
| `reference` mode's `fileId` scheme collides with a real storage path            | Low      | Reserved prefix (`external/`) that the storage layer checks before resolving anything else.                                                                                                                                                     |
| Upstream merges a competing "mount mode" implementation                         | Low      | The fork keeps the feature behind `RETROASSEMBLY_RUN_TIME_ROM_SYNC_ENABLED` and the data model is independent of upstream's.                                                                                                                    |

## 8. Our install, as of 2026-11-24

Restored during this work, before any code changes:

- The official image was already at the newest published tag
  (`v7.261011.1345`) — but it ships **Node v26.5.0**, which is affected by
  **CVE-2026-111497** (Brotli decoder RCE, fixed in Node 26.12.1). Upstream
  fixed this in PR #81 but **no new image was ever published**. Our fork's
  image is built from `main` and ships **Node v26.12.1**.

  > **The tag name is a trap.** Upstream's tag scheme is `v7.YYMMDD.HHMM`, so
  > `v7.261011.1345` is **11 October 2026**, not 10 November. The tag's commit
  > is dated `2026-10-11`; the Node 26.12.1 bump (PR #81) is dated
  > `2026-11-09` and is **not an ancestor of the tag**:
  >
  > ```bash
  > git merge-base --is-ancestor 43ff037a v7.261011.1345  # → non-zero, NOT in tag
  > ```
  >
  > Docker Hub's `latest` is byte-identical to that tag (same sha256 digest,
  > pushed 2026-10-11T13:48Z), so `latest` is _newer_ than the newest
  > published **release notes** (`v7.260719.1051`, 2026-07-19) but still
  > **older than the source**. The `latest` tag tracks the newest git tag, and
  > the CI workflow only rebuilds on a tag push — so no image has ever
  > carried the Node CVE fix.

- The empty library was repaired by merging the stranded install's data
  (1,285 ROMs, 263 states, 130 launch records) into the live database under the
  account Kyle currently logs in with, and moving the 1.3 GB of blobs into the
  live data directory. The app's own `raw-metadata` migration then re-cached
  metadata for all 1,279 matchable ROMs on first boot.
- Verified: the library page reports "1285 games for 2 platforms in total", a
  ROM downloads correctly (464,270 bytes, `PK\x03\x04` ZIP magic), and the
  search API returns 86 Mario titles with full metadata.

Backups: `/home/kyle/retroassembly-backups/` on `watch`
(`live-data-20261124-084657.tar.gz`, `old-retroassembly.sqlite.20261124-084657`,
`compose.yml.20261124-084657`, `retroassembly-local.tar.gz`).

## 9. Open questions for the operator

1. **Prune default.** When a file disappears from the source directory, should
   the library entry be removed (true sync) or kept (append-only)? Default in
   this spec is **off** — append-only — because it is the non-destructive
   choice.
2. **`.zip` ambiguity.** RetroAssembly's own collection is mostly `.zip`, and
   `.zip` is accepted by **31 of the 32** platforms, so a flat directory of
   zips cannot be auto-detected. Options: require platform subfolders
   (recommended), or let the operator pin a platform per root.
3. **Multi-user sharing.** Should a source directory be visible to _all_ users
   of the instance (requires the sentinel-ownership work of Phase 3), or only to
   the account that runs the sync? v1 is per-user.
4. **BIOS files.** Platform BIOSes are stored in _user preferences_
   (`preference.emulator.platform[p].bioses`) and uploaded through a separate
   route. Should a source directory also supply BIOSes (matched by the `md5`
   values already listed in `platformMap[p].bioses`), or is BIOS handling out
   of scope for v1?
