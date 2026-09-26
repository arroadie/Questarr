# Library identification: agent execution plan

Status: ready for contract/design work; implementation has not started.

Source: [Game Identification and Fingerprinting Proposal](./LIBRARY_ITEM_IDENTIFIER.md).

## 1. Objective and boundaries

Import an existing library into Questarr without relocating its contents, explain which games/releases were recognized, and fingerprint observed content for subsequent recognition, duplicate detection, and verification.

Keep Game, Release, optional Build, Artifact, and observed Installation distinct. Adapters collect evidence; a central resolver decides identity. A fingerprint alone does not establish a published version without a trusted reference mapping.

The source proposal defines the overall architecture. This plan adds execution boundaries and resolves gaps needed for implementation. Tasks that change fingerprint semantics must first amend the contract and its golden vectors.

### Delivery milestones

| Milestone | User outcome | Included tasks |
| --- | --- | --- |
| M1: usable local import | Scan, inspect candidates, confirm identity, preserve multiple copies, rescan, verify content | T00–T10 |
| M2: reference matching | Identify supported ROM/disc artifacts from user-imported DAT catalogs; group supported media sets | T11–T12 |
| M3: similarity | Find structurally similar installations with explainable evidence | T13 |
| M4: platform expansion | Add independently tested provider/platform adapters | T14 |

M1 includes SHA-256, qtree-v1, qmanifest-v1, Steam evidence, provenance, optional sidecars, durable jobs, and review UI. qset-v1 lands in M2; qstructure-minhash-v1 lands in M3. This stages the proposal's recommended fingerprint suite rather than dropping it.

Not in these milestones: moving/organizing files, launcher repair, running discovered software, automatic archive extraction during discovery, shared public fingerprint service, content-defined chunking, qsample, cross-container CHD equivalence, or encrypted format access workarounds.

## 2. Repository integration map

Verify these entry points against the checkout before each task; names below describe the inspected baseline, not immutable APIs.

| Existing area | Integration |
| --- | --- |
| `server/library-scanner.ts` | Existing immediate-child discovery and IGDB matching. Replace name-only automatic association in the new flow; retain endpoint compatibility through a facade. |
| `server/services/ImportManager.ts` | Successful import already knows the game; create/update a library item from provenance after import succeeds. |
| `server/services/ImportStrategies.ts` | Reuse relevant safe filesystem helpers; scanning must not call transfer/reorganization operations. |
| `server/services/PathMappingService.ts` | Translate downloader paths before provenance registration; do not use metadata to choose paths. |
| `server/path-security.ts` | Reuse/extend root containment checks for passive traversal and parsing. |
| `shared/schema.ts`, `server/storage.ts` | Add domain schemas, persistence interfaces, and both storage implementations. |
| `server/migrate.ts`, `migrations/` | Follow actual migration registration conventions; test populated and empty databases. |
| `server/routes.ts`, `server/routes/import.ts` | Existing root-folder, scan, file, deletion, and import routes must remain compatible. |
| `client/src/pages/settings.tsx`, `client/src/components/ImportSettings.tsx` | Root configuration and import settings entry points. |
| `server/__tests__/`, `client/__tests__/`, `tests/e2e/` | Extend relevant coverage and add synthetic identification fixtures. |

Proposed new implementation locations: `shared/library-identification.ts`, `server/services/library-identification/`, and a dedicated library-identification route module. T00 chooses exact names and route mounting to match repository conventions.

## 3. Contracts to freeze before implementation

T00 produces `docs/proposal/LIBRARY_ITEM_IDENTIFIER_CONTRACT.md`. It must settle all of the following; downstream agents consume that document.

### Domain and ownership

- Use existing string Questarr IDs; numeric IDs in proposal examples are illustrative.
- A discovered LibraryItem may have a null game/release/build association. Unknown items must persist without creating placeholder Games.
- Separate logical Artifact identity from local occurrence/location. Equal hashes may share artifact identity while retaining distinct library items and paths.
- Persist unresolved native identifiers as evidence before a target entity exists. A Steam AppID can be recognized while its IGDB Game mapping remains unresolved.
- Define referential constraints so a Build belongs to the selected Release and a Release belongs to the selected Game.
- Define actor authorization using existing root-folder access policy. Roots currently lack an explicit owner field; do not infer private ownership from a requesting user's ID. Prevent cross-user association or information leakage.
- Define duplicate location uniqueness, identifier namespace normalization, scoped provider build IDs, catalog source/version keys, and deletion behavior. No global uniqueness assumption on short native IDs or build numbers.
- Existing `game.libraryPath` and `game_files` remain compatible projections during M1. Never create one permanent relational row per scanned file. Specify deterministic projection selection for multiple copies and avoid overwriting an existing managed path.
- Backfill legacy paths without claiming that existing title-based matches are verified; preserve them as legacy associations with recorded evidence.

### Fingerprint semantics

- SHA-256 streams bytes with bounded buffers. qtree means structure and size, not exact content.
- Define byte-exact canonical encoding for qtree, qmanifest, and later qset. The proposal's illustrative separators and digest concatenation leave newline handling, binary versus hexadecimal digests, and framing unspecified. Use explicit unambiguous framing and golden vectors before publishing v1.
- Normalize relative paths to NFC, `/` separators, and preserve case; sort by UTF-8 byte order, not locale order. Absolute paths/times/permissions never enter fingerprints.
- Reject or report canonical-path collisions, including distinct paths that normalize to the same NFC name. Never silently drop entries.
- Define integer size encoding, empty-tree behavior, unsupported filename encoding, and symlink treatment. M1 skips symlinks with a recorded warning; do not fingerprint links as ordinary files.
- Fingerprints record kind, algorithm, algorithm version, profile ID/version, and inclusion policy. Compare only compatible scopes/profiles.
- Store raw observed inventory separately from identity-profile inclusion. A filtered qmanifest proves exact equality only of included files. Exclude Questarr sidecars from all fingerprint inputs.
- Full verification bypasses cache. Cached results are not fresh integrity proofs; same-size/same-mtime changes can evade cache hints.
- Incomplete/unreadable/changing inventories do not publish a complete exact manifest. Record completeness and scan errors.

### Jobs and matching

- Durable job states: queued, running, completed, completed-with-errors, failed, cancelled. Persist stage, counters, limits, and restart behavior; Socket.IO is notification transport, not durable truth.
- Levels 0–1 are default discovery; anchors and full hashing are explicit or configured background work. Define budgets and cancellation checks.
- Snapshot/revision tokens prevent stale job results or stale manual decisions replacing newer observations. Detect files changing during hashing and retry boundedly or mark unstable.
- Separate identity match state, content comparison state, accessibility, and job state. Hash disagreement alone does not prove mods; `IDENTIFIED_MODIFIED` requires a compatible known baseline and interpretable evidence.
- Strong contradictory evidence produces CONFLICT before priority selection. Names or MinHash alone never auto-link. Exact content matches without a trusted Game mapping identify content only.
- User decisions persist across rescans; changed authoritative evidence creates a conflict rather than silently replacing the decision.
- Sidecars are untrusted claims, even when schema-valid. Check local IDs, portable identifiers, stale fingerprints, and contradictions; foreign database IDs must not accidentally resolve to an unrelated local game.
- No external catalog redistribution or automatic uploads. Catalog ingestion starts with user-provided sources and retains source/version/checksum provenance.

## 4. Task graph and execution rules

```text
T00 → T01 → T02
T00 → T03 → T04
T00 → T05
T02 + T03 + T04 + T05 → T06
T06 → T07
T06 + T07 → T08
T06 + T07 + T08 → T09 → T10 [M1]
T10 → T11 → T12 [M2]
T10 → T13 [M3]
T10 + relevant media/catalog contracts → T14 [M4]
```

T02, T03, and T05 may proceed independently once contracts are frozen. T04 follows T03. Parallel work is optional and must use isolated branches/checkouts with declared file ownership. Serialize migration numbering, shared schema/storage edits, route mounting, and integration commits. Do not let agents redefine shared contracts in separate branches.

One task is one reviewable change or a small documented sequence of changes. The orchestrator starts tasks only after dependencies pass. It updates the ledger, reviews artifacts, runs integration checks, and releases the next tasks. Task completion is based on acceptance evidence, not an agent's self-reported status.

## 5. Work packages

### T00 — Freeze design and contracts

Dependencies: none. Owner role: architecture/integration.

Deliver the contract from section 3, repository integration notes, DTOs/error semantics, resource-budget defaults, test fixture format, and an approved canonical fingerprint specification. Specify library item listing, scan start/status/cancel, evidence/candidates, manual association, verification, and catalog APIs. Distinguish existing endpoints retained from proposed endpoints.

Acceptance: every unresolved section-3 decision has a concrete documented choice; at least one fully worked canonical hash vector is independently reproducible; no change to production behavior. Do not approve new algorithms by copying implementation-generated expectations.

### T01 — Shared domain and validation

Dependencies: T00. Owner role: domain.

Create types/Zod validation for entities, identifiers, fingerprints, profiles, adapter evidence, match decisions, jobs, manifests, and API DTOs. Include nullable associations and observed components. Keep runtime contracts available to server and client without importing server modules.

Acceptance: invalid scope/entity combinations, malformed hashes, unsupported versions, and oversized metadata reject predictably. Existing TypeScript consumers remain valid. Validate proposal sidecar examples against the actual string-ID model.

### T02 — Persistence and migration

Dependencies: T01. Owner role: persistence.

Add releases/builds/artifacts, library items and locations, identifiers, fingerprints, evidence/decisions, compressed manifests, bounded cache, and jobs according to T00. Add lookup indexes and transactional association operations. Implement database and memory storage behavior. Make any legacy backfill restartable/idempotent.

Acceptance: fresh and populated migrations succeed; multiple copies and unresolved items persist; restart retains review decisions and jobs; unauthorized access is denied; foreign key deletion does not delete library files. Legacy library/download routes retain expected responses. Document storage retention and manifest cleanup.

### T03 — Safe inventory and adapter framework

Dependencies: T00. Owner role: scanner.

Implement root-scoped discovery, bounded recursive inventory, profiles, adapter probes/evidence collection, GenericDirectory and SingleFile adapters. Preserve the current immediate-child candidate boundary for generic roots; add explicit Steam-root discovery later. Multiple adapters can contribute. One adapter or item failure must not abort an entire root scan.

Acceptance: unreadable items, symlinks, devices/pipes, excessive depth/file counts, disappearing entries, and Unicode collisions yield explicit bounded outcomes. No file execution or mutation. Limits never masquerade as a complete scan. Root paths come from trusted configuration, not sidecars or filenames.

### T04 — Deterministic hashes, manifests, and cache

Dependencies: T03 and T00. Owner role: fingerprinting.

Implement streaming artifact SHA-256, qtree-v1, qmanifest-v1, compressed versioned manifest serialization, incremental cache, and cancellation. Distinguish raw inventory from profile-selected inventory. Persist through T02 interfaces when available.

Acceptance: independently specified golden vectors pass; moving a directory leaves hashes stable; permissions/times alone do not change hashes; changing one included byte changes qmanifest; same-size byte changes preserve qtree; excluded screenshots do not change identity-profile hashes. Forced verification detects same-size/same-mtime changes. Unstable/unreadable files cannot yield a complete exact result. Report peak memory and bytes read on a generated large-file fixture.

### T05 — Candidate finder and conservative resolver

Dependencies: T00. Owner role: matching.

Implement candidate generation separately from final resolution. Consume provenance, validated sidecar claims, native IDs, exact known fingerprints, and title candidates. Record evidence/source/version/rule used. Reuse existing IGDB integration for metadata candidates without making names authoritative.

Acceptance: name-only matches remain reviewable; strong conflicts produce CONFLICT; unknown exact hashes do not invent Game mappings; provider ID does not establish Build; multiple exact candidates remain ambiguous; manual decisions survive rescans. Evidence and decisions are deterministic for a fixed snapshot.

### T06 — Durable scan orchestration and APIs

Dependencies: T02–T05. Owner role: integration/backend.

Join inventory, adapters, progressive fingerprints, resolver, and persistence. Add configured concurrency/I/O budgets, job polling and progress events, cancellation, restart recovery, scan generations, and per-root locking. Keep routes thin and return before long work. Adapt existing scanner endpoints and old progress consumers deliberately.

Acceptance: duplicate requests do not duplicate items; cancel/restart works during hashing; stale runs cannot overwrite newer results; disabled/inaccessible roots degrade clearly; IGDB outage still records local inventory. A missing root never marks all its items deleted. Only complete discovery can mark prior locations missing; nothing is deleted from disk.

### T07 — Import provenance and optional sidecars

Dependencies: T06. Owner role: import integration.

Register items from successful `ImportManager` imports using known Game identity; do not rediscover it. Enqueue fingerprints separately. Add versioned bounded sidecar parser/adapter and opt-in atomic writer. Define adjacent-sidecar behavior for single-file artifacts. Integrate import retries without duplicate records.

Acceptance: failed imports do not register successful installations; imported identity is available before full hashing; sidecar write failure does not undo a successful import; rescans ignore sidecar bytes; copied/stale/foreign sidecars do not silently bind to the wrong game; sidecar and native-ID disagreement is visible. Scan defaults remain read-only.

### T08 — Steam metadata adapter

Dependencies: T06–T07. Owner role: platform adapter.

Parse bounded local ACF metadata from configured Steam library roots, resolve contained install directories, and emit AppID, available Build/depot metadata, and components with provenance. Use known provider mappings where available; unresolved AppIDs remain useful evidence. Verify supported metadata semantics against official provider documentation during implementation.

Acceptance: multiple libraries, malformed manifests, absent metadata, AppID without Game mapping, DLC, and missing optional build fields have fixtures. Different builds never collapse solely by AppID. Metadata-derived paths cannot escape roots. Parser failure falls back to generic evidence without losing the item.

### T09 — Library import and review UI

Dependencies: T06–T08. Owner role: frontend.

Expose scan controls/progress, library items and copies, state filters, candidates, reasons, manual association/correction, known version versus unknown version, missing/inaccessible state, verification progress, and duplicate candidates. A structural match must not be labeled exact. Defer similarity-specific UI until T13.

Acceptance: users can import and resolve a synthetic existing library without moving files; conflicts and unknown items are actionable; cancellation/reload preserves state; multiple copies are visible; existing library screens work. Add meaningful component/E2E coverage. Launch `npm run dev:test`, drive the running page with a headless browser, and capture visual evidence before opening/updating any UI PR; include that evidence in its description per AGENTS.md.

### T10 — M1 acceptance, performance, and rollout

Dependencies: T09. Owner role: integration/review.

Run section-6 M1 matrix, relevant regression suites, migration checks, lint/type checks, build, and live UI validation. Document configuration, exclusions, cache trust, matching semantics, read-only behavior, recovery, and enabling/disabling the feature. Use an opt-in rollout that leaves existing associations and data readable when disabled.

Acceptance: all M1 gates pass with evidence. Measure initial metadata scan, warm rescan, cold full verification, cancellation latency, memory, and stored manifest size on a documented generated workload. Publish measurements and environment; do not invent NAS throughput targets. Confirm hashing is outside latency-sensitive request handlers and cannot block normal download/import operation indefinitely.

### T11 — External DAT reference catalogs

Dependencies: T10. Owner role: catalog.

Implement bounded user-supplied DAT ingestion with source/version/checksum provenance; indexed size/checksum lookup; optional CRC32/MD5/SHA-1 computed in one streaming pass when required. Treat legacy checksums as catalog evidence, not modern cryptographic integrity guarantees. External release metadata is independent of mapping to an IGDB Game.

Acceptance: synthetic No-Intro/Redump-style fixtures distinguish regions/revisions; same ROM renamed matches; duplicate/nonunique catalog entries remain explicit; replacement imports are atomic/versioned; malformed XML, entity expansion, and oversized input are bounded. No implicit external download or catalog redistribution.

### T12 — Ordered media sets and bounded archive payloads

Dependencies: T11. Owner role: media.

Freeze qset-v1 encoding and implement supported CUE/BIN and M3U set discovery, member roles/order, missing-member state, and multi-disc aggregation. Add payload scanning for an explicitly documented small archive-format set using bounded streaming APIs; keep container and payload fingerprints separate. Split sets and archives into separate changes if necessary.

Acceptance: renaming members with updated descriptors preserves logical set identity; disc/track order changes affect it; references outside roots fail; missing members cannot produce a complete set fingerprint; identical recompressed payloads share payload identity while container hashes differ. Enforce entry count, expanded bytes, compression ratio, nesting, time, and cancellation limits. CHD logical equivalence remains deferred.

### T13 — Structural similarity

Dependencies: T10. Owner role: similarity.

Freeze deterministic seeded 128-component qstructure-minhash-v1, path and path/size-bucket feature definitions, and sketch serialization. Retrieve bounded candidates scoped by compatible profiles/platforms; compare fuller evidence before resolution. Calibrate retrieval thresholds with labeled fixtures; keep fuzzy-only results in review.

Acceptance: seed/order determinism has golden vectors; patched installations retrieve their family; unrelated games with common layouts do not auto-link; empty/tiny sets are handled; every displayed similarity names what was compared. Add precision/recall reporting on a reproducible fixture corpus without treating fabricated example percentages as targets.

### T14 — Additional adapters

Dependencies: T10 and relevant T11/T12 contracts. Owner role: one adapter per task.

Implement GOG, Epic, macOS bundles, and selected console formats as separate packages using the same evidence contracts. Later handle MAME, Microsoft Store, Heroic, and other specialized sources. Each task names supported format versions, documented metadata sources, bounded parsers, profiles, fixtures, and unsupported cases before coding.

Acceptance: failures fall back to generic scanning; inaccessible/encrypted content is reported; no permission escalation or discovered-code execution; native Game/Release/Build claims remain separate; adapter tests include malformed inputs and contradictory evidence. Verify niche format details against primary documentation during that task.

## 6. Acceptance matrix

Use synthetic fixtures, not redistributed commercial game binaries. Mark tests for deferred milestones explicitly; do not pretend they passed in M1.

| Scenario | Expected outcome | Gate |
| --- | --- | --- |
| Same artifact renamed | Same bytes/artifact; separate occurrences retained | M1 |
| Directory moved; source disappears | Stable compatible fingerprints; prior location missing; new location preserved; no delete | M1 |
| Directory copied; both exist | Two copies, same content identity where verified | M1 |
| Same-size byte edit | Same qtree possible; different freshly verified qmanifest | M1 |
| Timestamp/permission edit | Canonical hashes unchanged | M1 |
| Excluded screenshot/mod/cache addition | Raw inventory changes; profile semantics remain explicit | M1 |
| Patch without known baseline | Identity retained; changed observation; modification/build not invented | M1 |
| Known baseline with supported modification evidence | Identified modified; reasons retained | M1 |
| Steam base game plus DLC | Stable base identity; component/content change | M1 |
| Provider-specific copies | Same Game only when mapping known; distinct Releases | M1 schema; adapter evidence in M4 |
| Similar titles | Review; no name-only merge | M1 |
| Sidecar versus native-ID contradiction | CONFLICT | M1 |
| Symlink escape/device/unreadable/changing file | Bounded warning/failure; no complete exact claim | M1 |
| Cancel/crash/restart/concurrent scans | Durable coherent state; no duplicate/stale overwrite | M1 |
| Root unavailable | Accessibility warning; no mass missing/deletion inference | M1 |
| Cross-user request | Existing access policy enforced; no foreign association | M1 |
| DAT checksum with region/revision | Reference identity and provenance; Game mapping may remain unresolved | M2 |
| Multi-disc set | Multiple ordered artifacts for one release; incomplete sets explicit | M2 |
| Recompressed archive | Different container identity; same verified payload identity | M2 |
| Patch/mod structural similarity | Candidate retrieved; similarity never alone proves identity | M3 |

## 7. Agent task envelope

Use this prompt template with any framework. Supply only tasks whose dependencies are complete.

```text
Implement task <ID> from docs/proposal/LIBRARY_ITEM_IDENTIFIER_PLAN.md.
Read AGENTS.md, the source proposal, the frozen contract, and dependency handoffs.
Objective: <task outcome>.
Allowed edit areas: <explicit files/directories>.
Dependencies/available contracts: <IDs and artifacts>.
Out of scope: <later tasks and excluded behavior>.
Acceptance criteria: <copy task criteria and relevant matrix rows>.
Validation: <targeted commands and fixtures>.
Preserve unrelated working-tree changes. Do not move/delete game files.
Do not change frozen fingerprint semantics or shared APIs silently.
If a contract is missing, propose the smallest explicit amendment and block only
dependent work; continue independent checks. Never infer approval from a timeout.
Deliver code, tests, documentation, and a handoff containing changed files,
validation results, limitations, and required follow-up. Do not claim checks ran
without command evidence. Do not publish/deploy unless separately authorized.
```

Commands should use the AGENTS.md-required `rtk` prefix. During preparation `rtk` was unavailable; record that environment limitation and restore/install the expected tooling through the framework's normal setup, or use the documented raw-command debugging exception where applicable. Do not silently make `rtk` a product dependency.

Suggested verification commands: `rtk npm run check`, `rtk npm run lint`, `rtk npm test -- <targeted test paths>`, `rtk npm run build`, and selected Playwright tests. Reserve the broader regression run for integration gates or changes that justify it. UI PRs additionally require actual running visual evidence.

## 8. Execution ledger and completion rule

Maintain a framework ledger with these fields:

```yaml
id: T00
status: pending # pending | ready | running | review | done | blocked
depends_on: []
owner: null
branch: null
allowed_paths: []
contract_revision: null
acceptance_evidence: []
handoff: null
blockers: []
```

Initial state: T00 ready; T01–T14 pending. A framework may import section 4 as its dependency graph and expand each work package into its task description.

Every handoff must include actual acceptance results, schema/API/algorithm changes, migration implications, fixture coverage, unresolved limitations, and visual evidence when applicable. The integration reviewer checks that claims match the diff and tests. Mark a milestone complete only when its acceptance matrix and compatibility checks pass; unfinished future milestones remain pending.

Planning budget, not a delivery commitment: M1 roughly 5–8 engineer-weeks; M2 another 2–4; M3 another 1–2; M4 varies by adapter. This exceeds the earlier lightweight feature estimate because the proposal adds a multi-entity model, durable progressive scans, secure adapters, exact manifests, and compatibility work. Re-estimate after T00 and after the first end-to-end scan; optional parallel execution reduces elapsed time only where the dependency graph permits it.
