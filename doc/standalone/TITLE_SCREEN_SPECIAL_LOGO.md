# `spo_title_screen_special_logo.ips` — Summary

Cosmetic: writes the text **SPECIAL EDITION** into the title screen below the main game logo. Uses the standard menu font (palette 2, priority 1) and renders every time the title screen background is initialized.

## What it does

Hooks the title-screen BG init routine `CODE_08C8AB` to call a stub that writes 14 tiles into the BG3 tilemap buffer at row 12, cols 6-22 (with a one-tile gap between SPECIAL and EDITION).

## Patch records

2 records, 115 bytes total (plus the SNES header checksum):

| File offset | Bytes | Effect |
|---|---|---|
| `0x44906` | 6 | Hook in `CODE_08C8AB`: `LDY #$1801; STY $4300` → `JSL $0D:FBDF; NOP; NOP` (the 2 displaced instructions are re-executed at the top of the stub) |
| `0x6FBDF` | 109 | Stub at `$0D:FBDF` — writes "SPECIAL EDITION" tile-by-tile into the BG3 tilemap buffer at WRAM `$5000`+ |

Tile entry format: `$28XX` = priority 1, palette 2, tile index XX. The space between SPECIAL and EDITION (col 13) is left unwritten — the background fill tile at that position acts as the gap.

## Free space consumed

- **Bank `$0D`**: 109 bytes at `$0D:FBDF-FC4B` (inside the disassembly's `%InsertGarbageData($0DFA69, ...)` zone)

## Compatibility

- **Apply on top of**: bare `spo.sfc` (MD5 `97fe7d7d2a1017f8480e60a365a373f0`)
- **Bundled into**: `spo_special_edition_v1.5.ips`
- **Conflicts with**: nothing in this repo
- **Cheat-code compatibility**: unaffected

## See also

For the full stub assembly, the BG3 tilemap address formula, and a discussion of the title-screen BG layout quirk (the `$5000` block renders as BG3 despite `$2109` settings suggesting otherwise), see the `spo_title_screen_special_logo.ips` entry in [doc/TECHNICAL.md § 6](../TECHNICAL.md) and [§ 10 (Title screen BG layout quirk)](../TECHNICAL.md#10-title-screen-bg-layout-quirk).
