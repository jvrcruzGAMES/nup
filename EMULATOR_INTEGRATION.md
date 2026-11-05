# Loading NUP v2 in Eglantine

These are integration instructions for future work, not implemented support.
The local Eglantine source inspected on 2026-11-05 supports NUP v1; v2 files
specified in [SPEC.md](SPEC.md) require a new parser and mount path.

## 1. Preserve v1 and add format dispatch

Relevant existing files in EglantineBuild:

- `src/core/file_sys/nup_container.cpp` / `.h`: current v1 parsing and VFS views.
- `src/core/loader/nup.cpp` / `.h`: `AppLoader_NUP`, metadata and launch path.
- `src/core/loader/deconstructed_rom_directory.*`: plaintext program loading.
- `src/core/file_sys/romfs.*`, `romfs_factory.*`: RomFS construction/mounting.
- `src/core/file_sys/patch_manager.*`: existing patch/mod integration.
- `src/yuzu/nup_dialogs.*`: current package UI operations, to audit for v2 assumptions.

Dispatch by exact magic: `NUP\x01` retains the existing implementation;
`NUP\x02` selects a separate v2 reader. Unsupported majors need a clear error.
Keep format-specific records and caches separate. Do not reinterpret v2 as NSP,
pass plaintext through NCA decryption, or weaken the v1 magic check.

## 2. Discover and select content

1. Parse and validate each selected/registered file according to SPEC sections 3–7.
2. Only base-kind packages are game-list launch entries, including promoted bases.
   Read names/icons/NACP from their metadata. Fall back to application ID.
3. Register updates by application ID, content index and required base snapshot ID.
   Only offer those matching the selected base. Let the user choose an update or
   disable updates; an automatic choice may select the highest compatible numeric
   version. Do not stack versions. Resolve same-version alternatives explicitly.
4. Register each DLC separately by application ID, content title ID and content
   index. Allow per-DLC enable/disable and version selection. Enforce its minimum
   effective application version; do not treat it as another update.
5. Opening an update/DLC directly should register it or explain the missing game;
   it must not launch as an executable base. A promoted update is already base-kind
   and launches normally, without requiring an original release.

File proximity and file names are not dependency authority. A content registry or
explicit user selections should locate packages; hash verification decides matches.
Installing should not mutate source NUPs or embed DLC into a base.

## 3. Build the effective VFS

Validate the base snapshot and any selected update before creating views. With no
update, the base index is the effective tree. With an update, **its target index is
the effective tree**: resolve literal entries locally, references against the exact
base, and deltas through a checked COPY/INSERT reader. An absent target path stays
absent. A generic directory union/overlay is incorrect for deletions.

Expose read-only `VirtualDir`/`VirtualFile` views for effective `exefs/` and `romfs/`.
Index delta commands for seeking, or materialize them into a verified disk cache;
avoid decompressing/materializing the entire game into RAM. Preserve empty files,
empty directories and case-sensitive names. Keep backing files alive while mounted.
Errors must stop the mount instead of substituting mismatched base content.

## 4. Launch and mount game content

The current `AppLoader_NUP` delegates effective ExeFS to
`AppLoader_DeconstructedRomDirectory` and registers a `RomFSFactory` through
`system.GetFileSystemController().RegisterProcess(...)`. Reuse that architectural
path after v2 resolution, rather than pretending the archive is an encrypted NCA.

Require usable effective `exefs/main` and `exefs/main.npdm`; validate program identity
against the manifest application/content-index mapping. Obtain executable ModuleId,
permissions and loader inputs from the effective files, not stale base TLVs.
Unsupported NPDM revisions/platform features still need explicit emulator support;
a valid container alone does not make a game executable.

Feed the effective RomFS directory through the existing directory-to-RomFS builder
or a corresponding filesystem adapter. A directory tree is not a ready-made binary
RomFS stream. Ensure `ReadRomFS`/filesystem service requests see the same effective
snapshot as the executable. Set up resolution before the secondary program loader
loads modules. Keep normal user mod/cheat layering above this reconstructed game
snapshot, avoiding a second application of the same installed update by PatchManager.

## 5. Metadata, version and DLC services

Use base metadata for the library name, publisher, icons and available control data.
Prefer a supplied validated NACP when usable; otherwise synthesize only the fields
supported by the emulator with documented defaults. Effective runtime control data,
if present, must be handled separately from library presentation. Do not manufacture
an update display version by decoding the NCM integer or reuse an old base version.
Keep saves associated with the application ID, including promoted dumps.

The inspected `AppLoader_NUP` currently warns that add-on mounting is unsupported.
Implement v2 DLC registration in the content-provider/AoC service path separately:
report enabled content IDs/indexes to add-on enumeration and mount each DLC's own
RomFS when requested. Do not overlay DLC into the game's RomFS, and do not assume
registering an encrypted-NCA cache entry is sufficient for plaintext NUP views.
Platform-specific upgrade packs may need further service semantics; package kind
alone does not implement those features.

## 6. Verification before calling support complete

Exercise these scenarios in future implementation tests:

- Existing v1 fixtures retain current loading; v2 and unknown majors never enter v1 parsing.
- Base-only and Ocarina-style promoted input launch with no external base lookup.
- Missing/empty `base/` imports behave identically; incomplete snapshots report errors.
- Bananza-style base plus selected update reconstructs the declared target hashes.
- Wrong snapshot with the same title/version is rejected; update chains are not accepted.
- Added, deleted, renamed, empty and file-to-directory replacement paths resolve exactly.
- COPY/INSERT seeks, truncated streams and corrupted payloads fail predictably.
- Each DLC enumerates/mounts independently; disabling it removes that add-on only.
- Updates/DLC with nonempty metadata are rejected; base metadata remains available.
- Resource limits and path/extent overflow checks handle malformed archives safely.

Until this work exists, Eglantine's v1 loader cannot load v2 NUP files. A future
v2-aware extraction tool could materialize the selected effective ExeFS/RomFS into
the emulator's existing loose-dump path as an interim workflow; no such tool is
provided by this documentation, and it would not by itself implement DLC services.
