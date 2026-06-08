# `spo_super_macho_man_fix.ips` — Summary

Fixes the original ROM's "SUPER MACHOMAN" → "SUPER MACHO MAN" typo across every screen in the game (TIME ATTACK and VERSUS opponent select, PERSONAL RECORDS, BEST TIME table, pre-fight profile...). The corrected spelling matches the US manual and every other appearance of the character — Punch-Out!! (1987, NES) and Punch-Out!! (2009, Wii).

## What's involved

The fix touches four distinct screens with two text encodings, two pointer tables, and a per-fighter code patch. The pre-fight profile screen has a fixed layout slot that the corrected longer name doesn't fit into — Nintendo originally avoided this by collapsing "MACHO MAN" into "MACHOMAN". The patch resolves it by shifting Super Macho Man's portrait sprite 4 pixels right (only on his profile screen) to give the name the room it needs.

## Patch records

9 records, 59 bytes total (plus the SNES header checksum):

| File offset | Bytes | Purpose |
|---|---|---|
| `0x043B53` | 2 | Banner-table pointer redirect (fighter 11 → `$08:D926`) |
| `0x045926` | 19 | New banner entry "SUPER MACHO MAN" |
| `0x0452AB` | 5 | Profile-renderer hook (`LDX #$38A8; STX $7A` → `JSL $0D:FD87; NOP`) |
| `0x06A392` | 2 | DA332 redirect (entry `$30` → `$0D:FD69`) |
| `0x06AE8F` | 1 | PR descriptor: SUPER WRAM-lo column shift |
| `0x06AE93` | 1 | PR descriptor: MACHO MAN WRAM-lo column shift |
| `0x06AF73` | 1 | TA title card: MACHO MAN WRAM-lo column shift |
| `0x06FD69` | 10 | New menu-font string "MACHO MAN" (length-prefixed) |
| `0x06FD87` | 18 | Per-fighter portrait X-shift stub |

## Free space consumed

- **Bank `$08`**: 19 bytes at `$08:D926`
- **Bank `$0D`**: 28 bytes total (10 at `$0D:FD69`, 18 at `$0D:FD87`)

All allocations sit inside disassembly-documented `%InsertGarbageData` regions, verified unreferenced by ROM-wide search.

## Compatibility

- **Apply on top of**: bare `spo.sfc` (MD5 `97fe7d7d2a1017f8480e60a365a373f0`)
- **Bundled into**: `spo_special_edition_v1.5.ips`
- **Conflicts with**: nothing in this repo
- **Cheat-code compatibility**: unaffected

## See also

For the full reverse-engineering details — banner-font letter map, screen-by-screen layout reasoning, profile-renderer flow, the per-fighter portrait stub assembly, and the explanation of why `$EF` and `$FF` behave as different "space" bytes — see [doc/TECHNICAL.md](../TECHNICAL.md) section 6 entry for `spo_super_macho_man_fix.ips`.
