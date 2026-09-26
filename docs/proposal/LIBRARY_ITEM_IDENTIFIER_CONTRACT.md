# Library Item Identifier Contract

Status: delivered as one squashed T00 contract commit — Part I / T00a (domain
and ownership), Part II / T00b (canonical fingerprint semantics), Part III /
T00c (candidate generation and match-decision contract), Part IV / T00d
(durable scan-job lifecycle and resource budgets), Part V / T00e (HTTP API and
DTO contract), T00f (open-question resolutions OQ-1–OQ-13, section 33), T00f2
(Part III matching/decision open-question resolutions OQ-14–OQ-20, section 33),
T00f3 (Part IV scan-job/resource open-question resolutions OQ-21–OQ-30, section
33), and T00f4 (Part V API/DTO open-question resolutions OQ-31–OQ-38, section
33).

Source documents (copied into this branch unchanged):

- [Game Identification and Fingerprinting Proposal](./LIBRARY_ITEM_IDENTIFIER.md)
- [Library identification: agent execution plan](./LIBRARY_ITEM_IDENTIFIER_PLAN.md)

This document is the T00 deliverable required by plan
[section 3 "Contracts to freeze before implementation"](./LIBRARY_ITEM_IDENTIFIER_PLAN.md)
and plan [section 5 "T00 — Freeze design and contracts"](./LIBRARY_ITEM_IDENTIFIER_PLAN.md).
It comprises the following parts:

- **Part I (T00a)** — domain and ownership decisions (sections 1–10).
- **Part II (T00b)** — canonical fingerprint bytes, framing, profiles,
  completeness, cache trust, and golden vectors (sections 11–24).
- **Part III (T00c)** — candidate generation versus final resolution, evidence
  classes and provenance, identifier/exact-hash mapping, untrusted sidecar
  decisions, strong conflicts, manual-association persistence, identity/content/
  accessibility/job state separation, revision/snapshot protection, and
  explainable results (sections 25–32).
- **Part IV (T00d)** — durable job states/transitions, stages and progress
  counters, scan levels, default budgets and concurrency/I/O limits, cancellation
  points, restart recovery, per-root locking and duplicate starts, scan
  generations and snapshot binding, changing-file retries and incomplete
  outcomes, missing/inaccessible-root rules, and notification versus durable
  state (sections 35–48).
- **Part V (T00e)** — HTTP API and DTO contract: route names, listing/detail,
  scan start/status/cancel, evidence/candidates, manual association/correction,
  verification, catalog endpoints, pagination, auth, revision/`If-Match`
  stale-write semantics, error codes, and a retained-versus-new compatibility
  map (sections 49–61).
- **Still out of scope** (owned by downstream tasks): UI; external catalog
  _ingestion_ (matching _against_ catalogs is in Part III, the ingestion pipeline
  is T11); the standalone sidecar _parser_ implementation; and the reserved
  `qset` / `qrelease-set` / `qstructure-minhash` / `qtree-casefold` fingerprint
  kinds, which are explicitly not specified here (section 23).

Part II does not change any Part I decision marked **D** or **INV**; where it
extends or narrows a Part I statement this is flagged in the "Compatibility with
T00a" note in section 11 and in the question list (section 33). Part III does
not change any Part I **D**/**INV** or Part II **F**/**INV**; its
"Compatibility with prior parts" note is in the Part III introduction. Part IV
does not change any Part I **D**/**INV**, Part II **F**/**INV**, or Part III
**M**/**INV**; its "Compatibility with prior parts" note is in the Part IV
introduction. Part V does not change any Part I **D**/**INV**, Part II
**F**/**INV**, Part III **M**/**INV**, or Part IV **J**/**INV**; its
"Compatibility with prior parts" note is in the Part V introduction.

> Delivery note: this contract is the single squashed T00 commit; the former
> per-slice commits (T00a–T00e, T00f–T00f4) were squashed into it. Its parts are
> not reviewed, merged, or consumed standalone.

---

## 1. Scope

In scope for this slice:

- The identity levels Game / Release / optional Build / Artifact / Installation
  and their ownership and referential rules.
- Occurrence (Installation) identities versus logical Artifact identities.
- Questarr entity ID type and nullable associations.
- Provider/native identifier scope, namespaces, and uniqueness.
- Root identity and authorization.
- Legacy `games.libraryPath` / `game_files` projection and backfill.
- Deletion and duplicate semantics.

Out of scope for this slice (do not implement or freeze here):

- Fingerprint kinds, byte-exact canonical encoding, framing, golden vectors.
- Job lifecycle, scheduling, budgets, cancellation, snapshots/revisions.
- HTTP routes, DTO shapes, error envelopes, UI.
- Candidate generation and matching rules (Rules A–G), match-state transitions.
- External catalog ingestion details, sidecar (`.questarr.json`) parsing rules.

> This out-of-scope list scopes **Part I (T00a)** only. Later parts of this same
> contract supply the remaining items: Part II supplies the fingerprint items
> (kinds, byte-exact encoding, framing, golden vectors); Part III supplies
> candidate generation and matching rules; Part IV supplies job lifecycle,
> scheduling, budgets, cancellation, and snapshots/revisions; and Part V supplies
> HTTP routes, DTO shapes, and error envelopes. Only UI, external catalog
> _ingestion_, and the sidecar _parser_ implementation remain out of scope for
> the whole T00 contract (the reserved fingerprint kinds are listed in section
> 23).

---

## 2. Entity levels and ownership

The proposal
([section 2](./LIBRARY_ITEM_IDENTIFIER.md)) defines a deliberately layered
model in which each level has a distinct meaning:

```text
Game
└── Release
    └── Build        (optional)
        └── Artifact
            └── Installation
```

Proposal section 2 further requires that an Installation never be assumed equal
to a canonical Build merely because a directory resembles one, and
[section 44](./LIBRARY_ITEM_IDENTIFIER.md) presents the SQLite/Drizzle
conceptual model using the table name `library_items` for the observed item.

### Decisions

- **D2.1 — Two axes, not one.** The model has a _logical identity chain_
  (`Game → Release → optional Build → Artifact`) and an _occurrence axis_
  (`Installation`). The occurrence is the concrete thing Questarr observes in a
  library; the logical chain is what it may resolve the occurrence to. See
  proposal [sections 2 and 4](./LIBRARY_ITEM_IDENTIFIER.md).
- **D2.2 — Naming.** In this contract, **Installation** is the domain term for a
  single observed item and is the thing persisted as a `library_items` row
  (plan/proposal spelling). A **LibraryItem** and an **Installation** are the
  same concept; the persistence name stays `library_items` to match
  proposal section 44.
- **D2.3 — Formal meaning of each level.**
  - `Game` — the conceptual work; roughly the IGDB level.
  - `Release` — a platform/region/edition-specific release of exactly one Game.
  - `Build` — an optional software revision of exactly one Release. The Build
    level is optional because not every platform exposes a useful build concept
    ([proposal §2 "Build"](./LIBRARY_ITEM_IDENTIFIER.md)).
  - `Artifact` — one immutable distributable or media object; location
    independent, and the level at which exact identity can exist.
  - `Installation` — one observed collection/files present under a Questarr
    root, with its own occurrence identity (D3.1).
- **D2.4 — Ownership.** `Game` rows are user-owned today (`games.userId`,
  `games.user_id` FK). Releases, Builds, Artifacts, and Installations are
  Questarr-owned rows derived from observation; they are **not** given a
  per-user owner column in this slice. User visibility of an Installation is
  earned through its association to a user-owned Game, not through a row-level
  owner (see section 6).

---

## 3. Occurrence identity vs logical artifact identity

Plan section 3 ("Domain and ownership") requires:

> Separate logical Artifact identity from local occurrence/location. Equal hashes
> may share artifact identity while retaining distinct library items and paths.

### Decisions

- **D3.1 — Occurrence identity is its own identity.** Every Installation
  (`library_items` row) has its own immutable string ID, independent of any
  Artifact. Two Installations with byte-identical content are two distinct rows
  with two distinct IDs.
- **D3.2 — Artifact identity is logical and shared.** A first-class `Artifact`
  entity exists. Zero, one, or many Installations may reference the same
  Artifact. An Artifact is location independent; installing/copying the same
  bytes twice creates two Installations and does not create a second Artifact
  identity.
- **D3.3 — Association is nullable and one-directional.** `library_items`
  references an Artifact by a nullable `artifact_id`. An Installation with no
  exact/derived identity yet is still persisted with `artifact_id = null`; it
  is never blocked from existing and never force-created into an Artifact.
- **D3.4 — Artifact association may be unresolved.** The proposal
  ([§2 "Artifact"](./LIBRARY_ITEM_IDENTIFIER.md)) says an Artifact "can usually
  be exactly fingerprinted," which is not the same as always. An Artifact may
  exist without a Build/Release/Game association, and an Installation may
  reference an Artifact whose higher-level associations are still null.
- **D3.5 — Proposal section 44 is illustrative.** The
  [section 44](./LIBRARY_ITEM_IDENTIFIER.md) sketch has no `artifacts` table and
  no `library_items.artifact_id`. This contract makes the Artifact entity
  explicit; the concrete columns/indexes are a T02 persistence obligation
  (section 7) and must not contradict D3.1–D3.4.
- **D3.6 — Artifact creation is lazy (resolved T00f, OQ-7).** An `Artifact` row
  is created only when an exact identity value exists for an Installation — a
  computed `sha256` (single file, section 16) or a compatible canonical
  `qmanifest-v1` (section 15). It is never created eagerly for every
  Installation, and an Installation without an exact identity persists with
  `artifact_id = null` (D3.3). Equal compatible exact-identity values map
  multiple Installations to one Artifact (D3.2/D9.3); the sharing key is the
  full fingerprint reference (kind, algorithm, algorithm version, profile,
  inclusion-policy version, digest), never a path. A `qtree` value, an
  `INCOMPLETE`/`FAILED` inventory, or a `cached`/`mixed` value that is not a
  fresh content proof can never create or resolve an Artifact (F14.5, F19.2,
  F22.5, INV-37).

---

## 4. String IDs and nullable associations

Plan section 3 requires string Questarr IDs (numeric IDs in proposal examples are
illustrative) and nullable associations with no placeholder entities.

### Decisions

- **D4.1 — All Questarr-owned entity IDs are strings.** Existing tables already
  use `text(...).primaryKey()` and `randomUUID()` IDs (for example `games.id`,
  `root_folders.id` in `shared/schema.ts`; ID creation in `server/storage.ts`).
  New entities (`game_releases`, `game_builds`, `artifacts`, `library_items`,
  `game_identifiers`) use the same non-empty opaque string ID convention
  (UUID v4 via `randomUUID()` unless a later amendment to this contract states
  otherwise).
  Numeric IDs such as `gameId = 123` or `questarrId: 123` in the proposal are
  **illustrative only** and must never appear as persisted IDs or be parsed as
  numbers.
- **D4.2 — Association columns on an Installation are nullable.**
  `library_items.game_id`, `.release_id`, `.build_id`, and `.artifact_id` are all
  nullable.
- **D4.3 — No placeholder Games.** A discovered Installation may persist with
  `game_id = NULL`. Questarr must not create a placeholder/temporary `Game` to
  satisfy a NOT NULL constraint. This is required by plan section 3 and is the
  mechanism behind proposal [§48 "UNKNOWN"](./LIBRARY_ITEM_IDENTIFIER.md).
- **D4.4 — Logical-chain referential constraints.**
  - `game_releases.game_id` is NOT NULL → FK to `games.id` (a Release belongs to
    exactly one Game).
  - `game_builds.release_id` is NOT NULL → FK to `game_releases.id` (a Build
    belongs to exactly one Release).
  - If `library_items.build_id IS NOT NULL` then `release_id IS NOT NULL` and it
    equals the Build's `release_id`.
  - If `library_items.release_id IS NOT NULL` then `game_id IS NOT NULL` and it
    equals the Release's `game_id`.
  - These are the concrete form of plan section 3's "Define referential
    constraints so a Build belongs to the selected Release and a Release belongs
    to the selected Game."
- **D4.5 — Historical proposal field names.** Proposal section 44 uses
  `game_id`, `release_id`, `build_id`; the actual Questarr column names follow
  the existing `snake_case` convention in `shared/schema.ts`.

### Invariants

- **INV-1** Every Questarr-owned entity ID is a non-empty string; no code path
  assumes a numeric ID.
- **INV-2** An Installation row exists and is queryable while
  `game_id`, `release_id`, `build_id`, and `artifact_id` are all NULL.
- **INV-3** `library_items.build_id NOT NULL ⇒ release_id NOT NULL` and the
  Release matches the Build's parent.
- **INV-4** `library_items.release_id NOT NULL ⇒ game_id NOT NULL` and the Game
  matches the Release's parent.
- **INV-5** `game_builds.release_id` and `game_releases.game_id` are never NULL.
- **INV-6** Removing an association (e.g. unlinking a Game) clears the dependent
  narrower associations in the same transaction so INV-3/INV-4 cannot be
  violated.

---

## 5. Provider / native identifiers: scope and uniqueness

The proposal [section 4](./LIBRARY_ITEM_IDENTIFIER.md) keeps semantic
identifiers separate from derived fingerprints and suggests:

```ts
interface GameIdentifier {
  namespace: string;
  value: string;
  scope: "game" | "release" | "build" | "artifact";
  source: string;
  metadata?: Record<string, unknown>;
}
```

Plan section 3 additionally requires persisting unresolved native identifiers as
evidence before a target entity exists (for example a Steam AppID recognized
while its IGDB mapping is unresolved) and forbids any global uniqueness
assumption on short native IDs or build numbers.

### Decisions

- **D5.1 — Scope is the entity level, not a hash.** `scope ∈ {game, release,
build, artifact}` and it constrains the target entity type. A scope never
  points at an Installation (the occurrence axis is not an identifier scope).
- **D5.2 — Namespaces are normalized strings.** Namespaces are lowercase,
  trimmed, `.`-separated tokens (e.g. `igdb`, `steam.app`, `steam.build`, `gog`,
  `psn`, `ps2.serial`, `switch.title`). Producers must not invent synonyms for
  the same provider; a canonical namespace registry is a T01 concern.
- **D5.3 — Values are normalized per namespace.** Numeric provider IDs are
  canonical decimal strings without leading zeros; serial-style IDs are
  trimmed and upper-cased (e.g. `SLUS-xxxxx`); unknown namespaces preserve the
  value verbatim after trimming. Normalization is applied before storage and
  before uniqueness checks.
- **D5.4 — Identifier evidence may precede the entity.** `game_identifiers` may
  hold a claim with `entity_type = NULL` / `entity_id = NULL` (unresolved
  evidence). It must be possible to record and later resolve a claim without
  inventing a placeholder Game/Release/Build/Artifact.
- **D5.5 — Resolved uniqueness is scoped; there is no global uniqueness.**
  - A _resolved_ identifier (`entity_id IS NOT NULL`) is unique on
    `(namespace, scope, value)`.
  - Unresolved evidence claims are **not** globally unique; the same native ID
    observed on two different Installations is two evidence rows (deduplicated
    by their owning occurrence/observation, later slice).
  - The same numeric `value` may legitimately exist in different namespaces or
    scopes. Provider build numbers are scoped to their provider/release, so a
    bare build number must never be treated as globally unique.
- **D5.6 — Provider build IDs live at build scope.** A `steam.build` (or
  equivalent) identifier is build-scope and resolves only within its provider
  release context; it is never used as a global lookup key on its own.
- **D5.7 — `source` is provenance, not identity.** `source` records where the
  claim came from (importer, sidecar, filesystem evidence). Two claims with the
  same namespace/value/scope but different `source` are not collapsed into one;
  contradictions are surfaced rather than overwritten.
- **D5.8 — Foreign IDs are validated, never trusted.** Questarr IDs embedded in
  sidecars are validated against the local database, and portable identifiers
  are checked for local consistency before they may resolve an entity. This
  follows proposal [§34](./LIBRARY_ITEM_IDENTIFIER.md) and
  [§49](./LIBRARY_ITEM_IDENTIFIER.md) ("Validate Questarr IDs from sidecars
  against the database"). Detailed sidecar handling is T07. **OQ-5 is resolved
  by deferral (T00f):** the validation _mechanics_ are owned by T07, and the
  frozen safe M1 behavior is that an unvalidated portable identifier never
  selects or creates an entity and remains evidence only (M28.3, INV-53).
- **D5.9 — Polymorphic target, application-enforced.** Because a single
  `game_identifiers` table can target several entity tables, SQLite/Drizzle
  cannot express the cross-table FK. The `(entity_type, entity_id)` pair is
  polymorphic and its integrity is enforced by the storage layer, not the
  database (OQ-3, resolved T00f).
  - `entity_type ∈ {game, release, build, artifact}`; for a resolved row
    `entity_type` equals `scope` (D5.1/INV-7), and a mismatch is rejected.
  - Every write that sets `entity_id` verifies, inside the same transaction,
    that the target row exists in the table named by `entity_type`; otherwise
    the write is rejected as unresolved evidence (`entity_type = NULL`,
    `entity_id = NULL`).
  - When a target entity is deleted, its resolved identifier rows are
    downgraded to unresolved evidence in the same transaction rather than left
    dangling; D8.2/D8.3 clear associations without deleting identifier history.
    T02 additionally runs an orphan sweep that finds any `entity_id` missing
    from its target table and downgrades it with a recorded warning; the sweep
    never re-points an identifier to a different entity.
- **D5.10 — An Installation may carry many identifiers.** Identifiers attach to
  the logical entity levels; a single Installation reads them through its
  Artifact/Build/Release/Game associations, and unresolved identifiers are
  recorded as evidence without a target.

### Invariants

- **INV-7** `scope` is one of `game`, `release`, `build`, `artifact`; there is no
  `installation` scope.
- **INV-8** No two resolved identifier rows share the same
  `(namespace, scope, value)`.
- **INV-9** Unresolved evidence rows (`entity_id IS NULL`) impose no global
  uniqueness and never silently overwrite a resolved row.
- **INV-10** A lookup by native identifier always constrains namespace and
  scope; no lookup is performed on `value` alone.
- **INV-11** A provider build number is resolved only through its provider
  release context, never as a global key.

---

## 6. Root authorization and ownership

Relevant baseline (inspected on this branch):

- `user_settings.libraryRoot` is **per user** (`shared/schema.ts`).
- `root_folders` is a **global** table with a unique `path` and an explicit
  `allowDelete` opt-in; it has **no owner/user column** (`shared/schema.ts`).
- Root-folder and scan routes require authentication but apply no per-user
  ownership filter (`server/routes.ts`, `/api/root-folders*`,
  `/api/library/scan*`).
- Game ownership is enforced separately (`resolveOwnedGame` in
  `server/routes.ts` returns 403 when `game.userId !== req.user.id`).
- File deletion is guarded by root containment
  (`server/path-security.ts` `assertWithinRoots`, plus
  `isWithinDeletableRootFolder` in `server/root-folders.ts`).

Plan section 3 requires: define actor authorization using the existing
root-folder access policy; roots currently lack an explicit owner field; do not
infer private ownership from a requesting user's ID; prevent cross-user
association or information leakage.

### Decisions

- **D6.1 — Root identity has two kinds.**
  - `LIBRARY_ROOT` — a user's configured `user_settings.libraryRoot`,
    implicitly owned by exactly that user.
  - `ROOT_FOLDER` — a global `root_folders` row, owned by no user.
    Every Installation references exactly one root identity plus a root-relative
    path (D7.1).
  - **Persistence shape (resolved T00f, OQ-1).** Root identity is a normalized
    `library_roots` table carrying a stable opaque string `id` (D4.1), not
    columns duplicated onto `library_items`. A row has: `id` (text primary
    key); `root_kind` (`LIBRARY_ROOT` | `ROOT_FOLDER`); `owner_user_id`
    (nullable FK to `users.id`, NOT NULL exactly when
    `root_kind = LIBRARY_ROOT`); `root_folder_id` (nullable FK to
    `root_folders.id`, NOT NULL for an active `ROOT_FOLDER`); a canonical
    absolute `path`; and timestamps. Uniqueness is one row per source —
    `(owner_user_id)` for `LIBRARY_ROOT`, `(root_folder_id)` for `ROOT_FOLDER`.
    `library_items` then holds `root_id` (NOT NULL FK → `library_roots.id`)
    plus `relative_path` (NOT NULL), and never a bare absolute path. The `id`
    is generated once on first observation of the source row and reused for its
    lifetime; re-pointing a `LIBRARY_ROOT`'s `path` updates that row and marks
    prior occurrences inaccessible rather than re-keying them. A `ROOT_FOLDER`
    whose `root_folders` row is removed keeps its `library_roots` row with
    `root_folder_id = NULL` and is reported `ROOT_UNRESOLVED` (D8.5/INV-24).
    This matches proposal [§44](./LIBRARY_ITEM_IDENTIFIER.md)'s
    `library_root_id` reference. M1 writes a `LIBRARY_ROOT` row for each
    configured library root and a `ROOT_FOLDER` row for each enabled root
    folder; no Installation is persisted without a `root_id`.
- **D6.2 — Do not add per-user ownership to roots in this slice.** `root_folders`
  remains owner-less. No owner column is inferred from `req.user.id` for a root
  or for an Installation.
- **D6.3 — Preserve the existing access policy.** Reading/configuring
  root-folders and starting scans stays at the current authenticated (not
  per-user-filtered) level. This slice does not tighten or broaden those routes;
  any change to them is a later API decision (the new library-item API is
  proposed in Part V and implemented by T06).
- **D6.4 — Ownership accrues at Game association.** An Installation becomes
  user-visible as "mine" only when it is associated with a Game whose
  `games.userId` equals the acting user. Unassociated Installations are shared
  discovery data, not private library content.
- **D6.5 — Association requires ownership.** Associating an Installation to an
  existing Game requires the acting user to own that Game (same rule as
  `resolveOwnedGame`, 403 otherwise). This prevents cross-user association.
- **D6.6 — No information leakage.** An Installation associated with another
  user's Game must not disclose that Game's identity (title, identifiers,
  metadata) to non-owners. The exact read-projection rules are a later API slice;
  the domain rule is that visibility follows the associated Game's owner.
- **D6.7 — Import provenance keeps its owner.** When an import succeeds,
  `ImportManager` already knows the owning Game; the resulting Installation
  records Questarr provenance and inherits the Game's owner through that Game
  association. It does not acquire a row-level owner from the scanning user.
- **D6.8 — Path translation does not choose paths.** Downloader path
  translation via `PathMappingService` happens before provenance registration;
  metadata must never be used to select a filesystem path. Paths are always
  observed under a configured root.

### Invariants

- **INV-12** `root_folders` has no owner column and roots are not filtered per
  user in this slice.
- **INV-13** No Installation row receives an owner derived from the requesting
  user's ID.
- **INV-14** An Installation can be associated to a Game only when the acting
  user owns that Game.
- **INV-15** Non-owners cannot read another user's Game identity through an
  Installation.
- **INV-16** Every Installation's absolute location is inside exactly one
  configured root; no scanned metadata can select an arbitrary path
  (proposal [§49](./LIBRARY_ITEM_IDENTIFIER.md)).

---

## 7. Legacy path projection and backfill

Plan section 3 requires:

> Existing `game.libraryPath` and `game_files` remain compatible projections
> during M1. Never create one permanent relational row per scanned file.
> Specify deterministic projection selection for multiple copies and avoid
> overwriting an existing managed path.

> Backfill legacy paths without claiming that existing title-based matches are
> verified; preserve them as legacy associations with recorded evidence.

Baseline: `games.libraryPath` is a nullable text path; `game_files` is a
per-file table used by the download/import flow (`shared/schema.ts`); the
scanner writes `libraryPath` for matched games (`server/library-scanner.ts`).

### Decisions

- **D7.1 — Unique occurrence location.** An Installation is uniquely identified
  by `(root_id, normalized relative path)`, where `root_id` is the
  `library_roots.id` of D6.1. The relative path is stored relative to its root,
  never as an absolute path.
- **D7.2 — Location normalization.** Root-relative paths are normalized to NFC,
  use `/` separators, and preserve case (consistent with plan section 3's path
  normalization). Distinct paths that normalize to the same NFC name are a
  canonical collision: report/reject, never silently merge or drop.
- **D7.3 — No per-scanned-file rows.** Questarr must never create one permanent
  relational row per scanned file for identification. `game_files` stays the
  existing per-file projection of the _import/download_ flow and is not
  repurposed as a scan database. File-level inventory is transient/manifest data
  (later slice), not a permanent row per file. **OQ-6 is resolved (T00f):**
  existing `game_files` rows are _not_ backfilled or projected into
  Installations. Identification backfill reads only `games.libraryPath` (D7.7),
  and import provenance comes from `ImportManager` (D6.7), never from
  `game_files`, because `game_files` is per-file and `downloadId`-scoped
  (`shared/schema.ts`). `game_files` keeps serving the download/import UI
  unchanged throughout M1.
- **D7.4 — `games.libraryPath` stays a compatible projection during M1.**
  Existing readers of `games.libraryPath` (health check, delete flow, scanner)
  keep working. The identification model is additive.
- **D7.5 — Deterministic projection selection.** When more than one Installation
  is associated with a Game and a single `libraryPath` must be projected, choose
  in this order:
  1. an Installation whose normalized path already equals the current
     `libraryPath`;
  2. otherwise an Installation with an authoritative/verified association over a
     legacy/probable one;
  3. otherwise the Installation whose root is the owner's configured
     `LIBRARY_ROOT` (D6.1) over a `ROOT_FOLDER`;
  4. otherwise the most recently observed, then lexicographically smallest
     normalized relative path.
     The chosen Installation and the reason are recorded so the projection is
     reproducible and explainable. The exact persisted form is a T02 decision.
- **D7.6 — Never overwrite a managed path.** Projection must not replace a
  non-null `libraryPath` that still resolves to an existing path with a
  different discovered path unless an explicit user/config action requests it.
- **D7.7 — Backfill is legacy, not verification.** Backfill creates an
  Installation for each non-null `games.libraryPath`, marks the association as
  `legacy` (title-based match), and records evidence. It must not mark the
  association verified/authoritative, and must not be presented as content
  verification.
- **D7.8 — Backfill is idempotent and restartable.** Re-running backfill keys on
  `(root identity, normalized relative path)` and creates no duplicate
  Installation and no duplicate association evidence.
- **D7.9 — Root inference during backfill.** For a `libraryPath`, root identity
  is inferred by containment: the owning user's configured `libraryRoot` first,
  then a matching enabled `root_folders` path. Precedence is explicit because a
  path can sit under both.
  - **Containment predicate (resolved T00f, OQ-2).** After canonicalization, a
    path `P` is contained in root `R` iff `P == R` or `P` begins with
    `R + "/"` — the same boundary rule as `isWithinDeletableRootFolder` in
    `server/root-folders.ts`. When several roots match, precedence is: (1) the
    owning user's `LIBRARY_ROOT`; then (2) the `ROOT_FOLDER` with the longest
    canonical matching prefix. Two enabled roots with the same canonical path
    are a collision, recorded as an error and never guessed between.
  - **Identification precedence is independent of deletion.** Choosing the
    `LIBRARY_ROOT` (or any root) for identification never grants delete rights:
    explicit file deletion still requires the independent `allowDelete` and
    `assertWithinRoots` checks (D8.6). Identification and deletion evaluate root
    membership separately, so the delete-time `allowDelete` behavior is not
    affected by this precedence.
  - If neither contains the path, the item is recorded with an unresolved root
    rather than guessed.

### Invariants

- **INV-17** `(root_id, normalized relative path)` is unique across
  Installations.
- **INV-18** Re-running backfill is a no-op with respect to row counts.
- **INV-19** Backfill never sets a verified/authoritative association from a
  title match alone.
- **INV-20** Backfill never creates one row per scanned file.
- **INV-21** Projection never overwrites a still-valid managed `libraryPath`
  without explicit user/config intent.

---

## 8. Deletion semantics

Baseline behaviors to preserve:

- Deleting a Game requires ownership (`resolveOwnedGame`).
- Files are deleted only with an explicit `deleteFiles=true` and only when the
  path is inside the configured library root or inside a root folder whose
  `allowDelete` is set; otherwise the response reports
  `outside-library-root` (`server/routes.ts`).
- `game_files.gameId` has `onDelete: "cascade"` (download-derived rows).
- Deleting a `root_folders` row currently removes just the row
  (`server/routes.ts`, `DELETE /api/root-folders/:id`) and never touches disk.

### Decisions

- **D8.1 — Files are never deleted by a database cascade.** No FK cascade in the
  identification model deletes a filesystem path. Only the explicit delete flow
  (with containment checks) removes files. This preserves the T02 acceptance
  requirement "foreign key deletion does not delete library files."
- **D8.2 — Deleting a Game preserves observations.** Deleting a Game clears the
  Installation's associations (`game_id`, and therefore the dependent
  `release_id`/`build_id`, per INV-6) instead of deleting the Installation or
  its Artifact. The files still exist on disk, so the occurrence must survive.
  The cleared association is retained as an immutable evidence/decision-history
  entry recording the prior target, actor, timestamp, reason, and the
  observation revision it was made against; it is never destructively dropped
  (OQ-4, resolved T00f). It is history, not a live association, and never
  re-links on its own. The concrete storage shape is T01/T02 (OQ-15, resolved by
  deferral in T00f2); the M1 rule "retain the clearing event" is frozen here.
- **D8.3 — Artifacts are shared and not game-scoped for deletion.** Deleting a
  Game never deletes an Artifact or a Release/Build that is not exclusively
  owned through that Game association. Orphan cleanup, if any, is a separate
  retention concern (later slice).
- **D8.4 — Deleting a Release/Build/Artifact is restricted.** Deletion is
  blocked or association-clearing while Installations still reference the
  entity; it never removes library files.
- **D8.5 — Deleting a root does not delete observations.** Removing a
  `root_folders` row leaves its Installations in place, marked with an
  unresolved/orphaned root and retaining their last-known relative path. This
  keeps `DELETE /api/root-folders/:id` non-destructive to library data.
- **D8.6 — Explicit file deletion stays contained.** When a user explicitly
  deletes files, containment is enforced by `assertWithinRoots` /
  `isWithinDeletableRootFolder` before any `fs` removal. Discovered files outside
  a deletable root are never removed.

### Invariants

- **INV-22** No DB deletion cascade removes a filesystem path.
- **INV-23** Deleting a Game leaves its Installations and Artifacts in place
  with associations cleared.
- **INV-24** Deleting a root leaves its Installations present (root unresolved).
- **INV-25** File removal happens only through the explicit, containment-checked
  delete flow.

---

## 9. Duplicate and occurrence semantics

- **D9.1 — Same location ⇒ same occurrence.** Two observations of the same
  `(root, normalized relative path)` update one Installation; they do not create
  duplicates. (This is the occurrence form of INV-17; the snapshot/revision
  mechanics that detect changes are a later slice.)
- **D9.2 — Different location ⇒ different occurrence.** Identical content under
  two roots or two relative paths yields two Installations (D3.1).
- **D9.3 — Same content ⇒ shared Artifact, not collapsed rows.** Equal exact
  identity may map multiple Installations to one Artifact while every
  Installation retains its own row, ID, root, and relative path (plan section 3).
- **D9.4 — Duplicate detection is per occurrence first.** "Is this item already
  imported elsewhere?" (proposal [§1](./LIBRARY_ITEM_IDENTIFIER.md)) is answered
  by occurrence rows plus shared Artifact identity; it must not be implemented by
  silently merging or moving files. No cross-root deduplication deletes data.
- **D9.5 — Legacy duplicates are preserved.** Multiple Games pointing at the
  same or near-identical paths remain distinct associations during M1; the
  projection rules in D7.5 choose deterministically without deleting the others.
- **D9.6 — Multiple copies with equal fingerprints persist.** Multiple copies of
  the same Artifact are expected and supported; there is no "one Installation per
  Artifact" rule.

### Invariants

- **INV-26** No scan or backfill path produces two Installations for the same
  `(root, normalized relative path)`.
- **INV-27** No deduplication logic merges or deletes distinct occurrences or
  files implicitly.

---

## 10. Repository integration references

Plan section 2 ("Repository integration map") lists the entry points that later
tasks must verify against the checkout. Verified present on this branch:

| Area                      | Path                                                                        | Contract relevance                                                                                                                    |
| ------------------------- | --------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| Existing scanner          | `server/library-scanner.ts`                                                 | Existing immediate-child discovery and IGDB name-matching that the new flow replaces behind a facade (D7, section 6).                 |
| Import orchestration      | `server/services/ImportManager.ts`                                          | Provenance source: a successful import knows the Game (D6.7).                                                                         |
| Import filesystem helpers | `server/services/ImportStrategies.ts`                                       | Reuse safe helpers; scanning must not transfer/reorganize files (D8.6).                                                               |
| Path translation          | `server/services/PathMappingService.ts`                                     | Translate downloader paths before provenance; never use metadata to pick paths (D6.8).                                                |
| Root containment          | `server/path-security.ts`                                                   | `assertWithinRoots` is the containment primitive for reads and explicit deletes (D8.6, INV-25).                                       |
| Root folders              | `server/root-folders.ts`                                                    | `isWithinDeletableRootFolder`; global roots without owner (D6.2, D8.5).                                                               |
| Domain schema             | `shared/schema.ts`                                                          | `games`, `game_files`, `root_folders`, `user_settings.libraryRoot`; string ID and nullable-association conventions (section 4, D7.4). |
| Persistence               | `server/storage.ts`                                                         | Storage interface plus memory and DB implementations; `randomUUID()` ID convention (D4.1).                                            |
| Persistence               | `server/migrate.ts`, `migrations/`                                          | Drizzle journal-driven migration convention (`migrations/meta/_journal.json`, `db:generate`, `db:migrate`) for T02 (section 7).       |
| Routes                    | `server/routes.ts`, `server/routes/import.ts`                               | Existing root-folder, scan, file, deletion, and import endpoints must remain compatible (D6.3, D8.6).                                 |
| Settings UI               | `client/src/pages/settings.tsx`, `client/src/components/ImportSettings.tsx` | Root configuration and import-settings entry points (later slice).                                                                    |
| Tests                     | `server/__tests__/`, `client/__tests__/`, `tests/e2e/`                      | Extend coverage and add synthetic identification fixtures (plan section 2).                                                           |

Ownership/authorization references confirmed in the baseline:

- `resolveOwnedGame` in `server/routes.ts` → 403 on `game.userId !== req.user.id`
  (D6.5).
- `games.userId` FK and `user_settings.libraryRoot` in `shared/schema.ts`
  (D2.4, D6.1).
- `root_folders` has no owner column; `path` is unique (D6.2, INV-12).
- ID generation is `randomUUID()` in `server/storage.ts` (D4.1).
- Migration registration is the Drizzle journal in
  `migrations/meta/_journal.json`, applied by `runMigrations()` in
  `server/migrate.ts` (section 7).

Proposed new implementation locations from plan section 2, frozen here as the
T00 choice (plan section 2: "T00 chooses exact names and route mounting"): the
shared runtime contracts module `shared/library-identification.ts`, the
`server/services/library-identification/` service package, and a dedicated
library-identification route module registered beside the existing imports
routes in `server/routes.ts` (P49.1). T06 registers the mount consistent with
repository conventions.

### Test fixture format (plan T00 deliverable)

Plan [section 5 "T00 — Freeze design and contracts"](./LIBRARY_ITEM_IDENTIFIER_PLAN.md)
requires T00 to deliver a **test fixture format** so downstream tasks (T03–T09,
T11–T14) express synthetic identification fixtures in one comparable shape. T00
freezes only the format, not the cases; each owning task supplies its own
fixtures.

- **Synthetic content only.** A fixture uses generated bytes/trees or a
  descriptor alone; redistributed commercial game data is never a fixture (plan
  section 6). Fixtures are never registered as user library content.
- **Descriptor.** Each fixture carries a machine-readable descriptor declaring:
  the fixture id; the profile id/version and inclusion-policy version in use
  (F21.1); each entry's expected decision (`included`/`excluded` plus reason,
  F20.1) and class; the expected completeness (F19.1) and expected warning/error
  classes (F18.1); the expected fingerprint references or section 24
  golden-vector id where a digest is asserted; the expected
  `identity_state`/`content_state`/`accessibility_state` (M30.1); for job
  fixtures the expected durable job outcome (J35.1/J43.4); and the owning
  milestone (M1–M4).
- **Frozen expectations only.** A fingerprint expectation uses the byte-exact
  definitions (sections 11–16) and a section 24 vector or documented preimage; a
  fixture never encodes implementation-generated expectations (plan T00
  acceptance).
- **Negative cases are first-class.** Canonical collision (F13.7),
  symlink/special/unsupported names (section 17), unreadable/changing entries
  (section 18), over-limit scans (F19.3), and stale/foreign sidecars (section 28)
  each have a fixture with their expected bounded outcome.
- **Milestone marking.** Deferred-milestone fixtures (M2–M4) are marked and are
  never reported as M1 passes (plan section 6). Paths inside a fixture are
  root-relative (F13.1) and NFC (F13.4). The on-disk location and descriptor
  filename are T03/T06 conventions; this contract freezes only the descriptor's
  required fields and the synthetic-only rule.

---

---

# Part II — T00b: canonical fingerprint contract

This part implements the "Fingerprint semantics" bullets of plan
[section 3](./LIBRARY_ITEM_IDENTIFIER_PLAN.md) and turns the proposal's
illustrative encoding ([§6–10](./LIBRARY_ITEM_IDENTIFIER.md),
[§14](./LIBRARY_ITEM_IDENTIFIER.md), [§31–33](./LIBRARY_ITEM_IDENTIFIER.md),
[§45](./LIBRARY_ITEM_IDENTIFIER.md), [§51](./LIBRARY_ITEM_IDENTIFIER.md)) into a
byte-exact, unambiguous specification. It adds no runtime code, jobs, APIs, UI,
matching rules, or catalog behavior.

## 11. Fingerprint model, kinds, and canonical references

### Decisions

- **F11.1 — Frozen v1 kinds.** Three fingerprint kinds are frozen by this
  document:
  - `sha256` — exact hash of one file's raw bytes (Artifact content), proposal
    [§6.1](./LIBRARY_ITEM_IDENTIFIER.md). See section 16.
  - `qtree` — structural tree fingerprint (normalized paths + sizes), proposal
    [§7](./LIBRARY_ITEM_IDENTIFIER.md). See section 14.
  - `qmanifest` — exact installation manifest (normalized paths + sizes +
    content digests), proposal [§8](./LIBRARY_ITEM_IDENTIFIER.md). See section 15.
    All other proposal fingerprint names are reserved and unspecified (section 23).
- **F11.2 — Signature fields.** Every fingerprint record carries, at minimum:
  `kind`, `algorithm`, `algorithm_version`, `profile_id` (nullable),
  `profile_version` (nullable), `inclusion_policy_version`, `digest_hex`,
  `completeness`, `cache_status`, `filtered_inventory_count`,
  `filtered_inventory_total_size`, and `computed_at` (metadata, never hashed).
- **F11.3 — v1 algorithm.** `algorithm = "sha256"` and
  `algorithm_version = 1` for all frozen kinds. The human label is
  `<kind>-v<algorithm_version>` (`qtree-v1`, `qmanifest-v1`), as in proposal
  [§7, §8, §45](./LIBRARY_ITEM_IDENTIFIER.md). The bare `sha256` name in
  proposal [§51](./LIBRARY_ITEM_IDENTIFIER.md) denotes kind `sha256`; its
  canonical label is `sha256-v1`.
- **F11.4 — Digest encoding.** Every stored and compared digest is lowercase
  hexadecimal, exactly 64 characters for SHA-256, with no `0x` prefix, no
  upper-case hex, and no truncation.
- **F11.5 — Canonical reference string.** A fingerprint is referenced as
  `qfp:<kind>:<algorithm_version>:<algorithm>:<digest_hex>` and, when the kind is
  profile-scoped, appends `:<profile_label>:<inclusion_policy_version>`.
  Examples:
  - `qfp:sha256:1:sha256:9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08`
  - `qfp:qtree:1:sha256:25ac60465b8d6c8522ff26d2cb30a28d036cfe8a1aaecf1d7cf8735b0ab985dd:generic-pc-v1:1`
  - `qfp:qmanifest:1:sha256:5cd833bd71ad61eddac7fba65a66835dd88149f9121ff9cb7f34c326e04222b0:generic-pc-v1:1`
    Proposal [§45](./LIBRARY_ITEM_IDENTIFIER.md)'s `qfp:tree:1:sha256:<digest>` is
    the same shape with an abridged kind and no profile; the spelling in this
    contract is normative.
- **F11.6 — Compatibility gates comparison.** Two fingerprints may be compared
  only when they are _compatible_ (section 21). A digest match across
  incompatible fingerprints is not an identity claim.
- **F11.7 — Fingerprints are derived, never authored.** A fingerprint is never
  accepted from an untrusted source (sidecar, catalog, filename) as a local
  observation; it is always computed from the local filtered inventory. Sidecar
  handling is T07.

### Compatibility with T00a

- No Part I decision marked **D** or **INV** is changed by Part II.
- Part I's out-of-scope list (section 1) deferred fingerprint encoding to a
  later slice; Part II is that slice. This changes slice scope, not any **D** or
  **INV**.
- **D7.2** (NFC normalization, `/` separators, case preserved, collisions
  reported and never merged) is confirmed and made byte-exact in section 13.
  F13.6 adds an explicit rule for non-UTF-8 names, which D7.2 did not address;
  it does not contradict it.
- Part I **OQ-7** ("Artifact creation trigger") is resolved by **D3.6** (T00f);
  the concrete `artifacts` columns remain a T02 obligation. F14.5/F15.6/F16.4
  narrow it: `qtree` alone can never create or resolve an Artifact, and a
  `cached`/`mixed` identity fingerprint (section 22) is not a fresh content
  proof.

### Invariants

- **INV-28** Every published v1 fingerprint has a frozen kind, algorithm,
  algorithm version, canonical digest encoding, and (when profile-scoped) a
  recorded profile and inclusion-policy version.

---

## 12. Canonical byte framing

### Decisions

- **F12.1 — Canonical input is bytes.** Every canonical preimage in Part II is a
  byte string. Text components are UTF-8; separators are raw bytes, never host
  newline or locale encodings.
- **F12.2 — NUL is the field separator.** The single byte `0x00` separates
  fields and terminates paths. A filesystem path can never contain `0x00`, so
  NUL is an unambiguous, total terminator on every supported platform. Paths may
  contain any other byte, including `0x0A` (LF); no quoting or escaping scheme
  is used.
- **F12.3 — LF is a record terminator, not a line separator.** Where a record
  terminator is used (`qtree-v1`), it is the literal byte `0x0A`. It is never
  derived from the host separator and never translated.
- **F12.4 — Integer size encoding.** File sizes are unsigned decimal ASCII with
  no sign, no units, no separators, and no leading zeros except the single value
  `0`. The representable range is `0 … 2^64 − 1` (at most 20 digits). A size is
  the logical byte length (`st_size` of the regular file); sparse holes are not
  normalized and disk-block consumption is never used.
- **F12.5 — Embedded digest encoding.** When a digest appears inside a preimage
  it is encoded as its 64-character lowercase hex string (hexadecimal ASCII),
  not as raw bytes. Fixed width (64) makes concatenation self-delimiting.
- **F12.6 — Sort order.** Records are ordered by the unsigned lexicographic
  order of the UTF-8 bytes of the normalized path, never by locale collation,
  Unicode code points, or casefold. Because normalization collisions are
  rejected (section 13), no two records share a sort key.
- **F12.7 — Preamble framing.** A preimage is `preamble 0x00 body`, where
  `preamble` is the exact ASCII kind-version token (`qtree-v1` or
  `qmanifest-v1`). Nothing else — no length field, no trailing NUL, no newline —
  is added around the preamble or the body beyond what each section specifies.
- **F12.8 — Reserved kinds reuse this framing (resolved T00f, OQ-13).** The
  reserved media kinds `qset-v1`/`qrelease-set-v1` (section 23) must reuse the
  framing primitives of this section unchanged — the kind-version preamble plus
  `0x00`, the NUL field separator, the LF record terminator, decimal-ASCII
  integer sizes, lowercase-hex embedded digests, and UTF-8 byte-order sort.
  Section 12 is never forked or redefined; T12 may only add a kind-specific
  preamble token, field order, and golden vectors on top of it. This decision
  changes no frozen preimage and therefore adds no golden vector. M1 computes
  and consumes neither reserved kind; both remain invalid until T12 freezes them
  (INV-41).

### Invariants

- **INV-29** A fingerprint digest depends only on canonical bytes defined here;
  host newline translation, locale, time, permissions, ownership, inode,
  device, and scan time never affect it.

---

## 13. Path normalization and collision handling

Root-relative path normalization underlies every fingerprint and the occurrence
key (Part I D7.1–D7.2). This section makes it byte-exact.

### Decisions

- **F13.1 — Paths are root-relative.** A path entering a fingerprint is relative
  to its root identity (Part I D6.1); an absolute path never enters a
  fingerprint.
- **F13.2 — Separators.** On POSIX hosts only `/` separates segments, and `\` is
  an ordinary filename byte. On Windows hosts both `\` and `/` separate and are
  canonicalized to `/`. A producer records the host class it normalized under so
  a POSIX filename containing `\` is never silently reinterpreted.
- **F13.3 — Segment canonicalization.** Empty segments (`//`) collapse to one
  separator; `.` segments are removed; the path has no leading `/` and no
  trailing `/`. A trailing `/` only denotes a directory, which is never a leaf.
- **F13.4 — Unicode normalization.** Paths are normalized to Unicode NFC.
  Filename case is preserved and never folded; the casefolded variant is a
  separate reserved fingerprint (section 23).
- **F13.5 — No escape.** `..` segments that would escape the root are rejected.
  A discovered path that does not resolve inside its root is never fingerprinted
  and is recorded as an error (Part I INV-16).
- **F13.6 — Valid UTF-8 required.** POSIX filenames are arbitrary bytes. A path
  that is not valid UTF-8 cannot be NFC-normalized deterministically; it is
  classified `unsupported-filename-encoding`, excluded from the filtered
  inventory, its raw bytes are recorded in the raw inventory as a warning, and
  it never enters a digest. It is reported, never silently dropped (plan
  [section 3](./LIBRARY_ITEM_IDENTIFIER_PLAN.md): "Never silently drop
  entries").
  - **Blast radius (resolved T00f, OQ-9).** The exclusion is confined to the
    affected entry, not the whole installation. An unsupported _file_ name
    excludes only that file; an unsupported _directory_ name excludes that
    directory and its subtree (its descendants cannot receive canonical paths),
    recorded as one warning covering the excluded subtree. The remaining entries
    may still publish a `qtree`/`qmanifest` as `COMPLETE_WITH_WARNINGS`
    (F17.6), and any exact match is scoped to the included files only (F15.6,
    F20.2) and must not be presented as a whole-installation content proof.
    Review surfaces the excluded count; nothing is silently merged, dropped, or
    silently promoted to a complete exact claim (INV-30/INV-31).
- **F13.7 — Canonical collision is a hard error.** A canonical collision exists
  when two distinct on-disk entries (different raw byte strings, including NFC
  versus NFD spellings of the same name) normalize to the same byte path. On
  collision the item is not fingerprinted: both raw paths and the shared
  normalized form are recorded, the inventory is `INCOMPLETE` (section 19), and
  entries are never merged, deduplicated, or dropped. This is the byte-exact
  form of Part I D7.2. Names differing only by case are not a collision (F13.4).
- **F13.8 — Reserved path.** The Questarr sidecar `**/.questarr.json` is
  excluded from every fingerprint input (section 20) and is never treated as
  game content.

### Invariants

- **INV-30** Two distinct occurrences never share a normalized path; a
  collision is reported and blocks fingerprinting, never silently merged.
- **INV-31** A fingerprint input path is root-relative, NFC, `/`-separated,
  case-preserving, valid UTF-8, and free of `.`/`..`/empty segments.

---

## 14. `qtree-v1` encoding

`qtree-v1` captures structure and size only and never reads file content
(proposal [§7](./LIBRARY_ITEM_IDENTIFIER.md)).

### Encoding (normative)

```text
canonical_entries =
    for each included file, in sort order (F12.6):
        utf8(normalized_relative_path) 0x00 decimal_ascii(size) 0x0A

qtree_v1_preimage = "qtree-v1" 0x00 canonical_entries
qtree_v1_digest   = SHA-256(qtree_v1_preimage)     # 32 bytes -> 64-char lowercase hex
```

### Decisions

- **F14.1 — Record framing.** The proposal's illustrative `path\0size` is
  refined with a mandatory `0x0A` record terminator. The terminator makes the
  size field self-delimiting: without it, a size followed by a path beginning
  with a digit would be ambiguous. The grammar above is injective, so distinct
  filtered inventories always produce distinct preimages.
- **F14.2 — Stat only.** `qtree-v1` requires only path and size. It must not
  read content, follow symlinks, or use timestamps, inode, device, UID/GID,
  permissions, creation time, or scan time (proposal
  [§7](./LIBRARY_ITEM_IDENTIFIER.md)).
- **F14.3 — Empty inventory.** With no included files, `canonical_entries` is
  empty and
  `qtree_v1_digest = SHA-256("qtree-v1\0") = 20131f0c717331825d2a1d7b556f6692a6e179c6b7cd9a444fc6f8fdd2cc2e4d`.
  This constant is recorded only for an inventory observed as complete and empty
  (section 19). A truncated or failed scan must not record it; it records no
  qtree and `completeness = INCOMPLETE`/`FAILED`. Because the empty digest is not
  discriminating, it must never by itself be used to associate an Artifact.
- **F14.4 — Derivable metadata is not hashed.** File count and total size are
  stored alongside the fingerprint (proposal
  [§9](./LIBRARY_ITEM_IDENTIFIER.md)) but are not part of the preimage; they are
  derivable from the manifest, and hashing them would add redundant framing to
  keep in sync.
- **F14.5 — Structural equality is not content equality.** A `qtree-v1` match
  means "same included paths and sizes", never "same bytes" (proposal
  [§7–8](./LIBRARY_ITEM_IDENTIFIER.md)), and may never create or resolve an
  Artifact on its own.

### Invariants

- **INV-32** A `qtree-v1` digest depends only on the fixed preamble, the
  normalized included paths, and their sizes.

---

## 15. `qmanifest-v1` encoding

`qmanifest-v1` is the exact-content installation fingerprint (proposal
[§8](./LIBRARY_ITEM_IDENTIFIER.md)).

### Encoding (normative)

```text
fileDigest(f)      = SHA-256(raw bytes of f)            # 64-char lowercase hex
leafInput(f)       = "F" 0x00 utf8(normalized_relative_path)
                          0x00 decimal_ascii(size)
                          0x00 fileDigest_hex
leafDigest(f)      = SHA-256(leafInput(f))              # 64-char lowercase hex

manifest_body      = leafDigest_hex(f1) || leafDigest_hex(f2) || ...   (sort order F12.6)
manifest_preimage  = "qmanifest-v1" 0x00 manifest_body
qmanifest_v1_digest = SHA-256(manifest_preimage)
```

### Decisions

- **F15.1 — Leaf framing.** The leaf preimage is unambiguous: `F`, NUL, path
  (terminated by NUL), size (terminated by NUL), then exactly 64 hex characters.
  A path cannot contain NUL and a decimal size cannot contain NUL, so the parse
  is unique.
- **F15.2 — Digests are embedded as hex, not raw bytes.** `fileDigest` and the
  concatenated `leafDigest` values are 64-character lowercase hex (F12.5). This
  explicitly resolves the plan's "binary versus hexadecimal digests" gap and
  matches the stored `sha256:<hex>` form in proposal
  [§9](./LIBRARY_ITEM_IDENTIFIER.md).
- **F15.3 — Concatenation framing.** `leafDigest_hex` is fixed width (64), so
  the manifest body is self-delimiting and needs no separator between leaf
  digests. Distinct ordered leaf-digest sequences always produce distinct
  bodies.
- **F15.4 — Size is domain separation.** Size is included even though it is
  implied by the file digest, because it domain-separates leaf entries and keeps
  the leaf preimage aligned with the proposal.
- **F15.5 — Empty inventory.** With no included files,
  `qmanifest_v1_digest = SHA-256("qmanifest-v1\0") = 4ebb24df7a25d3a4a6788180b96430918b116bdd3b8bfac741106a01f8ac5f6b`.
  The completeness guard of F14.3 applies equally.
- **F15.6 — Exact equality is scoped to included files.** A `qmanifest-v1`
  match proves exact equality only of the identity-filtered included files under
  a compatible profile (section 21). It does not prove the bytes are an
  officially published build (proposal [§8](./LIBRARY_ITEM_IDENTIFIER.md)) and
  it does not cover excluded entries (section 20).
- **F15.7 — Profile dependence.** The included set is profile-defined, so a
  `qmanifest-v1` value is meaningful only together with its profile and
  inclusion-policy versions (section 21).

### Invariants

- **INV-33** A `qmanifest-v1` digest depends only on the fixed preamble, the
  normalized included paths, their sizes, and their content digests, framed as
  above.

---

## 16. Artifact content SHA-256

Proposal [§6.1](./LIBRARY_ITEM_IDENTIFIER.md) makes `SHA256(file bytes)` the
primary exact fingerprint for a single-file Artifact; plan
[section 3](./LIBRARY_ITEM_IDENTIFIER_PLAN.md) requires it to stream with
bounded buffers.

### Decisions

- **F16.1 — Definition.** `sha256(f) = SHA-256` of the exact byte sequence of
  regular file `f`, encoded as 64-character lowercase hex. No transformation is
  applied: no BOM stripping, text mode, newline conversion, Unicode
  normalization, decompression, sparse-hole filling, or padding.
- **F16.2 — Streaming.** Bytes are read once, in order, through a bounded
  buffer. The buffer size is an implementation detail and must not affect the
  digest; the whole file is never required in memory.
- **F16.3 — Reference.** The reference is
  `qfp:sha256:1:sha256:<digest_hex>`. Kind `sha256` has no profile and no
  inclusion policy; it is comparable on `kind`/`algorithm`/`algorithm_version`
  alone (section 21).
- **F16.4 — Relationship to installation fingerprints.** `sha256` is the
  Artifact-level content identity for one file. For a directory, exact content
  identity is `qmanifest-v1`; `qtree-v1` is never an exact content identity. A
  single-file installation may have both a `sha256` and a one-entry
  `qmanifest-v1`; they are different values and neither substitutes for the
  other (the one-entry leaf preimage frames `"F"\0path\0size\0hex`, which is not
  the raw file hash).
  - **Single-file Artifact backing (resolved T00f, OQ-11).** The `artifacts`
    row for a single-file installation is backed by `sha256` (F16.3), because it
    is the profile-independent content identity. A one-entry `qmanifest-v1` may
    be computed and stored for installation-level exact equality, but it does
    not define Artifact identity and never substitutes for the `sha256`. For a
    directory Installation the Artifact identity is `qmanifest-v1` under a
    compatible profile (F15.6/F21.2). The Artifact-creation trigger itself is
    D3.6.
- **F16.5 — Other digest algorithms are reserved.** SHA-1, MD5, and CRC32 are
  needed only to match external DAT databases (proposal
  [§6.1](./LIBRARY_ITEM_IDENTIFIER.md), plan T11) and are not v1 identity kinds.

### Invariants

- **INV-34** An artifact `sha256` is computed over raw file bytes with no
  transformation and is independent of filename, path, profile, and cache
  state.

---

## 17. Symlinks, special files, and unsupported names

Plan [section 3](./LIBRARY_ITEM_IDENTIFIER_PLAN.md): "M1 skips symlinks with a
recorded warning; do not fingerprint links as ordinary files."

### Decisions

- **F17.1 — Symlinks are never followed.** A symlink is never dereferenced,
  never included as an ordinary file, and never traversed as a directory,
  regardless of whether its target is inside or outside the root. The result
  never depends on link targets.
- **F17.2 — Symlinks are recorded, not hashed.** Each symlink is recorded in the
  raw inventory with its normalized path, its `readlink` target (best effort),
  class `SYMLINK`, decision `excluded`, and reason `symlink`. It never enters
  the filtered inventory or any digest.
  - **Target storage (resolved T00f, OQ-12).** The raw target string is never
    persisted verbatim in any shared or non-owner-readable projection. Only a
    non-revealing classification is stored: whether the target is relative or
    absolute and whether it resolves inside its root. When it resolves inside
    the root, the root-relative normalized target may be stored; otherwise a
    redacted marker (for example `outside-root`) is stored with no raw path. A
    target that could disclose an absolute path outside the root (proposal
    [§49](./LIBRARY_ITEM_IDENTIFIER.md)) is therefore never exposed, consistent
    with INV-15 and the read-projection rules (resolved T00f4, OQ-33). Symlink
    handling remains excluded from every digest (INV-35).
- **F17.3 — Hard links are ordinary files.** A hard link is a regular file with
  `st_nlink > 1`; each path is fingerprinted normally, and `device`/`inode`
  identity is never merged across paths.
- **F17.4 — Non-regular files are excluded.** FIFOs, sockets, block devices,
  character devices, and any other non-regular, non-directory entry are recorded
  as `special-file` exclusions with a warning and never hashed.
- **F17.5 — Unsupported names are recorded.** `unsupported-filename-encoding`
  paths (F13.6) are recorded as warnings and excluded, never hashed.
- **F17.6 — Policy skips are warnings, not errors.** Symlink/special/unsupported
  skips are expected policy outcomes; on their own they do not make an inventory
  incomplete. The item becomes `COMPLETE_WITH_WARNINGS` (section 19) and the
  warning set is stored. Warnings do not change the digest, because excluded
  entries are not in the filtered inventory.
- **F17.7 — No loops to detect.** Because symlinked directories are never
  descended, symlink cycles are structurally impossible.

### Invariants

- **INV-35** No symlink, hard-link inode identity, or special file contributes
  to any fingerprint.

---

## 18. Unreadable entries and observation stability

Plan [section 3](./LIBRARY_ITEM_IDENTIFIER_PLAN.md): "Incomplete/unreadable/
changing inventories do not publish a complete exact manifest. Record
completeness and scan errors."

### Decisions

- **F18.1 — Errors are classified, not collapsed.** Every failure to account
  for an entry is recorded with a stable class: `permission-denied`,
  `io-error`, `not-found` (disappeared between enumeration and access),
  `unsupported`, `limit-exceeded`, `collision`, `unstable-file`.
- **F18.2 — Unreadable directory ⇒ structural incompleteness.** If a directory
  cannot be opened, its children are unknown; both `qtree` and `qmanifest` are
  `INCOMPLETE` and neither is published as comparable.
- **F18.3 — File stat fails ⇒ no entry.** Without `st_size` the file cannot
  appear in either fingerprint; the inventory is `INCOMPLETE`.
- **F18.4 — File read fails ⇒ qtree may stand, qmanifest may not.** `qtree`
  needs only `st_size`, so a stat-successful but unreadable regular file can
  still yield a `COMPLETE` qtree while `qmanifest` is `INCOMPLETE` and withheld.
  A qtree must never be presented as content verification.
- **F18.5 — Changing files are unstable.** During a `qmanifest` read, if the
  byte count read differs from the observed size, or a post-read `stat` shows a
  changed size or `mtime`, the entry is `unstable-file`: the read is retried a
  bounded number of times and then abandoned. A partial or truncated digest is
  never emitted.
- **F18.6 — No retry storms.** Retry counts and per-item time bounds are
  configured limits; exhausting them yields `INCOMPLETE`, never a silently
  shorter manifest.

### Invariants

- **INV-36** No fingerprint is published from an unstable or partially read
  file.

---

## 19. Inventory completeness

Completeness is a property of an inventory/fingerprint record and is distinct
from job state (Part IV).

### Decisions

- **F19.1 — Enum.**
  - `COMPLETE` — every included entry was fully and stably accounted for, with
    no warnings, no errors, and no limits hit.
  - `COMPLETE_WITH_WARNINGS` — every included entry was accounted for and only
    expected policy skips occurred (symlinks, special files, unsupported names).
  - `INCOMPLETE` — at least one included entry could not be accounted for
    (unreadable directory or file, collision, limit/depth/file-count bound, or
    unstable after retries).
  - `FAILED` — no usable inventory was produced.
- **F19.2 — Only complete inventories publish comparable fingerprints.**
  `qtree`/`qmanifest` records with `COMPLETE` or `COMPLETE_WITH_WARNINGS` may be
  stored and compared as exact equality. `INCOMPLETE`/`FAILED` records may retain
  a partial raw inventory as evidence but must never be compared as an exact
  match and must never back an Artifact association.
- **F19.3 — Limits are visible.** Depth, entry-count, and byte/time budgets that
  trigger an early stop produce `INCOMPLETE` with `limit-exceeded`; a bounded
  scan is never presented as a complete one.
- **F19.4 — Counts and warnings are recorded.** The record stores
  `filtered_inventory_count`, `filtered_inventory_total_size`, the warning list,
  and the error list, so a digest can be interpreted without re-walking the
  tree.
- **F19.5 — Warnings do not change equality.** Two scans over identical included
  files yield the same digest even if one also skipped a symlink; equality is
  defined over the filtered inventory only (section 20).
- **F19.6 — Persistence and retention (resolved T00f, OQ-10).** Completeness,
  the warning list, the error list, and the filtered counts are recorded per
  _observation revision_ (M30.9) and are never overwritten in place by a later
  scan. The raw inventory is retained separately from the filtered inventory
  (F20.1) as revision-scoped evidence and is serialized/compressed in the same
  manner as the manifest (proposal [§9](./LIBRARY_ITEM_IDENTIFIER.md)), rather
  than as one permanent row per scanned file (D7.3, INV-20). The exact
  table/column names, blob format, and cleanup policy are T02 storage shape
  (OQ-10 shape, M30.9 token resolved by T00f2, OQ-24 retention); the safe M1
  rule frozen here is revision-scoped retention with counts and warnings/errors
  carried on every published fingerprint (F11.2, F19.4) and no per-file rows.

### Invariants

- **INV-37** An `INCOMPLETE`/`FAILED` inventory never yields a comparable exact
  fingerprint and never associates an Artifact.

---

## 20. Raw inventory versus identity-filtered inventory

Plan [section 3](./LIBRARY_ITEM_IDENTIFIER_PLAN.md): "Store raw observed
inventory separately from identity-profile inclusion." Proposal
[§14](./LIBRARY_ITEM_IDENTIFIER.md): the raw manifest may record excluded files.

### Decisions

- **F20.1 — Two inventories, always distinct.**
  - The **raw inventory** records every enumerated entry with its normalized
    path, kind, class, and per-profile decision (`included`/`excluded` plus a
    reason), including entries excluded from identity.
  - The **filtered inventory** is exactly the entries whose decision is
    `included`.
- **F20.2 — Only the filtered inventory is hashed.** `qtree`/`qmanifest` are
  computed over the filtered inventory in sort order (F12.6); raw entries never
  enter a digest.
- **F20.3 — Raw inventory is evidence.** It is stored for explainability, change
  detection, and review; it is not identity and is not compared for equality
  across installations except as supporting evidence.
- **F20.4 — Sidecars are excluded from every fingerprint.** `.questarr.json`
  (proposal [§34](./LIBRARY_ITEM_IDENTIFIER.md)) is excluded from both the
  filtered set and every digest, in all profiles, because it is Questarr-
  generated metadata and would be self-referential. It may appear in the raw
  inventory as an excluded entry.
- **F20.5 — Filtering is profile-driven and versioned.** Which entries are
  included is the profile's decision (section 21); the volatile-path heuristics
  (`save/`, `logs/`, `*.log`, …) from proposal
  [§14](./LIBRARY_ITEM_IDENTIFIER.md) are profile content, not algorithm
  content, and must be versioned with the profile.

### Invariants

- **INV-38** An excluded or raw-only entry never contributes to any fingerprint
  digest.

---

## 21. Fingerprint profiles and compatibility

### Decisions

- **F21.1 — Profile identity.** A profile-scoped fingerprint carries
  `profile_id` (lowercase `[a-z0-9][a-z0-9-]*`, no trailing hyphen),
  `profile_version` (positive integer), and `inclusion_policy_version`
  (positive integer). The canonical label is
  `${profile_id}-v${profile_version}` (for example `generic-pc-v1`, proposal
  [§14](./LIBRARY_ITEM_IDENTIFIER.md)). The initial v1 profile ids are
  `generic-pc`, `steam`, `gog`, `playstation-disc`, `switch`, `rom`, and `mame`;
  their exact inclusion tables and classes (`CORE`, `CONTENT`, `EXECUTABLE`,
  `METADATA`, `DLC`, `MOD`, `USER_DATA`, `CACHE`, `UNKNOWN`) are T03, but their
  compatibility contract is frozen here.
- **F21.2 — Compatibility predicate.** Fingerprints `a` and `b` are compatible
  iff `a.kind == b.kind`, `a.algorithm == b.algorithm`,
  `a.algorithm_version == b.algorithm_version`, and, for profile-scoped kinds,
  `a.profile_id == b.profile_id`, `a.profile_version == b.profile_version`, and
  `a.inclusion_policy_version == b.inclusion_policy_version`. Kind `sha256` is
  profile-independent and compares on the first three fields only.
- **F21.3 — No cross-profile equality.** A digest match between incompatible
  fingerprints is not an identity claim and must never be surfaced as one
  (plan [section 3](./LIBRARY_ITEM_IDENTIFIER_PLAN.md): "Compare only
  compatible scopes/profiles").
- **F21.4 — Version bump rule.** Changing inclusion, exclusion, or
  classification rules for an existing profile requires a new
  `profile_version` (or a new `inclusion_policy_version` for algorithm-level
  policy changes) and new golden vectors. A published profile version is
  immutable.
- **F21.5 — Profile is recorded, not inferred.** The profile used for a
  fingerprint is stored with it; changing the active profile later does not
  retroactively relabel stored fingerprints.

### Invariants

- **INV-39** Two fingerprints are compared for equality only when compatible;
  profile and inclusion-policy versions are part of identity.

---

## 22. Cache trust

Proposal [§10](./LIBRARY_ITEM_IDENTIFIER.md); plan
[section 3](./LIBRARY_ITEM_IDENTIFIER_PLAN.md): "Full verification bypasses
cache. Cached results are not fresh integrity proofs; same-size/same-mtime
changes can evade cache hints."

### Decisions

- **F22.1 — Cache contents.** The cache stores per-file content digests only. It
  never stores `qtree`/`qmanifest` values and never stores identity.
- **F22.2 — Cache key.** A content-digest cache entry is keyed by
  `(root identity, normalized relative path, size, mtime_ns)`. `device` and
  `inode` may be stored as additional hints but are never sufficient for a hit
  and never part of any fingerprint.
- **F22.3 — Modes.** `identity` mode may reuse cache entries. `verify` mode
  bypasses the cache and rehashes every included file.
- **F22.4 — Status.** A fingerprint records
  `cache_status ∈ {fresh, cached, mixed, none}`: `fresh` = every content digest
  computed in this run, `cached` = all reused, `mixed` = both, `none` = no
  content digests needed (`qtree`).
- **F22.5 — Trust level.** A `cached`/`mixed` fingerprint is a valid identity
  fingerprint but is not a fresh content-integrity proof. Only `fresh` (or a
  `verify`-mode run) supports a "content verified now" claim.
- **F22.6 — Invalidation.** A cache entry is invalidated by any change to size
  or `mtime_ns`. If `mtime` is unavailable or its resolution cannot distinguish
  changes (coarse filesystem granularity), the entry is treated as a miss and
  the file is rehashed. Same-size/same-mtime evasion is a documented limitation,
  not a correctness guarantee.
  - **Coarse-resolution detection (resolved T00f, OQ-8).** The scanner records
    a per-root `mtime_resolution ∈ {fine, coarse, unavailable}` using passive
    observation only. It is `coarse` when the platform/filesystem does not
    return sub-second timestamps (`mtime_ns` unsupported) or when every observed
    `mtime_ns` in the root is an exact whole-second multiple on a platform known
    to store second-granularity times (for example FAT/exFAT and some network
    shares); it is `unavailable` when no usable `mtime` is reported.
  - For a root marked `coarse` or `unavailable`, every identity-mode (F22.3)
    cache entry is treated as a miss and the file is rehashed, because a
    same-second modification is indistinguishable. `verify` mode rehashes
    regardless. The resolution is recorded with the fingerprint's cache status
    (F22.4/F22.5), so a `cached`/`mixed` result from a coarse root is never
    promoted to a fresh content-integrity proof. This decision changes no
    fingerprint bytes.
- **F22.7 — Cache is not hashed.** No cache key, `mtime`, `device`, or `inode`
  ever enters a fingerprint preimage (proposal
  [§7, §10](./LIBRARY_ITEM_IDENTIFIER.md)).

### Invariants

- **INV-40** Cache state never contributes to a digest; a `cached`/`mixed`
  fingerprint is never presented as a fresh content-integrity proof.

---

## 23. Reserved fingerprint kinds (not specified here)

The following names appear in the proposal but are deliberately **not**
specified in this document. No encoding, preamble, framing, or golden vector is
frozen for them, and no downstream task may treat them as valid v1 fingerprints
until their owning task freezes them with the same rigor.

- **`qset-v1`** — ordered member content hashes for multi-file disc/archive sets
  (proposal [§31](./LIBRARY_ITEM_IDENTIFIER.md)). Reserved for the media task
  (plan T12, "Ordered media sets and bounded archive payloads"). Filenames must
  not participate; logical role/order and member content hashes must be
  explicitly framed when it is specified.
- **`qrelease-set-v1`** — aggregate over ordered artifact fingerprints (proposal
  [§32](./LIBRARY_ITEM_IDENTIFIER.md)). Reserved for T12.
- **`qstructure-minhash-v1`** — structural similarity sketch (proposal
  [§11–13](./LIBRARY_ITEM_IDENTIFIER.md),
  [§51](./LIBRARY_ITEM_IDENTIFIER.md)). Reserved for the structural-similarity
  task (plan T13).
- **`qtree-casefold-v1`** — casefolded structural comparison (proposal
  [§7](./LIBRARY_ITEM_IDENTIFIER.md)). Reserved; it must never replace the
  case-preserving `qtree-v1`.

The frozen kinds `sha256` (section 16), `qtree` (section 14), and `qmanifest`
(section 15) must not be conflated with these reserved kinds. In particular, a
multi-disc set is not a `qmanifest-v1`, and a converted container's own hash is
not a logical-media `qset-v1` (proposal
[§33](./LIBRARY_ITEM_IDENTIFIER.md)).

### Invariants

- **INV-41** No reserved fingerprint name is valid until its owning task
  publishes byte-exact encoding and golden vectors; reserved names never alias a
  frozen kind.

---

## 24. Golden vectors

These vectors are part of the frozen contract. They were computed **only from
the byte definitions above**, with standard tools (macOS `shasum -a 256` and
`openssl dgst -sha256`, cross-checked with Python `hashlib` and Node `crypto`),
not from any Questarr implementation. An implementation whose `qtree-v1`,
`qmanifest-v1`, or artifact `sha256` values differ from these is non-conformant.

Reproduce any digest from its preimage hex:

```sh
printf '%s' '<preimage_hex>' | xxd -r -p | shasum -a 256
# portable alternative:
python3 -c 'import hashlib,sys; print(hashlib.sha256(bytes.fromhex(sys.argv[1])).hexdigest())' '<preimage_hex>'
```

### GV-1 — `qtree-v1`, two files

Filtered inventory (already in sort order):

| normalized path | size |
| --------------- | ---- |
| `bin/game.exe`  | 4    |
| `data/main.pak` | 5    |

Preimage bytes:

```text
"qtree-v1" 0x00 "bin/game.exe" 0x00 "4" 0x0A "data/main.pak" 0x00 "5" 0x0A
```

```text
preimage_hex = 71747265652d76310062696e2f67616d652e65786500340a646174612f6d61696e2e70616b00350a
qtree-v1     = 25ac60465b8d6c8522ff26d2cb30a28d036cfe8a1aaecf1d7cf8735b0ab985dd
```

### GV-2 — `qmanifest-v1`, same two files

Contents: `bin/game.exe` = `test`, `data/main.pak` = `hello`.

```text
fileDigest(bin/game.exe) = 9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08
fileDigest(data/main.pak) = 2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824

leaf1 preimage_hex = 460062696e2f67616d652e65786500340039663836643038313838346337643635396132666561613063353561643031356133626634663162326230623832326364313564366331356230663030613038
leaf1              = f4c39686aeb6aa62c7b3aa15c77acc4bdc61e3f5094e4990232e6922675204c5

leaf2 preimage_hex = 4600646174612f6d61696e2e70616b00350032636632346462613566623061333065323665383362326163356239653239653162313631653563316661373432356537333034333336323933386239383234
leaf2              = f40daa0766a31bc0147418ec1b4663ee8fb7eed0f64fd7a54b9c4f27a6871fb7

manifest preimage_hex = 716d616e69666573742d7631006634633339363836616562366161363263376233616131356337376163633462646336316533663530393465343939303233326536393232363735323034633566343064616130373636613331626330313437343138656331623436363365653866623765656430663634666437613534623963346632376136383731666237
qmanifest-v1           = 5cd833bd71ad61eddac7fba65a66835dd88149f9121ff9cb7f34c326e04222b0
```

The `manifest preimage_hex` is `"qmanifest-v1" 0x00` followed by the ASCII hex
spelling of `leaf1` and then of `leaf2` (each 64 characters).

### GV-3 — empty inventory

```text
qtree preimage_hex     = 71747265652d763100
qtree-v1 (empty)       = 20131f0c717331825d2a1d7b556f6692a6e179c6b7cd9a444fc6f8fdd2cc2e4d

qmanifest preimage_hex = 716d616e69666573742d763100
qmanifest-v1 (empty)   = 4ebb24df7a25d3a4a6788180b96430918b116bdd3b8bfac741106a01f8ac5f6b
```

Both constants are recorded only for an inventory observed as complete and
empty (F14.3, F19.2).

### GV-4 — NFC normalization and UTF-8 byte-order sort

On-disk path `cafe` + U+0301 (combining acute) + `/x` has raw bytes
`63616665cc812f78`; NFC-normalized it is `café/x` = `636166c3a92f78`. Filtered
inventory after normalization, already in sort order (`Z` = `0x5A` sorts before
`c` = `0x63` in unsigned byte order, unlike locale collation):

| normalized path | size |
| --------------- | ---- |
| `Zeta.txt`      | 1    |
| `café/x`        | 2    |

```text
preimage_hex = 71747265652d7631005a6574612e74787400310a636166c3a92f7800320a
qtree-v1     = 1e6c1910c60120b1ceebe7d1320b9f5ad0e61b695f1d60cd273ce712395fbee0
```

### GV-5 — artifact `sha256` (single file)

Content `test` has bytes `74657374`.

```text
sha256("test") = 9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08
```

Reproduce with `printf 'test' | shasum -a 256`.

### GV-6 — canonical collision (no fingerprint published)

Entries `café/x` (NFC bytes `636166c3a92f78`) and `cafe` + U+0301 + `/x` (NFD
bytes `63616665cc812f78`) both normalize to `636166c3a92f78`. Per F13.7 this is
a canonical collision: no `qtree-v1`/`qmanifest-v1` is published for the item,
the inventory is `INCOMPLETE`, and neither entry is merged or dropped. This
vector intentionally has no digest.

### Verification record

The vectors were reproduced with three independent standard-tool paths and all
three agreed on every digest:

```text
shasum -a 256 / openssl dgst -sha256   (BSD/macOS and OpenSSL)
python3 hashlib.sha256                  (CPython 3)
node crypto.createHash('sha256')        (Node.js)
```

---

# Part III — T00c: candidate generation and match-decision contract

This part implements the **matching and decision** bullets of plan
[section 3 "Jobs and matching"](./LIBRARY_ITEM_IDENTIFIER_PLAN.md) and freezes
the proposal's matching behaviour ([§3–5](./LIBRARY_ITEM_IDENTIFIER.md),
[§34–39](./LIBRARY_ITEM_IDENTIFIER.md), [§47–48](./LIBRARY_ITEM_IDENTIFIER.md))
into concrete rules. It adds **no runtime code**, changes **no fingerprint
encoding**, and specifies **no** job lifecycle, HTTP API/DTO/error envelope, UI,
or catalog ingestion.

Bounded scope:

- **In scope:** candidate generation versus final resolution; evidence classes
  and provenance; identifier and exact-hash mapping; untrusted sidecar
  _decisions_; strong conflicts; manual-association persistence;
  identity/content/accessibility/job state separation; revision/snapshot
  protection for decisions; explainable result examples; a compact decision
  table; explicit unresolved issues (all Part III questions OQ-14–OQ-20 were
  closed by T00f2; see section 32).
- **Out of scope (retained by other owners):** durable job states and
  scheduling/budgets/cancellation (Part IV of this contract; M30.1 records the
  separation only), HTTP routes and DTO shapes (T06), UI (T09), external catalog
  _ingestion_ (T11), the sidecar _parser_ implementation and portable-identifier
  validation mechanics (T07; OQ-5 resolved by deferral in T00f), numeric fuzzy
  thresholds (T13), and all
  reserved fingerprint kinds (section 23).

### Compatibility with prior parts

- No Part I decision marked **D**/**INV** and no Part II decision marked
  **F**/**INV** is changed by Part III.
- Part III consumes fingerprints only through the compatibility gate
  (F21.2/F21.3, INV-39) and the completeness gate (F19.2, INV-37).
- F11.7 ("a fingerprint is never accepted from an untrusted source") is extended,
  not narrowed: sidecar fingerprint values are _claims_, never local
  observations (section 28).
- Part I D3.3/D4.3 (nullable associations, no placeholder Games) and INV-14
  (association requires ownership) continue to bind every rule here.

---

## 25. Candidate generation versus final resolution

Proposal [§36](./LIBRARY_ITEM_IDENTIFIER.md): candidate generation favors recall;
final resolution favors precision. A wrong automatic merge is substantially worse
than asking the user to review an ambiguous item.

### Decisions

- **M25.1 — Two separate operations.** Candidate generation and final resolution
  are distinct phases with distinct outputs. A _candidate_ is a hypothesis and
  carries no state; only resolution may produce a match state (section 30).
- **M25.2 — Candidate inputs are evidence.** Candidate generation may use native
  identifiers, exact and structural fingerprints, title tokens, platform/region/
  edition/year, and external catalog lookups, but every input is recorded as
  evidence with provenance (section 26). Candidate generation may be broad and
  fuzzy; it must never write an association.
- **M25.3 — Rank is not truth.** Candidates may be ordered and deduplicated by an
  explicit, recorded key, but a candidate is never "the answer" by rank, score, or
  order alone.
- **M25.4 — Resolution is per candidate.** For each candidate, resolution gathers
  supporting and contradicting evidence, evaluates strong conflicts _first_
  (section 29), then applies the decision table (section 31). Resolution favors
  precision.
- **M25.5 — No forced pick.** Resolution may yield no candidate (UNKNOWN) or
  several plausible candidates (AMBIGUOUS) without selecting one.
- **M25.6 — Names and fuzzy similarity alone never auto-link.** Title/filename
  tokens and any similarity measure (FUZZY/WEAK) may generate and rank candidates
  but can never by themselves create an association (proposal
  [§37 Rule G](./LIBRARY_ITEM_IDENTIFIER.md); plan section 3). **OQ-17 is
  resolved by deferral (T00f2):** M1 freezes no numeric fuzzy threshold and
  enables no Rule F-style automatic association. A future calibrated threshold
  (T13, M3) may only _corroborate_ a candidate; it still may not substitute for
  the trusted mapping M25.7 requires.
- **M25.7 — Automatic association needs a trusted mapping.** A _trusted mapping_
  is one of:
  - a resolved Questarr import provenance reference to a Game (D6.7);
  - a schema-valid sidecar reference that resolves to a local Game **and** passes
    the consistency checks in section 28;
  - a native/provider identifier resolved through its namespace+scope to exactly
    one known Release/Build (D5.x);
  - a known-artifact/manifest exact match whose record maps to a
    Release/Build/Game with recorded source/version provenance.

  Automatic association requires at least one **AUTHORITATIVE**, **EXACT**, or
  **STRONG** evidence item that carries such a trusted mapping. SUPPORTING,
  FUZZY, and WEAK evidence may only corroborate a PROBABLE state.

- **M25.8 — Candidate sets are recomputed per revision in M1 (resolved T00f2,
  OQ-16).** Candidate generation is deterministic for a fixed observation
  revision: it derives candidates from the recorded evidence and fingerprints of
  that revision. M1 recomputes candidates on demand for the current revision and
  does **not** persist a durable candidate row set (P53.3). A later T05/T06
  optimization may cache a candidate set, but a cached set is revision-scoped, is
  invalidated by any newer observation revision (M30.10), and never itself
  establishes identity (INV-42/INV-44). PROBABLE and AMBIGUOUS items retain the
  evidence and the observation revision so their candidate list is deterministically
  reproducible for review.

### Invariants

- **INV-42** Candidate generation never writes `library_items.game_id` /
  `.release_id` / `.build_id` or an `artifacts` row.
- **INV-43** No automatic association exists without at least one
  AUTHORITATIVE/EXACT/STRONG evidence item carrying a trusted mapping.
- **INV-44** Candidate rank/score order is never sufficient to establish identity.

---

## 26. Evidence classes, provenance, and strength

Proposal [§35](./LIBRARY_ITEM_IDENTIFIER.md) defines evidence classes rather than
a single magic score. Proposal [§47](./LIBRARY_ITEM_IDENTIFIER.md) requires every
resolved item to retain _why_ it matched.

### Decisions

- **M26.1 — Frozen evidence classes (strongest first).** `AUTHORITATIVE`,
  `EXACT`, `STRONG`, `SUPPORTING`, `FUZZY`, `WEAK`. Every evidence item records
  its class and a mandatory provenance (M26.3).
- **M26.2 — Class ↔ provenance.**
  - `AUTHORITATIVE` — Questarr-managed import provenance (D6.7), or a validated
    sidecar reference that passes section 28. Always carries a trusted mapping.
  - `EXACT` — a locally computed frozen fingerprint equality under F21.2, or an
    external DAT checksum match whose catalog source/version/checksum is recorded
    (T11). Carries a trusted mapping only when the matched record maps to a
    Release/Build/Game.
  - `STRONG` — a native/provider identifier (Steam AppID, console title ID, disc
    serial, GOG product ID, provider build ID, …) resolved through its
    namespace+scope to exactly one entity (D5.x). Build identity additionally
    requires its provider release context (D5.6, INV-11).
  - `SUPPORTING` — platform, region, edition, version, executable metadata, disc
    number, component/DLC classification.
  - `FUZZY` — structural similarity (qtree structural comparison, MinHash,
    path-set similarity, partial content overlap) or any similarity score.
  - `WEAK` — folder/filename, normalized title, release year, generic metadata.
- **M26.3 — Provenance is mandatory.** Each evidence item records: class; kind/
  type; the value or fingerprint reference; `source` (importer, sidecar path +
  hash, filesystem observation, catalog source/version); the observation revision
  it came from (M30.9); and whether it is local or external. Evidence without
  provenance is not usable for resolution.
- **M26.4 — Evidence is immutable per observation.** Correcting evidence creates a
  new evidence item/revision; it never rewrites history.
- **M26.5 — Contradiction is first-class.** Evidence of any class is retained even
  when it is outweighed; a stronger item never deletes a weaker one.
- **M26.6 — Class is not a probability.** A class describes kind/provenance, not a
  numeric confidence. Any score a producer derives must be explainable as the
  evidence list; no bare numeric confidence stands alone (proposal §47).
- **M26.7 — External evidence is not redistributed.** Matching against external
  catalogs is allowed; automatic uploads and redistribution are not (plan
  section 3). Only the recorded source/version/checksum and the local comparison
  result are stored; the catalog payload is not the observation.

### Invariants

- **INV-45** Every evidence item has a class in the frozen set and a recorded
  provenance.
- **INV-46** No resolution consumes evidence lacking provenance or derived from an
  incompatible fingerprint (F21.3) or an incomplete inventory (F19.2).
- **INV-47** Contradictory evidence is preserved, never overwritten by later
  evidence.

---

## 27. Identifier and exact-hash mapping rules

Proposal [§37](./LIBRARY_ITEM_IDENTIFIER.md) Rules C–E and plan section 3's
"Exact content matches without a trusted Game mapping identify content only".

### Decisions

- **M27.1 — Exact content maps to content, not to a Game, by itself.** A `sha256`
  (F16.x) or `qmanifest-v1` (F15.x) match is EXACT **content** evidence. If the
  matched record has no trusted Game/Release/Build mapping, the result identifies
  _content only_: Questarr must not create or select a `Game`, `Release`, or
  `Build`, and must not invent a placeholder (D4.3). The match may persist as
  unresolved content evidence only.
- **M27.2 — Structural match maps to structure, not content.** `qtree-v1`
  equality (F14.5) is never exact content identity and never alone creates or
  resolves an Artifact.
- **M27.3 — Identifier mapping is namespace + scope exact.** A native identifier
  resolves only through `(namespace, scope, value)`; no value-only lookup
  (INV-10) and no numeric-proximity or fuzzy identifier matching.
- **M27.4 — Native ID → Release.** An identifier whose scope resolves to exactly
  one known Release selects that Release; the Build stays unknown unless
  separately identified (Rule D).
- **M27.5 — Native ID + revision → Build.** A serial/title ID together with a
  provider build/revision identifier resolves a specific Build only within its
  provider release context (Rule E; D5.6; INV-11).
- **M27.6 — Exact hash → Artifact/Release.** A known artifact `sha256`, or a
  known canonical `qmanifest-v1` matched under a compatible profile, selects the
  mapped Artifact and, when the mapping records them, its Release/Build (Rule C).
- **M27.7 — Hash disagreement alone does not prove mods.** A local fingerprint
  differing from a known baseline value is **content-difference evidence only**.
  `IDENTIFIED_MODIFIED` requires all three of:
  1. identity already established by an AUTHORITATIVE/EXACT/STRONG trusted
     mapping (M25.7);
  2. a _compatible_ known baseline for the same Release/Build/profile
     (F21.2, INV-39);
  3. interpretable difference evidence (for example additional files, or a
     differing `qmanifest-v1` over included content with a recorded count/size
     delta).
     A bare digest mismatch, a missing baseline, an incompatible profile (F21.3), or
     an incomplete inventory (F19.2) yields content state `DIFFERS_FROM_KNOWN`
     and/or `UNVERIFIED`, **never** "modified".
- **M27.8 — Identifier claims may precede entities.** An unresolved identifier is
  stored as evidence with no target (D5.4) and never invents an entity. Resolving
  it later is an explicit step.
- **M27.9 — Foreign/native IDs are validated, not trusted.** A claim from a
  sidecar or catalog selects an entity only after validation (D5.8).
- **M27.10 — DLC/components do not change base identity.** A DLC/component
  identifier is SUPPORTING/STRONG _component_ evidence and never re-identifies the
  base Game/Release; the base Release remains stable while the component set
  changes (proposal [§39](./LIBRARY_ITEM_IDENTIFIER.md)).
- **M27.11 — External DAT match scope.** A DAT checksum match is EXACT only for
  the mapped media/ROM content, only when source/version/checksum provenance is
  recorded, and it does not by itself prove installation completeness or a Build
  (T11 ingestion). **OQ-18 is deferred to T11 (M2) with frozen safe M1 behavior
  (T00f2):** M1 runs no catalog matching, so no catalog evidence exists in M1.
  The exact source/version/checksum key set and the non-SHA algorithms
  (CRC32/MD5/SHA-1) are T11; this contract freezes only that they must be
  recorded and that a match without them is not EXACT.

### Invariants

- **INV-48** An EXACT content match without a trusted mapping never creates or
  selects a Game/Release/Build and never invents a placeholder.
- **INV-49** A fingerprint/hash mismatch alone never sets a modified
  (`IDENTIFIED_MODIFIED`) content state.
- **INV-50** Identifier resolution is always constrained by namespace and scope;
  no value-only or fuzzy identifier lookup.
- **INV-51** DLC/component identifiers never change the base Game/Release
  identity.

---

## 28. Untrusted sidecar handling

Proposal [§34](./LIBRARY_ITEM_IDENTIFIER.md): a `.questarr.json` sidecar is
strong future-scan evidence. Plan section 3: sidecars are untrusted claims, even
when schema-valid; check local IDs, portable identifiers, stale fingerprints, and
contradictions; foreign database IDs must not accidentally resolve to an
unrelated local game.

### Decisions

- **M28.1 — Untrusted even when valid.** A sidecar is an untrusted claim even when
  schema-valid and correctly versioned. It is excluded from every fingerprint
  input (F13.8, F20.4).
- **M28.2 — Questarr IDs are validated locally.** A `questarrId` is usable only
  after it resolves to an existing local entity under the acting scope
  (D6.5/INV-14). A `questarrId` that does not resolve is _unresolved evidence_; it
  must never be matched to an unrelated local Game by numeric or title proximity.
- **M28.3 — Portable identifiers are claims.** Sidecar namespace/value
  identifiers are validated for local consistency (known namespace/scope, value
  normalized per D5.3) and recorded with `source = sidecar`. The validation
  mechanics are T07 (OQ-5, resolved by deferral in T00f); the decision rule is
  frozen here: an unvalidated portable identifier cannot select or create an
  entity and is usable only as evidence/review hint (D5.8, INV-53). **OQ-19 is
  deferred to T07 (M1) with frozen safe M1 behavior (T00f2):** M1 validates
  local IDs (M28.2) and cross-checks the sidecar's claimed same-profile exact
  fingerprint against locally recomputed evidence (M28.4/M28.5); optional
  anti-copy signatures or extra consistency signals are T07, and a sidecar that
  cannot be locally validated is never AUTHORITATIVE.
- **M28.4 — Sidecar fingerprints are claims, not observations.** A fingerprint
  value in a sidecar is a claim from the past, never a locally computed
  fingerprint (F11.7). It is never stored as a local fingerprint and never proves
  current content. It may be used only as a staleness hint.
- **M28.5 — Stale/copied sidecar.** Two cases must be distinguished.
  - **Stale/copied fingerprint claim.** When the sidecar's claimed same-profile
    exact fingerprint no longer matches the locally recomputed compatible
    fingerprint (F21.2), the claim is stale and is **not** AUTHORITATIVE: it
    contributes only as a non-authoritative hint and resolves no higher than
    PROBABLE pending review, and a copied sidecar must not re-associate a
    different installation.
  - **Identity contradiction.** When the sidecar's claimed identity contradicts
    another trusted identity source — a validated native identifier (M27.3) or an
    exact-identity mapping — this is a **strong conflict** under section 29
    (M29.1/M29.2/INV-55): the sidecar claim is not AUTHORITATIVE, no association
    is auto-written (M29.6), and the item is `CONFLICT` for review rather than
    PROBABLE. This matches the plan section-6 matrix row "Sidecar versus native-ID
    contradiction → CONFLICT".
- **M28.6 — Partial sidecar.** A sidecar whose referenced entities are partly
  unresolved contributes only the evidence it can support; unresolved parts stay
  unresolved and never block persistence of the observation (D3.3/D4.3).
- **M28.7 — Writing is configurable and never retroactive.** Writing a sidecar
  never changes fingerprints and never upgrades a decision retroactively; a
  decision is not authoritative merely because a sidecar was later written.

### Invariants

- **INV-52** A sidecar fingerprint claim is never treated as a locally computed
  fingerprint or as exact content proof.
- **INV-53** An unresolvable sidecar/native ID never resolves to an unrelated
  local entity.
- **INV-54** A validated sidecar reference is AUTHORITATIVE only while it does not
  contradict locally recomputed exact evidence.

---

## 29. Strong conflicts

Plan section 3: contradictory strong evidence produces CONFLICT **before**
priority selection. Proposal [§48](./LIBRARY_ITEM_IDENTIFIER.md): Questarr must
surface disagreement rather than silently picking one.

### Decisions

- **M29.1 — Conflict before priority.** When two or more evidence items of class
  AUTHORITATIVE, EXACT, or STRONG support mutually incompatible targets,
  resolution stops and the state is `CONFLICT`; no priority order or "highest
  score" picks a winner.
- **M29.2 — Examples.** A sidecar naming Game A while a validated Steam AppID
  resolves to Game B; an exact `qmanifest-v1` mapping to Build B while a native
  serial maps to Release A. Both are conflicts, not ties to break by rank.
- **M29.3 — Persisted and surfaced.** All sides, their provenance, and their
  observation revisions are retained; the item becomes review-required. A later
  explicit user decision may resolve it (section 30), recorded as a new decision,
  never inferred silently.
- **M29.4 — Weak/fuzzy disagreement is not a conflict.** Disagreement among
  SUPPORTING/FUZZY/WEAK evidence lowers confidence to PROBABLE/AMBIGUOUS but does
  not itself raise `CONFLICT`; only the three trusted classes can conflict.
- **M29.5 — Conflict vs modified.** `CONFLICT` is _identity_ disagreement between
  trusted sources. `IDENTIFIED_MODIFIED` is _content_ disagreement under an
  agreed identity (M27.7). A native-ID/content mismatch with matching identity is
  a content state, not a conflict.
- **M29.6 — Conflict blocks auto-link.** No automatic association is written
  while a conflict exists. An existing manual decision contradicted by new
  trusted evidence is retained but flagged conflicting (M30.6), never overwritten.

### Invariants

- **INV-55** `CONFLICT` is produced before any priority/score selection when
  trusted evidence is contradictory.
- **INV-56** A conflict is never auto-resolved by rank, score, or recency.
- **INV-57** `CONFLICT` and `IDENTIFIED_MODIFIED` are distinct and never
  interchangeable.

---

## 30. Decision durability: state separation, manual associations, revisions

### State separation

- **M30.1 — Four orthogonal axes.** Plan section 3 requires separating identity
  match state, content comparison state, accessibility, and job state:
  - `identity_state` ∈ {`IDENTIFIED`, `IDENTIFIED_MODIFIED`, `PROBABLE`,
    `AMBIGUOUS`, `UNKNOWN`, `CONFLICT`} (proposal [§48](./LIBRARY_ITEM_IDENTIFIER.md)),
    governed by sections 25–29;
  - `content_state` ∈ {`MATCHES_KNOWN`, `DIFFERS_FROM_KNOWN`, `UNVERIFIED`}
    alongside inventory completeness (F19.1);
  - `accessibility_state` ∈ {`ACCESSIBLE`, `PARTIALLY_ACCESSIBLE`, `INACCESSIBLE`,
    `MISSING`, `ROOT_UNRESOLVED`};
  - `job_state` — durable job lifecycle (`queued`, `running`, `completed`,
    `completed-with-errors`, `failed`, `cancelled`); **out of scope here** and
    owned by Part IV of this contract. Socket.IO is notification transport, not
    durable truth (plan section 3).
- **M30.2 — No axis implies another.** Identity may hold while content differs
  (→ `IDENTIFIED_MODIFIED`) or while the item is inaccessible. A failed/cancelled
  job does not erase identity; accessibility does not change identity.
- **M30.3 — `IDENTIFIED_MODIFIED` is exact.** It is exactly identity=IDENTIFIED +
  content=`DIFFERS_FROM_KNOWN` with a compatible baseline and interpretable
  evidence (M27.7). It is not a synonym for inaccessible, incomplete, or conflict.

### Manual association persistence

- **M30.4 — Manual decisions are first-class records.** A decision records:
  target (`game_id`/`release_id`/`build_id`, or an explicit "unidentified/
  rejected" choice); decision kind (confirm/reject/unlink/set); actor; timestamp;
  the evidence considered; and the observation revision it was made against
  (M30.10). **OQ-15 is deferred to T01/T02 (M1) with frozen safe M1 behavior
  (T00f2):** the concrete table/column shape is T01/T02, but the frozen M1 rule
  is that decisions are **append-only** records — a new decision never mutates
  or deletes a prior one (M30.7) — and the actor is captured at decision time as
  the acting user's string ID plus display name, not resolved live. Superseded
  decisions are retained exactly as D8.2/OQ-4 require.
- **M30.5 — Decisions survive rescans.** A rescan does not clear a manual
  association. Re-observation updates the observation/evidence but keeps the
  decision until explicitly changed or contradicted (plan section 3).
- **M30.6 — Changed authoritative evidence ⇒ conflict, not silent replacement.**
  If new trusted evidence contradicts an existing manual decision, the decision is
  retained and flagged `CONFLICT` for review; the system never silently overwrites
  the user's choice (plan section 3).
- **M30.7 — Resolution is explicit and recorded.** A user may resolve a conflict
  (choose a side, mark "not this", or clear), producing a new decision record;
  prior decision/evidence history is retained (OQ-4, resolved in T00f: retained
  as immutable history, never destructively dropped).
- **M30.8 — Authorization.** A manual association requires that the acting user
  own/can access the target Game (D6.5/INV-14) and respects root access policy
  (section 6); decisions never cause cross-user association or leakage.

### Revision and snapshot protection

- **M30.9 — Observation revision token.** Every observation carries a monotonic
  revision token per Installation (occurrence) identifying the exact observed
  inventory, completeness, and fingerprint set used. **OQ-14 is resolved
  (T00f2):** the token is a per-Installation monotonically increasing integer
  (`observation_revision`), **not** a content-addressed snapshot ID. Allocation
  is transactional with the observation commit, starts at 1, and never resets;
  an observation is an event, so re-observing identical content still advances
  the token. A content digest may be stored _alongside_ the token for the
  manifest/raw-inventory store (F19.6), but never replaces it. Prior revisions
  are retained revision-scoped; the exact table, cleanup count, and retention
  window are T02 (OQ-10/OQ-24), and the safe M1 rule is that the current
  revision and any revision still referenced by a decision, job, or candidate
  result (M30.10) are never garbage-collected. This preserves the stale-write
  guard of M30.10/INV-61.
- **M30.10 — Decisions and job outputs are revision-bound.** A decision,
  candidate set, or resolution result records the observation revision it was
  computed against. A stale (older-revision) job result or decision must not be
  applied over a newer observation; it is discarded or recomputed, never silently
  applied (plan section 3).
- **M30.11 — Decisions remain valid across compatible revisions.** A manual
  decision made at revision N stays associated while later revisions do not change
  authoritative evidence relevant to it. When a newer revision changes
  authoritative evidence, M30.6 (conflict) applies.
- **M30.12 — Unstable observations do not decide.** Fingerprints from unstable/
  changing files or incomplete inventories (section 18, F19.2) never back an
  automatic association; a resolution computed from them is provisional and
  revision-scoped.
- **M30.13 — Stale sidecar vs rescan.** A rescan whose recomputed compatible exact
  fingerprint disagrees with a sidecar claim treats the sidecar as stale (M28.5);
  it does not resurrect an old manual decision that the newer evidence
  contradicts.
- **M30.14 — Accessibility state transitions (resolved T00f2, OQ-20).**
  `accessibility_state` uses exactly the five M30.1 values and moves only as
  follows. `ROOT_UNRESOLVED` and `INACCESSIBLE` describe the _root_, so they
  take precedence over any per-path state and never delete an observation
  (D8.5/INV-24/J44.1). `MISSING` is the only inference about a path and requires
  a complete current-generation pass (J42.3/INV-73/INV-75); a failed, cancelled,
  `INCOMPLETE`, or inaccessible pass leaves every prior accessibility value
  unchanged (J44.2). Returning state is re-observed, never inferred (J44.3).
  Accessibility is independent of identity/content/job and never changes them
  (M30.2/INV-58).

  | From                                | Observed condition                                                       | To                                               |
  | ----------------------------------- | ------------------------------------------------------------------------ | ------------------------------------------------ |
  | any                                 | root removed from configuration (D8.5)                                   | `ROOT_UNRESOLVED`                                |
  | any                                 | root present but unreadable/unreachable (J44.1)                          | `INACCESSIBLE`                                   |
  | `ROOT_UNRESOLVED`/`INACCESSIBLE`    | root re-resolves and reads (J44.3)                                       | re-observed: `ACCESSIBLE`/`PARTIALLY_ACCESSIBLE` |
  | `MISSING`                           | path re-observed present (J44.3)                                         | `ACCESSIBLE`                                     |
  | any (root readable, path present)   | all included entries readable                                            | `ACCESSIBLE`                                     |
  | any (root readable, path present)   | some included entries unreadable/changing                                | `PARTIALLY_ACCESSIBLE`                           |
  | `ACCESSIBLE`/`PARTIALLY_ACCESSIBLE` | path absent and current pass `COMPLETE`/`COMPLETE_WITH_WARNINGS` (J42.3) | `MISSING`                                        |
  | any                                 | path absent, but pass failed/cancelled/`INCOMPLETE`/inaccessible         | unchanged (no missing inference)                 |

### Invariants

- **INV-58** `identity_state`, `content_state`, `accessibility_state`, and
  `job_state` are stored and reported independently; none is derived from
  another.
- **INV-59** `IDENTIFIED_MODIFIED` requires identity=IDENTIFIED **and** a
  comparable known baseline **and** interpretable difference — never a bare
  mismatch.
- **INV-60** A manual decision survives rescans unless explicitly changed or
  contradicted by trusted evidence (then `CONFLICT`).
- **INV-61** No stale (older-revision) decision or job result overwrites a newer
  observation.
- **INV-62** Manual association is subject to the same ownership/access rules as
  any association (INV-14).

---

## 31. Explainable results and compact decision table

### Decisions

- **M31.1 — Always explain _why_.** Every resolved item exposes: the state, the
  chosen target, the ordered evidence list with class and provenance, the
  conflicts, the content and accessibility states, and the observation revision
  (proposal [§47](./LIBRARY_ITEM_IDENTIFIER.md)). No unexplained numeric
  confidence is presented.
- **M31.2 — Result shape (illustrative; API/DTO shape is a later slice).**

  ```json
  {
    "identity_state": "IDENTIFIED_MODIFIED",
    "content_state": "DIFFERS_FROM_KNOWN",
    "accessibility_state": "ACCESSIBLE",
    "observation_revision": 12,
    "target": { "game_id": "…", "release_id": "…", "build_id": null },
    "baseline": { "build_ref": "qfp:qmanifest:1:sha256:…:steam-v1:1" },
    "evidence": [
      {
        "class": "STRONG",
        "type": "identifier",
        "namespace": "steam.app",
        "value": "1091500",
        "source": "filesystem",
        "revision": 12
      },
      {
        "class": "EXACT",
        "type": "fingerprint",
        "ref": "qfp:qmanifest:1:sha256:…:steam-v1:1",
        "source": "local",
        "revision": 12,
        "matches": "known-build"
      },
      {
        "class": "FUZZY",
        "type": "structural",
        "similarity": 0.94,
        "source": "local",
        "revision": 12
      }
    ],
    "conflicts": [],
    "reasons": ["Steam AppID resolves Release", "known baseline differs: 472 additional mod files"]
  }
  ```

- **M31.3 — Compact decision table.** Summary of sections 25–30; the sections are
  normative where they differ.

  | Evidence present                                                        | Trusted mapping | Content vs known      | Automatic outcome                       | `identity_state`                                                         |
  | ----------------------------------------------------------------------- | --------------- | --------------------- | --------------------------------------- | ------------------------------------------------------------------------ |
  | Questarr import provenance (AUTHORITATIVE)                              | yes             | any                   | auto-link to Game                       | `IDENTIFIED` (or `IDENTIFIED_MODIFIED` when the baseline differs, below) |
  | Validated sidecar → local Game, no contradiction (AUTHORITATIVE)        | yes             | any                   | auto-link to Game                       | `IDENTIFIED`                                                             |
  | Exact `sha256`/`qmanifest-v1` → known Release/Build (EXACT)             | yes             | matches               | auto-link artifact + Release/Build      | `IDENTIFIED`                                                             |
  | Exact content match, no Game mapping                                    | no              | matches               | identify content only; no Game invented | `PROBABLE`                                                               |
  | Native ID → unique Release (STRONG)                                     | yes             | —                     | auto-link Release; Build unknown        | `IDENTIFIED`                                                             |
  | Native ID + build/revision in provider context (STRONG)                 | yes             | matches baseline      | auto-link Release/Build                 | `IDENTIFIED`                                                             |
  | Native ID (STRONG) + compatible baseline differs, interpretable         | yes             | differs               | keep identity; mark modified            | `IDENTIFIED_MODIFIED`                                                    |
  | Native ID (STRONG) + bare mismatch / no baseline / incompatible profile | yes             | differs or unverified | no "modified" claim                     | `IDENTIFIED` with content `UNVERIFIED`                                   |
  | Structural fuzzy + title + supporting, no STRONG+ anchor                | no              | —                     | never auto-link                         | `PROBABLE` / `AMBIGUOUS`                                                 |
  | Name/title only                                                         | no              | —                     | never auto-link                         | `PROBABLE` / `AMBIGUOUS` / `UNKNOWN`                                     |
  | Two trusted sources disagree                                            | —               | —                     | stop; review                            | `CONFLICT`                                                               |
  | No meaningful evidence                                                  | no              | —                     | none                                    | `UNKNOWN`                                                                |

  Accessibility transitions (M30.14) and candidate-set recomputation (M25.8) add
  no row: they are orthogonal to identity and never alter it (INV-58). The table
  only states which trusted evidence may auto-associate; it never authorizes a
  link sections 25–29 forbid (INV-64).

- **M31.4 — Explainability for decisions and conflicts.** Manual decisions and
  conflicts also retain their evidence and disagreement so the UI can present
  both (proposal §47/§48).

### Invariants

- **INV-63** Every resolved item exposes its evidence with class and provenance;
  no bare numeric confidence stands alone.
- **INV-64** The decision table never authorizes an automatic link that sections
  25–29 forbid.

---

## 32. Open issues (Part III)

All Part III matching/decision questions **OQ-14–OQ-20 were closed by T00f2**
(section 33), each as either a concrete M1 rule or an explicit named-milestone
deferral with a frozen safe M1 behavior. No Part III question remains open in
this contract; any later change to a Part III **M** decision or **INV-42–INV-64**
is governed by the change-control rules (section 34).

---

## 33. Resolved questions (OQ-1–OQ-38)

Part I questions (OQ-1–OQ-7) and Part II fingerprint questions (OQ-8–OQ-13)
were closed by T00f; Part III matching/decision questions (OQ-14–OQ-20) were
closed by T00f2; Part IV scan-job/resource questions (OQ-21–OQ-30) were closed
by T00f3; Part V API/DTO questions (OQ-31–OQ-38) were closed by T00f4. Each is
now either a concrete M1 rule or an explicit deferral to a named later milestone
with a safe M1 behavior; the normative text lives in the cited decision. No
frozen canonical preimage changed and no golden vector is added or altered by any
of these resolutions. Strong-conflict-before-priority (M29.1/INV-55),
untrusted sidecars (M28.x/INV-52), manual-decision persistence
(M30.5/M30.6/INV-60), and no-name/fuzzy-auto-link (M25.6/M25.7/INV-42) are
unchanged by the Part III resolutions below, the Part IV resolutions below change
no **J** transition, **INV-65–INV-76**, fingerprint, or matching rule, and the
Part V resolutions below change no existing **P** decision's meaning, no
**INV-77–INV-86**, route, error code, fingerprint, matching, or job rule.

- **OQ-1 — Root-identity persistence shape — RESOLVED (T00f).** A normalized
  `library_roots` table with a stable string `id`; `library_items.root_id` +
  `relative_path`. Normative rule: **D6.1** (one row per source; a `ROOT_FOLDER`
  whose source row is removed keeps its identity as `ROOT_UNRESOLVED` per
  D8.5). Rationale: matches proposal [§44](./LIBRARY_ITEM_IDENTIFIER.md)'s
  `library_root_id`, gives one stable identity per source row, and avoids
  duplicated root columns on every item.
- **OQ-2 — Overlapping roots precedence — RESOLVED (T00f).** Containment is
  equality or an `R + "/"` prefix; precedence is the owner `LIBRARY_ROOT` first,
  then the longest `ROOT_FOLDER` prefix; delete-time `allowDelete` is evaluated
  independently. Normative rule: **D7.9**. Rationale: mirrors
  `isWithinDeletableRootFolder` and confirms identification never grants delete
  rights.
- **OQ-3 — Polymorphic identifier integrity — RESOLVED (T00f).** Application-
  enforced transactional target checks with `entity_type = scope`, plus
  downgrade-to-unresolved (never re-point) on target deletion and a T02 orphan
  sweep. Normative rule: **D5.9**. Rationale: SQLite cannot express the
  cross-table FK, so integrity stays in the storage layer with no dangling
  references.
- **OQ-4 — Association-history retention — RESOLVED (T00f).** A cleared
  association is retained as an immutable evidence/decision-history entry
  (prior target, actor, timestamp, reason, revision); storage shape is T01/T02
  (OQ-15). Normative rule: **D8.2** (also M30.7). Rationale: preserves _why_
  without leaving a live link.
- **OQ-5 — Sidecar portable-identifier resolution — DEFERRED to T07 (M1) with
  frozen safe M1 behavior (T00f).** Validation mechanics are T07; until
  validated, a portable identifier is evidence only and never selects or creates
  an entity. Normative rule: **D5.8** and **M28.3**. Rationale: sidecars are
  untrusted (M28.x) and T07 owns the parser; the safety rule keeps M1 sidecar
  handling non-authoritative.
- **OQ-6 — `game_files` migration path — RESOLVED (T00f).** `game_files` rows
  are not projected into Installations; backfill reads only `games.libraryPath`
  and import provenance comes from `ImportManager`. Normative rule: **D7.3**
  (with D7.7). Rationale: `game_files` is per-file and `downloadId`-scoped
  (proposal [§9](./LIBRARY_ITEM_IDENTIFIER.md), D7.3/INV-20).
- **OQ-7 — Artifact creation trigger — RESOLVED (T00f).** Lazy: an Artifact is
  created only when an exact `sha256`/compatible `qmanifest-v1` identity exists;
  `qtree` and non-fresh/incomplete fingerprints never create or resolve one, and
  there are no eager per-Installation rows. Normative rule: **D3.6**.
  Rationale: honors D3.3 (nullable artifact) and F14.5.
- **OQ-8 — Coarse `mtime` detection — RESOLVED (T00f).** Passive per-root
  `mtime_resolution`; `coarse`/`unavailable` roots treat identity cache entries
  as misses and rehash, `verify` always rehashes. Normative rule: **F22.6**.
  Rationale: portable and side-effect-free; same-second evasion stays a
  documented limitation.
- **OQ-9 — Non-UTF-8 name blast radius — RESOLVED (T00f).** Only the affected
  entry (or, for an unsupported directory name, that subtree) is excluded; the
  rest may publish `COMPLETE_WITH_WARNINGS` with matches scoped to included
  files. Normative rule: **F13.6**. Rationale: consistent with F17.6 policy
  skips and INV-30/INV-31.
- **OQ-10 — Completeness and inventory persistence shape — DEFERRED to T02 (M1)
  with frozen safe M1 behavior (T00f).** Revision-scoped retention of
  completeness/warnings/errors/counts, raw inventory stored separately and
  compressed, no per-file rows. Normative rule: **F19.6**. Rationale: storage
  shape is T02; the observable contract (F11.2/F19.4/F20.1) is frozen now.
- **OQ-11 — Single-file Artifact identity — RESOLVED (T00f).** Single-file
  Artifact identity is `sha256`; a one-entry `qmanifest-v1` may coexist for
  installation equality but never defines or substitutes Artifact identity.
  Normative rule: **F16.4**. Rationale: keeps Artifact identity
  profile-independent and distinct from the framed leaf digest.
- **OQ-12 — Symlink target storage — RESOLVED (T00f).** Never persist raw
  targets in shared/non-owner-readable projections; store an inside-root
  relative target or a redacted marker. Normative rule: **F17.2**. Rationale:
  proposal [§49](./LIBRARY_ITEM_IDENTIFIER.md) path-disclosure safety and
  INV-15; symlinks remain excluded from digests.
- **OQ-13 — Reserved media kinds reuse — RESOLVED (T00f).** `qset-v1`/
  `qrelease-set-v1` must reuse the section 12 framing unchanged; T12 adds only a
  kind-specific preamble/field order/vectors; M1 uses neither. Normative rule:
  **F12.8**. Rationale: no frozen hash bytes change and no new golden vector is
  required.

### Part III matching/decision resolutions (T00f2)

- **OQ-14 — Revision/snapshot token persistence shape — RESOLVED (T00f2).** The
  observation token is a per-Installation monotonic integer
  (`observation_revision`), not a content-addressed snapshot ID; it is allocated
  transactionally with the observation, starts at 1, and never resets. Prior
  revisions are retained revision-scoped, and a revision referenced by a live
  decision/job/candidate is never collected. Normative rule: **M30.9**.
  Rationale: an observation is an event, so equal content re-observed still
  advances the token, which makes the stale-write guard (M30.10/INV-61)
  unambiguous; a content digest may be stored alongside, never as the token.
- **OQ-15 — Decision schema and history — DEFERRED to T01/T02 (M1) with frozen
  safe M1 behavior (T00f2).** Exact decision storage/columns/actor columns are
  T01/T02. The frozen M1 rule is that decisions are append-only first-class
  records (M30.4) with the actor captured at decision time, and superseded
  decisions are retained as history (M30.7, D8.2, INV-60). Normative rule:
  **M30.4/M30.7** and **D8.2/INV-60**. Rationale: M1 must persist decisions
  across rescans; the storage shape can change later without changing the
  durable semantics.
- **OQ-16 — Candidate-set persistence — RESOLVED (T00f2).** M1 recomputes
  candidates on demand for the current observation revision and persists no
  durable candidate row set; a later persisted cache (T05/T06) is
  revision-scoped, invalidated by a newer revision, and never authoritative.
  Normative rule: **M25.8**. Rationale: candidates are stateless hypotheses
  (M25.1) and rank is not truth (M25.3/INV-44); recompute keeps AMBIGUOUS review
  deterministic without a new durable identity surface.
- **OQ-17 — Fuzzy corroboration thresholds — DEFERRED to T13 (M3) with frozen
  safe M1 behavior (T00f2).** M1 freezes no fuzzy threshold and enables no
  Rule F-style auto-link; FUZZY/WEAK/SUPPORTING evidence always needs an
  AUTHORITATIVE/EXACT/STRONG trusted mapping to auto-associate (M25.7), and a
  future threshold may only corroborate. Normative rule: **M25.6/M25.7** and
  **M26.2**. Rationale: a wrong automatic merge is worse than review (M25.5),
  and numeric thresholds need labeled fixtures (T13).
- **OQ-18 — External catalog matching provenance — DEFERRED to T11 (M2) with
  frozen safe M1 behavior (T00f2).** M1 performs no catalog matching, so no
  catalog evidence exists in M1; the frozen requirement is that any catalog match
  records source/version/checksum and algorithm and is EXACT only for the mapped
  content (M27.11). Normative rule: **M27.11** (and P56.3). Rationale: catalog
  ingestion and its exact key/algorithm set are T11 (M2).
- **OQ-19 — Sidecar anti-copy strengthening — DEFERRED to T07 (M1) with frozen
  safe M1 behavior (T00f2).** M1 validates local IDs (M28.2) and treats a sidecar
  fingerprint as a claim checked against locally recomputed evidence
  (M28.4/M28.5); optional signatures or extra consistency signals are T07, and an
  unvalidatable sidecar is never AUTHORITATIVE. Normative rule:
  **M28.3/M28.5** and **INV-54**. Rationale: sidecars stay untrusted (M28.1) and
  T07 owns the parser; the M1 safety rule already prevents a copied sidecar from
  re-associating a different installation.
- **OQ-20 — Accessibility state transitions — RESOLVED (T00f2).** M1 uses the
  five M30.1 accessibility states with the M30.14 transition table: root removal
  yields `ROOT_UNRESOLVED`, an unreadable root yields `INACCESSIBLE`, only a
  complete current-generation pass may mark a path `MISSING`, and returning state
  is re-observed. Normative rule: **M30.14** (with D8.5, J44.1–J44.3,
  INV-58/INV-75). Rationale: reuses the frozen job-root outcomes and forbids
  mass-missing inference; accessibility never changes identity or content.

### Part IV scan-job/resource resolutions (T00f3)

- **OQ-21 — Budget calibration — DEFERRED to T10 (M1) with frozen safe M1
  behavior (T00f3).** The J38.2 values are the frozen, configurable, bounded M1
  defaults; T10 measures and publishes environment-specific figures and is the
  only owner that may replace a default (same-change contract amendment, section
  34). M1 never presents a provisional default as a measured target and invents
  no NAS throughput figure (plan T10). Normative rule: **J38.1/J38.2/J38.5**
  (with J38.4/INV-69). Rationale: unmeasured numbers must not become promises;
  bounded placeholders plus explicit ceilings keep M1 safe while T10 measures.
- **OQ-22 — Anchor selection set — DEFERRED to T03/T08 (M1) with frozen safe M1
  behavior (T00f3).** M1 freezes no anchor set because L2 is off by default
  (J37.3); when L2 is enabled the set is adapter/profile-selected and bounded by
  `max_anchor_file_bytes`, and L2 alone never establishes exact identity.
  Normative rule: **J37.1/J37.2** (with J38.2/INV-68). Rationale: the default M1
  path needs no anchor set; T03 owns profiles/inventory and T08 the Steam
  adapter's anchor designation.
- **OQ-23 — Lease/heartbeat and retry numbers — DEFERRED to T02/T06 (M1) with
  frozen safe M1 behavior (T00f3).** M1 freezes `lease_seconds=30`, a heartbeat
  at least once per `lease_seconds`, expiry → `failed` (`interrupted`), and
  `max_job_attempts=1` (single manual restart, no automatic retry loop); exact
  heartbeat interval, clock-skew handling, stale-lock reclamation timing, and
  retry policy are T02/T06. Normative rule: **J40.1/J40.2/J40.4/J41.3**.
  Rationale: restart must be safe and bounded in M1 without freezing unmeasured
  timing detail.
- **OQ-24 — Job retention — DEFERRED to T02 (M1) with frozen safe M1 behavior
  (T00f3).** M1 retains terminal jobs, counters, and partial raw inventories at
  least until superseded by a later generation or an explicit cleanup, and never
  collects a revision referenced by a live job/decision/candidate (M30.9); the
  retention window and cleanup shape are T02. Normative rule: **J35.1** (with
  F19.6/M30.9). Rationale: retention shape is storage, but the frozen rule keeps
  the stale-write guard (M30.10/INV-61) from losing a referenced revision.
- **OQ-25 — Global concurrency default — DEFERRED to T10 (M1) with frozen safe
  M1 behavior (T00f3).** M1 freezes `max_concurrent_roots` default 2, configurable
  and bounded ≥ 1, with one active job per root regardless (J41.1); T10 calibrates
  the value for NAS/IO-bound hosts. Normative rule: **J38.3/J38.5**. Rationale: a
  bounded provisional default keeps M1 usable while T10 owns measured tuning.
- **OQ-26 — Cancellation during persistence — DEFERRED to T02 (M1) with frozen
  safe M1 behavior (T00f3).** Whether an in-flight persist batch completes or is
  rolled back is T02, but no torn batch is observable and a cancelled run
  publishes or compares no exact fingerprint. Normative rule:
  **J39.2/J39.3/J39.4** (with INV-70). Rationale: atomicity mechanics are storage,
  while the observable safety rule can be frozen now.
- **OQ-27 — Job/lock storage shape — DEFERRED to T02 (M1) with frozen safe M1
  behavior (T00f3).** Exact tables/columns are T02; M1 requires a durable
  uniqueness constraint on root identity plus non-terminal state (one active job,
  INV-72), transactional generation allocation (J42.1), and a durable lease.
  Normative rule: **J35.2/J41.1/J42.1**. Rationale: storage shape is T02, but the
  single-active-job guarantee must be durable rather than in-memory.
- **OQ-28 — Multi-process coordination — DEFERRED to T02/T06 (M1) with frozen
  safe M1 behavior (T00f3).** M1 targets one scanning process per database; the
  durable lock/constraint prevents a second non-terminal job even if issued by
  another process, but M1 promises no distributed failover, leadership election,
  or simultaneous multi-process scanning of one root. Normative rule:
  **J41.1/J41.3/J41.4** (with INV-72). Rationale: avoid overpromising
  coordination M1 does not implement while keeping the single-job guarantee
  durable.
- **OQ-29 — Duplicate-request policy — RESOLVED (T00f3).** A duplicate start for a
  root with an active job joins it (`joined: true`, P52.2) and never creates a
  second job; options matching the applied budget join, differing options are
  rejected `409 scan-options-mismatch` (P52.3) and never mutate the running
  budget. Finer queue/ignore policy is tracked as OQ-34 (resolved by T00f4;
  Part V below). Normative rule:
  **J41.2** (with J38.1/P52.2/P52.3/INV-72). Rationale: Part V already froze the
  API default, so M1 no longer needs an open API decision; the residual finer
  policy is tracked separately.
- **OQ-30 — Stage granularity and progress denominators — DEFERRED to T02/T09
  (M1) with frozen safe M1 behavior (T00f3).** The J36.1 stage list is the M1
  durable granularity; M1 exposes raw counters plus `progress_known` and
  synthesizes no percentage (J36.3); finer sub-stages/denominators and UI
  presentation are T02/T09. Normative rule: **J36.1/J36.3**. Rationale: durable
  stages must be stable for persistence now, while decomposition and presentation
  can evolve.

### Part V API/DTO resolutions (T00f4)

- **OQ-31 — API-key/integration exposure — RESOLVED (T00f4).** M1 adds **no**
  library-item read to the `/api/integration/*` API-key surface; that surface
  stays exactly as frozen in section 59/D7.4–D7.5 (`libraryPath` projection
  only, key management JWT-only), and the new library-item/catalog endpoints are
  JWT-only behind the default-deny `requireAuthenticationForApi`/
  `authenticateToken` gate (P49.2). Any future API-key exposure is a T06 API
  decision that must use the OQ-33 redacted projection (P49.7) and never weaken
  INV-12–INV-16/INV-77. Normative rule: **P49.2** (with section 59). Rationale:
  no extension use case is frozen, the default-deny gate already makes new routes
  JWT-only, and per-user root content must not reach long-lived keys without an
  ownership/redaction design.
- **OQ-32 — Cursor pagination — DEFERRED to T06 (M1) with frozen safe M1
  behavior (T00f4).** M1 uses `limit`/`offset` exactly as P51.2 (`limit` default
  20, clamped 1..100; `offset` ≥ 0; the same `validatePaginationParams`
  semantics) and every list endpoint returns a stable deterministic total order
  (jobs newest-generation-first per P52.5; a documented unique tiebreaker for
  items/evidence/decisions) so offset paging is deterministic within a snapshot.
  M1 accepts no `cursor` parameter. Cursor paging for very large histories is
  T06 and must be additive (a token alongside, not replacing, the offset
  envelope). Normative rule: **P51.2** (with P52.5). Rationale: the retained
  helper is offset-based and M1 scale is modest; a stable order is the
  precondition for a correct future cursor, and freezing a token format before
  the T02 storage shape exists would over-constrain it.
- **OQ-33 — Path/hash redaction projection — RESOLVED (T00f4).** Visibility
  first follows the frozen gate: an item associated with another user's Game is
  omitted from lists and `404` on detail (P51.4/D6.6/INV-15). For a visible item,
  a caller is **entitled** to its location/hash material iff it owns the
  associated Game, owns the item's `LIBRARY_ROOT` (D6.1), or the item's root is a
  global, owner-less `ROOT_FOLDER` (D6.2/INV-12). A non-entitled caller receives
  the redacted projection of P49.7: `relative_path` = `null`; symlink targets
  reduced to F17.2's non-revealing classification; fingerprint material
  (`fingerprint_ref`, digest values, `baseline`) suppressed; `etag` omitted. Item
  identity/state/counts/`root` (kind + opaque id) and evidence class/provenance/
  timestamps remain, and `display_name` is kept only when it is not path-derived
  (a path-derived label becomes a non-path placeholder); a foreign `game` stays
  `null` (D6.6). No field ever carries an absolute path (D7.1/INV-16). Normative
  rule:
  **P49.6/P49.7** (with D6.1–D6.6/F17.2/INV-15/INV-78). Rationale: unassociated
  items are shared discovery data (D6.4) and global root folders are already
  readable by any authenticated user (D6.3), but a per-user `LIBRARY_ROOT` is
  implicitly private (D6.1), so its paths and hashes are withheld while the item
  stays discoverable.
- **OQ-34 — Duplicate-start policy — RESOLVED (T00f4).** M1 freezes the
  join-or-reject policy and adds **no queue**: a start that matches the active
  job's applied budget joins it (`joined: true`, P52.2), and differing
  levels/budgets are rejected `409 scan-options-mismatch` (P52.3) without
  mutating the running budget (J38.1). M1 never queues, parks, or silently
  ignores a differing request. Finer queue/priority policy is T06 and must still
  create no second non-terminal job (INV-72/INV-80). Normative rule:
  **J41.2/P52.2/P52.3** (extends OQ-29). Rationale: queueing needs durable
  ordering semantics beyond M1; join-or-reject is already the frozen API default.
- **OQ-35 — Candidate set exposure — RESOLVED (T00f4).** M1 always recomputes
  candidates on demand for the current `observation_revision` and serves no
  persisted candidate set, so candidate responses carry no ETag/`If-None-Match`
  and no candidate identity; the response exposes the `observation_revision` it
  was computed at so a client detects staleness (P53.3/P53.5 with M25.8/OQ-16). A
  persisted/cached candidate set with ETags remains T05/T06 and is never
  authoritative (M25.1/M25.3/INV-44). Normative rule: **P53.3/P53.5** (with
  M25.8). Rationale: OQ-16 already froze on-demand recomputation; an ETag over a
  stateless recomputation would falsely imply durable candidate identity.
- **OQ-36 — Catalog ingestion API — DEFERRED to T11 (M2) with frozen safe M1
  behavior (T00f4).** M1 freezes only the P56.1 CRUD/entries envelope and its
  `source_type` of exactly `"user-file"`: no upload/import endpoint beyond that
  shape, no remote fetch, no redistribution, and no payload execution
  (P56.2/P56.5/INV-86). M1 performs no catalog matching at all (OQ-18), and any
  later catalog match is EXACT only with recorded source/version/checksum
  provenance (P56.3/M27.11). Accepted source formats, upload size limits,
  remote-fetch policy, whether `DELETE /api/catalogs/:id` is blocked while
  dependent matches exist, and superseded-provenance retention are T11; deletion
  never touches library files or existing catalog-match evidence
  (P56.4/D8.1). Normative rule: **P56.1–P56.5** (with M26.7/M27.11/INV-86).
  Rationale: ingestion mechanics are explicitly T11's deliverable; freezing only
  the safe envelope keeps the API additive.
- **OQ-37 — Bulk and idempotency — DEFERRED to T06/T09 (M1) with frozen safe M1
  behavior (T00f4).** M1 exposes **no bulk** decision/rescan endpoint; every
  mutating item request targets one item and is revision-bound by P57 (no
  precondition → `428 revision-required`; stale → `412 stale-revision`), which
  makes a retried write safe by rejecting the stale retry rather than applying it
  twice. Scan start/verify retries are safe by join-on-duplicate and
  job/generation identity (J41.2/P52.2/INV-80). Explicit idempotency keys/replay
  storage and any bulk operation are T06/T09; a bulk operation must preserve
  per-item revision binding and ownership (`403` per P54.2), return per-item
  results, and never bypass INV-61/INV-79. Normative rule: **P57.6** (with
  J41.2/P52.2). Rationale: M1 has revision preconditions and single-item writes,
  which already make retries safe, and no idempotency-key machinery exists.
- **OQ-38 — Notification contract — RESOLVED (T00f4).** M1 reuses the existing
  default Socket.IO namespace and `${basePath}/socket.io/` path (no new
  namespace), retains the legacy `library-scan-progress` event unchanged
  (section 59), and adds the advisory `library-identification:job` event whose
  payload is exactly the P52.6 `JobDto` — field names/values identical to the
  durable job record (J35.2/J36.2) and carrying no root paths or hashes
  (J45.4/INV-85). The event name is the version boundary and `generation` plus
  monotonic counters are the reconciliation keys (J42.1/J36.2/J45.3); M1 makes
  additive changes only, and a breaking payload change needs a new event name
  and is T06. This contract (Part V) owns the payload field schema; T06 owns the
  emitter. Normative rule: **P52.8/P52.9** (with J45.1–J45.4/INV-85). Rationale:
  the frozen transport is advisory and durable-first, so event versioning must
  not become a second source of truth.

---

## 34. Change control

- This document is the complete, single-commit T00 deliverable (Part I T00a +
  Part II T00b + Part III T00c + Part IV T00d + Part V T00e +
  T00f/T00f2/T00f3/T00f4 open-question resolutions), squashed from the former
  per-slice commits; its parts are not reviewed, merged, or consumed standalone.
- T00f resolved Part I/II open questions **OQ-1–OQ-13** (section 33) without
  changing any canonical preimage or golden vector: OQ-1, OQ-2, OQ-3, OQ-4,
  OQ-6, OQ-7, OQ-8, OQ-9, OQ-11, OQ-12, and OQ-13 are concrete M1 rules;
  OQ-5 is deferred to T07 and OQ-10 to T02, each with a frozen safe M1
  behavior. T00f left OQ-14–OQ-20 open; T00f2 (below) resolved them.
- T00f2 resolved the Part III matching/decision open questions **OQ-14–OQ-20**
  (section 33) without changing any canonical preimage, golden vector, or
  fingerprint/matching byte encoding: OQ-14 (M30.9) and OQ-16 (M25.8) and OQ-20
  (M30.14) are concrete M1 rules; OQ-15 is deferred to T01/T02, OQ-17 to T13,
  OQ-18 to T11, and OQ-19 to T07, each with a frozen safe M1 behavior. No Part
  III question remains open (section 32).
- T00f3 resolved the Part IV scan-job/resource open questions **OQ-21–OQ-30**
  (section 33) without changing any canonical preimage, golden vector,
  fingerprint/matching byte encoding, **J** transition, or the meaning of
  **INV-65–INV-76**: OQ-29 is the only question settled without a T10/T02/T03/T06/
  T08/T09 deferral, and OQ-21–OQ-28 and OQ-30 are deferred to a named later task
  with a frozen safe M1 behavior. Budgets stay explicitly configurable and bounded
  (J38.1/J38.5), the J38.2 values stay provisional pending T10 measurement, and
  M1 keeps a single scanning process per database (J41.4). No Part IV question
  remains open (section 48).
- T00f4 resolved the Part V API/DTO open questions **OQ-31–OQ-38** (section 33)
  without changing any canonical preimage, golden vector, fingerprint/matching byte
  encoding, **J** transition, route, or the meaning of **INV-77–INV-86**: OQ-31
  (no integration exposure), OQ-33 (redaction projection, P49.7), OQ-34
  (join-or-reject, no queue), OQ-35 (recompute, no candidate ETag, P53.5), and
  OQ-38 (notification contract, P52.9) are concrete M1 rules; OQ-32 (cursor paging)
  is deferred to T06, OQ-36 (catalog ingestion) to T11, and OQ-37
  (bulk/idempotency) to T06/T09, each with a frozen safe M1 behavior. The
  compatibility map (section 59) and the error-code table (section 58) are
  unchanged; INV-87/INV-88 are added without altering INV-77–INV-86. No Part V
  question remains open (section 61).
- Any later decision that changes a rule marked **D**, **INV**, **F**, or **M**
  in this document must amend this document in the same change (plan section 3:
  "Tasks that change fingerprint semantics must first amend the contract and its
  golden vectors").
- A change to any canonical preimage (Part II sections 11–16) or to a profile's
  inclusion rules (section 21) requires new or updated golden vectors (section 24) and a new `profile_version`/`inclusion_policy_version` where applicable. A
  published v1 profile version and its vectors are immutable.
- A change to any matching rule (Part III **M** decisions or **INV-42–INV-64**)
  must amend this document in the same change; because Part III adds no bytes or
  golden vectors, it introduces no new fingerprint vectors.
- A change to any job rule (Part IV **J** decisions or **INV-65–INV-76**) must
  amend this document in the same change; Part IV adds no fingerprint bytes or
  golden vectors.
- A change to any API rule (Part V **P** decisions or **INV-77–INV-88**) must
  amend this document in the same change; Part V adds no fingerprint bytes,
  golden vectors, matching rules, or production behaviour.
- No fingerprint kind may be published or consumed without a frozen, byte-exact
  encoding and at least one independently reproducible golden vector.
- The reserved kinds (`qset-v1`, `qrelease-set-v1`, `qstructure-minhash-v1`,
  `qtree-casefold-v1`) remain invalid until their owning task freezes them; this
  document must not be treated as specifying them.
- Sections beyond the frozen scope — UI, catalog _ingestion_, the sidecar
  parser, and numeric fuzzy thresholds — remain intentionally absent and owned
  by downstream tasks. HTTP API/DTO and error-envelope
  behaviour is now frozen by Part V (sections 49–61) as proposed,
  non-implemented endpoints; durable job behaviour is now frozen by Part IV
  (sections 35–48). Its budget fields are configurable and bounded but their
  defaults remain provisional pending T10 measurement; OQ-21–OQ-30 are resolved
  in section 33, and the Part V API/DTO questions OQ-31–OQ-38 are resolved there
  as well.

---

# Part IV — T00d: durable scan-job lifecycle and resource-budget contract

Implements the **jobs** bullets of plan [section 3](./LIBRARY_ITEM_IDENTIFIER_PLAN.md) and T06
([plan section 5](./LIBRARY_ITEM_IDENTIFIER_PLAN.md)), turning proposal [§43](./LIBRARY_ITEM_IDENTIFIER.md)'s
progressive-scan levels and [§49](./LIBRARY_ITEM_IDENTIFIER.md)'s passive-scan constraints into rules. Adds
**no runtime code**, changes **no fingerprint encoding or matching rule**, and specifies **no** HTTP API/DTO shape,
UI, catalog ingestion, or sidecar parser.

- **In scope:** job states/transitions; stages and counters; levels; default budgets and concurrency/I/O limits;
  cancellation points; restart recovery; per-root locking; generations and snapshot binding; changing-file retries
  and incomplete outcomes; missing/inaccessible roots; notification versus durable state; the transition table;
  worked examples; invariants; question resolutions (section 33; section 48).
- **Out of scope (other owners):** HTTP routes/DTOs (T06 API surface), UI (T09), catalog ingestion (T11), sidecar
  parser (T07), fuzzy thresholds (T13), reserved fingerprint kinds (section 23), measured performance targets (T10).

### Compatibility with prior parts

- No Part I **D**/**INV**, Part II **F**/**INV**, or Part III **M**/**INV** is changed. `job_state` values are exactly
  M30.1's six; this part fixes only their transitions and persistence, satisfying M30.1's "out of scope here".
- Completeness/warnings/errors stay governed by section 19; cache modes by section 22; observation revisions by
  M30.9–M30.13. No job output bypasses the completeness gate (F19.2/INV-37) or compatibility gate (F21.2/INV-39).
  Part IV groups its invariants in section 47.

---

## 35. Durable job states and transitions

Plan section 3: "Persist stage, counters, limits, and restart behavior; Socket.IO is notification transport, not
durable truth."

### Decisions

- **J35.1 — States.** `queued`, `running`, `completed`, `completed-with-errors`, `failed`, `cancelled` (as M30.1).
  `cancelling` is not a state; stopping is the durable flag `cancel_requested` (J39.1). The last four are terminal; a
  rescan is a new job + generation (J42.1). Terminal job records, counters, and partial raw inventories are retained
  at least until superseded by a later generation or an explicit cleanup, and a revision referenced by a live job,
  decision, or candidate result is never collected (M30.9); the exact retention window and cleanup shape are T02
  (OQ-24, resolved T00f3).
- **J35.2 — Durable record.** `job_id`, root (D6.1), generation (J42.1), stage (J36.1), state, `cancel_requested`,
  attempts, lease (owner + timestamp), counters (J36.2), applied budget (J38.1), timestamps; each transition commits
  before its notification (J45.2). Storage is T02 (OQ-27).
- **J35.3 — Single writer.** One worker advances a job, via the per-root lock (J41.1) plus the lease (J40.1). State
  never encodes identity/content/accessibility (INV-58).
- **J35.4 — Transition table (normative).** Only these transitions are allowed:

  | From → To                           | Trigger                                                                 | Constraint              |
  | ----------------------------------- | ----------------------------------------------------------------------- | ----------------------- |
  | (none) → `queued`                   | scan accepted                                                           | lock held first (J41.1) |
  | `queued` → `running`                | lease acquired, `prepare`                                               | start + heartbeat       |
  | `queued` → `cancelled`              | cancel before start                                                     | no work                 |
  | `queued` → `failed`                 | root unresolved                                                         | J44.1                   |
  | `running` → `completed`             | stages done, discovery complete, no warnings or errors (`COMPLETE`)     | F19.1                   |
  | `running` → `completed-with-errors` | warnings/`COMPLETE_WITH_WARNINGS`/unstable/`INCOMPLETE`/adapter failure | J43.4                   |
  | `running` → `cancelled`             | cancel at a check point                                                 | J39.2                   |
  | `running` → `failed`                | fatal error, lost lease, restart                                        | J40.2                   |
  | terminal → any                      | —                                                                       | forbidden               |

---

## 36. Stages and progress counters

### Decisions

- **J36.1 — Stages.** `prepare` (root resolve + lock), `discover` (L0/1), `anchors` (L2, optional), `full-hash` (L3,
  optional), `resolve`, `persist`, `done` — durable and monotonic per job; optional stages may be skipped (J37.3).
  This list is the frozen M1 durable granularity; finer sub-stage, indeterminate-stage, and UI denominator detail is
  T02/T09 (OQ-30, resolved T00f3).
- **J36.2 — Counters.** Monotonic per job: `roots_done`, `items_discovered`, `entries_enumerated`, `entries_included`,
  `files_hashed`, `bytes_read`, `bytes_hashed`, `anchors_hashed`, `items_resolved`, `items_persisted`, `warnings`,
  `errors`. None decreases or crosses generations; terminal values persist (M31.1).
- **J36.3 — No fabricated percentages.** Progress is counters + `progress_known`; an unknown denominator makes a stage
  indeterminate and no percentage may be synthesized. Reaching a count is not `COMPLETE` (F19.1). The M1 contract
  exposes raw counters only; any derived percentage is a T09 presentation choice that must obey this rule.

---

## 37. Scan levels and progressive work

Plan section 3: levels 0–1 are default discovery; anchors and full hashing are explicit/configured background work.

### Decisions

- **J37.1 — Levels.** `level0-metadata` (names, sizes, counts, manifests, sidecar claims, native metadata);
  `level1-structural` (`qtree-v1` + count + total size, section 14; no content bytes; reserved `qstructure-minhash-v1`
  excluded); `level2-anchors` (`sha256` of a bounded, adapter-selected anchor set); `level3-full-hash` (`sha256` of
  every included file + `qmanifest-v1`, sections 15–16). All levels use bounded streaming buffers (J38.2), never load
  a whole file/tree into memory, and never execute content (proposal §49). The L2 anchor _set_ is adapter/profile-
  selected and bounded by `max_anchor_file_bytes`; M1 freezes no anchor set because L2 is off by default (J37.3), and
  the per-adapter selection is T03/T08 (OQ-22, resolved T00f3).
- **J37.2 — Level → artifact.** L1 yields the structural fingerprint only. Only a complete L3 run in `identity`/
  `verify` mode (F22.3) backs an exact Artifact (F19.2/INV-37); L2 alone never establishes exact identity (INV-68).
- **J37.3 — Default work.** A normal scan runs L0+L1 + resolution; L2 only on adapter request or configuration; L3 only
  on explicit verification, configured background/idle verification, or a DAT/catalog requirement. Applied levels are
  recorded on the job. A later level never invalidates a stored lower-level result (F21.5).

---

## 38. Default budgets, concurrency, and I/O limits

Proposal [§49](./LIBRARY_ITEM_IDENTIFIER.md); plan T06 (concurrency/I/O budgets); plan T03 (limits never masquerade as a complete scan).

### Decisions

- **J38.1 — Budget record.** Each job persists its applied budget (`max_depth`, `max_entries`, `max_anchor_file_bytes`,
  `max_workers`, `read_buffer_bytes`, `max_in_flight_bytes`, `max_attempts_per_entry`, `checkpoint_entries`,
  `lease_seconds`, `max_bytes_per_job`, stage toggles), so results stay interpretable after a config change. Every
  budget field is **external configuration**, not a compiled constant, and each has a documented **hard ceiling**
  that no configuration may exceed; the persisted applied budget is the validated snapshot used by the job (J35.2).
- **J38.2 — Defaults (provisional M1 values, configurable and bounded, not measurements — OQ-21/OQ-25).**

  | Budget                   | Default               | Effect                                                         |
  | ------------------------ | --------------------- | -------------------------------------------------------------- |
  | `max_depth`              | 32                    | recursion bound; exceed → `INCOMPLETE` (`limit-exceeded`)      |
  | `max_entries`/root       | 250 000               | bounds enumeration memory/time                                 |
  | `max_anchor_file_bytes`  | 64 MiB                | anchor hashing bounded; larger candidates skipped + warning    |
  | `max_bytes_per_job`      | 0 (unbounded)         | full verification has no a-priori size; bounded by workers/I/O |
  | `max_workers` (hash)     | 2                     | concurrent file hashes per job                                 |
  | `read_buffer_bytes`      | 1 MiB                 | streaming chunk; no whole-file reads                           |
  | `max_in_flight_bytes`    | 16 MiB                | concurrent read-buffer ceiling per job                         |
  | `max_attempts_per_entry` | 2 (initial + 1 retry) | changing-file retry bound (J43.2)                              |
  | `checkpoint_entries`     | 500                   | persist state/counters at least this often                     |
  | `lease_seconds`          | 30                    | heartbeat/lease window (J40.1)                                 |
  | stage toggles            | L0+L1 on; L2/L3 off   | default discovery (J37.3)                                      |

  These values are the frozen M1 defaults. They are **provisional pending T10 measurement** (plan T10: "do not invent
  NAS throughput targets") and may only be replaced by T10's calibrated, environment-stated figures; a per-request
  override is honoured only within the J38.5 ceilings and is recorded in the applied budget (J38.1).

- **J38.3 — Ceilings and yielding.** One `running` job per root (J41.1); at most `max_concurrent_roots` (provisional M1
  default 2, configurable and bounded ≥ 1) roots concurrently per process; within a job at most `max_workers` hashes
  and ≤ `max_in_flight_bytes` buffered reads (OQ-25, resolved T00f3). Hashing is preemptible (J39.2) and runs below
  interactive import/download work, so no scan blocks normal operation indefinitely (plan T10).
- **J38.4 — Limits visible.** Hitting a budget stops the affected work, records the limit, and marks the inventory
  `INCOMPLETE` (F19.3/INV-37/INV-69); a bounded run is never reported as complete.
- **J38.5 — Configurable, bounded, provisional (OQ-21).** Budgets come from configuration and validated per-request
  overrides, never from compiled constants. A value outside a field's hard ceiling is rejected (`400 invalid-request`
  at the API, P52.1), never silently clamped, and the ceiling can never be disabled or set to "unlimited". The J38.2
  defaults are the M1 values; T10 owns their calibration and environment-specific publication, and only T10 may change
  a frozen default (this document is amended in the same change, section 34). No budget value affects fingerprint
  bytes, matching, or completeness semantics (F19.3).

---

## 39. Cancellation points

Plan T06: "cancel/restart works during hashing".

### Decisions

- **J39.1 — Cooperative, durable stop.** A stop request persists `cancel_requested=true` before any effect; it is
  idempotent and never a state alone (J35.1).
- **J39.2 — Check points.** Evaluate `cancel_requested` at least: before each stage; before each directory enumeration;
  between included entries; between hash tasks; every bounded number of read chunks while hashing; before each persist
  batch; and before any stage transition. Work between checks is bounded by J38.2, so no gap exceeds one bounded read,
  enumeration, or persist batch (measured latency T10/OQ-21).
- **J39.3 — Cancelled outcome.** Unwind to `cancelled` after the current bounded unit; counters, warnings, and any
  partial raw inventory are retained as evidence, but the inventory is `INCOMPLETE` and no exact fingerprint is
  published or compared (INV-37/INV-70). Files are never deleted or mutated.
- **J39.4 — Persist batches are atomic (OQ-26).** A check point is evaluated before each persist batch; cancellation
  never exposes a partially-written batch. Whether an in-flight batch completes or is rolled back is a T02 storage
  atomicity decision, but the observable invariant is frozen here: no torn batch is visible to readers and no
  cancelled run publishes or compares an exact fingerprint (J39.3/INV-70).

---

## 40. Restart recovery

### Decisions

- **J40.1 — Lease/heartbeat.** A `running` job holds a lease (owner + timestamp) refreshed at least every
  `lease_seconds`; an unrefreshed lease expires. M1 freezes the provisional values `lease_seconds=30` and a heartbeat
  at least once per `lease_seconds` (J38.2); exact heartbeat interval, clock-skew handling, and stale-lock reclamation
  timing are T02/T06 (OQ-23, resolved T00f3).
- **J40.2 — Interrupted jobs fail.** On startup or lease expiry, a `queued`/`running` job with an expired lease becomes
  `failed` (`interrupted`); counters/warnings stay as evidence. It is never left `running` and never resumes mid-file
  (INV-71).
- **J40.3 — Recovery is a new generation.** Recovery enqueues a new job + generation (J42.1) rather than resuming in
  place, so a changed snapshot is never mixed with stale observations (M30.10). The new run may reuse cached content
  digests (section 22); a `verify` run still bypasses cache (F22.3/F22.5).
- **J40.4 — Bounded retry.** Automatic retry only within `max_job_attempts` (frozen M1 value 1: a single manual restart
  and no automatic retry loop); a lost-lease failure is never retried in place (J40.2), and exact retry policy is
  T02/T06 (OQ-23, resolved T00f3).

---

## 41. Per-root locking and duplicate requests

Plan T06: "duplicate requests do not duplicate items… per-root locking".

### Decisions

- **J41.1 — One active job per root.** At most one non-terminal job per root identity, durable across restarts. The
  durable store enforces this with a uniqueness constraint on root identity plus non-terminal state (T02; OQ-27,
  resolved T00f3), and the job/lease tables and generation allocation are T02 storage shape (INV-72). Locking is per
  root identity, so unrelated roots scan concurrently up to the global ceiling (J38.3).
- **J41.2 — Duplicate start is joined.** A start request for a root with an active job returns that job (+ counters);
  it never creates a second job or duplicates items. A request with different level/budget options must not mutate the
  running job's budget (J38.1); the frozen M1 response is reject with
  `409 scan-options-mismatch` (P52.3), and finer queue/ignore/no-op policy is T06
  (`OQ-34`, resolved T00f4, extends the resolved OQ-29). Options that match the
  applied budget join the job.
- **J41.3 — Release and reclaim.** The lock releases exactly with the terminal transition (J35.1); a lock held by an
  expired lease (J40.1) is reclaimable only after the prior job is transitioned per J40.2.
- **J41.4 — Process scope (OQ-28).** M1 scans from a single process per database. Because the per-root lock and the
  OQ-27 uniqueness constraint are durable, a duplicate start cannot create a second non-terminal job for a root even
  if issued by another process; M1 nevertheless makes **no** promise about distributed failover, leadership election,
  or safe simultaneous multi-process scanning of one root. Those coordination semantics are deferred to T02/T06 (OQ-28,
  resolved T00f3) and must be added without weakening INV-72.

---

## 42. Scan generation and snapshot binding

Plan section 3/T06: snapshot/revision tokens prevent stale job results or decisions replacing newer observations.

### Decisions

- **J42.1 — Generation.** Each accepted scan start allocates a `generation`, monotonic per root identity; every job,
  observation, fingerprint, candidate set, and result records it, bound to the observation revision(s) current at
  start (M30.9) and therefore revision-scoped (M30.10).
- **J42.2 — Stale writes rejected.** A job commits only for its bound generation/revision; a write older than the
  target's current observation is rejected or reconciled, never applied (INV-61). The store enforces this
  transactionally (T02), and a stale generation can never resurrect a superseded generation's observations,
  fingerprints, or decisions (P57.3).
- **J42.3 — Missing-marking gate.** Only a `COMPLETE`/`COMPLETE_WITH_WARNINGS` discovery pass (F19.1) of the current
  generation may mark prior locations `MISSING` (D8.5); all other passes never mark anything missing (INV-73/INV-75).
- **J42.4 — Manual decisions outrank scans.** A manual association (M30.4/M30.5) is not overwritten; changed
  authoritative evidence creates `CONFLICT` (M30.6/INV-60).
- **J42.5 — Superseding scans.** A new generation while a job runs is serialized by the per-root lock (J41); it
  proceeds once the prior job is terminal and supersedes older uncommitted work.

---

## 43. Changing files, retries, and incomplete outcomes

Plan section 3: detect files changing during hashing and retry boundedly or mark unstable (sections 18–19).

### Decisions

- **J43.1 — Change detection.** A content hash is valid only if size and `mtime_ns` are stable across the read, the
  kind did not change, and the file did not disappear; otherwise the entry is _unstable_. An entry that disappears
  between enumeration and hashing is treated as unreadable/changed (section 18), recorded, non-fatal, and contributing
  to `INCOMPLETE`; the scan continues.
- **J43.2 — Bounded retry.** An unstable included entry is retried up to `max_attempts_per_entry` (default 2, J38.2)
  with short backoff; cache reuse is not permitted for an unstable entry (F22.6; J43.3).
- **J43.3 — Unstable ⇒ incomplete.** If still unstable, the entry is recorded in warnings/errors and the inventory
  becomes `INCOMPLETE` (F19.1); no complete exact fingerprint is published and no Artifact association is made
  (INV-37/INV-74). No retry is ever unbounded (`max_attempts_per_entry` and job budgets bound every retry).
- **J43.4 — Job outcome.** Finishing with warnings, unstable/unreadable entries, or an `INCOMPLETE` inventory
  terminates `completed-with-errors` (never bare `completed`); a fatal error terminates `failed`. Both retain evidence.

---

## 44. Missing and inaccessible roots

Plan T06: disabled/inaccessible roots degrade clearly; a missing root never marks all its items deleted (D8.5/M30.1).

### Decisions

- **J44.1 — Pre-scan root check.** Before `discover`, resolve the root and verify it exists and is readable. Unresolved
  → `ROOT_UNRESOLVED`, job `failed`; present-but-unreadable → `INACCESSIBLE`, job `completed-with-errors`. No discovery
  runs in either case.
- **J44.2 — No mass-missing inference.** A failed, cancelled, `INCOMPLETE`, or inaccessible-root pass never marks prior
  locations missing; prior items keep their last-known accessibility and content state (INV-75). A partial or
  inaccessible observation may still be retained as revision-scoped evidence (F19.6) and may be `PARTIALLY_ACCESSIBLE`
  (M30.14), but it is never comparable and never changes missing state (F19.2/INV-37).
- **J44.3 — Disabled and returning roots.** A configured-disabled root is skipped without missing-marking, and the
  omission is reported. When an available root returns, the next `COMPLETE` generation re-observes it; reappearing
  locations update from actual observation, never inference.
- **J44.4 — Metadata outage is independent.** An IGDB/catalog outage does not fail local discovery or hashing; local
  inventory, fingerprints, and accessibility are recorded, and resolution may stay `PROBABLE`/`UNKNOWN` (T06
  acceptance; Part III).
- **J44.5 — Read-only scanning.** Scanning never executes discovered content, never follows symlinks outside the root,
  ignores device/socket/pipe entries, and never mutates or deletes files even when marking locations missing (proposal
  §49).

---

## 45. Notification versus durable state

Plan section 3: "Socket.IO is notification transport, not durable truth."

### Decisions

- **J45.1 — Transport is advisory.** Events are hints; no transition, counter, or result depends on delivery, ordering,
  or receipt.
- **J45.2 — Durable-first.** State/counter commits precede their notification (INV-65); a notification always reflects
  already-durable state.
- **J45.3 — Recover by polling; idempotent consumers.** A client that missed events resumes from the durable job record
  (state, stage, counters, warnings, errors); events need not be replayed, may duplicate or arrive out of order, and
  consumers reconcile by counters/generation, ignoring anything that moves state backwards (J36.2, J42.2).
- **J45.4 — No secrets in events.** Notifications carry counters/state/identifiers only; root paths and hashes follow
  the durable records' access policy (section 6; D6.6).

---

## 46. Failure and recovery examples

Illustrative; sections 35–45 are normative where they differ.

| #   | Event                                      | Durable outcome                                                                                            | Recovery                                              |
| --- | ------------------------------------------ | ---------------------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| E1  | Cancel during `full-hash`                  | `cancel_requested` → `cancelled`; counters/raw inventory kept; `INCOMPLETE`; no Artifact (J39.3)           | New generation; cache may reuse digests (J40.3)       |
| E2  | Crash mid-`discover`; restart              | Expired lease → `failed` (`interrupted`); no job `running` (J40.2)                                         | New job/generation; stale writes blocked (J42.2)      |
| E3  | Second scan request, same root             | Request joins active job; no duplicate items (J41.2)                                                       | Client polls existing job (J45.3)                     |
| E4  | Bytes change during hashing                | Bounded retry then `unstable`; `INCOMPLETE`; `completed-with-errors` (J43.3/J43.4)                         | Later scan re-observes; no exact fingerprint (INV-74) |
| E5  | Root unmounted mid-scan                    | Root `INACCESSIBLE`; job `completed-with-errors`/`failed`; prior items keep state; nothing missing (J44.1) | Complete generation when root returns (J44.3)         |
| E6  | Older-generation result after newer commit | Older write rejected/reconciled; newest observation preserved (J42.2/INV-73)                               | Older result discarded/recomputed (M30.10)            |

---

## 47. Part IV invariants (INV-65–INV-76)

- **INV-65** A notification never precedes the durable transition it reports.
- **INV-66** `job_state` is one of the six frozen values; cancellation is `cancel_requested` until commit.
- **INV-67** Counters are per-job and monotonic; none is shared across jobs or generations.
- **INV-68** L2 anchor evidence never substitutes for an L3 exact fingerprint; only a complete L3 run backs an Artifact.
- **INV-69** A budget-limited run records the limiting budget and yields `INCOMPLETE`/`completed-with-errors`; no complete exact fingerprint. Budgets are configurable and bounded, and this holds for every applied budget (J38.4/J38.5).
- **INV-70** A cancelled job publishes no complete exact fingerprint and associates no Artifact; files stay untouched, and no partially-written persist batch is observable (J39.2–J39.4).
- **INV-71** No restart leaves a job falsely `running`; an interrupted job is durably terminal (never applies stale results, INV-61).
- **INV-72** No two non-terminal jobs share a root identity; a duplicate request cannot start a second concurrent scan. The guarantee is durable, not in-memory (J41.1/J41.4).
- **INV-73** No result older than the current observation may overwrite newer state or mark locations missing.
- **INV-74** An unstable or changing included entry never yields a complete exact fingerprint or Artifact association.
- **INV-75** A missing/inaccessible/disabled root never causes mass missing-marking or deletion; only a complete current-generation pass may change missing state (INV-73).
- **INV-76** Losing all notifications loses no durable state; the job record alone reconstructs status.

---

## 48. Open issues (Part IV)

All Part IV scan-job/resource questions **OQ-21–OQ-30 were closed by T00f3**
(section 33), each as either a concrete M1 rule or an explicit named-milestone
deferral with a frozen safe M1 behavior. OQ-29 is the only question settled
without a deferral (J41.2/P52.2/P52.3), while OQ-21's frozen M1 behavior is the
configurable/bounded budget rule (J38.5); the rest are deferred to their named
owners with bounded,
configurable, provisional M1 behavior (budgets and concurrency to T10; storage,
retention, cancellation atomicity, and locking shape to T02; multi-process scope
to T02/T06; anchors to T03/T08; stages/denominators to T02/T09). No Part IV
question remains open in this contract; any later change to a Part IV **J**
decision or **INV-65–INV-76** is governed by the change-control rules (section
34). See section 33 for the per-question rationale.

---

# Part V — T00e: HTTP API and DTO contract

Implements the T00 requirement (plan [section 5 "T00"](./LIBRARY_ITEM_IDENTIFIER_PLAN.md)) to
"Specify library item listing, scan start/status/cancel, evidence/candidates, manual association,
verification, and catalog APIs" and to "Distinguish existing endpoints retained from proposed
endpoints", plus the plan's "DTOs/error semantics" deliverable. Adds **no runtime code**, changes
**no fingerprint/M/J rule**, and specifies **no** UI, catalog _ingestion_ mechanics, sidecar parser,
or matching algorithm. All routes are **proposed** (T06) unless the compatibility map (section 59)
marks them retained.

- **In scope:** mount/auth conventions; listing/detail; scan start/status/cancel; evidence and
  candidates; manual association/correction; verification; catalog endpoints; request/response DTO
  fields; pagination; revision/`If-Match` stale-write behaviour; error codes; retained-versus-new
  map; invariants; question resolutions (section 33; section 61).
- **Out of scope (other owners):** durable job persistence (T02/Part IV), candidate/threshold
  algorithms (T05/T13/Part III), catalog ingestion (T11), sidecar parser (T07), UI (T09).

### Compatibility with prior parts

- No Part I **D**/**INV**, Part II **F**/**INV**, Part III **M**/**INV**, or Part IV **J**/**INV**
  is changed. Part V serializes frozen states (M30.1), evidence fields (M26.3), the durable job
  record and counters (J35.2/J36.2), generations/revisions (M30.9, J42.1–J42.2), and cancellation
  (J39.1) — it does not redefine them.
- The error envelope in P49.4 **extends** the existing `{ error, details? }` shape with a
  machine-readable `code`; retained endpoints are unchanged (section 59; INV-83/INV-84).

---

## 49. API surface and conventions

- **P49.1 — Mounting.** New routes mount under the existing global `basePath`
  (`config.server.basePath`; `server/routes.ts`): `<basePath>/api/library-items/…` and
  `<basePath>/api/catalogs/…`, in a dedicated route module (plan section 2) registered alongside
  the existing `/api/imports` and `/api/import-tasks` mounts.
- **P49.2 — Authentication.** JWT via the existing default-deny gate
  `requireAuthenticationForApi`/`authenticateToken`. Long-lived API keys stay confined to
  `/api/integration/*`; exposing library items to the extension is **not added in M1** (OQ-31,
  resolved T00f4), and that surface stays as frozen in section 59/D7.4–D7.5.
- **P49.3 — Content type.** JSON only; no HTML; all responses set `Cache-Control: no-store` (per-user/root content).
- **P49.4 — Error envelope.** `{ "error": string, "code": string, "details"?: unknown }`; Zod
  failures reuse the existing `respondWithZodError` `details` array (400 `invalid-request`). `204`
  for empty success, `202` for accepted asynchronous work.
- **P49.5 — Rate limiting.** Mutating endpoints reuse `sensitiveEndpointLimiter`; its `429` is
  reported as `{ error, code: "rate-limited" }` (retained limiter, new body field).
- **P49.6 — No path/hash leakage.** Fields derived from a filesystem path or fingerprint follow the
  section 6/J45.4 access policy; the exact redaction projection is P49.7 (OQ-33, resolved T00f4).
- **P49.7 — Redaction projection (OQ-33, resolved T00f4).** For a visible item, a caller is
  **entitled** to its location/hash material iff it owns the associated Game, owns the item's
  `LIBRARY_ROOT` (D6.1), or the item's root is a global, owner-less `ROOT_FOLDER`
  (D6.2/INV-12). An entitled caller receives the full projection. A non-entitled caller receives:
  `relative_path` = `null`; symlink targets reduced to F17.2's non-revealing classification;
  `fingerprint_ref`, fingerprint digest values, and `baseline` suppressed; `etag` omitted.
  Retained for both: `id`, `root` (kind + opaque id), the M30.1 state fields,
  `observation_revision`, `has_conflict`, `evidence_count`, `last_observed_at`, and evidence
  `class`/`type`/`source`/`local`/`revision`/`observed_at`. `display_name` is retained only when it
  is not derived from the item's path; a path-derived label is replaced by a non-path placeholder.
  A `game` reference is present only when the caller owns it, otherwise `null` (D6.6). No field ever
  contains an absolute path (D7.1/INV-16).

## 50. Common DTO fields

- **P50.1 — IDs.** All IDs are opaque non-empty strings (D4.1); never numbers (INV-1).
- **P50.2 — State fields.** `identity_state`, `content_state`, `accessibility_state`, `job_state`
  carry exactly the M30.1 value sets and are independent (INV-58).
- **P50.3 — Revisions.** `revision` is the per-Installation observation revision (M30.9);
  `generation` is the per-root scan generation (J42.1). Both are integers, not timestamps.
- **P50.4 — Root reference.** `RootRef = { kind: "LIBRARY_ROOT" | "ROOT_FOLDER", id: string }`
  (D6.1); never a client-supplied absolute path (INV-16).
- **P50.5 — Nullable associations.** `game`, `release_id`, `build_id`, `artifact_id` are always
  present and may be `null` (D4.2/D4.3).
- **P50.6 — Scalars.** Timestamps are ISO-8601 UTC strings; sizes/counters/bytes are integers.

## 51. Listing and detail

- **P51.1 — `GET /api/library-items`.** Query: `q`, `identity_state`, `content_state`,
  `accessibility_state`, `associated` (`true`/`false`), `game_id`, `root_kind`, `root_id`,
  `has_conflict`, `limit`, `offset`, `sort`.
- **P51.2 — Pagination.** `limit` default `20`, clamped `1..100`; `offset` `>= 0`, default `0`
  (matches `validatePaginationParams`). Envelope:
  `{ items: LibraryItemSummary[], total, limit, offset }`. Every list endpoint returns a stable
  deterministic total order, so offset paging is deterministic within a snapshot; M1 accepts no
  `cursor` parameter. Cursor paging for very large histories is deferred to T06 (OQ-32, resolved
  T00f4).
- **P51.3 — `LibraryItemSummary`:** `id` (Installation ID, D3.1); `display_name`; `root` (`RootRef`,
  D6.1); `relative_path` (string \| null; root-relative normalized, D7.2; redaction P49.7/OQ-33);
  `game` (`{ id, title }` \| null; `null` = unassociated D4.3; a foreign Game is never disclosed,
  D6.6/INV-15); `release_id`/`build_id`/`artifact_id` (string \| null, D4.2–D4.4);
  `identity_state`/`content_state`/`accessibility_state` (M30.1, independent); `observation_revision`
  (M30.9); `has_conflict` (section 29); `evidence_count`; `last_observed_at` (ISO-8601 or null).
- **P51.4 — Visibility.** An item associated with another user's Game is omitted from lists and
  returns `404` on detail; unassociated items are shared discovery data (D6.4/D6.6, INV-15).
- **P51.5 — `GET /api/library-items/:id`.** Returns `LibraryItemDetail`: the summary plus `target`
  (`{ game_id?, release_id?, build_id? }`), `baseline`, `conflicts[]`, `reasons[]`, `revision`, and
  `etag`, mirroring the illustrative M31.2 result. Evidence/candidates may be embedded up to a small
  default; full lists use P53. A non-entitled caller receives the P49.7 redacted projection, with
  `etag` omitted. Unknown/not-visible id → `404 not-found`, never a body that leaks
  existence.

## 52. Scan start, status, and cancel

- **P52.1 — `POST /api/library-items/scan`.** Body `ScanStartRequest`:
  `{ root: RootRef, levels?: Level[], budgets?: Partial<AppliedBudget> }`, where `Level` is one of
  `level0-metadata`/`level1-structural`/`level2-anchors`/`level3-full-hash` (J37.1). Omitted
  `levels` uses the J37.3 default; overrides are recorded in the job's `applied_budget` (J38.1).
- **P52.2 — `202` `JobDto`.** A start on a root with an active non-terminal job **joins** that job
  (`joined: true`) and never creates a second job (J41.2/INV-72). A start whose root id does not
  resolve at all → `404 root-not-found`.
- **P52.3 — Options mismatch.** Levels/budgets differing from the active job's applied budget →
  `409 scan-options-mismatch`; the running budget is never mutated (J38.1). M1 adds **no queue**: a
  differing request is never queued, parked, or silently ignored; finer queue/priority policy is T06
  (OQ-34, resolved T00f4; extends OQ-29).
- **P52.4 — Root outcomes are job state.** A resolvable root that is missing/unreadable is accepted
  and surfaces J44.1 outcomes (`failed`/`completed-with-errors`, `accessibility_state`
  `ROOT_UNRESOLVED`/`INACCESSIBLE`); no discovery runs.
- **P52.5 — `GET /api/library-items/scan`** lists jobs (paginated, P51.2, newest generation first);
  **`GET /api/library-items/scan/:jobId`** returns one `JobDto`.
- **P52.6 — `JobDto`:** `job_id`; `kind` (`"scan"`/`"verify"`, P55); `root` (`RootRef`, J35.2);
  `generation` (J42.1); `state` (M30.1/J35.1); `stage` (J36.1); `cancel_requested` (J39.1);
  `counters` (J36.2 keys, monotonic); `progress_known` (J36.3; `false` = indeterminate);
  `warnings_summary`/`errors_summary` (`{ count, sample[] }`, bounded, no root paths — J45.4);
  `applied_budget` (J38.1); `created_at`/`started_at`/`finished_at` (ISO-8601 or null);
  `links` (`{ self, item? }`, optional).
- **P52.7 — `POST /api/library-items/scan/:jobId/cancel`.** Sets durable `cancel_requested=true`
  and returns `202 { job_id, cancel_requested: true, state }`; idempotent, no state alone
  (J39.1/J35.1). A terminal job → `409 job-terminal`. Never deletes or mutates files (J39.3).
- **P52.8 — Notifications.** Retain the legacy `library-scan-progress` event; add the advisory
  `library-identification:job` event carrying `JobDto`. Durable state commits before the event
  (J45.2); consumers reconcile by counters/generation and reconstruct status by polling P52.5
  (J45.3/INV-76).
- **P52.9 — Event namespace and versioning (OQ-38, resolved T00f4).** Both events use the existing
  default Socket.IO namespace and `${basePath}/socket.io/` path; M1 adds no namespace. The event
  name is the compatibility boundary, and the payload is exactly the P52.6 `JobDto` (same field
  names/values as J35.2/J36.2, no root paths/hashes — J45.4). `generation` (J42.1) plus monotonic
  counters (J36.2) are the reconciliation keys (J45.3); consumers ignore unknown fields and never
  treat an event as authoritative. M1 changes are additive only; a breaking payload change needs a
  new event name and is T06. This contract owns the payload field schema; T06 owns the emitter.

## 53. Evidence and candidates

- **P53.1 — `GET /api/library-items/:id/evidence`.** Paginated. `EvidenceDto` fields (M26.3):
  `class` (M26.1 set), `type`, `namespace`?, `value`?/`fingerprint_ref`?, `source`, `local`
  (boolean), `revision`, `target`?, `observed_at`. Evidence is immutable per observation (M26.4);
  no endpoint rewrites history.
- **P53.2 — `GET /api/library-items/:id/candidates`.** Returns
  `{ candidates: CandidateDto[], resolution: ResolutionDto }`. `CandidateDto`: `candidate_id`,
  `target` (Game/Release/Build refs), `trusted_mapping` (M25.7), `label` (M31.3 row),
  `evidence_refs[]`, `rank_key`, `conflicts_with[]`. A candidate is a stateless hypothesis; rank or
  order is never identity (M25.1/M25.3/INV-44).
- **P53.3 — Recompute by default.** M1 recomputes candidates on demand for the current revision
  (M25.8); serving a later persisted/cached set with ETags is deferred to T05/T06 (OQ-35, resolved
  T00f4).
- **P53.4 — `POST /api/library-items/:id/resolve`.** Runs Part III resolution at
  `expected_revision` and returns `ResolutionDto { identity_state, content_state, target, baseline,
conflicts, reasons }` (M31.2). It writes nothing unless an automatic association is permitted
  (INV-42/INV-43), so it is retry-safe; stale revision → `412 stale-revision` (section 57).
- **P53.5 — No candidate ETag in M1 (OQ-35, resolved T00f4).** Because M1 recomputes candidates
  (P53.3/M25.8), candidate responses carry no `ETag`/`If-None-Match` and no durable candidate id;
  the response exposes the `observation_revision` it was computed at so a client can detect
  staleness. A persisted candidate cache with validators is later (T05/T06), revision-scoped, and
  never authoritative (M25.1/M25.3/INV-44).

## 54. Manual association and correction

- **P54.1 — `POST /api/library-items/:id/decisions`.** Body `DecisionRequest`:
  `{ kind: "confirm"|"set"|"reject"|"unlink"|"resolve-conflict", target?: { game_id, release_id?, build_id? }, expected_revision, note? }`
  (M30.4/M30.7). `reject`/`unlink` clear associations in the same transaction (INV-6/D8.2).
- **P54.2 — Ownership.** `set`/`confirm` require the acting user to own the target Game
  (D6.5/INV-14) → `403 forbidden`; no cross-user association or leakage (D6.6).
- **P54.3 — Response.** `200 { decision: DecisionDto, item: LibraryItemDetail }`. `DecisionDto`:
  `decision_id`, `kind`, `target`, `actor` (`{ id, username }`), `created_at`, `evidence_refs[]`,
  `revision`.
- **P54.4 — Durability.** Decisions survive rescans (M30.5/INV-60); contradicting trusted evidence
  sets the item `CONFLICT` and retains the decision (M30.6/INV-60); the API never silently overwrites
  a user choice.
- **P54.5 — `GET /api/library-items/:id/decisions`** returns paginated decision history (retention
  shape OQ-15, deferred to T01/T02 in T00f2).
- **P54.6 — Preconditions.** Missing `If-Match`/`expected_revision` → `428 revision-required`;
  stale → `412 stale-revision` (section 57). Decisions record the revision they were made against
  (M30.10).

## 55. Verification

- **P55.1 — `POST /api/library-items/:id/verify`.** Body
  `{ scope: "identity"|"content"|"full", expected_revision }` → `202 JobDto` (`kind: "verify"`).
  Verification runs L3 and **bypasses cache** (J37.2/F22.3/F22.5); `full` re-hashes every included
  file.
- **P55.2 — Outcomes.** An unstable/incomplete/limit-bounded run publishes no complete exact
  fingerprint and yields content `UNVERIFIED`/`DIFFERS_FROM_KNOWN` only with a compatible baseline
  and interpretable evidence (M27.7/INV-59/INV-74); it never fabricates `IDENTIFIED_MODIFIED`.
- **P55.3 — Read-only.** Verification never executes content, follows symlinks outside the root, or
  mutates/deletes files (J44.5); results are durable-first and surfaced on the item detail after
  commit, with the job event advisory (P52.8).

## 56. Catalog endpoints

- **P56.1 — Catalog CRUD (shape frozen here; ingestion is T11).**

  | Method & path                   | Body / query               | Response                                        |
  | ------------------------------- | -------------------------- | ----------------------------------------------- |
  | `GET /api/catalogs`             | pagination                 | `{ items: CatalogDto[], total, limit, offset }` |
  | `POST /api/catalogs`            | `CatalogCreateRequest`     | `201 CatalogDto`                                |
  | `GET /api/catalogs/:id`         | —                          | `CatalogDto`                                    |
  | `DELETE /api/catalogs/:id`      | —                          | `204`; never touches library files (D8.1)       |
  | `GET /api/catalogs/:id/entries` | pagination, `media_scope?` | `{ items: CatalogEntryDto[], … }`               |

  `CatalogDto`: `id`, `namespace`, `name`, `source_type` (`"user-file"`), `source_version`,
  `source_checksum`, `imported_at`, `entry_count`, `status`
  (`"importing"|"ready"|"failed"`), `error`. `CatalogEntryDto`: `entry_key`, `media_scope`, `title`,
  `checksums[]` (`{ algorithm, value }`), `source_version`.

- **P56.2 — User-provided only.** No automatic remote fetch/upload/redistribution in this slice (M26.7); remote-fetch policy is T11 (OQ-36, resolved T00f4).
- **P56.3 — Matching provenance.** Catalog matches are EXACT only with recorded source/version/
  checksum provenance (M27.11); the API surfaces the catalog/source keys used on the evidence and
  baseline.
- **P56.4 — Deletion.** Removing a catalog row never deletes library items, artifacts, or existing
  catalog-match evidence; whether delete is blocked or allowed while catalog matches depend on it,
  and retention of superseded provenance, are T11 (OQ-36, resolved T00f4).
- **P56.5 — M1 ingestion scope (OQ-36, resolved T00f4).** M1 freezes only the P56.1 envelope and
  `source_type` `"user-file"`; it adds no upload/import endpoint, no remote fetch, no
  redistribution, and no payload execution (P56.2/INV-86), and performs no catalog matching
  (OQ-18). Accepted source formats, upload limits, remote-fetch policy, delete-while-dependent
  behavior, and superseded-provenance retention are T11; catalog deletion never deletes library
  items, artifacts, or catalog-match evidence (P56.4/D8.1).

## 57. Revision, `If-Match`, and stale-write semantics

- **P57.1 — Revisions and ETag.** `LibraryItemDetail` exposes `revision` and a strong `ETag`
  encoding item plus revision (opaque; exact format T02).
- **P57.2 — Carrier.** Mutating item endpoints accept `If-Match: <etag>` or body
  `expected_revision`, which must agree if both are present (`400 invalid-request` otherwise);
  `If-Match: *` is not accepted.
- **P57.3 — Stale write is rejected.** A revision/ETag older than the item's current observation →
  `412 stale-revision` carrying the current revision; the write is rejected or explicitly
  reconciled, never merged silently (M30.10/J42.2/INV-61). This is the API face of the internal
  store guard (T02).
- **P57.4 — Missing precondition.** No `If-Match` and no `expected_revision` → `428 revision-required` for mutating item/decision/verify/resolve requests.
- **P57.5 — Exempt operations.** Scan start/cancel are not revision-gated: the server allocates the
  generation (J42.1) and cancel is an idempotent durable flag (J39.1); both still honour root access
  policy (section 6) and per-root locking (J41.1).
- **P57.6 — No bulk writes or idempotency keys in M1 (OQ-37, resolved T00f4).** M1 mutating item
  endpoints are single-item and revision-bound (P57.2–P57.4); a retried request is rejected as
  stale rather than applied twice. M1 exposes no bulk decision/rescan endpoint and accepts no
  idempotency-key header. Bulk operations and explicit idempotency/replay storage are T06/T09 and
  must preserve per-item revision binding and ownership (`403` per P54.2), return per-item results,
  and never bypass INV-61/INV-79.

## 58. Error codes

| `code`                                   | HTTP | Meaning                                                               |
| ---------------------------------------- | ---- | --------------------------------------------------------------------- |
| `invalid-request`                        | 400  | Zod/body/query validation failure (`details`).                        |
| `unauthorized`                           | 401  | Missing/expired JWT.                                                  |
| `forbidden`                              | 403  | Action not permitted (e.g. associating a Game the user does not own). |
| `not-found` / `root-not-found`           | 404  | Unknown or not-visible item/decision/entry; root id does not resolve. |
| `job-terminal` / `scan-options-mismatch` | 409  | Terminal-job cancel; start options differ from the active job.        |
| `stale-revision`                         | 412  | Precondition older than current observation (P57.3/INV-61).           |
| `payload-too-large`                      | 413  | Request exceeds a bound.                                              |
| `revision-required`                      | 428  | Missing `If-Match`/`expected_revision`.                               |
| `rate-limited`                           | 429  | Existing limiter.                                                     |
| `catalog-import-failed`                  | 422  | Catalog source rejected/invalid.                                      |
| `internal`                               | 500  | Unexpected server error.                                              |

Retained endpoints keep `{ error, details? }` and are **not** changed to add `code` in this slice
(INV-84); `code` belongs to the new envelope (P49.4). No T00f4 resolution (section 33) adds a new
`code`: OQ-34 reuses `409 scan-options-mismatch`, and OQ-37 reuses `412 stale-revision`/
`428 revision-required`.

## 59. Compatibility map

**Retained endpoints (behaviour frozen; no shape change in this slice).** The identification model
is additive (D7.4); these keep working behind the scanner facade (plan section 2).

| Method & path                                                                                                                                        | Current shape                    | Contract                                                              |
| ---------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------- | --------------------------------------------------------------------- |
| `POST /api/library/scan`                                                                                                                             | `202 { accepted, rootFolderId }` | Legacy immediate-child + IGDB name flow; unchanged.                   |
| `GET /api/library/scan/status`                                                                                                                       | `ScanProgress[]`                 | In-memory progress; unchanged.                                        |
| `GET /api/library/scan/unmatched`                                                                                                                    | `UnmatchedEntry[]`               | Unchanged.                                                            |
| `POST /api/library/scan/unmatched/match`                                                                                                             | match result                     | Unchanged; still owner-checked.                                       |
| `GET/POST/PATCH/DELETE /api/root-folders`, `POST /api/root-folders/:id/health-check`                                                                 | as today                         | D6/D8 policy unchanged (INV-12).                                      |
| `GET/POST/DELETE /api/game-files`, `GET /api/game-files/by-download/:downloadId`                                                                     | as today                         | D7.3/D7.4 projection unchanged.                                       |
| `GET /api/games`, `GET /api/games/:id`, `POST /api/games/match-and-add`, `POST /api/games/library-health-check`, `DELETE /api/games/:id?deleteFiles` | as today                         | D7.4/D8.6; deletion containment unchanged.                            |
| `/api/imports/*`, `/api/import-tasks/*`                                                                                                              | as today                         | D6.7 provenance unchanged.                                            |
| `/api/integration/*`                                                                                                                                 | as today                         | `libraryPath` projection only (D7.4/D7.5); API-key surface unchanged. |
| Socket `library-scan-progress`                                                                                                                       | `ScanProgress`                   | Advisory; unchanged (P52.8).                                          |

**Proposed new endpoints (T06; not implemented by this contract).**

| Method & path                                                                             | Purpose                              |
| ----------------------------------------------------------------------------------------- | ------------------------------------ |
| `GET /api/library-items`                                                                  | Paginated listing (P51).             |
| `GET /api/library-items/:id`                                                              | Detail (P51).                        |
| `POST /api/library-items/scan`                                                            | Start/join scan (P52).               |
| `GET /api/library-items/scan`                                                             | List jobs (P52).                     |
| `GET /api/library-items/scan/:jobId`                                                      | Job status (P52).                    |
| `POST /api/library-items/scan/:jobId/cancel`                                              | Idempotent cancel (P52).             |
| `GET /api/library-items/:id/evidence`                                                     | Evidence list (P53).                 |
| `GET /api/library-items/:id/candidates`                                                   | Candidates + resolution (P53).       |
| `POST /api/library-items/:id/resolve`                                                     | Recompute resolution (P53).          |
| `POST /api/library-items/:id/decisions`                                                   | Manual association/correction (P54). |
| `GET /api/library-items/:id/decisions`                                                    | Decision history (P54).              |
| `POST /api/library-items/:id/verify`                                                      | Verification job (P55).              |
| `GET/POST /api/catalogs`, `GET/DELETE /api/catalogs/:id`, `GET /api/catalogs/:id/entries` | Catalog API (P56, T11).              |
| Socket `library-identification:job`                                                       | Advisory job event (P52.8).          |

The route names and mount points above are frozen by this contract (plan section 2: T00 chooses exact names and route mounting); T06 registers them consistent with repository conventions.

The T00f4 resolutions (section 33) add **no** route and change no retained shape: OQ-31 keeps
`/api/integration/*` unchanged (P49.2), and the proposed table above is unchanged (INV-83).

## 60. Part V invariants (INV-77–INV-88)

- **INV-77** New endpoints do not weaken root authorization or ownership (INV-12–INV-16).
- **INV-78** No endpoint discloses another user's Game identity through an item (INV-15/D6.6).
- **INV-79** Every mutating item write is revision-bound; no stale write is applied (INV-61).
- **INV-80** A scan start never creates a second non-terminal job for a root identity (INV-72).
- **INV-81** Cancel is idempotent and never deletes or mutates files (INV-70/J39.3).
- **INV-82** Evidence/candidate reads and resolution never write an association unless Part III
  permits it (INV-42/INV-43).
- **INV-83** Retained endpoints keep their frozen response shapes in this slice.
- **INV-84** New endpoints carry a machine-readable `code`; retained endpoints are unchanged.
- **INV-85** Notification events are advisory; polling the durable job record reconstructs status
  (INV-76/J45.3).
- **INV-86** Catalog endpoints never upload, redistribute, or execute catalog payloads (M26.7/J44.5).
- **INV-87** A caller not entitled to an item's location/hash material receives no root-relative
  path, symlink target, or fingerprint digest for it, and a foreign Game is never disclosed
  (P49.7/OQ-33; INV-15/INV-78).
- **INV-88** M1 serves only recomputed candidates (no candidate ETag) and exposes no bulk write that
  bypasses revision binding (P53.5/P57.6/OQ-35/OQ-37; INV-61/INV-79).

## 61. Open issues (Part V)

All Part V API/DTO questions **OQ-31–OQ-38 were closed by T00f4** (section 33),
each as either a concrete M1 rule or an explicit named-milestone deferral with a
frozen safe M1 behavior. OQ-31 (no integration exposure, P49.2), OQ-33 (redaction
projection, P49.7), OQ-34 (join-or-reject, no queue, P52.3), OQ-35 (recompute, no
candidate ETag, P53.5), and OQ-38 (notification contract, P52.9) are concrete M1
rules; OQ-32 (cursor paging) is deferred to T06, OQ-36 (catalog ingestion) to
T11, and OQ-37 (bulk/idempotency) to T06/T09, each with a frozen safe M1
behavior. No Part V question remains open in this contract; any later change to a
Part V **P** decision or **INV-77–INV-88** is governed by the change-control
rules (section 34). See section 33 for the per-question rationale. The
route-compatibility map (section 59) and the error-code table (section 58) are
unchanged by these resolutions.
