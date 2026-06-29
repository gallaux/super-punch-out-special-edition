# `spo_sound_mode_incomplete.ips` — Summary

Adds a **SOUND MODE** entry to the title-screen menu and shifts the entire menu UI up by 4 rows. The menu shows NEW GAME / CONTINUE / DATA / CLEAR / SOUND MODE across rows 9/13/17/17/21. This patch is a proof-of-concept only — see **Known bugs** below.

<img width="256" height="224" alt="Super Punch-Out!! (USA) Sound Mode Incomplete_001" src="https://github.com/user-attachments/assets/1ec9d3e1-4bdd-4b89-8156-8ea9b5abf190" />

## What it does

- Shifts the title-screen menu UI up by 4 rows to make room for a 5th entry.
- Adds SOUND MODE text at row 21.
- The cursor highlight, OAM sprite, arrows, and MENU/<<SELECT decoration header all shift consistently.

## Known bugs

**(1) Pressing A on SOUND MODE triggers DATA CLEAR.**

The title-screen cursor uses a snap-to-text system: `$10`/`$11` (DP+`$10`/`$11`) encodes the tilemap position of the text under the cursor, not a simple item index. The D-pad navigation only has 4 hardcoded cursor stops (`$BF80`, `$BF90`, `$AFA0`, `$BFB0`). The SOUND MODE item is visible but the cursor can never reach it — D-pad wraps from CIRCUIT SELECT (`$BFB0`) directly back to NEW GAME, bypassing SOUND MODE. Pressing A while "on SOUND MODE" visually is actually pressing A while the cursor is on CIRCUIT SELECT, which falls through to the DATA CLEAR handler.

To fix: determine the correct cursor value for the SOUND MODE row empirically (Mesen WRAM watch on `$0C10`, navigate DOWN past CIRCUIT SELECT), then add D-pad navigation cases and an A-button dispatch case for that value.

**(2) Name-entry screen behaves incorrectly after selecting NEW GAME or CONTINUE.**

Adding a 4th title-screen item causes the stub at `$01:B194` to return with X = `$0003`, which the caller's `STX.b $14` writes as `$14 = 3`. The WRAM variable `$14` (DP+`$14` = `$0C14`) is used as the cursor upper bound by multiple screens — name entry reads it and navigates into out-of-bounds memory, corrupting the cursor and causing crashes or incorrect behavior. This is a fundamental consequence of adding an item to this particular menu.

**(3) The Sound Library (`CODE_00913B`) cannot be called from within the running game.**

`CODE_00913B` is the Sound Library entry point (SNES `$00:913B`, file `0x0113B`). It is a peer of the title screen in the game's state machine — not a subroutine. It sets up its own VRAM, PPU registers, NMI, and runs its own frame loop. Its exit path (`JMP CODE_009CB6`) is a full state-machine teardown that routes back to the boot loop, not a return to caller. Calling it mid-game produces garbled graphics or a black screen because the title-screen NMI handler keeps running and fights the Sound Library's PPU setup.

The Sound Library can only be launched cleanly after a full hardware reset sequence. See [doc/incomplete/AUDIO_MODE_INCOMPLETE.md](AUDIO_MODE_INCOMPLETE.md) for the complete investigation.

## How it works

The title-screen cursor init at `CODE_01DDD8` sets `$10/$11` to an initial snap position, then `CODE_01CAAE` scans the BG3 tilemap for text and snaps the cursor to the first item. Subsequent D-pad moves update `$10/$11` directly to hardcoded cursor stop values. The A-button handler at `CODE_01E0A3` dispatches on the full 16-bit `$10/$11` word.

The UI shift is accomplished by changing: the highlight bar base (row 9), the cursor Y base (OAM `$0B1F2`), the decoration tilemap position, and the arrow positions — all must be updated in lockstep.

The layout sub-table at `$0D:FE52` drives the BG1 tilemap positions for all 5 items. The dispatch stub at `$00:F61A` inlines `CODE_01CA71` (highlight system init) and sets `LDX #$0003` so the caller's `STX.b $14` writes item count 3 (4 items, 0-indexed).

## Patch records

11 records (plus the SNES header checksum):

| File offset | Bytes | Effect |
|---|---|---|
| `0x0B183` | `00` | Don't disable item 0 (NEW GAME) on no-save path |
| `0x0B194` | `22 1A F6 00` | JSL `$00:F61A` — replaces `JSR $CA71`, clobbers `$B197` |
| `0x0B198` | `EA EA` | NOP NOP — RTL landing pad |
| `0x0B19E` | `02` | Highlight base hi: row 9 start |
| `0x0B1BD` | `58` | Decoration tilemap hi: visual row 2 |
| `0x0B1C6` | `00` | Left arrow hi |
| `0x0B1C9` | `00` | Right arrow hi |
| `0x0B1F2` | `47` | Sprite Y base: row 9 |
| `0x6A3DA` | `52 FE` | DA3DA entry 0 layout pointer → `$0D:FE52` |
| `0x6FE52` | 33 bytes | Layout sub-table (8 entries + terminator) |
| `0x761A` | 31 bytes | Dispatch stub: inline `CODE_01CA71` + `STZ $23` + `LDX #$0003` + `RTL` |

**Layout sub-table (`$0D:FE52`, 33 bytes):**

```
01 1C 58 52   NEW GAME    row 9  col 12  ($5258)
02 1C 58 53   CONTINUE    row 13 col 12  ($5358)
03 1C 56 54   DATA        row 17 col 11  ($5456)
15 1C 62 54   CLEAR       row 17 col 17  ($5462)
4B 1C 56 55   SOUND       row 21 col 11  ($5556)
0C 1C 64 55   MODE        row 21 col 18  ($5564)
14 20 94 58   MENU        header         ($5894)
05 20 9C 58   <<SELECT    header         ($589C)
00            terminator
```

## Free space consumed

- **`$0D:FE52–$0D:FE72`** (33 B): layout sub-table. Within the 46-byte free gap at `$0D:FE52–$0D:FE7F`.
- **`$00:F61A–$00:F638`** (31 B): dispatch stub. Within `UNK_00F5D0` (`$00:F5D0–$00:FF8F`).

## Patch in asm form

```asm
; spo_sound_mode_incomplete
; Adds a SOUND MODE entry to the title-screen menu, shifting the UI up 4 rows.
; Proof-of-concept only — known bugs documented above.

;==============================================================================
; Title-screen menu init tweaks
;==============================================================================

org $01B183
    db $00          ; was: $01 — don't disable NEW GAME on no-save path

; Replace JSR $CA71 with JSL to dispatch stub in bank $00.
; The JSL's 4th byte overwrites $B197 (opcode of LDX #$0002).
; The stub compensates by inlining CA71 and ending with LDX #$0003,
; so the caller's STX.b $14 writes item count 3 (4 items, 0-indexed).
org $01B194
    JSL $00F61A     ; was: JSR $CA71 — clobbers $B197

org $01B198
    NOP : NOP       ; RTL landing pad (was: low bytes of LDX #$0002)

org $01B19E
    db $02          ; Highlight base hi: $0356 → $0256 (row 9), was $03

org $01B1BD
    db $58          ; Decoration tilemap hi: $5994 → $5894 (visual row 2), was $59

org $01B1C6
    db $00          ; Left arrow hi: $0190 → $0090, was $01

org $01B1C9
    db $00          ; Right arrow hi: $01AE → $00AE, was $01

org $01B1F2
    db $47          ; OAM cursor Y base: row 9, was $67

;==============================================================================
; Layout pointer: DA3DA entry 0 → new sub-table at $0D:FE52
;==============================================================================

org $0DA3DA
    dw $FE52        ; was: $AD48 (original title-screen layout)

;==============================================================================
; Layout sub-table at $0D:FE52 (33 bytes)
;==============================================================================

org $0DFE52
    db $01,$1C,$58,$52   ; NEW GAME    row 9  col 12
    db $02,$1C,$58,$53   ; CONTINUE    row 13 col 12
    db $03,$1C,$56,$54   ; DATA        row 17 col 11
    db $15,$1C,$62,$54   ; CLEAR       row 17 col 17
    db $4B,$1C,$56,$55   ; SOUND       row 21 col 11
    db $0C,$1C,$64,$55   ; MODE        row 21 col 18
    db $14,$20,$94,$58   ; MENU        header
    db $05,$20,$9C,$58   ; <<SELECT    header
    db $00               ; terminator

;==============================================================================
; Dispatch stub at $00:F61A (31 bytes)
; Inlines CODE_01CA71 (highlight system init), then sets item count.
; Called via JSL from $01:B194; returns with X=$0003 so caller's STX.b $14
; writes $14=3 (4 items, 0-indexed).
;==============================================================================

org $00F61A
    STZ $23                     ; persist flag
    LDX #$0000 : STX $12        ; CODE_01CA71 inline
    LDX #$0040 : STX $74        ; highlight D0 start
    LDX #$0100 : STX $49        ; highlight step per item
    LDX #$1400 : STX $0C
    LDX #$0C00 : STX $0E
    LDX #$0003                  ; item count (compensates clobbered $B197 LDX #$0002)
    RTL
```

(Plus the standard 4-byte SNES header checksum update at file `0x7FDC`.)

## Compatibility

- **Apply on top of**: original `Super Punch-Out!! (USA).sfc` ROM (MD5 `97fe7d7d2a1017f8480e60a365a373f0`)
- **Bundled into**: nothing — incomplete patch
- **Conflicts with**: nothing in this repo
- **Cheat-code compatibility**: unaffected

## See also

- [doc/incomplete/AUDIO_MODE_INCOMPLETE.md](AUDIO_MODE_INCOMPLETE.md) — Sound Library entry point, why it can't be called mid-game, and next steps for wiring it up
- [doc/TECHNICAL.md §6](../TECHNICAL.md) — `spo_sound_mode_incomplete.ips` section with deeper technical context
- [doc/TECHNICAL.md §7](../TECHNICAL.md) — Free space map (bank `$0D` and bank `$00`)
