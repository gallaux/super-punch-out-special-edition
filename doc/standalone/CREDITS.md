# `spo_credits.ips` — Summary

Adds a fourth **CREDITS** entry to the Records View select screen. Selecting CREDITS launches the game's ending-cutscene credits roll. After the credits finish the game stays on the final screen and requires reset — that's the original cutscene's terminal behavior, not introduced by this patch.

## What it does

The Records View select screen originally has three entries: BEST TIME, BEST SCORE, PERSONAL RECORDS — which list the three Championship circuits' records (Minor, Major, World). This patch:

- Adds a fourth entry **CREDITS** at row 22.
- Tightens the layout so the four entries sit at rows 10/14/18/22 with the header at row 2 (matching the Championship circuit-select layout).
- Hooks the A-button confirm dispatch so selecting CREDITS launches the ending credits cutscene (`JML $00:B29C`).
- **Fixes a no-record artifact** (see below) so the credits roll looks correct even when launched before all opponents have been beaten.

## No-record artifact fix

When the credits roll is launched before all 16 opponents have been beaten, fighters with no saved record would display a garbage Japanese glyph in the `YOUR BEST` time slot instead of a valid time. This is a cosmetic artifact only.

**Root cause:** The credits animation engine in bank `$01` runs a per-frame state machine (`CODE_01DAC4`) that reads each fighter's time digits from SRAM via `CODE_01D52B`. `CODE_01D52B` reads 5 SRAM bytes at fighter index × 5 and writes them as digit tiles to `$7E40B0`–`$7E40BE` (BG3 tilemap, DMA'd to VRAM `$7000`+ each frame). For an unbeaten fighter those SRAM bytes are uninitialized (`$FF`), and tile `$FF` in the credits font is a Japanese glyph. In the original game this path is only reached after World Circuit completion (all fighters beaten), so it was never a problem.

**Fix:** a 3-byte hook at `$01:DB27` (file `0x0DB27`) redirects `JSR CODE_01D52B` to a stub at `$01:FEC2` (file `0x0FEC2`). The stub calls `CODE_01D52B` normally, then checks whether `$7E40B0` is `$FF` and replaces it with `$03` (tile "3") if so. Tile `$03` is the value Nintendo's own no-record path writes at `CODE_01DAC4` — the leading digit of the hardcoded placeholder time `3'00"00`, which is the duration of a full round.

## Patch records

12 records, 137 bytes total (plus the SNES header checksum):

| File offset | Bytes | Effect |
|---|---|---|
| `0x0BFA6`, `0x0BFAB`, `0x0BFDB`, `0x0BFE9`, `0x0BFF2`, `0x0BFF5`, `0x0C022` | 1-2 | Init-routine bytes: highlight bar base, palette-cycle bases, arrow positions, cursor sprite anchor — all shifted in lockstep to move the layout from rows 12/16/20/24 → rows 10/14/18/22 |
| `0x0C9CA` | 4 | Replace start of confirm dispatcher `CODE_01C9CA` with `JML $0D:FB39` (per-fighter dispatch stub) |
| `0x0DB27` | 3 | Hook: `JSR CODE_01D52B` → `JSR $FEC2` (no-record artifact fix) |
| `0x0FEC2` | 18 | Stub: call `CODE_01D52B`, check `$7E40B0 == $FF`, overwrite with `$03` if so |
| `0x6A3EC` | 2 | DA3DA[`$12`] redirect to `$0D:FB0C` (new screen descriptor) |
| `0x6A47A` | 2 | DA332[`$A4`] redirect to `$0D:FB31` (CREDITS tile string) |
| `0x6AE62`, `0x6AE66`, `0x6AE6A` | 1 each | Header descriptor row shifts (RECORDS / VIEW / MODE) |
| `0x6FB0C` | 37 | New screen layout sub-table |
| `0x6FB31` | 8 | "CREDITS" tile string |
| `0x6FB39` | 41 | A-button dispatch stub (handles 4 cursor indices, including the new CREDITS path) |

## Free space consumed

- **Bank `$0D`**: 86 bytes at `$0D:FB0C–$0D:FB61` (descriptor + tile string + dispatch stub)
- **Bank `$01`**: 18 bytes at `$01:FEC2–$01:FED3` (no-record artifact fix stub)

## Compatibility

- **Apply on top of**: bare `spo.sfc` (MD5 `97fe7d7d2a1017f8480e60a365a373f0`)
- **Bundled into**: `spo_special_edition_v1.5.ips`
- **Conflicts with**: `spo_sound_mode_ui_incomplete.ips` (an experimental patch in `patches/incomplete/`) — both want the same `$0D:FB0C+` free-space region. Don't combine.
- **Cheat-code compatibility**: unaffected

## See also

For the full layout-shift rationale (why the cursor anchor `$53` is the load-bearing one — the `D082` text-snap scan loop), the descriptor sub-table layout, and the A-button dispatch stub assembly, see the `spo_credits.ips` entry in [doc/TECHNICAL.md § 6](../TECHNICAL.md).
