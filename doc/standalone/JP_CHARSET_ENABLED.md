# `spo_jp_charset_enabled.ips` — Summary

Makes the Japanese character set L/R-cycling always active on the name-entry screen, with no hidden button combo required. The screen still opens on the Western set by default — players who don't want Japanese characters won't notice any change. L and R cycle through three sets (Japanese-1, Japanese-2, Western), exactly like the Japanese version of the game.

## What it does

The original US/EUR ROM has a hidden mode where L/R cycles between three character sets, activated by holding a specific button combo (X+A or Start+X+A) at the New Game cursor. The Japanese version has this mode on by default. This patch removes the gate so the cycling is always available.

## Patch records

3 records, 3 bytes total (plus the SNES header checksum) — all single-byte tweaks at SNES `$01:DF83-DF95` (file `0xDF83-0xDF95`):

| File offset | Old | New | Effect |
|---|---|---|---|
| `0xDF83` | `F0` (BEQ) | `80` (BRA) | Always take the Japanese-mode-enabled branch |
| `0xDF92` | `9C` | `B7` | `JSR $D79C` → `JSR $D7B7` — load full Western UI tiles instead of just swapping the Japanese set (avoids corrupted tilemap on US ROM) |
| `0xDF95` | `00` | `02` | Initial `$1F = $02` (Western set index, matches the `$D7B7` load) |

## Free space consumed

None. All three records are in-place edits.

## Compatibility

- **Apply on top of**: bare `spo.sfc` (MD5 `97fe7d7d2a1017f8480e60a365a373f0`)
- **Bundled into**: `spo_special_edition_v1.5.ips`
- **Conflicts with**: nothing in this repo
- **Cheat-code compatibility**: unaffected

## See also

For the byte-by-byte explanation of the second tweak (why we redirect to `JSR $D7B7` instead of just enabling the Japanese path), see the `spo_jp_charset_enabled.ips` entry in [doc/TECHNICAL.md § 6](../TECHNICAL.md).
