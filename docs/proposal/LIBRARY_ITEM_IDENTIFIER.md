# Questarr Game Identification and Fingerprinting Proposal

## 1. Purpose

Questarr needs a reliable way to identify games discovered through:

- Questarr-managed downloads and post-processing
- Existing local game libraries
- PC game installations
- ROM collections
- Disc images
- Console game dumps/packages
- Multi-file games
- Modified or modded installations
- Multiple releases, revisions, regions, editions, and storefront versions of the same game

The identification system must answer several different questions that should **not** be collapsed into one hash:

1. **What game is this?**
2. **What platform/release is this?**
3. **What revision/build is this?**
4. **Are these exact same bytes?**
5. **Is this installation a modified version of a known game?**
6. **Have I already imported this item somewhere else?**

The proposed system therefore uses a **Game Fingerprint Bundle** rather than one universal game hash.

---

# 2. Core Design Principle

Game identity should be represented at several levels:

```text
Game
└── Release
    └── Build
        └── Artifact
            └── Installation
```

These have deliberately different meanings.

### Game

The conceptual work.

Examples:

```text
Cyberpunk 2077
The Legend of Zelda: Ocarina of Time
Final Fantasy VII
```

This is roughly the level at which IGDB identifies the game.

---

### Release

A platform/region/edition-specific release.

Examples:

```text
Cyberpunk 2077 — Windows — Steam
Cyberpunk 2077 — Windows — GOG
Final Fantasy VII — PlayStation — USA
Final Fantasy VII — PlayStation — Japan
Ocarina of Time — Nintendo 64 — USA — Rev 1
```

Some ecosystems have strong native identifiers for this level.

---

### Build

A particular software revision.

Examples:

```text
Cyberpunk 2077 Steam build 12345678
Cyberpunk 2077 v2.31
PS2 release revision 1.01
Switch title update v65536
```

Not every platform exposes a useful build concept.

The Build entity should therefore be optional.

---

### Artifact

One immutable distributable object or media object.

Examples:

```text
game.iso
game.chd
game.nsp
game.xci
game.rom
installer.exe
Disc 1.bin + Disc 1.cue
Steam depot manifest
```

An artifact can usually be exactly fingerprinted.

---

### Installation

The actual collection of files Questarr sees in a library.

For example:

```text
Cyberpunk 2077/
├── bin/
├── archive/
├── engine/
├── mods/
├── red4ext/
└── screenshots/
```

An installation may differ from the canonical build because of:

- Mods
- DLC
- user configuration
- patches
- generated caches
- shader caches
- saves
- screenshots
- language packs
- launcher files

An installation therefore must never be assumed to be equivalent to a canonical Build merely because its directory resembles one.

---

# 3. Provenance Before Fingerprinting

Questarr should avoid rediscovering information it already knows.

If Questarr performs:

```text
Search
→ Download
→ Post-process
→ Import
```

then the post-processing job already knows the associated Questarr game.

The importer should carry that identity through the pipeline.

For example:

```text
DownloadJob
    gameId = 123
    igdbId = 109754
    requestedPlatform = "PC"
    sourceRelease = "Example.Release.Name"
```

After import:

```text
LibraryItem
    gameId = 123
    provenance = QUESTARR_IMPORT
```

Questarr should then fingerprint the files for later recognition, integrity, deduplication, and move detection.

It should **not** fingerprint the files and then attempt to rediscover that they belong to game `123`.

This gives us the priority:

```text
1. Questarr provenance
2. Questarr sidecar
3. Native/platform identifier
4. Exact content fingerprint
5. Known external fingerprint database
6. Structural fingerprint
7. Fuzzy structural/content matching
8. Metadata/name matching
9. User review
```

---

# 4. Identifiers and Fingerprints Are Different Things

Questarr should keep semantic identifiers separate from derived hashes.

## Identifier

An ID created by an ecosystem.

Examples:

```text
igdb:109754
steam:1091500
gog:<product-id>
psn:CUSA12345
ps2:SLUS-xxxxx
switch:<title-id>
```

Suggested model:

```ts
interface GameIdentifier {
  namespace: string;
  value: string;

  scope:
    | "game"
    | "release"
    | "build"
    | "artifact";

  source: string;

  metadata?: Record<string, unknown>;
}
```

Example:

```json
{
  "namespace": "steam.app",
  "value": "1091500",
  "scope": "release",
  "source": "steam"
}
```

---

## Fingerprint

Something Questarr derives from files.

Examples:

```text
SHA-256
tree hash
manifest hash
structural MinHash
sample hash
multi-disc set hash
```

Suggested model:

```ts
interface GameFingerprint {
  kind: string;
  algorithm: string;
  version: number;
  value: string;

  scope:
    | "artifact"
    | "installation"
    | "build";

  metadata?: Record<string, unknown>;
}
```

Keeping these separate makes queries and semantics substantially clearer than treating a Steam AppID as another "hash."

---

# 5. Questarr Fingerprint Bundle

A LibraryItem may have several fingerprints.

Example:

```json
{
  "identifiers": [
    {
      "namespace": "steam.app",
      "value": "1091500",
      "scope": "release"
    }
  ],

  "fingerprints": [
    {
      "kind": "tree",
      "algorithm": "sha256",
      "version": 1,
      "value": "..."
    },
    {
      "kind": "manifest",
      "algorithm": "sha256",
      "version": 1,
      "value": "..."
    },
    {
      "kind": "structure-minhash",
      "algorithm": "minhash128",
      "version": 1,
      "value": "..."
    }
  ]
}
```

No single member of the bundle is required to mean everything.

---

# 6. Fingerprint Types

## 6.1 Exact File Hash

For a single-file game:

```text
SHA256(file bytes)
```

This should be the primary fingerprint for:

- ROM files
- ISO images
- CHDs
- installers
- package files
- cartridge dumps
- individual disc images

Use **SHA-256 as Questarr's default exact hash in v1**.

Reasons:

- Native Node support
- No additional binary dependency
- Cross-platform
- Widely understood
- Compatible with external preservation databases
- Cryptographically strong
- Usually I/O-bound rather than CPU-bound when reading games from NAS storage

Additional hashes such as SHA-1, MD5, and CRC32 can be calculated when required to match existing DAT databases.

No-Intro DAT information commonly includes file size and CRC32/MD5/SHA-1/SHA-256, along with version and serial information.

Redump similarly uses disc metadata such as serial/version together with content checksums to distinguish and validate releases.

---

# 7. `qtree-v1`: Structural Tree Fingerprint

Directories need a cheap fingerprint that does not require reading hundreds of gigabytes.

`qtree-v1` fingerprints the **shape** of an installation.

For every included file:

```text
normalized-relative-path
file-size
```

Example canonical input:

```text
archive/pc/content/basegame_1_engine.archive\01238472639
archive/pc/content/basegame_2_mainmenu.archive\0938476234
bin/x64/cyberpunk2077.exe\064248320
engine/config/base/general.ini\04832
```

Entries are sorted lexicographically.

Then:

```text
SHA256(
    "qtree-v1\0" +
    canonical entries
)
```

The fingerprint must NOT include:

- absolute path
- timestamps
- inode
- UID/GID
- permissions
- creation date
- scan date

Otherwise simply moving the library would alter the fingerprint.

### Path normalization

For `qtree-v1`:

1. Strip the library root.
2. Convert path separators to `/`.
3. Remove redundant `./`.
4. Normalize Unicode to NFC.
5. Preserve filename case.
6. Sort by canonical UTF-8 path.
7. Never allow `..` to escape the library root.
8. Do not follow symbolic links outside the scanned root.

Questarr may additionally create:

```text
qtree-casefold-v1
```

for case-insensitive comparison, but it should not replace the canonical fingerprint because Linux directories may legitimately contain names differing only by case.

---

# 8. `qmanifest-v1`: Exact Installation Manifest

The tree fingerprint tells us:

> These installations have the same structure and sizes.

The manifest fingerprint tells us:

> These installations contain the same bytes.

Calculate:

```text
fileDigest = SHA256(file)
```

Then:

```text
leafDigest =
    SHA256(
        "F\0" +
        relativePath +
        "\0" +
        fileSize +
        "\0" +
        fileDigest
    )
```

Sort leaves by normalized path.

Then:

```text
manifestDigest =
    SHA256(
        "qmanifest-v1\0" +
        leafDigest1 +
        leafDigest2 +
        ...
    )
```

An exact `qmanifest-v1` match means the compared installations contain the same included files.

It does **not** by itself prove that those bytes correspond to an officially published build unless Questarr has a canonical build manifest to compare against.

That distinction should remain explicit.

---

# 9. Manifest Storage

Do not store every installation file as a permanent database row by default.

A large library could easily contain millions of files.

Instead store:

```text
LibraryItem
    qtree
    qmanifest
    structural sketch
    file count
    total size
    manifest blob
```

The detailed manifest can be serialized and compressed.

Example:

```json
[
  ["bin/game.exe", 18374623, "sha256:..."],
  ["data/main.pak", 5363827362, "sha256:..."]
]
```

Compression should work particularly well because paths repeat many prefixes.

Per-file rows can be introduced later if a feature genuinely needs them.

---

# 10. Fingerprint Cache

Full hashing must be incremental.

Questarr should maintain a hash cache approximately keyed by:

```text
library
relative path
size
mtime
```

On supported local filesystems it may additionally record:

```text
device
inode
```

These properties are only cache hints.

They are **not part of the fingerprint**.

For an unchanged file:

```text
same path
same size
same sufficiently precise mtime
→ reuse previous digest
```

A verification scan can ignore that cache and rehash everything.

This matters enormously for games containing 50–150 GB of data.

---

# 11. Fuzzy Game Fingerprinting

Exact hashes fail as soon as a game receives a patch.

Questarr therefore needs a fuzzy representation of an installation.

Do **not** fuzzy-hash the directory as one concatenated byte stream.

Instead treat a game installation as a **set of features**.

---

## 11.1 Structural Feature Set

Generate tokens such as:

```text
bin/x64/cyberpunk2077.exe
archive/pc/content/basegame_1_engine.archive
archive/pc/content/basegame_2_mainmenu.archive
engine/config/base/general.ini
```

This produces:

```text
QPathSet
```

Two releases can then be compared using Jaccard similarity:

```text
|A ∩ B|
-------
|A ∪ B|
```

This works particularly well for:

```text
same game
different patch
mostly same files
```

while avoiding the cost of hashing all content.

---

# 12. Structural MinHash

Storing and comparing entire file sets against every known game is inefficient.

Questarr should eventually create:

```text
qstructure-minhash-v1
```

A 128-component MinHash sketch is a reasonable initial implementation.

It represents the normalized path set.

This allows Questarr to cheaply ask:

```text
Which known installations have a similar file structure?
```

Example conceptual result:

```text
Candidate                               Similarity

Cyberpunk 2077 known installation       0.987
Cyberpunk 2077 older build              0.954
Cyberpunk 2077 GOG installation         0.901
The Witcher 3                           0.041
```

MinHash is **candidate discovery evidence**, not identity proof.

Questarr should retrieve promising candidates and then compare richer evidence.

---

# 13. Path + Size Structural Sketch

A second structural representation can use:

```text
normalized-path + size-bucket
```

instead of only the path.

This helps distinguish:

```text
same directory layout
but substantially different contents
```

Use a size bucket rather than exact size for fuzzy matching.

For example:

```text
0
1–4 KiB
4–16 KiB
16–64 KiB
64–256 KiB
256 KiB–1 MiB
1–4 MiB
4–16 MiB
16–64 MiB
64–256 MiB
256 MiB+
```

Exact size remains available in the real manifest.

---

# 14. Volatile Files and Fingerprint Profiles

Not every file should influence fuzzy identification.

Common exclusions include:

```text
save/
saves/
logs/
cache/
shadercache/
screenshots/
crashes/
temp/
tmp/
mods/
workshop/
```

and files like:

```text
*.log
*.tmp
```

However these rules are not universally valid.

Therefore Questarr should use **fingerprint profiles**:

```text
generic-pc-v1
steam-v1
gog-v1
playstation-disc-v1
switch-v1
rom-v1
mame-v1
```

Each profile decides:

```ts
interface FingerprintProfile {
  include(path: string): boolean;
  classify(path: string): FileClass;
}
```

Possible classes:

```text
CORE
CONTENT
EXECUTABLE
METADATA
DLC
MOD
USER_DATA
CACHE
UNKNOWN
```

The raw filesystem manifest may still record excluded files.

They simply should not necessarily participate in identity fingerprints.

---

# 15. Anchor Evidence

Questarr should support adapter-selected **anchor files**.

An anchor is a file likely to contain strong identity information.

Examples:

```text
Steam app manifest
PARAM.SFO
SYSTEM.CNF
Info.plist
game executable
console metadata file
store metadata JSON
disc metadata
ROM header
```

An adapter should return evidence rather than forcing all games through the same assumptions.

Example:

```ts
interface IdentityEvidence {
  identifiers: GameIdentifier[];
  metadata: Record<string, unknown>;
  anchors: FingerprintAnchor[];
}
```

This is preferable to defining a universal "core hash."

---

# 16. Platform Adapter Architecture

Platform knowledge should live behind adapters.

Suggested interface:

```ts
interface GameIdentityAdapter {
  id: string;
  priority: number;

  probe(context: ScanContext): Promise<ProbeResult>;

  identify(
    context: ScanContext
  ): Promise<IdentityEvidence>;

  fingerprintProfile?(
    context: ScanContext
  ): FingerprintProfile;
}
```

Example adapters:

```text
QuestarrSidecarAdapter
SteamAdapter
GogAdapter
EpicAdapter
MicrosoftStoreAdapter
MacBundleAdapter

PlayStationAdapter
NintendoAdapter
XboxAdapter

MameAdapter
RomAdapter
DiscImageAdapter

GenericPcAdapter
GenericDirectoryAdapter
```

The scanner should allow multiple adapters to contribute evidence.

A directory does not need to belong exclusively to one adapter.

---

# 17. PC — Steam

Steam is one of the strongest PC cases.

Where available, Questarr should extract:

```text
Steam AppID
install directory
installed build information
depot information
manifest identifiers
branch information
```

Steam officially uses AppIDs for applications and SteamPipe creates versioned depot manifests and global BuildIDs. Steam also exposes a local `appmanifest_[appid].acf` describing installed application state.

Recommended mapping:

```text
Steam AppID
    → strong Release identifier

Steam BuildID
    → Build identifier

Depot manifest IDs
    → Build/component identifiers

qmanifest
    → exact observed installation
```

Important:

```text
Steam AppID != exact build
```

DLC and optional depots can cause two valid installations of the same AppID to have different filesystem contents.

Steam metadata should therefore establish identity while content fingerprints describe what is actually installed.

---

# 18. PC — GOG

GOG assigns products product IDs, and GOG's developer tooling manages builds and updates for those products.

The GOG adapter should look for supported local product/build metadata where available and preserve:

```text
GOG product ID
GOG build/version
installed DLC/components
```

Mapping:

```text
GOG product ID
    → Release/provider identity

GOG build information
    → Build identity

filesystem fingerprints
    → observed installation
```

Questarr should avoid making private/internal GOG database formats a permanent API contract.

Provider-specific parsing should be isolated behind the adapter so it can evolve independently.

---

# 19. PC — Epic Games

Use an Epic adapter when launcher metadata is available.

Potential identifying evidence includes provider-level values such as:

```text
namespace
catalog item
app name
manifest/build metadata
```

Because launcher metadata formats may change, the parser should be treated as versioned and best-effort.

If Epic metadata cannot be parsed:

```text
→ fall back to GenericPcAdapter
```

rather than failing the scan.

---

# 20. PC — Microsoft Store / Xbox PC

Microsoft-installed games should be handled through a dedicated adapter capable of consuming accessible package/game metadata when available.

Potential identity sources include:

```text
package identity
product identity
application identity
Gaming Services metadata
```

Filesystem access restrictions must not cause the whole scan to fail.

The adapter should be allowed to return:

```text
recognized but inaccessible
```

rather than attempting unsafe permission workarounds.

---

# 21. Standalone PC Games

Standalone games are the difficult case.

There may be no globally unique ID.

Use:

```text
1. embedded application metadata
2. executable metadata
3. known sidecars/manifests
4. directory structure
5. structural similarity
6. content anchors
7. title metadata
```

Directory name alone must never produce an automatic identity decision.

For example:

```text
Doom/
```

is insufficient.

---

# 22. macOS Games

For macOS application bundles, useful evidence includes:

```text
CFBundleIdentifier
CFBundleName
CFBundleShortVersionString
CFBundleVersion
code-signing metadata
```

`CFBundleIdentifier` is generally much stronger evidence than the visible `.app` filename.

The filesystem fingerprint can then describe the actual bundle.

---

# 23. Single-File ROM Platforms

For classic cartridge/ROM platforms, exact hashing should dominate.

Examples:

```text
NES
SNES
Genesis / Mega Drive
Game Boy
Game Boy Color
Game Boy Advance
Nintendo 64
Nintendo DS
```

Use:

```text
internal header metadata where useful
+
exact file checksum
+
DAT lookup
```

No-Intro demonstrates why this model is effective: exact checksums can be associated with serial, revision, version, and other release metadata.

Recommended identity:

```text
platform
native serial/game code if present
ROM revision
SHA-256
external DAT identity
```

Filename should merely assist candidate selection.

---

# 24. Headerless or Weak-Header ROMs

Some formats provide little reliable identity metadata.

For those:

```text
exact content hash
    → primary identity
```

External DAT databases become particularly valuable.

Examples include systems where internal titles are incomplete, inconsistent, or duplicated.

---

# 25. PlayStation 1 / PlayStation 2

Disc releases frequently contain useful serial/product identifiers.

Questarr should extract native serial information where feasible and combine it with exact media fingerprints.

Example identity:

```text
platform = ps2
serial = SLUS-xxxxx
region = US
revision = ...
disc = 1
```

The serial identifies a release far better than:

```text
"Metal Gear Solid 3"
```

The exact disc content hash then differentiates revisions and verifies the artifact.

Redump specifically records serial, version, edition, and checksums for disc releases.

---

# 26. PSP / PS3 / Vita / Later PlayStation Formats

Where package/disc metadata exposes identifiers such as title IDs, content IDs, or disc IDs, capture them through the PlayStation adapter.

Conceptually:

```text
native title/disc ID
    → Release identity

version metadata
    → Build/revision evidence

artifact/manifest hash
    → exact observed content
```

Questarr should parse metadata that is already available to the scanner.

Identification must not depend on bypassing platform encryption or access controls.

---

# 27. Nintendo Disc and Package Platforms

Nintendo platforms frequently expose useful native identifiers at the media/package level.

Examples may include:

```text
game codes
disc IDs
title IDs
revision values
```

The Nintendo adapter should normalize these into typed identifiers rather than putting them into filenames.

No-Intro, for example, explicitly distinguishes serial/game-code and revision information for several Nintendo platforms.

Example:

```json
{
  "namespace": "nintendo.game_code",
  "value": "XXXXX",
  "scope": "release"
}
```

Exact hashes remain important for distinguishing specific dumps or revisions.

---

# 28. Xbox Platforms

The Xbox adapter should extract native title/package identifiers from supported containers or executable metadata where they are available.

Use:

```text
native title ID
    → Release identity

version/build metadata
    → Build evidence

content fingerprint
    → Artifact identity
```

As with other consoles, exact format parsing belongs inside the adapter.

The central matcher should not contain Xbox-specific rules.

---

# 29. Arcade / MAME

MAME is fundamentally a manifest/set problem.

A MAME item may consist of:

```text
multiple ROMs
BIOS dependency
parent ROM
clone ROM
CHD
device ROM
```

Identity should therefore incorporate:

```text
MAME machine/set identifier
DAT/version source
member ROM CRC/SHA-1
CHD identity
```

The version of the reference DAT matters because sets evolve between MAME releases.

Example:

```text
namespace = mame.set
value = sf2
metadata.datVersion = "0.xxx"
```

Do not treat a ZIP container hash as the canonical game identity.

Recompressing the same ROM set would otherwise make it appear to be a different game.

---

# 30. Archives

Questarr must distinguish:

```text
container identity
```

from:

```text
payload identity
```

Example:

```text
game.zip
game.7z
```

could contain identical ROMs while the archive bytes differ because of:

- compression level
- archive ordering
- timestamps
- compressor implementation

Therefore:

```text
SHA256(archive)
```

is useful for exact artifact identity but should not be the only game identity.

For supported archive formats Questarr may generate:

```text
archive-container-sha256
payload-manifest
payload-qtree
payload-qmanifest
```

Payload hashing should stream entries without extracting them to arbitrary filesystem paths.

---

# 31. Multi-File Disc Images

Formats such as:

```text
CUE + BIN
M3U + discs
CCD + IMG + SUB
```

need a **set fingerprint**.

For example:

```text
Disc 1:
    track1.bin sha256 A
    track2.bin sha256 B

Disc 2:
    track1.bin sha256 C
```

Create:

```text
qset-v1 =
    SHA256(
        ordered logical role +
        member content hashes
    )
```

Logical ordering matters.

The filename itself should not.

This means:

```text
Final Fantasy VII (Disc 1).bin
```

can be renamed without changing identity.

---

# 32. Multi-Disc Games

A Release can have several Artifacts:

```text
Final Fantasy VII
├── Disc 1
├── Disc 2
└── Disc 3
```

Do not model those as three games.

Instead:

```text
Release
    artifacts:
        DISC_1
        DISC_2
        DISC_3
```

Create an optional aggregate:

```text
qrelease-set-v1
```

from the ordered artifact fingerprints.

---

# 33. CHD and Converted Formats

Converted archival formats create an important distinction:

```text
physical/logical media identity
vs
container identity
```

For example:

```text
BIN/CUE
→ CHD
```

The resulting CHD will not have the same file hash as the original BIN/CUE set.

Adapters should preserve any reliable logical-media hashes provided by the container format/tooling in addition to the container SHA-256.

This allows Questarr eventually to recognize:

```text
same disc content
different storage representation
```

without pretending the files themselves are byte-identical.

---

# 34. Questarr Sidecar

After Questarr confidently identifies an imported game, it should optionally write:

```text
.questarr.json
```

into the game root.

Example:

```json
{
  "schemaVersion": 1,

  "game": {
    "questarrId": 123,
    "igdbId": 109754
  },

  "release": {
    "platform": "PC",
    "source": "steam"
  },

  "identifiers": [
    {
      "namespace": "steam.app",
      "value": "1091500"
    }
  ],

  "fingerprints": {
    "tree": {
      "version": 1,
      "sha256": "..."
    }
  }
}
```

This should become very strong identification evidence on future scans.

`.questarr.json` must itself be excluded from fingerprints.

Benefits:

```text
library moved to another disk
Questarr database restored
directory renamed
library mounted on another server
```

and Questarr can still immediately recognize the item.

Sidecar writing should be configurable.

---

# 35. Matching Engine

Identification should be based on evidence, not a single magic score.

Suggested evidence classes:

```text
AUTHORITATIVE
EXACT
STRONG
SUPPORTING
FUZZY
WEAK
```

Examples:

### AUTHORITATIVE

```text
Questarr-managed import provenance
valid Questarr sidecar
```

### EXACT

```text
known artifact SHA-256
known canonical manifest exact match
known ROM DAT checksum match
```

### STRONG

```text
Steam AppID
console title ID
disc serial
GOG product ID
provider build ID
```

### SUPPORTING

```text
platform
region
edition
version
executable metadata
disc number
```

### FUZZY

```text
structural MinHash
path-set similarity
partial content overlap
```

### WEAK

```text
folder name
filename
normalized title
release year
```

---

# 36. Candidate Generation vs Final Resolution

These are separate operations.

## Candidate generation

Find plausible games using:

```text
native IDs
exact hashes
DAT databases
provider IDs
title tokens
structural sketches
```

Candidate generation should favor recall.

---

## Final resolution

Evaluate evidence for each candidate.

Resolution should favor precision.

A wrong automatic merge is substantially worse than asking the user to review an ambiguous game.

---

# 37. Matching Rules

Recommended conservative rules:

### Rule A — Questarr provenance

```text
Questarr import references Game X
→ Game X
```

No fuzzy matching necessary.

---

### Rule B — Sidecar

```text
valid sidecar references Game X
→ Game X
```

Verify enough filesystem evidence to detect a stale/copied sidecar if practical.

---

### Rule C — Exact known artifact

```text
SHA-256 matches known artifact
→ exact artifact/release
```

---

### Rule D — Exact native identifier

```text
native ID maps uniquely to Release X
→ Release X
```

Build remains unknown unless separately identified.

---

### Rule E — Native ID + revision

```text
serial/title ID
+
revision/build identifier
→ specific Build
```

---

### Rule F — Structural fuzzy match

Require multiple forms of corroboration before automatic linking.

Example:

```text
same platform
AND
high structural similarity
AND
compatible title metadata
AND
at least one anchor matches
```

Only then consider automatic association.

---

### Rule G — Name only

```text
folder name ~= title
```

should create a candidate but never automatically establish identity.

---

# 38. Modified Installations

The model should explicitly represent:

```text
identified game
+
modified installation
```

rather than forcing:

```text
identified
OR
unknown
```

Example:

```text
Game:
    Cyberpunk 2077

Release:
    Steam

Build:
    likely 2.31

Installation state:
    MODIFIED

Reasons:
    native Steam AppID matches
    expected core structure matches
    472 additional mod files
    qmanifest differs
```

This is a major benefit of keeping identity separate from exact content fingerprints.

---

# 39. DLC

DLC should not cause Questarr to identify the installation as a different game.

Adapters should be able to return components:

```ts
interface InstalledComponent {
  type:
    | "base-game"
    | "dlc"
    | "language"
    | "optional-content"
    | "mod";

  identifier?: GameIdentifier;
}
```

The base Release remains stable while the installation's component set changes.

---

# 40. Huge Packed Game Files

Modern games frequently contain files such as:

```text
data.pak        70 GB
textures.pak    35 GB
audio.pak       18 GB
```

One small patch can change the SHA-256 of the entire file.

Do **not** attempt to solve this in the initial implementation.

For v1:

```text
native identity
+
tree structure
+
manifest
+
structural MinHash
```

is sufficient.

A later fingerprint could introduce content-defined chunking:

```text
large file
→ FastCDC-style chunks
→ chunk hashes
→ chunk-set sketch
```

which would estimate how much content two giant files share.

This should be considered **future work**, because it substantially increases scanning and storage complexity.

---

# 41. Cheap Sample Hash

An optional intermediate fingerprint may be useful for very large files.

Example:

```text
qsample-v1:
    filesize
    first 1 MiB
    middle 1 MiB
    last 1 MiB
```

Hash those together.

This can cheaply answer:

```text
are these large files probably the same?
```

It must never be treated as cryptographically exact identity.

Suggested evidence classification:

```text
SUPPORTING
```

not:

```text
EXACT
```

---

# 42. Import Pipeline

Recommended scan pipeline:

```text
DISCOVER
   ↓
CLASSIFY ITEM
   ↓
RUN ADAPTER PROBES
   ↓
READ PROVENANCE / SIDECAR
   ↓
EXTRACT NATIVE IDENTIFIERS
   ↓
COMPUTE CHEAP STRUCTURAL FINGERPRINT
   ↓
GENERATE CANDIDATES
   ↓
RESOLVE IF POSSIBLE
   ↓
SELECTIVE HASHING IF NECESSARY
   ↓
FULL HASHING IF NECESSARY
   ↓
MATCH / REVIEW
   ↓
STORE LIBRARY ITEM
   ↓
OPTIONALLY WRITE SIDECAR
```

---

# 43. Progressive Scanning

Do not read every byte during the initial library scan.

## Level 0 — Metadata

Read:

```text
directory name
file names
file sizes
platform manifests
sidecars
native metadata
```

Very cheap.

---

## Level 1 — Structural fingerprint

Calculate:

```text
qtree-v1
file count
total size
structural MinHash
```

Still very cheap because file contents are not read.

---

## Level 2 — Anchor hashing

Hash:

```text
known metadata files
executables
adapter-selected anchors
small identity-bearing files
```

Moderate cost.

---

## Level 3 — Exact content verification

Calculate:

```text
all file SHA-256 hashes
qmanifest-v1
```

Potentially expensive.

Run when:

```text
user requests verification
candidate remains ambiguous
external DAT matching requires it
background idle verification is enabled
```

---

# 44. SQLite / Drizzle Data Model

Questarr currently uses SQLite with Drizzle configuration, so the design should avoid schemas that require millions of relational file rows.

A conceptual model:

```text
game_releases
-------------
id
game_id
platform
region
edition
source

game_builds
-----------
id
release_id
version
provider_build_id

library_items
-------------
id
game_id
release_id nullable
build_id nullable

library_root_id
relative_path

item_type
scan_state
match_state

file_count
total_size

last_scanned_at
created_at
updated_at
```

Identifiers:

```text
game_identifiers
----------------
id
entity_type
entity_id

namespace
value
scope
source

metadata_json
```

Fingerprints:

```text
game_fingerprints
-----------------
id
library_item_id

kind
algorithm
algorithm_version

scope
value

profile
metadata_json

created_at
```

Optional detailed manifest storage:

```text
library_manifests
-----------------
library_item_id
format_version
compression
manifest_blob
```

---

# 45. Fingerprint Versioning

Every Questarr-defined fingerprint must be versioned.

Never store:

```text
tree_hash
```

with an implicit algorithm.

Store:

```text
kind = tree
algorithm = sha256
algorithm_version = 1
profile = generic-pc-v1
```

or encode it as:

```text
qfp:tree:1:sha256:<digest>
```

Eventually Questarr will discover a normalization rule that needs changing.

Versioning allows:

```text
qtree-v1
qtree-v2
```

to coexist.

Existing libraries do not need to be invalidated immediately.

---

# 46. External Fingerprint Sources

Questarr should support imported reference databases.

Initial useful sources:

```text
No-Intro
Redump
MAME DATs
```

A generic interface:

```ts
interface FingerprintDatabase {
  id: string;

  lookup(
    fingerprints: GameFingerprint[]
  ): Promise<ExternalMatch[]>;
}
```

External matches should retain provenance:

```json
{
  "source": "redump",
  "sourceVersion": "...",
  "matchedBy": "sha1",
  "externalId": "..."
}
```

This makes matching explainable and allows database updates.

---

# 47. Explainable Matching

Every resolved item should retain **why** it matched.

Example:

```json
{
  "resolution": "matched",

  "gameId": 123,

  "evidence": [
    {
      "type": "identifier",
      "namespace": "steam.app",
      "value": "1091500",
      "strength": "strong"
    },
    {
      "type": "structure",
      "similarity": 0.987,
      "strength": "fuzzy"
    },
    {
      "type": "title",
      "similarity": 1.0,
      "strength": "weak"
    }
  ]
}
```

The UI can then display:

```text
Matched to Cyberpunk 2077

Steam AppID match
File structure 98.7% similar
Title match
```

rather than presenting an unexplained "97% confidence."

---

# 48. Match States

Suggested states:

```text
IDENTIFIED
IDENTIFIED_MODIFIED
PROBABLE
AMBIGUOUS
UNKNOWN
CONFLICT
```

Examples:

### IDENTIFIED

Strong deterministic evidence.

### IDENTIFIED_MODIFIED

Identity is known but installation differs from a known build.

### PROBABLE

Evidence strongly suggests a candidate but automatic criteria were not met.

### AMBIGUOUS

Several plausible candidates.

### UNKNOWN

No meaningful candidate.

### CONFLICT

Strong evidence disagrees.

Example:

```text
.questarr.json says Game A
Steam AppID says Game B
```

Questarr should surface this rather than silently picking one.

---

# 49. Security Requirements

Library contents are untrusted input.

The scanner must:

- Never execute discovered binaries.
- Never execute scripts to determine game identity.
- Never follow symlinks outside the configured root.
- Use bounded archive parsing.
- Protect against archive path traversal.
- Protect against decompression bombs.
- Limit pathological recursion.
- Gracefully handle unreadable files.
- Ignore device files, sockets, and pipes.
- Validate all parsed metadata.
- Avoid shell commands constructed from filenames.
- Treat sidecars as untrusted JSON.
- Validate Questarr IDs from sidecars against the database.
- Never allow scanned metadata to select arbitrary filesystem paths.

Fingerprinting should be passive.

---

# 50. Recommended Initial Platform Strategy

## Tier 1 — First implementation

Implement extremely well:

```text
Questarr-managed imports
Questarr sidecars
generic directories
single-file games
Steam
generic PC
No-Intro style ROM matching
Redump style disc matching
```

This covers a very large portion of likely libraries while establishing the architecture.

---

## Tier 2

Add:

```text
GOG
Epic
PS1
PS2
PSP
PS3
GameCube
Wii
Nintendo DS
3DS
Wii U
Switch metadata
Xbox / Xbox 360
```

Each becomes an adapter.

---

## Tier 3

Add specialized handling:

```text
MAME
multi-disc aggregation
CHD logical identity
archive payload identity
Microsoft Store / Gaming Services
Heroic metadata
itch.io
other launchers
```

---

## Tier 4

Advanced similarity:

```text
content MinHash
content-defined chunking
large archive similarity
cross-container logical media fingerprints
```

Do not block the first useful release on Tier 4.

---

# 51. Recommended v1 Fingerprints

I would commit v1 to only these core fingerprints:

```text
sha256
qtree-v1
qmanifest-v1
qset-v1
qstructure-minhash-v1
```

Plus native identifiers.

That is sufficient to establish the architecture without overengineering.

---

# 52. Testing Matrix

The feature should have integration fixtures proving at least the following.

### Same ROM, different filename

```text
Mario.sfc
game123.sfc
```

Expected:

```text
same artifact
same release
```

---

### Same game directory moved

```text
/library-a/Game
/library-b/Game
```

Expected:

```text
same fingerprints
same identity
```

---

### Same game patched

Expected:

```text
same Game
same Release
different qmanifest
possibly different Build
high structural similarity
```

---

### Same game with mods

Expected:

```text
same Game
same Release
IDENTIFIED_MODIFIED
```

---

### Steam game with DLC installed

Expected:

```text
same base Release
different component set
different installation manifest
```

---

### Same game Steam vs GOG

Expected:

```text
same Game
different Releases/provider identities
```

---

### Two games with identical/similar names

Expected:

```text
title alone does not merge them
```

---

### Recompressed ROM archive

Expected:

```text
different archive SHA-256
same payload identity
```

when payload scanning is enabled.

---

### Multi-disc game

Expected:

```text
one Release
multiple Artifacts
stable ordered set fingerprint
```

---

### Modified file timestamp

Expected:

```text
fingerprint unchanged
```

---

### Modified file permissions

Expected:

```text
fingerprint unchanged
```

---

### One modified byte

Expected:

```text
qmanifest changes
```

---

### Added screenshot

With appropriate PC profile:

```text
exact raw installation differs
identity-oriented structure unaffected
```

---

### Symlink escaping game directory

Expected:

```text
Questarr does not follow it outside root
```

---

# 53. Suggested Implementation Sequence for Codex

## Phase 1 — Domain model

Introduce:

```text
LibraryItem
GameRelease
GameBuild
Identifier
Fingerprint
MatchEvidence
```

Do not change existing game behavior unnecessarily.

---

## Phase 2 — Scanner abstraction

Implement:

```text
LibraryScanner
GameIdentityAdapter
FingerprintProfile
FingerprintService
```

Start with:

```text
QuestarrSidecarAdapter
GenericDirectoryAdapter
SingleFileAdapter
```

---

## Phase 3 — Basic fingerprints

Implement and test:

```text
SHA-256 streaming
qtree-v1
qmanifest-v1
qset-v1
hash cache
```

Golden test vectors should ensure future versions of Questarr produce identical fingerprints.

---

## Phase 4 — Provenance and sidecars

Carry Game identity from download through post-processing.

Generate optional:

```text
.questarr.json
```

after successful import.

---

## Phase 5 — Steam

Implement strong PC identification using:

```text
AppID
installation metadata
build metadata where available
```

Steam's existing versioned depot/build model makes it a good first platform-specific adapter.

---

## Phase 6 — ROM and disc fingerprint databases

Add generic DAT ingestion.

Support:

```text
CRC32
MD5
SHA-1
SHA-256
serial
revision
region
```

This allows No-Intro, Redump, and similar catalogs to plug into one architecture instead of creating special database tables for each project.

---

## Phase 7 — Fuzzy structural matching

Add:

```text
path-set similarity
qstructure-minhash-v1
candidate retrieval
evidence reporting
manual review UI
```

Do not initially auto-match solely from MinHash similarity.

Gather real-world results first.

---

## Phase 8 — Additional platform adapters

Implement each independently.

An adapter failure must always degrade gracefully to generic fingerprinting.

---

# 54. One Important Rule for the Codebase

Platform adapters should produce **evidence**, not final decisions.

Bad architecture:

```ts
SteamAdapter.identifyGame(): Game
```

Better:

```ts
SteamAdapter.collectEvidence(): IdentityEvidence
```

Then:

```text
Matcher
```

decides how the evidence maps to Questarr entities.

This prevents platform-specific assumptions from spreading through the application.

---

# 55. Final Architecture

The intended flow is:

```text
                         ┌──────────────────┐
                         │ Questarr Import  │
                         │ Provenance       │
                         └────────┬─────────┘
                                  │
                                  ▼
┌─────────────┐          ┌──────────────────┐
│ Filesystem  │─────────▶│ Library Scanner  │
└─────────────┘          └────────┬─────────┘
                                  │
               ┌──────────────────┼──────────────────┐
               │                  │                  │
               ▼                  ▼                  ▼
        ┌────────────┐     ┌────────────┐     ┌────────────┐
        │ Platform   │     │ Fingerprint│     │ Metadata   │
        │ Adapters   │     │ Service    │     │ Extractors │
        └─────┬──────┘     └─────┬──────┘     └─────┬──────┘
              │                  │                  │
              └──────────────────┼──────────────────┘
                                 ▼
                       ┌──────────────────┐
                       │ Identity Evidence│
                       └────────┬─────────┘
                                │
                                ▼
                       ┌──────────────────┐
                       │ Candidate Finder │
                       └────────┬─────────┘
                                │
                                ▼
                       ┌──────────────────┐
                       │ Match Resolver   │
                       └────────┬─────────┘
                                │
              ┌─────────────────┼──────────────────┐
              │                 │                  │
              ▼                 ▼                  ▼
        IDENTIFIED          AMBIGUOUS           UNKNOWN
              │                 │
              │                 ▼
              │            User Review
              │
              ▼
        Library Item
```

---

# 56. Summary

Questarr should not attempt to create one universal "game pHash."

Instead it should build a **Game Fingerprint Bundle** that represents different aspects of identity:

```text
Native/provider identifier
    → WHAT release is this?

Exact file hash
    → ARE these exact artifact bytes?

qtree
    → DOES this installation have the same structure?

qmanifest
    → DOES this installation contain the same bytes?

structural MinHash
    → WHAT known installation does this resemble?

Questarr provenance/sidecar
    → WHAT did Questarr already establish this to be?
```

The most important architectural separation is:

```text
Game identity
≠
Release identity
≠
Build identity
≠
Artifact identity
≠
Installation state
```

With those separated, Questarr can correctly represent all of the following without hacks:

```text
same game, different platform

same game, Steam vs GOG

same release, newer patch

same build, different directory

same game, modded installation

same disc, renamed file

same ROM, recompressed archive

same game, multiple discs

same game, DLC installed

unknown game resembling a known installation
```

That should be the foundation of Questarr's game-identification system.