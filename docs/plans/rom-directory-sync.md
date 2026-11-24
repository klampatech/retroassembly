# Implementation Plan — ROM Directory Sync

Companion to [`docs/specs/rom-directory-sync.md`](../specs/rom-directory-sync.md).

**Repo:** `github.com/klampatech/retroassembly` (fork of `arianrhodsandlot/retroassembly`)
**Local:** `~/Development/retroassembly` · `origin` = fork, `upstream` = upstream
**Target:** `watch` — `http://watch:8000/`, data at `/home/kyle/retroassembly/data`

> Branch discipline: every phase lands on its own branch → PR → **Kyle merges**.
> Never push to `main`. The agent opens the PR and reports the URL.

---

## Ground rules

- **Style** (from `agents.md`): single quotes, no semicolons, 2-space indent,
  lower-snake-case filenames, function declarations over arrows, **no
  `.reduce` / `.then` / `.catch`** — use `async/await` + `attemptAsync` from
  `es-toolkit`. `HTTPException` from `hono/http-exception` for API errors.
- **Lint is a gate.** `pnpm vp check` — treat warnings as errors. Type checking
  is included in lint; do **not** run `pnpm tsc` separately.
- **Tests.** Unit: `pnpm test:unit` (`node --test tests/unit/*.test.ts`).
  E2E: `pnpm test`. CI runs `setup → lint → test` on every push.
- **Migrations.** `drizzle-kit generate` then commit the SQL + journal. The
  Node runtime applies migrations automatically at boot
  (`src/utils/server/migration/initalization.ts`).

---

## Phase 0 — Fork hygiene and image delivery

**Why first:** today the image is built on `m5` and `scp`'d to `watch` as a
tarball. The fork needs a repeatable path to a deployable image, and `watch`
should pull rather than receive a 75 MB tarball over SSH.

| # | Task | File |
|---|------|------|
| 0.1 | Add a GHCR publish workflow (fork-only; upstream's `docker-publish.yaml` pushes to Docker Hub and is left alone). Triggers on tags `v*`; builds `linux/amd64`; pushes `ghcr.io/klampatech/retroassembly`. | `.github/workflows/ghcr-publish.yaml` |
| 0.2 | Add `docker-compose.yml` at the repo root showing the data volume **and** an example read-only ROM source volume. | `docker-compose.yml` |
| 0.3 | Document the fork's divergence from upstream in the readme ("This fork adds …"), with a pointer to the spec. | `readme.md` |
| 0.4 | Tag and publish the current `main` as `v7.261124.1` so `watch` can pull a real image instead of the ad-hoc tarball. | — |

**Acceptance**
- `docker pull ghcr.io/klampatech/retroassembly:v7.261124.1` succeeds.
- `docker compose -f docker-compose.yml up -d` on `watch` starts the app and
  `curl -s localhost:8000/` returns `302`.

**Verification**
```bash
# on watch
docker pull ghcr.io/klampatech/retroassembly:v7.261124.1
docker images ghcr.io/klampatech/retroassembly
```

---

## Phase 1 — Path-based digest and streaming storage

**Why:** the import path has a file on disk, not a web `File`. This phase makes
the existing ingestion code path-agnostic **and** stops ROMs being buffered in
memory (upstream issue #68).

| # | Task | File |
|---|------|------|
| 1.1 | Add `getFilePartialDigestFromPath(path, size)`. Must produce the **identical** digest as `getFilePartialDigest(file)` for the same bytes: SHA-256 over first 1 KiB + last 1 KiB + size, truncated to 16 hex chars. Read the tail with a positioned stream, not by slurping the file. | `src/utils/server/file.ts` |
| 1.2 | Extend `storage.put()` to accept `{ kind: 'path'; path: string }` in addition to a `Blob`. Stream with `createReadStream` → `pipeline` → `createWriteStream`; never buffer. | `src/utils/server/storage.ts` |
| 1.3 | Refactor `createRom()` to take a discriminated source instead of a `File`: `{ kind: 'file'; file }` \| `{ kind: 'path'; path; size; name }`. Preserve the existing upload behavior byte-for-byte. | `src/controllers/roms/create-roms.ts` |
| 1.4 | Unit: digest parity (path vs `File`), streaming put writes the exact bytes, put is idempotent. | `tests/unit/file-digest.test.ts`, `tests/unit/storage-put.test.ts` |

**Acceptance**
- `pnpm test:unit` passes.
- `pnpm vp check` clean.
- The existing upload path still passes the e2e suite unchanged.

**Risk:** medium — `createRom` is on the hot path for every upload. The refactor
must be behavior-preserving; the e2e `rom-management.spec.ts` is the guard.

---

## Phase 2 — Platform detection

| # | Task | File |
|---|------|------|
| 2.1 | Add `platformAliases`: for each of the **32** platforms in `platformMap`, collect the platform key, `libretroName`, the localized display name, and a curated alias list (`snes` → `snes`, `sfc`, `super nes`, `super nintendo entertainment system`, `nintendo - super nintendo entertainment system`, …). Lowercased. Collapse the four duplicate pairs (`snes`/`sfc`, `nes`/`famicom`, `genesis`/`megadrive`, `gb`/`gbc`) to one canonical key each. | `src/constants/platform-aliases.ts` |
| 2.2 | Build `extensionToPlatforms`: every extension in every `platformMap[p].fileExtensions` → the platforms that accept it. A single-platform mapping is unambiguous; `.zip` (31 of 32 platforms) is not. | `src/constants/platform-aliases.ts` |
| 2.3 | `detectPlatform({ relativePath, rootPlatform })` → `PlatformName \| 'ambiguous' \| 'unsupported'`. Order: pinned root platform → matching path segment (alias) → unambiguous extension → ambiguous. | `src/utils/server/rom-source/detect-platform.ts` |
| 2.4 | Unit: every alias resolves to its platform; `.smc` → `snes` (after the `snes`/`sfc` pair is collapsed); `.gba` → `gba`; `.zip` → ambiguous; unknown extension → unsupported; a pinned root platform always wins. | `tests/unit/detect-platform.test.ts` |

**Acceptance**
- `detectPlatform('snes/Super Mario World (U).smc')` → `snes`.
- `detectPlatform('Super Mario World (U).smc')` → ambiguous (`.smc` maps to both `snes` and `sfc` — resolve by preferring `snes`/`sfc` as aliases of one another, documented in the alias table).
- `detectPlatform('stuff.bin')` → unsupported.

---

## Phase 3 — Scanner

| # | Task | File |
|---|------|------|
| 3.1 | Walk a root recursively. Collect: absolute path, path relative to root, basename, extension, size, mtime. | `src/utils/server/rom-source/scan.ts` |
| 3.2 | Skip: hidden files/dirs, unsupported extensions, symlinks whose `realpath` escapes the root's `realpath`, files above `maxFileSize`, and anything past `maxDepth` / `maxFiles`. | same |
| 3.3 | Classify each candidate with `detectPlatform`. Collect **ambiguous** and **unsupported** candidates into separate buckets rather than dropping them silently. | same |
| 3.4 | Enforce root containment: the resolved root `realpath` must be the container of every candidate's `realpath`. | same |
| 3.5 | Unit: nested dirs; hidden files; escaping symlink; ambiguous `.zip`; depth/size/count limits; containment violation is rejected. | `tests/unit/scan.test.ts` |

**Acceptance**
- Scanning a 10,000-file tree does not exceed the configured memory budget
  (the scan is streaming — it yields candidates rather than building an array of
  the whole tree).

---

## Phase 4 — Schema and manifest

| # | Task | File |
|---|------|------|
| 4.1 | Add `romSourceRootsTable` and `romSourcesTable` to the Drizzle schema, following the repo's conventions (`baseSchema`, soft delete via `status`). `UNIQUE(rootId, relativePath)` on `rom_sources`. | `src/databases/schema.ts` |
| 4.2 | `drizzle-kit generate`; commit the SQL and the updated journal. | `src/databases/migrations/` |
| 4.3 | Seed the `rom_source_roots` rows from `RETROASSEMBLY_RUN_TIME_ROM_SOURCE_ROOTS` on boot (idempotent upsert by path; never clobber a user's per-root settings). | `src/utils/server/rom-source/roots.ts` |
| 4.4 | Add the env knobs to `getRunTimeEnv()` with safe defaults: `RETROASSEMBLY_RUN_TIME_ROM_SYNC_ENABLED` (`'false'`), `RETROASSEMBLY_RUN_TIME_ROM_SOURCE_ROOTS` (`''`), `RETROASSEMBLY_RUN_TIME_ROM_SOURCE_MAX_FILES` (`'20000'`), `RETROASSEMBLY_RUN_TIME_ROM_SOURCE_MAX_DEPTH` (`'8'`), `RETROASSEMBLY_RUN_TIME_ROM_SOURCE_MAX_FILE_SIZE` (`'4294967296'`). | `src/constants/env.ts` |

**Acceptance**
- The app boots on an existing database and the migration applies cleanly.
- Booting twice does not duplicate root rows.
- `getDirectories()` gains a `romSourceRoots` accessor.

---

## Phase 5 — Import service, sync job, API

| # | Task | File |
|---|------|------|
| 5.1 | `importCandidate()`: compute digest + md5 (≤ 64 MiB), materialize the file into storage (`hardlink` → `copy` → `symlink` fallback), call `createRom()` with the path source, write the manifest row. Manifest row is written **after** the blob is in place, so an interrupted import is safe to re-run. | `src/utils/server/rom-source/import.ts` |
| 5.2 | `planSync()`: diff scanner output against the manifest → `{ new, changed, unchanged, pruned, orphaned, skipped, errors }`. Pure function over the manifest and the scan result; no I/O beyond the scan itself. | `src/utils/server/rom-source/sync.ts` |
| 5.3 | `applySync()`: execute a plan. Changed files re-materialize and update `roms.fileId` **in place** (preserving `roms.id` so favorites/states/history survive); the superseded blob is removed if unreferenced. Prune soft-deletes the row and removes the app's copy — **never** touching the source. | same |
| 5.4 | Job registry: in-process `Map`, status + counters + per-file outcomes, pollable by id. | `src/utils/server/rom-source/jobs.ts` |
| 5.5 | Controllers: `listRomSources`, `previewRomSource`, `syncRomSource`, `getRomSourceJob`. All supervisor-only via `RETROASSEMBLY_RUN_TIME_SUPERVISER_USER_IDS`. | `src/controllers/rom-sources/*.ts` |
| 5.6 | Routes under the library router (already behind `authMiddleware`): `GET /api/v1/rom-sources`, `POST /api/v1/rom-sources/preview`, `POST /api/v1/rom-sources/sync`, `GET /api/v1/rom-sources/sync/:jobId`. The client sends a **root id**, never a path. | `src/api/routes/library/rom-sources.ts` |
| 5.7 | Unit + integration: import a fixture tree; assert rows, blobs, metadata; re-import is a no-op; mutate a file → row updates and keeps its id; delete a file → prune on/off; assert the source tree hash is unchanged after every operation. | `tests/unit/rom-source-sync.test.ts` |

**Acceptance**
- `POST /api/v1/rom-sources/sync` returns a job id immediately; the job endpoint
  reports progress and a final summary.
- A second sync of an unchanged tree reports `new: 0, changed: 0, pruned: 0`.
- The source directory's content hash is identical before and after.

**Risk:** high — this is the bulk of the change. Land behind
`RETROASSEMBLY_RUN_TIME_ROM_SYNC_ENABLED=false` (default) so it cannot affect a
running instance.

---

## Phase 6 — UI

| # | Task | File |
|---|------|------|
| 6.1 | A **Sync** entry in the library settings menu, shown only when the feature flag is on and at least one root is configured. | `src/pages/library/components/library-layout/layout-menu/settings-dialog/` |
| 6.2 | Dialog: list roots with path, platform, mode, last sync time, game count; **Preview** (shows the plan table), **Sync now** (live progress), and a per-file error list. | `src/pages/library/.../rom-source-dialog.tsx` |
| 6.3 | Add the new i18n keys to **all 12 locale files** (`en, de, es, fr, it, ja, ko, pt, ru, cs, zh-cn, zh-tw`). | `src/locales/` |
| 6.4 | E2E: dialog renders; preview returns counts; sync imports a fixture and the library shows the game; a game launches. | `tests/e2e/tests/rom-source-sync.spec.ts` |

**Acceptance**
- `pnpm test` passes with the new spec.
- Every locale file has the same key set.

---

## Phase 7 — `reference` mode (serve in place)

**Why second:** `link` mode covers the common case. `reference` is needed when
the source is on a read-only mount on a different filesystem, where hardlink is
impossible and copying gigabytes is unacceptable.

| # | Task | File |
|---|------|------|
| 7.1 | Reserved `fileId` prefix `external/` for ids that resolve outside the app's storage. `storage.get()` / `head()` resolve it through the realpath containment check; `put()` and `delete()` are hard no-ops for it. | `src/utils/server/storage.ts` |
| 7.2 | `importCandidate()` in `reference` mode: no blob is materialized; the `fileId` is `external/<rootId>/<relativePath>` and the manifest records the digest for change detection. | `src/utils/server/rom-source/import.ts` |
| 7.3 | Change detection in `reference` mode compares the manifest digest against a freshly computed digest (2 KiB read per file) — cheap enough to run on every resync. | `src/utils/server/rom-source/sync.ts` |
| 7.4 | Unit: `external/` ids resolve; traversal attempts are rejected; `delete()` on an external id does not unlink the source; a read-only mount works end to end. | `tests/unit/storage-external.test.ts` |

**Acceptance**
- A `:ro` bind-mounted source directory is served correctly and the source is
  provably untouched.
- `storage.delete()` on an external id is a no-op.

---

## Phase 8 — Shared across users (optional)

**Why optional:** only needed if a source directory should be visible to every
account on the instance, rather than only to the account that ran the sync.

| # | Task | File |
|---|------|------|
| 8.1 | Sentinel owner id (`__shared__`) for rows imported from a root marked `shared = 1`. | `src/utils/server/rom-source/shared.ts` |
| 8.2 | A single `romOwnershipCondition(userId)` helper, applied at **every** one of the 10 ROM query sites: `count-roms.ts`, `create-roms.ts`, `delete-roms.ts`, `get-rom-content.ts`, `get-rom-platform-count.ts`, `get-rom.ts`, `get-roms.ts`, `get-roms-with-states.ts`, `search-roms.ts`, `update-rom.ts`, plus `create-state.ts`. | `src/controllers/roms/*`, `src/controllers/states/create-state.ts` |
| 8.3 | A **guard test** that greps the source for `romTable.userId` and fails if any site bypasses the helper. This is the failure mode that made PR #63 risky. | `tests/unit/rom-ownership-guard.test.ts` |

**Acceptance**
- The guard test fails if a new ROM query is added without the helper.
- A shared root's games appear for every user; per-user state stays private.

---

## Phase 9 — Deployment on `watch`

| # | Task |
|---|------|
| 9.1 | Build and publish the image; pull it on `watch`. |
| 9.2 | Add the source volume to the compose file. The source mount is **read-only** (`:ro`). |
| 9.3 | Enable the feature flag and configure the roots. |
| 9.4 | Dry-run the sync against the live collection; review the plan; then apply. |
| 9.5 | Verify: game count, a spot-check of titles/metadata, a game launching in the browser, and a source-tree hash comparison proving nothing was written. |
| 9.6 | Update the readme with the self-hosting instructions and the new env vars. |

**Verification**
```bash
# on watch
sha256sum /srv/media/games/roms/snes/*.smc | sha256sum    # before
docker compose up -d && docker logs -f retroassembly
curl -s -X POST localhost:8000/api/v1/rom-sources/preview \
  -H "Authorization: Bearer $TOKEN" -H 'content-type: application/json' \
  -d '{"rootId":"<id>"}'
sha256sum /srv/media/games/roms/snes/*.smc | sha256sum    # after — identical
```

---

## Dependency order

```
Phase 0 (fork + CI)          ── independent, do first
Phase 1 (path digest + storage) ──▶ Phase 5 (import/sync)
Phase 2 (platform detection)   ──▶ Phase 3 (scanner) ──▶ Phase 5
Phase 4 (schema)               ──▶ Phase 5
Phase 5                        ──▶ Phase 6 (UI)
Phase 7 (reference mode)       ── after Phase 5
Phase 8 (shared users)         ── after Phase 5, optional
Phase 9 (deploy)               ── after Phase 6 (and 7 if wanted)
```

Phases 1–4 are independent of each other and can be developed in parallel;
Phase 5 is the integration point.

## Suggested PR sequence

1. `feat/rom-source-path-plumbing` — Phase 1 (behavior-preserving refactor)
2. `feat/rom-source-platform-detection` — Phase 2
3. `feat/rom-source-scanner` — Phase 3
4. `feat/rom-source-schema` — Phase 4
5. `feat/rom-source-sync` — Phase 5
6. `feat/rom-source-ui` — Phase 6
7. `feat/rom-source-reference-mode` — Phase 7
8. `chore/ghcr-publish` — Phase 0.4 (can land first)

Each PR runs the full CI (`lint` + `test`) and is opened by the agent for
**Kyle to approve and merge**.
