# `spo_versus_hack.ips` — Summary

The core hack of this repo. Adds VERSUS MODE to the Mode Select menu, lets either controller pick on the opponent-select screen, and disables the Special Circuit security checksum lock.

## What it adds

- **VERSUS MODE on Mode Select** — 2-player matches launch directly from the menu, no secret button combo required.
- **Player 2 controls the opponent** in Versus matches — the same mechanic as the 2022-discovered secret mode, wired in permanently when entering from Versus.
- **Correct in-game screen labels** — opponent-select shows "VERSUS MODE" / "< PLAYER 2 SELECT >" when entering from Versus; Time Attack is unchanged.
- **Either controller can pick on opponent-select** — Controller 2's D-pad / A / Start mirror Controller 1 only on the Versus opponent-select screen, so P2 can choose their own fighter.
- **Special Circuit always unlocks correctly** — the World Circuit completion checksum that locks Special on save states / emulators / patched ROMs is bypassed.

## Patch records

This is a substantial patch — ~700 bytes of edits across banks `$00`, `$01`, and `$0D`. Records [1] through [34] in TECHNICAL.md (with [33b] for the mode-flag rewrite hook) cover every byte change with full reasoning: the Mode Select menu rewrite, opponent-select reuse, P2 boss-control hook, post-match routing, table-hide stubs, and the per-frame P2-mirrors-P1 input merge.

## Free space consumed

- **Bank `$01`**: ~50 bytes between `$8018-$8029` and `$FFB0-$FFDF` (consumed by header stub, trampolines, and dispatch handlers); 0 bytes free remain in bank `$01` after this patch (except `$01:FFE0-FFE3`, 4 bytes free).
- **Bank `$0D`**: ~600 bytes across multiple stubs and tables (Mode Select layout, VERSUS opponent-select descriptor, table-hide stubs, P2 control dispatch, post-match routing, P2-mirrors-P1 merge stub).

See [doc/TECHNICAL.md § 7](../TECHNICAL.md#7-free-space-map) for the authoritative bank-level free-space breakdown.

## Compatibility

- **Apply on top of**: bare `spo.sfc` (MD5 `97fe7d7d2a1017f8480e60a365a373f0`)
- **Bundled into**: `spo_special_edition_v1.5.ips`
- **Conflicts with**: `spo_disable_security_checksum.ips` writes the same value to file `0x003C23` (the Special Circuit lock-bypass byte) — applying both is a harmless no-op double-write.
- **Cheat-code compatibility**: unaffected

## See also

For the full reverse-engineering reference — every record, the Mode Select state machine architecture, the VERSUS opponent-select reuse strategy, the P2-mirrors-P1 input merge stub, the post-match routing, free-space inventory, and key-address quick reference — see [doc/TECHNICAL.md](../TECHNICAL.md), particularly:

- Section 2: The secret 2-player mode
- Section 3: Title screen state machine
- Section 4: Mode Select architecture
- Section 5: Versus Hack — patch-by-patch breakdown (records [1]–[34])
- Section 7: Free space map
