# NUP v2

A package for decrypted NX (Nintendo Switch 1/2) dumps, derived
from the NSP layout with the encryption and signatures removed.

Instead of using sigs and encryption it uses hashes that can be 
checked and verifed before being run by an emulator.

## Goal

Take the NSP structure - partition filesystem holding content files plus
content metadata - and strip everything that requires keys:

- no NCA-level crypto (XTS header, key areas, CTR/BKTR sections)
- no RSA signatures (fixed-key, NPDM, ticket/cert chain)
- content hashes kept instead, so an emulator can verify data integrity
  and validity later

Extra things that will exist in v2:

- update dumps for games will be diffed and only the diff stored
 so it can be applied to the exact matching base. If the base folder is absent
 or empty, each complete prepatched update snapshot is packaged as a base game
 instead, with no patch dependency. Each DLC is a separate NUP file.
- application metadata belongs to base NUPs; update and DLC NUPs retain an empty
 metadata section.

## Specification and loading

- [SPEC.md](SPEC.md): v2 wire format, separate base/update/DLC files, exact-base
  deltas, metadata ownership, promoted prepatched dumps, and v1 distinction.
- [EMULATOR_INTEGRATION.md](EMULATOR_INTEGRATION.md): proposed Eglantine loading
  instructions; no loader implementation yet.

## Status

Design draft. v2 uses `NUP\x02`; v1 retains `NUP\x01`.

## v1

The v1 format SPEC (own container, per-file zstd, TOC hashes) and its 
Python reference implementation are on the `meow` branch.
