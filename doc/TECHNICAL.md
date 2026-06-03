# Super Punch-Out!! Special Edition — Technical Reference

This document covers the full reverse-engineering and implementation details behind the patches described in [README.md](../README.md). It is intended for ROM hackers, the technically curious, and anyone who wants to continue this work.

The main patch is distributed as [`spo_versus_hack.ips`](../patches/standalone/spo_versus_hack.ips).

---

## Table of contents

1. [ROM structure](#1-rom-structure)
2. [The secret 2-player mode](#2-the-secret-2-player-mode)
3. [Title screen state machine](#3-title-screen-state-machine)
4. [Mode Select architecture](#4-mode-select-architecture)
5. [Versus Hack — patch-by-patch breakdown](#5-versus-hack--patch-by-patch-breakdown)
6. [Standalone patches — technical detail](#6-standalone-patches--technical-detail)
7. [Free space map](#7-free-space-map)
8. [Text encoding and screen descriptor system](#8-text-encoding-and-screen-descriptor-system)
9. [Key addresses quick reference](#9-key-addresses-quick-reference)

---

## 1. ROM structure

**Format:** LoROM (32 KB banks mapped at SNES $8000–$FFFF).

**File offset formula:**
```
file_offset = bank × 0x8000 + (SNES_addr − 0x8000)
```

Examples:
```
SNES $01:B194 → file 0x0B194
SNES $01:E16F → file 0x0E16F
SNES $0D:A3DA → file 0x6A3DA
SNES $0D:FA69 → file 0x6FA69
SNES $02:8408 → file 0x10408
```

**Direct Page register** is set to `$0C00` by `CODE_01B0F0` at title-screen entry and maintained throughout. All DP-relative references (e.g. `$10`, `$11`, `$23`) resolve to WRAM `$0C10`, `$0C11`, `$0C23`, etc.

**Original ROM MD5:** `97fe7d7d2a1017f8480e60a365a373f0`

---

## 2. The secret 2-player mode

The game contains a dormant 2-player mode. The P2 boss-control flag lives at WRAM `$0030`; when it equals `$08`, Controller 2 drives the opponent. The code that sets this flag lives at SNES `$81:80CC`:

```asm
$81:80CC  LDA $0607      ; game mode flag
          CMP #$02       ; is this a VERSUS session?
          BNE +$10       ; skip if not
          ...
$81:80D4  BNE +$99       ; skip flag-set if button combo not held
$81:80D6  LDA #$08
          STA <$30       ; set P2 boss-control flag
          LDA #$80
          STA <$3D       ; set supplemental flag
```

The `BNE +$99` at `$81:80D4` (file `0x80D4`) nearly always branches past the flag-set because it guards against a specific button combination on the character-info screen. The Versus route in this hack bypasses the info screen entirely, so without a fix this code never runs and the CPU controls both fighters.

The Game Genie code `DD62-DF00` and GameShark code `180D5:00` both fix this by zeroing the branch offset — the same byte this patch changes.

**Discovery:** this mechanic was first publicly documented in 2022.

---

## 3. Title screen state machine

Entry point: `CODE_01B0F0` (file `0xB0F0`) — sets Direct Page = `$0C00`.

The machine dispatches on two nested state variables, `$01` and `$03`, via indexed indirect jump tables:

```
CODE_01B12D: LDX $01 → JMP (DATA_01B132, X)
  $01=0 → CODE_01B138: LDX $03 → JMP (DATA_01B13D, X)
    $03=$00 → CODE_01B14D   menu init
    $03=$0A → CODE_01B24C   menu interaction (cursor lives here)
    $03=$0C → CODE_01B389   circuit select
    $03=$0E → CODE_01B5C3   data clear
  $01=2 → CODE_01B8A3       post-confirm (new game / circuit entry)
  $01=4 → CODE_01BB61       Mode Select + Championship sub-machine
```

**Interactive menu loop** (`CODE_01DFB4`, state `$03=$0A`, sub-state `$05=2`): calls `CODE_01D8F7` for button input, then dispatches:

```
A-button → CODE_01E0A3
DPad     → CODE_01E123 → up/down/left/right handlers
Start    → CODE_01E005
```

**Cursor position encoding:** the cursor is a 16-bit word stored across DP+`$10` (X, lo byte) and DP+`$11` (Y, hi byte). The A-button handler reads `LDX $10` (16-bit) and compares it against known cursor values:

| Value  | Menu item      |
|--------|----------------|
| `$BF80`| NEW GAME       |
| `$BF90`| CONTINUE       |
| `$AFA0`| DATA CLEAR     |
| `$BFB0`| circuit select |

---

## 4. Mode Select architecture

### Design decision: duplicating Time Attack rather than using the secret VS menu

The game contains a dormant VS menu that is never reachable through normal play (`CODE_00B29C` and surrounding code). Early prototypes of this hack attempted to launch directly into that hidden flow. It was abandoned for two reasons:

1. **Not part of the normal game loop.** The secret VS menu sits outside the state machine that governs the title screen and Mode Select. Entering it from Mode Select context causes a corrupt first frame because the VRAM state, CGRAM palette, and engine sub-states expected by that code are never set up by the normal flow. Recovering to a clean game loop after the match also proved unreliable through this path.

2. **The Time Attack path is clean and already correct.** The Time Attack opponent-select screen is a fully functional, menu-driven character picker that sits squarely inside the normal game-state machine. By duplicating its screen descriptor and routing Versus through the same state machine path (with `$0607=$02` as a mode flag), Versus gets a working opponent-select screen, clean back-out navigation, and a correct post-match return with no extra work.

The result is that Versus mode reuses the Time Attack infrastructure throughout, differentiated only by the `$0607` flag. The conditional stubs (trampoline 1, trampoline 2, the header stub at `$8018`, the back-out clear stub) all key off this flag to give Versus its own behavior while leaving Time Attack completely unchanged.

**Post-match behavior:** when a Versus match ends, the game returns to the intro cutscene that plays before the title screen — the same behavior observed with the original secret VS mode. This is a consequence of sharing the Time Attack state machine, which routes back to the top-level game loop from that point. Match times are not recorded to Time Attack records.

Reached via `$01=4`, `$03=2`. The sub-machine dispatches on `$05` via `DATA_01BC76`:

```
$05=0 → $BC82   init (sets disable flags, renders layout)
$05=4 → $BE8C   interactive input handler
$05=8 → $C905   confirm dispatch (maps cursor index → action)
```

### Layout system

Screen text is driven by a two-level table chain rooted at `DATA_0DA3DA` (file `0x6A3DA`). Entry 2 of that table is a bank-relative pointer to the Mode Select layout sub-table.

**Layout sub-table record format** (4 bytes each):
```
[text_idx][attr][WRAM_lo][WRAM_hi]
```
- `text_idx`: index into `DATA_0DA332` (text string pointer table)
- `attr`: `$1C` = selectable (highlight when cursor is on it), `$20` = always visible
- `WRAM lo/hi`: target address in the WRAM tilemap buffer

Terminator: `$00`.

**Tilemap address formula:**
```
WRAM_addr = $5000 + row × $40 + col × 2
```

### Cursor highlight system

Two independent systems — both must be consistent:

**System 1 — highlight renderer** (`CODE_01CC79`): loops `$14 + 1` times (where `$14` = item count − 1). For each item, reads the disable flag at DP+`$20+x`; if zero, renders the highlight tile at the current D0 position (advances by `$0100` per item). Purely visual.

**System 2 — interactive cursor** (`CODE_01DDD8` onward): tracks cursor position in DP+`$10`/`$11`, updates OAM sprite position, handles DPad input.

### Confirm dispatch — `CODE_01C905`

After the player confirms a selection, `$07E3` holds the 0-based cursor item index. `CODE_01C905` maps it to a state transition:

Original (4-item menu):
```
$07E3=0 → CHAMPIONSHIP  (STZ $03/$05; JMP $C752 → INC $01×2)
$07E3=1 → TIME ATTACK   (STA $0607=#$01; STZ $05; JMP $C74C → INC $03×2)
$07E3=2 → RECORDS VIEW  (LDA #$04; STA $03; BRA $C951)
$07E3=3 → BUTTON SETTINGS (LDA #$06; STA $03; BRA $C951)
```

Our patch inserts a JMP at `$C90E` to redirect all indices ≥ 1 through a 5-item stub in free space.

---

## 5. Versus Hack — patch-by-patch breakdown

All records are applied to the original unmodified ROM. Records are listed in file-offset order.

---

### [1] File `0x003C23` — Special Circuit lock bypass (2 bytes)

```
Old: 05 D5   ORA $D5
New: A9 00   LDA #$00
```

The World Circuit completion checksum prevents the Special Circuit from unlocking when using save states, emulators (SNES Classic, Switch NSO), or patched ROMs. At SNES `$00:BC23`, this routine stores its result to SRAM `$700080` (Special Circuit lock flag). Replacing `ORA $D5` with `LDA #$00` disables that checksum check — A is forced to zero before the store, so the lock flag is always written as unlocked regardless of checksum result.

---

### [2] File `0x008018` — Conditional VERSUS/TIME ATTACK header stub (18 bytes)

```asm
48           PHA               ; save Y-arg (A = TYA result = $0C or $0E)
AD 07 06     LDA $0607         ; read mode flag
C9 02        CMP #$02          ; VERSUS?
F0 04        BEQ +4            ; branch BEFORE PLA (avoids PLA clobbering Z flag)
68           PLA               ; non-VERSUS: restore Y-arg
4C FC D1     JMP $D1FC         ; tail-call descriptor renderer
68           PLA               ; VERSUS: restore Y-arg
A9 A4        LDA #$A4          ; override descriptor index to VERSUS descriptor
4C FC D1     JMP $D1FC         ; tail-call renderer
```

Called from `CODE_01BD0D` (file `0x0BD0F`), which originally called `JSR $D1FC` directly. Redirected to `JSR $8018`.

**Critical note:** PLA on SNES updates the N and Z processor flags from the pulled value — not from the preceding CMP. The BEQ must come before PLA, or it always sees Z=0 (since the pulled value `$0C`/`$0E` is always nonzero). An earlier iteration had this wrong; the current implementation has the BEQ correctly placed before PLA.

This stub sits in the dead-loop jump-table slots at `$8018`/`$801B`/`$801E` (self-referencing JMP dead loops, never called) plus 9 bytes of confirmed free space at `$8021–$8029`. Bank `$01` has zero confirmed-free bytes remaining after this stub.

---

### [3] File `0x0080D4` — P2 control: BPL→BRA (1 byte)

```
Old: 10   BPL
New: 80   BRA
```

Makes the branch that guards `CODE_01CC79`'s first iteration unconditional, ensuring the highlight-cursor loop always runs. (This is a separate 1-byte fix from the P2 boss-control patch below.)

---

### [4] File `0x0080D5` — P2 boss-control flag: always set in VERSUS (1 byte)

```
Old: 99   BNE +$99  (skip flag-set if button combo not held)
New: 00   BNE +$00  (never skip)
```

At SNES `$81:80D4`. The `CMP #$02` guard at `$81:80CB` ensures `$30=$08` is only written when `$0607=$02` (VERSUS mode), so TIME ATTACK and all other modes are completely unaffected.

---

### [5] File `0x008457` — Trampoline 1: conditional TA disable (12 bytes)

```asm
D0 09        BNE +9     ; if $700010,X ≠ 0 (has progress): skip to RTS
AD 07 06     LDA $0607
C9 02        CMP #$02
F0 02        BEQ +2     ; if VERSUS: skip to RTS (leave TA enabled)
E6 22        INC $22    ; TA + no progress: disable TIME ATTACK
60           RTS
```

Called via `JSR $8457` at `$01:BBE2` (replaces the original `BNE +02; INC $22` sequence). Ensures TIME ATTACK is only greyed out when there is no Championship progress AND we are not currently in VERSUS mode (since entering Versus then backing out would otherwise leave `$22=0`, making TA appear available even on a fresh save).

---

### [6] File `0x008463` — Back-out clear stub (6 bytes)

```asm
9C 07 06     STZ $0607   ; clear mode flag on back-out
4C 66 C7     JMP $C766   ; continue to original epilog
```

When the player backs out of opponent select, `BEEE: JMP $C766` is redirected here first. Without this, `$0607` stays `=$02` (VERSUS) after back-out, causing the conditional trampoline to skip `INC $22` on re-entry — making TIME ATTACK spuriously appear available on fresh saves and then deadlocking when selected.

---

### [7] File `0x00BB93` — Item count: 4 → 5 (1 byte)

```
Old: 03   LDX #$0003
New: 04   LDX #$0004
```

At SNES `$01:BB93`, sets `$14 = 4` (5 items, 0-indexed) for the highlight-cursor loop. The original rendered only 4 items (CHAMPIONSHIP through BUTTON SETTINGS).

---

### [8] File `0x00BB99` — Highlight base address (1 byte)

```
Old: 02   LDX #$024C   ($47 = $024C → D0 starts at $5000+$024C+2 = row 9)
New: 01   LDX #$014C   ($47 = $014C → D0 starts at row 5)
```

Shifts the cursor-highlight starting position up by 4 rows to match CHAMPIONSHIP's new position.

---

### [9] Files `0x00BBA3`, `0x00BBA7`, `0x00BBB3–BBB4` — Disable flag init (4 bytes)

These four patches rewrite the `LDX/STX` pairs that initialise items' `$20–$24` disable flags at menu init, for both the has-save and new-game paths. Net effect:

| Item | No progress | Has progress |
|---|---|---|
| CHAMPIONSHIP | enabled | enabled |
| VERSUS | enabled | enabled |
| TIME ATTACK | disabled | enabled |
| RECORDS VIEW | disabled | enabled |
| BUTTON SETTINGS | enabled | enabled |

---

### [10] File `0x00BBB3` + `0x00BBE5–BBF2` — Progress gate rewrite (14 bytes)

Replaces the original conditional `INC`/`STZ` block (`$01:BBE5–BBF2`) with:

```asm
INC $22        ; disable TIME ATTACK (conditional, via trampoline 1)
STZ $24        ; BUTTON SETTINGS always enabled (unconditional)
LDA $700010,X  ; read Minor Championship progress flag
BEQ +3         ; no progress: skip STZ $23
STZ $23        ; enable RECORDS VIEW only with progress
NOP
NOP
NOP
```

This gives the correct two-state unlock behavior based on `$700010,X` (the SRAM Minor Circuit progress flag).

---

### [11] File `0x00BC1D` — OAM cursor Y base (1 byte)

```
Old: 47   LDX #$472C ($53 = $47)
New: 27   LDX #$272C ($53 = $27)
```

OAM cursor Y formula: `Y = item_index × $20 + $53 + 1`. Adjusting `$53` from `$47` to `$27` shifts the cursor sprite 32 px up to align with CHAMPIONSHIP at its new row 5 position.

---

### [12] File `0x00BCAA` — Circuit count: JSR $D641 → JSR $FFD7 (2 bytes)

Redirects the JSR that reads available circuit count from SRAM to our trampoline 2.

---

### [13] File `0x00BD0F` — Header renderer: JSR $D1FC → JSR $8018 (2 bytes)

Redirects the opponent-select screen descriptor renderer call to our conditional stub.

---

### [14] File `0x00BEEF` — Back-out redirect: JMP $C766 → JMP $8463 (2 bytes)

Redirects the back-out button handler in opponent select to our clear stub.

---

### [15] File `0x00C8EE` — Confirm dispatch hook (4 bytes)

```
Old: C9 01 F0 0F   CMP #$01; BEQ +$0F
New: 5C B0 FF 02   JML $02:FFB0
```

At SNES `$01:C8EE`. The `BEQ $C8FA` at `$01:C8EC` already handles index 0 (NEW GAME) before we reach this point. The JML hands off A (= `$07E0`, cursor index 1+) to our 5-item dispatch stub in bank `$02`.

---

### [16] File `0x00C90E` — Mode Select confirm: JMP to dispatch stub (3 bytes)

```
Old: C9 01 F0   CMP #$01; BEQ ...  (top of 4-item chain)
New: 4C B0 FF   JMP $FFB0
```

At SNES `$01:C90E`, reached after `$07E3=0` (CHAMPIONSHIP) has already been handled by `BEQ $C91C` at `$C90C`. Hands A (= `$07E3`, item index 1+) to our 5-item stub in bank `$01` free space.

---

### [17] File `0x00FFB0` — 5-item Mode Select dispatch stub (22 bytes, SNES `$01:FFB0`)

```asm
C9 01        CMP #$01
F0 19        BEQ +$19   ; → $FFC3 (VERSUS entry)
C9 02        CMP #$02
F0 0F        BEQ +$0F   ; → $FFC7 (TIME ATTACK)
C9 03        CMP #$03
F0 0F        BEQ +$0F   ; → $FFCA (RECORDS VIEW: STZ $B200; JMP $C926)
A9 06        LDA #$06
85 03        STA $03     ; BUTTON SETTINGS ($03=6)
4C 51 C9     JMP $C951   ; → STZ $05/$07; PLB; RTL
; VERSUS entry ($FFC3):
5C 9C B2 00  JML $00:B29C  ; → VS mode launcher
```

---

### [18] File `0x00FFC7` — TIME ATTACK + RECORDS VIEW trampolines (6 bytes)

```asm
; $FFC7 — TIME ATTACK:
4C 26 C9     JMP $C926   ; STA $0607=#$01; STZ $05; JMP $C74C
; $FFCA — RECORDS VIEW:
4C 30 C9     JMP $C930   ; LDA #$04; STA $03; BRA $C951
```

---

### [19] File `0x00FFD7` — Trampoline 2: conditional circuit count (13 bytes, SNES `$01:FFD7`)

```asm
AD 07 06     LDA $0607
C9 02        CMP #$02
F0 03        BEQ +3      ; if VERSUS: skip to LDA #$04
4C 41 D6     JMP $D641   ; TA: tail-call SRAM circuit-count reader (RTS returns to caller)
A9 04        LDA #$04    ; VERSUS: force 4 circuits (all characters always available)
60           RTS
```

---

### [20] File `0x017FB0` — Back-out stub in bank `$02` (unused)

Older versions used a cross-bank JML stub here for DPad-down; no longer referenced (superseded by the in-bank `$8463` stub). Bytes remain from earlier iterations but are harmless.

---

### [21] File `0x06A38C` — DA332[`$2D`] pointer redirect (2 bytes)

```
Old: (prototype fighter name garbage)
New: 03 FB   → SNES $FB03
```

Repurposes text index `$2D` to point to the new "PLAYER 2" tile data.

---

### [22] File `0x06A3A0` — DA332[`$37`] pointer redirect (2 bytes)

```
Old: B0 FF   → old "VERSUS" tile data at $FFB0 (stale, superseded)
New: C3 FA   → current "VERSUS" tile data at $FAC3
```

---

### [23] File `0x06A3DE` — DA3DA entry 2 pointer (2 bytes)

```
Old: 6A AD   → original Mode Select layout sub-table at $AD6A
New: 86 FA   → new layout sub-table at $FA86
```

---

### [24] File `0x06A47E` — DA3DA entry `$A4` pointer (2 bytes)

```
Old: B7 FF   → stale VERSUS descriptor (superseded)
New: CA FA   → current 12-record VERSUS descriptor at $FACA
```

---

### [25] File `0x06FA86` — New Mode Select layout sub-table (61 bytes)

```
0D 1C 4E 51   CHAMPIONSHIP  row 5  col 7   ($514E)
0C 1C 6A 51   MODE          row 5  col 21  ($516A)
37 1C 54 52   VERSUS        row 9  col 10  ($5254)
0C 1C 64 52   MODE          row 9  col 18  ($5264)
0F 1C 50 53   TIME          row 13 col 8   ($5350)
12 1C 5A 53   ATTACK        row 13 col 13  ($535A)
0C 1C 68 53   MODE          row 13 col 20  ($5368)
10 1C 4E 54   RECORDS       row 17 col 7   ($544E)
0E 1C 5E 54   VIEW          row 17 col 15  ($545E)
0C 1C 6A 54   MODE          row 17 col 21  ($546A)
11 1C 4C 55   BUTTON        row 21 col 6   ($554C)
13 1C 5A 55   SETTING       row 21 col 13  ($555A)
0C 20 54 58   MODE          header row     ($5854)
05 20 5C 58   <<SELECT      header row     ($585C)
00            terminator
```

---

### [26] File `0x06FAC3` — "VERSUS" tile data (7 bytes)

```
06 1F 0E 1B 1C 1E 1C
```

Format: `[length][tile…]`. Length = 6. Tiles: V=`1F`, E=`0E`, R=`1B`, S=`1C`, U=`1E`, S=`1C`.

---

### [27] File `0x06FACA` — 12-record VERSUS opponent-select descriptor (49 bytes)

Full descriptor for the VERSUS opponent-select screen (used when `$0607=$02`). Contains all four circuit rows (Special, World, Major, Minor), the VERSUS MODE header at row 2, and the `< PLAYER 2 SELECT >` row.

Key records:
```
0A 1C 84 53   SPECIAL   row 14 col 2   ($5384)
0B 1C 94 53   CIRCUIT   row 14 col 10  ($5394)
37 0C 94 50   VERSUS    row 2  col 10  ($5094)
0C 0C A2 50   MODE      row 2  col 17  ($50A2)
05 20 60 59   <<SELECT  row 37 col 16  ($5960)
2D 20 50 59   PLAYER 2  row 37 col 8   ($5950)
00            terminator
```

---

### [28] File `0x06FB03` — "PLAYER 2" tile data (9 bytes)

```
08 19 15 0A 22 0E 1B EF 02
```

Length = 8. Tiles: P=`19`, L=`15`, A=`0A`, Y=`22`, E=`0E`, R=`1B`, space=`EF`, 2=`02`. Tile `EF` is the correct space glyph in this font (not `64`, which renders as a Japanese character).

---

### [29] File `0x06FFB0` — "VERSUS" tile data (7 bytes, bank `$0D`)

```
06 1F 0E 1B 1C 1E 1C
```

Same content as `$0x6FAC3`. This copy is referenced by the title-screen layout sub-table (separate from the opponent-select descriptor tile data).

---

### Versus opponent-select table-hide (records [30]-[33])

The opponent-select screen normally shows a BEST TIME / YOUR BEST table populated from the Time Attack save data. On Versus that table is meaningless. The following four patch records blank that table region (a 30×7 tile rectangle, both BG layers) when entering from Versus, so the screen looks like a clean opponent picker. The blanking is gated on `$0607 == $02` so Time Attack's score table renders normally.

### [30] File `0x0BE7A` — Init hook for Versus opponent-select table-hide (4 bytes)

```
Old: 22 DD F3 0E   JSL $0E:F3DD
New: 22 62 FB 0D   JSL $0D:FB62
```

Replaces the existing `JSL $0E:F3DD` near the end of the opponent-select init (just before the final DMA upload at `$01:B3BF`). The new target is a stub at `$0D:FB62` which calls `JSL $0E:F3DD` itself (preserving exact original behavior), then conditionally blanks the BEST TIME / YOUR BEST table region when `$0607 == $02` (Versus mode). See [33].

### [31] File `0x0BEC8` — Char-switch hook for Versus opponent-select table-hide (6 bytes)

```
Old: A2 0F 00 20 BC D7   LDX #$000F / JSR $D7BC
New: 22 98 FB 0D EA EA   JSL $0D:FB98 / NOP / NOP
```

Replaces the last two instructions of `CODE_01BE9B` (the char-switch render path that runs when `$0C80 == $86`). The stub at `$0D:FB98` inlines `D7BC`'s body (an 8-byte DMA list copy from `DATA_01D7D2` to `$0387`+ — required for both modes), then conditionally blanks BG3 only when `$0607 == $02`. The two NOPs after the JSL fall harmlessly through to `CODE_01BECE`. See [34].

### [32] File `0x06FB62` — Init blanking stub (54 bytes, SNES `$0D:FB62`)

```asm
22 DD F3 0E      JSL $0E:F3DD     ; original work, always
AD 07 06         LDA $0607        ; gate
C9 02            CMP #$02
D0 2A            BNE skip_to_RTL  ; not Versus → exit
08               PHP
C2 30            REP #$30         ; 16-bit M/X/Y
A2 82 54         LDX #$5482       ; BG1 row 18 col 1
A9 07 00         LDA #$0007       ; row counter (7)
48               PHA
row_loop:
  A0 1E 00         LDY #$001E     ; 30 cells per row
  A9 00 00         LDA #$0000     ; empty tile
  inner:
    9F 00 00 7E      STA.l $7E0000,x   ; BG1
    9F 00 08 7E      STA.l $7E0800,x   ; BG3 (BG1 + $0800)
    E8 E8 88         INX, INX, DEY
    D0 F3            BNE inner
  8A 18 69 04 00 AA  TXA, CLC, ADC #$0004, TAX  ; advance to next row col 1
  68 3A 48           PLA, DEC, PHA
  D0 E2              BNE row_loop
68 28            PLA, PLP
6B               RTL
```

Blanks a 30-col × 7-row rectangle (cols 1-30, rows 18-24) on both BG1 and BG3 by writing `$0000` (tile 0, palette 0 — universally transparent). BG3 tilemap is at WRAM `$5800-$5FFF`, exactly `$0800` above BG1 at `$5000`, so a single inner loop writes both layers without extra arithmetic.

### [33] File `0x06FB98` — Char-switch blanking stub (71 bytes, SNES `$0D:FB98`)

```asm
08               PHP
E2 30            SEP #$30          ; 8-bit (D7BC body expects this)
A2 0F            LDX #$0F          ; (replicates the LDX #$000F we removed)
A0 08            LDY #$08
d7bc_loop:
  BF D2 D7 01      LDA.l $01:D7D2,x   ; cross-bank read of bank-$01 table
  99 87 03         STA.w $0387,y
  CA 88            DEX, DEY
  D0 F5            BNE d7bc_loop
A9 80 8D 7D 03   LDA #$80, STA $037D
8D 87 03         STA $0387
AD 07 06         LDA $0607         ; gate
C9 02            CMP #$02
D0 24            BNE skip_to_PLP_RTL
C2 30            REP #$30          ; 16-bit
A2 82 5C         LDX #$5C82        ; BG3 row 18 col 1
A9 07 00         LDA #$0007
48               PHA
row_loop:
  A0 1E 00         LDY #$001E
  A9 00 00         LDA #$0000
  inner:
    9F 00 00 7E      STA.l $7E0000,x   ; BG3 only (X already at BG3 base)
    E8 E8 88         INX, INX, DEY
    D0 F7            BNE inner
  8A 18 69 04 00 AA  TXA, CLC, ADC #$0004, TAX
  68 3A 48           PLA, DEC, PHA
  D0 E6              BNE row_loop
68               PLA
28 6B            PLP, RTL
```

Inlines `CODE_01D7BC`'s body (DMA list setup at `$0387` — needed every char-switch in both modes), then conditionally blanks BG3 only when `$0607 == $02`. BG1 doesn't need re-blanking because the char-switch render path doesn't touch it. The `LDA.l $01:D7D2,x` long-indexed read accesses the bank-$01 lookup table directly, avoiding the need for a bank-$01 trampoline.

**Why the cross-bank inlining:** `CODE_01BE9B` runs in bank `$01`, and the natural fix would be a bank-$01 trampoline that does `JSR $D7BC` + blank + `JMP $BECE`. But bank `$01` has zero confirmed-free bytes. Inlining `D7BC`'s 14 bytes inside our bank-$0D stub avoids needing any bank-$01 space.

---

## 6. Standalone patches — technical detail

### [`spo_sandman_stats_fix.ips`](../patches/standalone/spo_sandman_stats_fix.ips)

**What it does:** Corrects Mr. Sandman's profile screen, which shows the wrong stats (Super Macho Man's age, weight, and record) due to a copy-paste error in the original source. After patching, Sandman shows his correct values from the Japanese version: age 30, weight 270 lbs, record 28-4. Those values are also confirmed in the game's manual.

**Problem:** Mr. Sandman's profile data at file `0x42CD1` is a copy of Super Macho Man's record (`$08AC89`). A copy-paste error in the original source.

**Data format:** `[country][0A][age 2 digits][0A][weight 2 digits][0A][record][00]`
Font encoding: digits `0–9` = `$10–$19`, dash = `$F4`. Weight is stored as 2 digits; the renderer appends a literal "0" ("27" → "270 lbs").

**Fix:** 10 bytes at file `0x42CD8`:
```
Old: 12 18 0A 12 13 0A 12 19 F4 13   ("28", "23", "29-3" — Macho's stats)
New: 13 10 0A 12 17 0A 12 18 F4 14   ("30", "27", "28-4" — correct JP values)
```

---

### [`spo_jp_charset_enabled.ips`](../patches/standalone/spo_jp_charset_enabled.ips)

**What it does:** Makes L/R cycling between three character sets (Japanese-1, Japanese-2, Western) always active in name entry, with no button combo required. The screen opens on the Western set by default, so players who don't want Japanese characters don't need to do anything differently. L and R cycle through all three sets as they do in the Japanese version.

The name-entry screen has a hidden mode where L/R cycles between three character sets (Japanese-1, Japanese-2, Western). Activated by holding X+A or Start+X+A at the New Game cursor. The JP version has it on by default.

Three single-byte changes at SNES `$01:DF83`/`$DF92`/`$DF95` (file `0xDF83–0xDF95`):

| File offset | Old | New | Effect |
|---|---|---|---|
| `0xDF83` | `F0` (BEQ) | `80` (BRA) | Always take the Japanese-mode-enabled branch |
| `0xDF92` | `9C` | `B7` | `JSR $D79C` → `JSR $D7B7` — load full Western UI tiles instead of just swapping the Japanese set (avoids corrupted tilemap on US ROM where Japanese-set UI tiles aren't pre-loaded) |
| `0xDF95` | `00` | `02` | Initial `$1F = $02` (Western set index, matching the `$D7B7` load) |

---

### [`spo_special_title_screen.ips`](../patches/standalone/spo_special_title_screen.ips)

**What it does:** Replaces the title-screen ring logo and background color palette with the Special Circuit variants. The SUPER PUNCH-OUT!! logo is unchanged; only the ring artwork and colors behind and around it are swapped. All in-game circuit screens are unaffected — only the title screen is altered.

Redirects two sites in the title-screen frame handler `CODE_008EE4`:

| File offset | Old | New | Effect |
|---|---|---|---|
| `0x0F25` | `02` | `08` | `LDX #$8002` → `LDX #$8008` — loads Special Circuit ring logo GFX instead of Minor Circuit |
| `0x0F68–0F6A` | `AE 18 80` | `A2 B6 99` | `LDX DATA_088000+$18` → `LDX #$99B6` — reads Special Circuit palette directly instead of via the pointer table, leaving the table entry (and all in-game Minor Circuit references) unchanged |

Bank `$0E` header table: `+$00` = MainMenu font, `+$02` = MinorCircuit, `+$04` = MajorCircuit, `+$06` = WorldCircuit, `+$08` = SpecialCircuit.

---

### [`spo_disable_security_checksum.ips`](../patches/standalone/spo_disable_security_checksum.ips)

The World Circuit completion checksum prevents the Special Circuit from unlocking when using save states, emulators (SNES Classic, Switch NSO), or patched ROMs. This patch disables that checksum check. Identical to patch record [1] in the Versus Hack.

IPS hex (15 bytes): `5041544348003c230002a900454f46`

### [`spo_credits.ips`](../patches/standalone/spo_credits.ips)

**What it does:** Adds a fourth `CREDITS` entry to the **Records View select screen** (the screen that asks which circuit's records to view) and tightens the layout so the four entries sit at rows 10/14/18/22 with the header at row 2 — matching the Championship circuit-select layout. Selecting `CREDITS` launches the game's ending-cutscene credits roll. After the credits finish the game stays on the final screen and requires reset; that is the original cutscene's terminal behavior, not introduced by this patch.

**Final on-screen layout:**

| Row | Element | Layer |
|---|---|---|
| 2 | RECORDS VIEW MODE | Layer 1 |
| 6 | `<` ITEM SELECT `>` | Layer 2 |
| 10 | BEST TIME | Layer 1 |
| 14 | BEST SCORE | Layer 1 |
| 18 | PERSONAL RECORDS | Layer 1 |
| 22 | CREDITS | Layer 1 |

**Records View screen architecture:**

The select screen's init at `CODE_01BFA2` (file `0x0BFA2`) runs a sequence of operations that *each* hardcode WRAM addresses paired to specific tile rows. Moving any element requires updating all of its paired hardcoded references in lockstep. The five coupled positions are:

| Init offset | Hardcoded value | Paired with |
|---|---|---|
| `0x0BFAB` (`$47`) | `$0310` | first entry row (highlight bar base) |
| `0x0BFDB` (`CBA8` X) | `$5A14` | subheader Layer 2 row (palette-cycle range) |
| `0x0BFE9` (`CB57` X) | `$00CE` | "RECORDS VIEW MODE" header row (palette cycle) |
| `0x0BFF2`/`0x0BFF5` (`D70C` X/Y) | `$0210`/`$022E` | subheader row (`<`/`>` brackets) |
| `0x0C022` (`$53` cursor anchor Y) | `$5F` | first entry row (cursor sprite scan start) |

**Why the cursor anchor is the load-bearing one — the `D082` scan loop:**

`CB57` (the palette-cycler called with `X=$0000, Y=$0200`) reads each tilemap entry, and for the empty-fill value `$0064` produces:

- `$00EF` written to the same position
- `$00FF` written to row+1 (shadow row)

Those are exactly the two sentinel values that `CODE_01D082` (the cursor's auto-snap-to-text scan) skips. After init the cursor sprite is positioned by:

1. Setting sprite Y to `$53 + 1` (`$5F + 1 = $60` = pixel row 12).
2. Calling `CAAE`, which loops calling `D082` — converts sprite (X,Y) to tilemap offset via `Y*8 + X/4`, reads the tile, advances X by 8 px and re-tries if the tile is `$EF`/`$FF`, exits when it finds real text.

If `BEST TIME` is anywhere other than row 12, the scan finds only the CB57-transformed `$EF`/`$FF` sentinels at row 12 and **loops forever** — the audio CPU keeps running (music plays) but the main CPU is stuck. The fix is to shift `$53` (file `0x0C022`) in lockstep with the first entry's row.

**Patch records:**

| File offset | Bytes | Effect |
|---|---|---|
| `0x0BFA6` | `03` | `$14=3` (4-item highlight loop, was 3) |
| `0x0BFAB` | `90 02` | `$47=$0290` highlight bar base (row 12 → row 10) |
| `0x0BFDB` | `94 59` | `CBA8` palette-cycle base `$5A14` → `$5994` (subhdr row 8 → 6) |
| `0x0BFE9` | `8E` | second `CB57` X arg `$00CE` → `$008E` (cycle row 2, not row 3) |
| `0x0BFF2` | `90 01` | arrow renderer X `$0210` → `$0190` (`<` row 8 → 6) |
| `0x0BFF5` | `AE 01` | arrow renderer Y `$022E` → `$01AE` (`>` row 8 → 6) |
| `0x0C022` | `4F` | cursor sprite anchor Y `$5F` → `$4F` (scan starts at row 10) |
| `0x0C9CA` | `5C 39 FB 0D` | `JML $0D:FB39` — replaces start of `CODE_01C9CA`, bypasses original 3-way dispatch |
| `0x6A3EC` | `0C FB` | `DA3DA[$12]` redirect to `$0D:FB0C` (replaces ptr to `$AE3F`) |
| `0x6A47A` | `31 FB` | `DA332[$A4]` redirect to `$0D:FB31` (CREDITS tile string) |
| `0x6AE62` | `8E` | descriptor `$14` RECORDS WRAM lo `$50CE` → `$508E` (row 3 → 2) |
| `0x6AE66` | `9E` | descriptor `$14` VIEW WRAM lo `$50DE` → `$509E` |
| `0x6AE6A` | `AC` | descriptor `$14` MODE WRAM lo `$50EC` → `$50AC` |
| `0x6FB0C` | 37 B | new screen descriptor (9 records + terminator) |
| `0x6FB31` | 8 B | "CREDITS" tile string (`07 0C 1B 0E 0D 12 1D 1C`) |
| `0x6FB39` | 41 B | A-button dispatch stub |

**New layout sub-table (`$0D:FB0C`, 37 bytes):**

```
1A 20 94 59   ITEM             Layer 2 row 6  col 10  ($5994)
05 20 9C 59     SELECT         Layer 2 row 6  col 14  ($599C)
1B 34 96 52   BEST             Layer 1 row 10 col 11  ($5296)
0F 34 A0 52   TIME             Layer 1 row 10 col 16  ($52A0)
1B 1C 96 53   BEST             Layer 1 row 14 col 11  ($5396)
1D 1C A0 53   SCORE            Layer 1 row 14 col 16  ($53A0)
1C 34 90 54   PERSONAL         Layer 1 row 18 col 8   ($5490)
10 34 A2 54   RECORDS          Layer 1 row 18 col 17  ($54A2)
A4 1C 98 55   CREDITS          Layer 1 row 22 col 12  ($5598)
00            terminator
```

The text-index `$A4` (`DA332[$A4]`) is repurposed: in the original ROM it pointed to garbage at `$B0A7` that is never used as menu text.

**A-button dispatch stub (`$0D:FB39`, 41 bytes):**

The original `CODE_01C9CA` is the confirm dispatcher for state `$05=0, $07=8` on this screen. It advances `$05` by 2/4/6 based on `$07E6` (cursor index 0/1/2) to enter the score-display state for Minor/Major/World circuits. Index 3 (CREDITS) was not handled and fell through to index 2's path — which is why the title-screen exhibition hack v9–v13 work in the parent doc accidentally launched the credits when their JML target was `$00:B29C`.

The stub replaces that dispatch wholesale by JMLing into bank `$0D` from the very first byte of `C9CA`, then implementing all four cases plus the original `$44 != 0` short-circuit:

```asm
A5 44              LDA $44
D0 21              BNE +$21         ; → JML $01:C94F (the $44 ≠ 0 path)
AD E6 07           LDA $07E6
F0 10              BEQ +$10         ; → index 0 INC pair
C9 01              CMP #$01
F0 08              BEQ +$08         ; → index 1 INC pair
C9 03              CMP #$03
F0 10              BEQ +$10         ; → index 3 JML
E6 05  E6 05       INC $05 ×2       ; index 2 fall-through (3 INC pairs total)
E6 05  E6 05       INC $05 ×2       ; index 1 enters here  (2 INC pairs total)
E6 05  E6 05       INC $05 ×2       ; index 0 enters here  (1 INC pair total)
64 07              STZ $07
AB                 PLB
6B                 RTL
5C 9C B2 00        JML $00:B29C     ; index 3: ending-cutscene credits roll
5C 4F C9 01        JML $01:C94F     ; $44 ≠ 0 path
```

**Free space consumed:** `$0D:FB0C–$0D:FB61` (86 bytes). Conflicts with `spo_sound_mode_ui_incomplete.ips`, which also uses `$0D:FB0C+` — the two patches cannot be applied together.

### [`spo_sound_mode_ui_incomplete.ips`](../patches/incomplete/spo_sound_mode_ui_incomplete.ips)

**What it does:** Adds a SOUND MODE entry to the title-screen menu and shifts the entire menu UI up by 4 rows to accommodate it. The menu now shows NEW GAME / CONTINUE / DATA / CLEAR / SOUND MODE across rows 9/13/17/17/21. Cursor highlight, OAM sprite, arrows, and the MENU/<<SELECT decoration header all shift consistently.

**Known bugs:** (1) Pressing A on SOUND MODE falls through to the DATA CLEAR handler — no A-button dispatch case has been wired for cursor index 3, and wiring the sound library entry requires a runtime trace to identify its trigger mechanism (it sits outside the normal title-screen state machine and requires a different VRAM tileset). (2) After selecting NEW GAME or CONTINUE, moving the cursor in the name entry screen causes a CPU deadlock. Root cause: adding a 4th title-screen item requires `$14=3` (the highlight renderer loops items 0–3); this value persists in WRAM and `CODE_01CAEA` (name-entry cursor navigation) reads `DP+$20+x` as disable flags in a loop — with `$14=3` still set, `CC79` over-iterates into an out-of-bounds layout entry, corrupting memory and freezing on the next cursor movement. This is the same mechanism as the v9 regression in the exhibition hack development. This patch is a proof-of-concept only, is **not** included in `spo_special_edition.ips`, and is not recommended for general use.

**Why JSL instead of JSR:** The original `JSR $CA71` (3 bytes) is replaced with `JSL $0D:FB2D` (4 bytes) to reach the stub in bank `$0D` without needing any bank `$01` relay stub. Bank `$01` has zero confirmed-free bytes remaining. The JSL's 4th byte (`$0D`) overwrites `$B197` — the opcode of the following `LDX #$0002`. The stub compensates by inlining that `LDX` before `RTL`, and `$B198–$B199` are patched to `NOP NOP` as a safe landing pad after `RTL`.

**Why the stub inlines `CODE_01CA71`:** The original `JSR $CA71` not only triggered our `STZ $23` but also ran `CODE_01CA71`, which initializes the highlight system (`$74` = D0 base, `$49` = step size, `$12`/`$0C`/`$0E`). Replacing the JSR with JSL to a stub that only did `STZ $23; LDX; RTL` skipped `CA71` entirely, leaving those variables uninitialized — the highlight bars rendered at garbage positions. The fix is to inline `CA71`'s body into the stub before the `LDX`/`RTL`.

**Patch records:**

| File offset | Bytes | Effect |
|---|---|---|
| `0x0B183` | `00` | Don't disable item 0 (NEW GAME) on no-save path |
| `0x0B194` | `22 2D FB 0D` | JSL `$0D:FB2D` — replaces `JSR $CA71`, clobbers `$B197` |
| `0x0B198` | `EA EA` | NOP NOP — RTL landing pad at the 2 bytes after JSL |
| `0x0B19E` | `02` | Highlight base hi: `$0356` → `$0256` (row 9 start) |
| `0x0B1BD` | `58` | Decoration tilemap hi: `$5994` → `$5894` (visual row 2) |
| `0x0B1C6` | `00` | Left arrow hi: `$0190` → `$0090` |
| `0x0B1C9` | `00` | Right arrow hi: `$01AE` → `$00AE` |
| `0x0B1F2` | `47` | Sprite Y base: `$67` → `$47` (row 9) |
| `0x6A3DA` | `0C FB` | Layout pointer → `$0D:FB0C` |
| `0x6FB0C` | 33 bytes | Layout sub-table (8 entries + terminator) |
| `0x6FB2D` | 31 bytes | Stub: inline `CA71` + `STZ $23` + `LDX #$0003` + `RTL` |

**Layout sub-table (`$0D:FB0C`, 33 bytes):**

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

Text indices `$4B` = "SOUND" (5 tiles, from credits screen — safe to reuse) and `$0C` = "MODE" (4 tiles — already used throughout Mode Select).

**Stub (`$0D:FB2D`, 31 bytes):**

```asm
64 23              STZ $23           ; persist flag across screen transitions
A2 00 00  86 12    LDX #$0000; STX $12
A2 40 00  86 74    LDX #$0040; STX $74   ; highlight D0 start
A2 00 01  86 49    LDX #$0100; STX $49   ; highlight step per item
A2 00 14  86 0C    LDX #$1400; STX $0C
A2 00 0C  86 0E    LDX #$0C00; STX $0E
A2 03 00           LDX #$0003            ; item count (compensates clobbered $B197)
6B                 RTL
```

**Free space consumed:** `$0D:FB0C–$0D:FB4B` (64 bytes). Free block remainder: `$0D:FB4C–$0D:FFE3` (~1,176 bytes).


## 7. Free space map

### Bank `$01` (file `0x8000–0xFFFF`)

**Zero confirmed-free bytes remain** in bank `$01`.

| Range | Size | Status |
|---|---|---|
| `0x8018–0x8029` | 18 B | Consumed by conditional header stub |
| `0x8457–0x8462` | 12 B | Trampoline 1 |
| `0x8463–0x8468` | 6 B | Back-out clear stub |
| `0x841F–0x842A` | 12 B | **NOT SAFE** — null entries in `DATA_01840B` (AI opcode table, opcodes `$14–$1E`) |
| `0x875F–0x8766` | 8 B | **NOT SAFE** — `$FF` fill inside `DATA_01872F` (128-byte game data table) |
| `0xFFB0–0xFFE3` | 52 B | Consumed by stubs |

### Bank `$02` (file `0x10000–0x17FFF`)

| Range | Size | Status |
|---|---|---|
| `0x17FB0–0x17FD3` | 36 B | Contains stale DPad-down stub from v9 era (no longer referenced) |

### Bank `$0D` (file `0x68000–0x6FFFF`)

The disassembly at line 78995 explicitly labels `$0DFA69–$0DFFE3` as garbage fill (`%InsertGarbageData`). This is **1,403 bytes of confirmed unused space**.

| Range | Size | Contents |
|---|---|---|
| `0x6FA69–0x6FA85` | 29 B | Free |
| `0x6FA86–0x6FAC2` | 61 B | VERSUS score descriptor |
| `0x6FAC3–0x6FAC9` | 7 B | "VERSUS" tile data |
| `0x6FACA–0x6FAF8` | 49 B | 12-record VERSUS descriptor |
| `0x6FB03–0x6FB0B` | 9 B | "PLAYER 2" tile data |
| `0x6FB0C–0x6FB61` | 86 B | `spo_credits.ips` descriptor + tile string + dispatch stub |
| `0x6FB62–0x6FB97` | 54 B | Versus opponent-select init blank stub (record [32]) |
| `0x6FB98–0x6FBDE` | 71 B | Versus opponent-select char-switch blank stub (record [33]) |
| `0x6FBDF–0x6FFE3` | ~1,029 B | **Free** |
| `0x6FFB0–0x6FFDE` | 47 B | Stale bytes from earlier iterations (harmless, unreferenced) |
| `0x6FFE4–0x6FFFF` | 28 B | Interrupt vectors — **untouchable** |

> The same `$0D:FB0C–$0D:FB4B` range is also written by `spo_sound_mode_ui_incomplete.ips` (64 B for the title-screen sound-mode layout table + stub). These two patches **cannot be applied together** — they are mutually exclusive layouts of the same free space.

---

## 8. Text encoding and screen descriptor system

### Font encoding (menus and profile screens)

Used by the menu text renderer for Mode Select, opponent-select, etc.:

```
0–9   → $00–$09
A–Z   → $0A–$23  (A=$0A, B=$0B, …, Z=$23)
space → $EF      (in menu font; $64 renders as a Japanese glyph in this context)
```

### Text string format (`DATA_0DA332`)

Each entry: `[length_byte][tile_byte × length]`. The renderer reads `data[0]` as the character count — it does **not** use the gap to the next pointer entry. Do not update the next pointer when adding a new string.

### Key DA332 indices

| Index | Content | File offset of tile data |
|---|---|---|
| `$05` | `  SELECT` (2 leading `$EF` spaces + SELECT) | original |
| `$07` | MINOR | original |
| `$08` | MAJOR | original |
| `$09` | WORLD | original |
| `$0A` | SPECIAL | original |
| `$0B` | CIRCUIT | original |
| `$0C` | MODE | original |
| `$0D` | CHAMPIONSHIP | original |
| `$0F` | TIME | original |
| `$10` | RECORDS | original |
| `$11` | BUTTON | original |
| `$12` | ATTACK | original |
| `$13` | SETTING | original |
| `$0E` | VIEW | original |
| `$38` | space (1 tile, `$EF`) | original — **do not touch pointer**; used as spacer throughout |
| `$49` | PLAYER | original |
| `$2D` | PLAYER 2 (8 tiles) | `0x6FB03` — **repurposed** |
| `$37` | VERSUS (6 tiles) | `0x6FAC3` — **repurposed** |
| `$A4` | CREDITS (7 tiles) | `0x6FB31` — **repurposed by `spo_credits.ips`** (originally pointed to garbage at `$B0A7`, never used as menu text) |

**Warning:** `text_idx=$00` is the descriptor terminator sentinel. It cannot be used as a live token index — the renderer stops on it.

### Screen descriptor table (`DATA_0DA3DA`)

2-byte LE bank-relative pointers, one per screen/context. The renderer (`CODE_01D1FC`) is called with a **byte offset** into this table — so an arg of `$0E` means "the entry at byte offset `$0E`", i.e. the 8th pointer (index 7).

Key entries used or modified by this hack:

| Byte offset | Entry index | Pointer (default) | Screen / context |
|---|---|---|---|
| `$00` | 0 | `$AD48` | title screen layout |
| `$04` | 2 | `$AD6A` | Mode Select layout (redirected to `$FA86` by `spo_versus_hack.ips`) |
| `$12` | 9 | `$AE3F` | Records View select layout (redirected to `$FB0C` by `spo_credits.ips`) |
| `$14` | 10 | `$AE60` | "RECORDS VIEW MODE" header (modified in place by `spo_credits.ips`) |
| `$A4` | 82 | `$AEB7` (stale) | VERSUS opponent-select layout (redirected to `$FACA` by `spo_versus_hack.ips`) |

The opponent-select call site at `CODE_01BD0D` (file `0x0BD0D`) uses `TYA` to derive the argument from Y, where `Y=$000C` (Special Circuit complete) or `Y=$000E` (otherwise), selecting between byte-offset entries 6 and 7.

---

## 9. Key addresses quick reference

### Routines

| SNES address | File offset | Description |
|---|---|---|
| `$01:B0F0` | `0xB0F0` | Title screen entry, sets DP=`$0C00` |
| `$01:B14D` | `0xB14D` | Menu init (state `$03=0`) |
| `$01:BFA2` | `0xBFA2` | Records View select screen init |
| `$01:C9CA` | `0xC9CA` | Records View select A-button confirm dispatch |
| `$01:CA71` | `0xCA71` | Cursor highlight init |
| `$01:CAAE` | `0xCAAE` | OAM cursor sprite positioner (calls `D082` scan loop) |
| `$01:CB57` | `0xCB57` | Tilemap palette-cycle (transforms `$0064` → `$EF`/`$FF`) |
| `$01:CC79` | `0xCC79` | Cursor highlight renderer (loops `$14+1` times) |
| `$01:D082` | `0xD082` | Cursor's auto-snap-to-text scan (skips `$EF`/`$FF`) |
| `$01:C8E9` | `0xC8E9` | Post-Start dispatcher — **do not patch** |
| `$01:DFB4` | `0xDFB4` | Per-frame input loop (interactive menu) |
| `$01:E0A3` | `0xE0A3` | A-button dispatch |
| `$01:E25F` | `0xE25F` | Cursor update done: play sound, update OAM, PLB;RTL |
| `$01:BB7D` | `0xBB7D` | Mode Select + Championship menu init |
| `$01:C905` | `0xC905` | `$07E3` → mode dispatch (48 bytes, partially overwritten) |
| `$01:C74C` | `0xC74C` | `INC $03; INC $03; PLB; RTL` |
| `$01:C752` | `0xC752` | `INC $01; INC $01; PLB; RTL` |
| `$01:D1FC` | `0xD1FC` | Screen descriptor renderer |
| `$00:B29C` | `0x0B29C` | VS mode launcher (routes to fighter select) |
| `$00:9937` | `0x01937` | VS RAM-only setup — **ends with RTS, not RTL; do not JSL** |
| `$00:913B` | `0x0913B` | **Dangerous** — frame-wait loop, deadlocks from title screen |

### RAM variables

| Address | Role |
|---|---|
| DP+`$01` = `$0C01` | State machine level 1 (`$01=0` → `$03` dispatch) |
| DP+`$03` = `$0C03` | State variable for `DATA_01B13D` dispatch |
| DP+`$05` = `$0C05` | Sub-state |
| DP+`$10` = `$0C10` | Cursor X (lo byte) |
| DP+`$11` = `$0C11` | Cursor Y (hi byte) |
| DP+`$14` = `$0C14` | Highlight cursor max index (item count − 1) |
| DP+`$20–$24` = `$0C20–$0C24` | Disable flags for items 0–4 (0=enabled, nonzero=disabled) |
| `$07E0` | Save-data initial selection (read by `CODE_01C8E9`) |
| `$07E3` | Current menu item index (0-based, updated per frame) |
| `$0607` | Mode flag (`$01`=TIME ATTACK, `$02`=VERSUS, `$00`=idle) |
| `$0602` | Player count (`$03` = 2 players) |
| `$0030` | P2 boss-control flag (`$08` = use Controller 2 for opponent) |
| `$700010,X` | SRAM Minor Circuit progress flag (X=`$0050` in this context) |
| `$700080` | SRAM Special Circuit lock flag (0=unlocked) |

### Data tables

| Address | File offset | Description |
|---|---|---|
| `$0D:A3DA` | `0x6A3DA` | Screen descriptor pointer table (82 entries) |
| `$0D:A332` | `0x6A332` | Text string pointer table |
| `$0D:FA69` | `0x6FA69` | Start of confirmed garbage zone (1,403 bytes free) |
| `$0D:FA86` | `0x6FA86` | New Mode Select layout sub-table |
| `$0D:FACA` | `0x6FACA` | 12-record VERSUS opponent-select descriptor |
| `$0D:FB0C` | `0x6FB0C` | New Records View select-screen descriptor (`spo_credits.ips`) |
| `$0D:FB31` | `0x6FB31` | "CREDITS" tile string |
| `$0D:FB39` | `0x6FB39` | A-button dispatch stub for Records View select |
| `$0D:FFE4` | `0x6FFE4` | Interrupt vectors — untouchable |
| `$01:B132` | `0x0B132` | `$01`-indexed dispatch table |
| `$01:B13D` | `0x0B13D` | `$03`-indexed title-screen state table |
| `$01:BB66` | `0x0BB66` | `$03`-indexed table for `$01=4` |
| `$01:BB73` | `0x0BB73` | `$05`-indexed Championship sub-machine table |
| `$01:BC76` | `0x0BC76` | `$05`-indexed Mode Select state machine table |
