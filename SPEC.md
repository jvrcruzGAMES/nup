# NUP v2 — decrypted NX submission package

Version: **2.0 draft**. Extension: `.nup`. Magic: **`4E 55 50 02`**.
Status: proposed wire format; no v2 producer or loader is implemented here.
MUST, MUST NOT, SHOULD and MAY express normative requirements.

## 1. Scope and package boundaries

NUP v2 stores decrypted Switch 1/2 filesystem dumps. Its organization follows
NSP: an indexed archive of content plus a content manifest. It is a new format,
not a PFS0/NSP or an NCA with a renamed extension. No Nintendo keys, tickets,
certificates, encrypted headers, RSA verification or BKTR processing are required.
NPDM and executable files are preserved verbatim; embedded signature bytes may
remain opaque, but NUP loading MUST NOT require their cryptographic verification.

Each file contains exactly one of:

| Kind | Value | Contents |
|---|---:|---|
| Base | 0 | Complete game snapshot, including a promoted prepatched dump |
| Update | 1 | Delta from one exact base snapshot to one complete target snapshot |
| DLC | 2 | Complete snapshot of one add-on content ID and content index |

Base, updates and DLC MUST be separate NUP files. Each DLC MUST have its own
file; DLC content MUST NOT be merged into the base or update filesystem.
Different versions of the same DLC are alternative files, not simultaneous mounts.
There are no nested NCA containers. Paths such as `exefs/main` and
`romfs/...` directly identify plaintext content.

Hashes provide corruption detection and exact dependency matching. They do not
prove publisher authenticity, ownership, executable safety or console validity:
an attacker can replace content and recompute unkeyed hashes.

## 2. Importing extracted dumps

Recognized input roots are `base/`, `updates/<decimal u32 version>/`, and
`dlc/<16 hexadecimal content-title-id>/<decimal u32 content-index>/`.
A bare directory containing `exefs/` or `romfs/` is also a base snapshot.
Only regular files and directories are accepted; reject symlinks and special files.
Exclude dump-tool `.listing` files by default before deciding whether a root is empty.

A missing base or a base containing no payload files, even if empty `exefs/` or
`romfs/` directories exist, means **there is no base**. In that case:

1. Treat each nonempty update directory as a complete game snapshot.
2. Emit a **base-kind** NUP for each snapshot, retaining its numeric version and
   setting `origin = 1` (promoted update). Each is independently launchable when
   its content is sufficient; it has no base dependency.
3. Put application metadata in each resulting base NUP. Do not emit an update-kind
   file or invent a dependency on an unavailable original release.
4. An importer MAY export only an explicitly selected snapshot; default export is
   all snapshots, with the highest numeric version recommended for launch.

This rule applies equally to an absent and an empty base folder. Promotion is a
packaging decision, not proof of completeness. If required program files are
missing, report an incomplete dump rather than silently searching for a base.

With a nonempty base, emit one base NUP and one separate update NUP per update
snapshot. The default import contract is that update directories are **complete
prepatched trees**: compare each directly against the base. If a source actually
contains an overlay or partial patch, the operator MUST explicitly select that
input mode and supply deletion information as needed. Materialize a complete target
snapshot before encoding it. Never infer snapshot-versus-overlay from directory size.

Nonempty but unusable bases MUST be reported; do not silently promote over them.
A DLC-only import is valid and emits only DLC files with empty metadata sections.
An empty content root MUST NOT produce a package.

Identifiers MUST come from validated dump metadata/NPDM or explicit operator input.
Do not infer a title ID from a game name or synthesize it by masking arbitrary bits.
`application_id` is the base application's ID, including for promoted dumps;
`content_title_id` preserves the original source title ID if known. A promoted
update may therefore retain a patch title ID while remaining base-kind.
Versions are opaque NCM u32 values; do not infer a human version string from bitfields.

## 3. Binary conventions and file layout

All integers are unsigned little-endian. Offsets are absolute unless explicitly
stated otherwise. No native structure padding is implied. SHA-256 produces 32 bytes.
All regions use checked arithmetic and must fit within the actual file length.

```
0x0000  Header                       0x100 bytes
0x0100  Index                        entry_count × 0x60 bytes
        Path pool                    packed UTF-8 paths
        Manifest                     0xA0 bytes
        Metadata                     0x10-byte header + TLVs (possibly none)
        zero alignment padding
        Payload                      per-entry stored extents
```

The first five regions are contiguous in that order. Payload begins at the next
16-byte boundary. Every nonempty extent starts at an absolute 16-byte boundary;
inter-extent padding is zero. Unreferenced payload data and trailing bytes are invalid.
No compression is required in this revision: stored literal bytes are raw. This
keeps random reads simple; future codecs require a new specified revision.
Game files that already contain compressed data are preserved unchanged.

### 3.1 Header (0x100 bytes)

| Offset | Type | Field |
|---:|---|---|
| 0x00 | bytes[4] | Magic `NUP` followed by byte `02` |
| 0x04 | u16 | Header size = 0x100 |
| 0x06 | u16 | Revision = 0 |
| 0x08 | u64 | Actual file size |
| 0x10 | u64 | Index offset = 0x100 |
| 0x18 | u64 | Entry count |
| 0x20 | u64 | Path pool offset |
| 0x28 | u64 | Path pool size |
| 0x30 | u64 | Manifest offset |
| 0x38 | u64 | Manifest size = 0xA0 |
| 0x40 | u64 | Metadata offset |
| 0x48 | u64 | Metadata size, at least 0x10 |
| 0x50 | u64 | Payload offset |
| 0x58 | u64 | Payload size |
| 0x60 | bytes[32] | Catalog hash |
| 0x80 | bytes[32] | Package ID |
| 0xA0 | bytes[96] | Reserved, zero |

Catalog hash = SHA-256 of exact bytes from index offset through the end of metadata,
including paths and manifest. It covers stored hashes and structural declarations.

Package ID = SHA-256(`header with bytes 0x80..0x9F zeroed` || catalog bytes).
It identifies this physical package's declarations; payload is bound indirectly by
stored hashes. Verify those hashes before trusting payload content. It is not the
snapshot ID (§5). Unknown revisions, header sizes or nonzero reserved fields MUST
be rejected by revision-0 readers.

### 3.2 Index entry (0x60 bytes)

The index describes the **entire resulting filesystem**, including unchanged
files in updates. Entries are sorted by path's UTF-8 byte sequence and unique.

| Offset | Type | Field |
|---:|---|---|
| 0x00 | u64 | Path offset relative to path pool |
| 0x08 | u32 | Path length in bytes |
| 0x0C | u8 | Node type: 0 file, 1 directory |
| 0x0D | u8 | Storage: 0 literal, 1 base reference, 2 delta |
| 0x0E | u16 | Reserved, zero |
| 0x10 | u64 | Result size |
| 0x18 | u64 | Stored extent absolute offset |
| 0x20 | u64 | Stored extent size |
| 0x28 | bytes[32] | SHA-256 of complete resulting file bytes |
| 0x48 | u64 | Base entry index, or UINT64_MAX |
| 0x50 | bytes[16] | Reserved, zero |

Directories have storage 0, result size/extent offset/extent size/hash all zero,
and base entry UINT64_MAX. Empty directories are preserved. The root is implicit;
include every other parent directory. A path cannot be both file and directory.

Paths are nonempty valid UTF-8, case-sensitive, slash-separated and relative.
No NUL, backslash, leading/trailing slash, empty component, `.` or `..` is allowed.
Do not normalize or case-fold source names silently. Producers SHOULD use NFC
and MUST report collisions introduced by any explicitly requested normalization.
The path pool is the concatenation of index paths in index order, without terminators
or gaps; each offset points to the next unused byte. Paths include the content root,
e.g. `exefs/main`, `romfs/System/...`, or `data/...`.

Literal files have stored size equal to result size and base entry UINT64_MAX.
An empty literal uses offset/size zero and SHA-256 of the empty byte string.
Base and DLC files MUST use literals only.

A base reference is valid only in update files. It names a file entry in the
required base's index; result size/hash MUST equal that entry's size/hash. Extent
offset/size are zero. It can reference a different source path (a rename).

A delta is valid only in updates. Its base entry identifies one file in the
required base; its extent holds the instruction stream below. Result size/hash
refer to reconstructed bytes, not instruction bytes. An added file can use literal
storage; a delta MUST have a valid base file even if no COPY is emitted.

### 3.3 Delta instruction stream

Instructions are packed without padding. Each starts with a one-byte opcode:

| Opcode | Following fields | Operation |
|---:|---|---|
| 0 | u64 source_offset, u64 length | COPY exact bytes from the selected base file |
| 1 | u64 length, bytes[length] | INSERT literal bytes |

Lengths MUST be nonzero. Instructions emit bytes sequentially. COPY ranges MUST
fit the base file; source offsets may move backwards and ranges may repeat.
The stream MUST end exactly at stored extent size and emit exactly result size.
Unknown opcodes, overflow, truncation or output hash mismatch are errors.
There is no END opcode and no in-place mutation of the base. Producers SHOULD
use a delta only when smaller than a literal and SHOULD coalesce adjacent operations.
Matching strategy and block size are producer choices, not wire-format requirements.

## 4. Manifest (0xA0 bytes)

This is mandatory structural content metadata, distinct from the optional
application metadata TLVs. Updates and DLC retain this manifest even though their
application metadata sections are empty.

| Offset | Type | Field |
|---:|---|---|
| 0x00 | bytes[4] | `NCM` followed by byte `02` |
| 0x04 | u8 | Kind: 0 base, 1 update, 2 DLC |
| 0x05 | u8 | Platform: 0 unspecified, 1 Switch, 2 Switch 2 |
| 0x06 | u8 | Origin: 0 normal, 1 promoted update |
| 0x07 | u8 | Reserved, zero |
| 0x08 | u64 | Application ID, nonzero |
| 0x10 | u64 | Content title ID, nonzero |
| 0x18 | u32 | Content version |
| 0x1C | u32 | Content index (0 for ordinary single-program base/update) |
| 0x20 | bytes[32] | Required base snapshot ID; zero for base/DLC |
| 0x40 | bytes[32] | Result snapshot ID |
| 0x60 | u32 | Minimum effective application version for DLC, otherwise zero |
| 0x64 | u32 | Flags, zero |
| 0x68 | bytes[56] | Reserved, zero |

Origin 1 is allowed only for base. Update dependency MUST be nonzero. A DLC's
application ID explicitly associates it with the game; its content title ID/index
identify the add-on. Do not rely on arithmetic ID conventions alone. Unknown kinds,
platforms, origins or flags are errors. Revision 0 models one program per base;
importers MUST reject multi-program sources rather than discard programs.

Each update depends directly on one base-kind snapshot, never another update.
Updates are alternatives, not a chain; select one compatible update. A promoted
base can be the dependency for a newly generated update, but a patch targeting an
original release cannot be applied merely because application ID/version match.

## 5. Snapshot identity and exact update semantics

Compute a snapshot ID from the *logical resulting tree*, independent of literal
versus delta representation and independent of application metadata:

```
SHA-256(
  bytes("NUP2SNAP") || application_id:u64 || content_index:u32 ||
  entry_count:u64 ||
  for each entry in path order:
    node_type:u8 || path_length:u32 || path_bytes ||
    result_size:u64 || result_sha256:bytes[32]
)
```

Directory size/hash are zero as specified above. No terminators or padding are
included. This hash binds file paths, empty directories, lengths and contents.
Readers MUST compare the recomputed ID with the manifest and verify referenced
base bytes, not just trust an unverified base's claimed ID.

An update index is a complete target namespace. A path absent from it is deleted;
unchanged files explicitly reference base entries. Never expose the union of base
and update paths: that would resurrect deleted files. File/directory replacements
and renames follow naturally from the target namespace. Missing target parents,
children below file nodes or duplicate paths are invalid.

Base dependency matching requires application ID, content index and exact snapshot
ID. Title/version matching alone is insufficient. DLC uses application ID and any
minimum application version requirement; its snapshot is independent of the base.

## 6. Application metadata section

Every NUP has a metadata section, including updates and DLC. The 0x10-byte header is:

| Offset | Type | Field |
|---:|---|---|
| 0x00 | bytes[4] | `MET` followed by byte `02` |
| 0x04 | u32 | TLV count |
| 0x08 | u64 | Combined TLV byte length |

Base files hold all available application metadata. Missing names/icons are allowed;
use application ID as the UI fallback. Update and DLC files MUST have count 0,
TLV byte length 0 and metadata size exactly 0x10. Their header is present and their
metadata content is empty. Required dependency/identity fields belong to the manifest,
not TLVs. Program files such as `main.npdm` still belong in content trees.

Each TLV has `type:u16, reserved:u16=0, payload_size:u32, payload[ size ]`, packed
without padding. Count and total length MUST agree exactly; reject partial records.
Types retain v1 meanings for migration:

| Type | Payload |
|---:|---|
| 0x0001 | Verbatim base snapshot `exefs/main.npdm`, if available |
| 0x0002 | Verbatim NACP, if available |
| 0x0003 | Default icon image bytes (sniff JPEG/PNG) |
| 0x0004 | UTF-8 display version of this base snapshot |
| 0x0005 | UTF-8 default title name |
| 0x0006 | UTF-8 publisher |
| 0x0007 | 32-byte executable ModuleId, if supported and validated |
| 0x0008 | UTF-8 source dump name |
| 0x0009 | Localized titles |
| 0x000A | Localized icons |

Localized TLVs contain packed records of `lang_len:u8, lang[lang_len],
data_len:u32, data[data_len]`. Tags are nonempty lowercase ASCII BCP-47 tags;
titles are UTF-8 and icons are image bytes. Duplicate types or duplicate language
tags within a localized payload are invalid. Unknown TLV types are skipped and
SHOULD be preserved when rewriting. Registered types 1–8 are singleton records;
9/10 each contain all their language records. A metadata NPDM MUST match the
base tree's file. Runtime permissions/build IDs come from the selected effective
ExeFS, so base metadata cannot override patched executable information.

Since updates have empty metadata, loaders MUST NOT display the base's display
version as the update's version. Show the effective numeric content version unless
an effective runtime control file supplies a display version. Numeric versions
remain opaque. No extra update metadata is needed to identify or load a patch.

## 7. Validation and read behavior

Before mounting, readers MUST:

1. Dispatch on all four magic bytes; validate header/revision and reserved fields.
2. Check actual length, contiguous catalog layout, alignment, bounded counts and
   all checked offset/length products before allocating or reading arrays.
3. Verify catalog hash and package ID before using catalog content.
4. Validate manifest, metadata rules, paths, parent topology and snapshot ID.
5. Validate every extent and dependency reference. Nonempty extents MUST NOT
   overlap each other or the catalog. No extent sharing is defined in revision 0.
6. For updates, locate and verify the exact required base and its file index.
7. Verify SHA-256 over resulting bytes before exposing a file as verified. A strict
   prelaunch verification pass MUST check all effective files; lazy readers MAY
   verify on first access but MUST report the package as incompletely verified.
   Execution files MUST be verified before execution. Failed verification MUST
   abort that read/launch, never fall back to an older file silently.

Readers must handle short reads, enforce configurable resource limits for path
length/count/instruction count/output size, and prevent filesystem traversal when
extracting. Validation errors must identify unsupported version, corrupt package,
missing base, mismatched base or incomplete program separately where practical.
Package ID/snapshot ID checks alone do not verify the actual payload bytes.

A literal supports direct bounded reads. Base references delegate to the base file.
Delta readers can build an instruction-to-result-offset index for random access or
materialize verified results into a temporary cache. Cache keys SHOULD include
package ID, base snapshot ID and result file hash; publish cache entries atomically.
Keep source packages open for the mount lifetime. Never rewrite the installed base.

## 8. Observed dump examples

Inspected locally on 2026-11-05; counts below include source regular files and are
observations, not completeness or game-identity verification.

| Dump root | Files | Logical bytes |
|---|---:|---:|
| Ocarina of Time `updates/131072` | 186592 | 26274757607 |
| Bananza `base` | 75242 | 9180838667 |
| Bananza `updates/458752` | 77346 | 9207479192 |
| Bananza `dlc/040074A01BF13001` (all index subtrees) | 7270 | 957636015 |

Ocarina has no `base/`: emit `oot.base.v131072.nup`, kind base, origin 1,
version 131072, zero dependency, complete ExeFS/RomFS, and populated metadata
where available. Do not emit a patch requiring a missing release.

Bananza has a nonempty base: emit `bananza.base.nup`,
`bananza.update.v458752.nup`, and
`bananza.dlc.040074a01bf13001.0.nup` for the observed index 0 subtree.
The update records the base snapshot ID and a full target index with references,
literals and optional deltas. It does not contain a second base or the DLC.
Only the base's metadata section contains TLVs. File names are illustrative;
loaders use manifest IDs, never file-name parsing.

The observed update tree is treated as a complete prepatched snapshot under the
import contract. File counts alone do not establish whether every expected asset
is present. A producer should report comparison statistics and launchability checks.

## 9. Distinction from v1 and migration

| Property | v1 | v2 revision 0 |
|---|---|---|
| First four bytes | `4E 55 50 01` | `4E 55 50 02` |
| Header | 0x80-byte superblock | 0x100-byte header |
| Contents | Several package trees in one file | Exactly one base, update or DLC |
| Index | TOC, node arrays, child slots | Flat path index and content manifest |
| Storage | Raw/per-file zstd | Raw literals, base references, COPY/INSERT |
| Update-only input | Legacy standalone flag/loader behavior | Explicit independent base-kind snapshot |
| Metadata | Per-package TLVs | Base TLVs; empty section in update/DLC |

v2 MUST NOT use v1 magic, TOC layout, UUID algorithm, codec numbers or package flags
as an implicit extension. A loader MUST NOT route an arbitrary `NUP` prefix or
`.nup` extension directly into the v1 parser. Dispatch major 1 to the unchanged
v1 parser, major 2 to v2, and reject other majors as unsupported.
The current Eglantine v1 container compares the exact four-byte magic, so v2 is
rejected rather than misinterpreted. Existing v1 packages remain unchanged and
loadable through the v1 path. Older loaders cannot be promised a friendly error.

Migration requires decoding and verifying v1 content, reconstructing complete
snapshots, then writing new packages. Merely changing magic bytes is invalid.
Split legacy DLC into separate files. Recompute all v2 identities. Reuse metadata
payloads after checking them against the new base; do not copy application metadata
into update/DLC files. Interpret legacy standalone-update flags according to actual
content completeness, not as proof that a base dependency exists.

## 10. Reference material

The v1 comparison uses `SPEC.md` from the repository's `origin/meow` ref as inspected
on 2026-11-05. [NSP.md](NSP.md) provides contextual NSP/NCA notes; it is not normative
for v2 bytes. This specification defines its own manifest and delta format and does
not depend on every statement in that reference being applicable to decrypted dumps.
See [EMULATOR_INTEGRATION.md](EMULATOR_INTEGRATION.md) for proposed loader integration.
