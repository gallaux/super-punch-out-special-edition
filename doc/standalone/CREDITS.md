# `spo_credits.ips` — Summary

Adds a fourth **CREDITS** entry to the Records View select screen. Selecting CREDITS launches the game's ending-cutscene credits roll. After the credits finish the game stays on the final screen and requires reset — that's the original cutscene's terminal behavior, not introduced by this patch.

## What it does

The Records View select screen originally has three entries: BEST TIME, BEST SCORE, PERSONAL RECORDS — which list the three Championship circuits' records (Minor, Major, World). This patch:

- Adds a fourth entry **CREDITS** at row 22.
- Tightens the layout so the four entries sit at rows 10/14/18/22 with the header at row 2 (matching the Championship circuit-select layout).
- Hooks the A-button confirm dispatch so selecting CREDITS launches the ending credits cutscene (`JML $00:B29C`).

## Patch records

10 records, 119 bytes total (plus the SNES header checksum):

| File offset | Bytes | Effect |
|---|---|---|
| `0x0BFA6`, `0x0BFAB`, `0x0BFDB`, `0x0BFE9`, `0x0BFF2`, `0x0BFF5`, `0x0C022` | 1-2 | Init-routine bytes: highlight bar base, palette-cycle bases, arrow positions, cursor sprite anchor — all shifted in lockstep to move the layout from rows 12/16/20/24 → rows 10/14/18/22 |
| `0x0C9CA` | 4 | Replace start of confirm dispatcher `CODE_01C9CA` with `JML $0D:FB39` (per-fighter dispatch stub) |
| `0x6A3EC` | 2 | DA3DA[`$12`] redirect to `$0D:FB0C` (new screen descriptor) |
| `0x6A47A` | 2 | DA332[`$A4`] redirect to `$0D:FB31` (CREDITS tile string) |
| `0x6AE62`, `0x6AE66`, `0x6AE6A` | 1 each | Header descriptor row shifts (RECORDS / VIEW / MODE) |
| `0x6FB0C` | 37 | New screen layout sub-table |
| `0x6FB31` | 8 | "CREDITS" tile string |
| `0x6FB39` | 41 | A-button dispatch stub (handles 4 cursor indices, including the new CREDITS path) |

## Free space consumed

- **Bank `$0D`**: 86 bytes at `$0D:FB0C-$0D:FB61` (descriptor + tile string + dispatch stub)

## Compatibility

- **Apply on top of**: bare `spo.sfc` (MD5 `97fe7d7d2a1017f8480e60a365a373f0`)
- **Bundled into**: `spo_special_edition_v1.5.ips`
- **Conflicts with**: `spo_sound_mode_ui_incomplete.ips` (an experimental patch in `patches/incomplete/`) — both want the same `$0D:FB0C+` free-space region. Don't combine.
- **Cheat-code compatibility**: unaffected

## See also

For the full layout-shift rationale (why the cursor anchor `$53` is the load-bearing one — the `D082` text-snap scan loop), the descriptor sub-table layout, and the A-button dispatch stub assembly, see the `spo_credits.ips` entry in [doc/TECHNICAL.md § 6](../TECHNICAL.md).
