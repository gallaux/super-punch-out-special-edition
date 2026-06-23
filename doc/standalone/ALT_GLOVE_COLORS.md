# `spo_alt_glove_colors.ips` — Summary

Adds a runtime glove-color selector to the player's gloves. By default each circuit gets a per-opponent color (mirroring how Punch-Out!! Wii dressed Little Mac in different glove tones across circuits), and the player can force the glove color for the current match by holding a button before the fight starts: on the opponent-select screen in Time Attack / Versus mode, or on the pre-fight opponent screen in Championship mode. Hold the button and press A or Start to enter the fight with the chosen color.

<img width="256" height="224" alt="Super Punch-Out!! Special Edition (USA)_009" src="https://github.com/user-attachments/assets/ab78a0d0-9501-48e9-920d-6cf59af7b7f4" />
<img width="256" height="224" alt="Super Punch-Out!! Special Edition (USA)_005" src="https://github.com/user-attachments/assets/6252da4c-fca4-42a6-baee-6fb14fe95ca7" />
<img width="256" height="224" alt="Super Punch-Out!! Special Edition (USA)_008" src="https://github.com/user-attachments/assets/9f9fd575-1868-4d6c-8939-15adf718f78c" />

## What it does

- **No button held** → use the opponent's circuit-default color.
  - **Minor circuit** (opponents 0-3) → vanilla green (Nintendo's original)
  - **Major circuit** (opponents 4-7) → blue
  - **World circuit** (opponents 8-11) → red
  - **Special circuit** (opponents 12-15) → yellow
- **L** → blue
- **R** → red
- **X** → yellow
- **Y** → force vanilla green (override even on a colored-default opponent)

Override applies only to the current match. The next fight reseeds from the opponent's default. All glove states render in the chosen mode's hue — rest, the powered-up cycling animation, the knock-out-punch / rapid-punch frames, the portrait HUD (rest + both powered-up frames), the BG-tile renders for the victory pose / knockdown / get-up animations, and the player sprite on the ending credits screen.

The "powered up" and "knock-out punch" terms come from the in-game training demo. Internal code symbols (`BLINK_*`, `SUPERARM_*`) keep their historical names; the prose uses the in-game terminology.

## Color reference

Each mode uses a 4-color glove ramp (palette indices 12-15) plus a coordinated 4-color knock-out-punch / rapid-punch ramp. Bytes are little-endian 15-bit BGR words; each 8-byte sequence is c12 c13 c14 c15.

| Mode | State | 8-byte sequence |
|---|---|---|
| **Blue** | Rest gloves | `E3 60 E3 69 A8 6A 50 6B` |
| | Powered-up | `6B 71 AB 7E C8 7F 50 7F` |
| | Knock-out-punch (orange flash) | `1C 01 BF 09 7F 12 5F 23` |
| **Red** | Rest gloves | `27 04 2E 04 74 0C FC 1C` |
| | Powered-up | `AF 14 F6 1C 9B 31 5F 4A` |
| | Knock-out-punch (gold flash) | `5C 02 FF 02 7F 13 FF 33` |
| **Yellow** | Rest gloves | `CA 00 F8 05 DD 12 BF 23` |
| | Powered-up | `52 11 BF 1E FD 37 BF 4F` |
| | Knock-out-punch (crimson flash) | `12 18 19 20 9F 28 DF 39` |

Mode 0 (vanilla green) is unchanged from the original ROM — the fight-init trampoline early-exits when the chosen mode is 0, so no palette write happens. Each knock-out-punch hue is hand-picked rather than mechanically derived: blue→orange (true complement), red→gold (analogous-warm flare), yellow→crimson (warm-to-warm darkening).

## How it works

The patch is a runtime WRAM patch, not a ROM-data patch. The original `Layer1_Contender.bin` and per-fighter palette tables stay untouched. At fight init the trampoline writes the chosen mode's palette directly into WRAM, and per-frame hooks at the powered-up writer, the knock-out-punch writer, and the portrait writer apply the matching per-mode palettes for those animation states.

The mode flag at WRAM `$7E:1FE0` is reseeded every fight init. There is no persistent state across matches.

### Glove animation state map

The player's glove animations span two rendering systems — sprites (most states) and BG layer 1 tiles (victory pose / knockdown / get-up). Each state has its own WRAM source:

| State | Render system | WRAM target | Source |
|---|---|---|---|
| Rest | sprite OAM palette 0 c12-c15 | `$7E:0518-$051F` | written by Hook 1 trampoline at fight init (MVN 1) |
| Powered-up (cycling charged-up animation) | sprite OAM palette 0 c12-c15 (same slot) | `$7E:0518-$051F` | engine snapshots rest, writes powered-up colors during animation, restores rest after — DD43 hook replaces the hardcoded immediates with a per-mode lookup |
| Knock-out-punch / rapid punches | sprite OAM palette 3 c12-c15 | `$7E:0578-$057F` | DDAA hook replaces the hardcoded immediates with a per-mode lookup; Hook 1 also pre-writes the chosen color into `$0578-$057F` (MVN 3) so the engine's snapshot/restore preserves it across attacks |
| Portrait HUD (rest + both powered-up frames) | sprite OAM palette 1 c12-c13 | `$7E:0538/$053A` | EC6E hooks at all 4 call sites detect frame and apply per-mode portrait color |
| Victory pose / knockdown / get-up | BG layer 1 with BG palette 2 c12-c15 | `$7E:0458-$045F` | Hook 1 trampoline writes per-mode rest colors here at fight init (MVN 2) |
| Ending credits screen (final stats / "THE END") | sprite OAM palette 4 c12-c15 | `$7E:0580-$058F` | Static ROM patch: `DATA_008547` (ColorEndScreen.bin) palette 4 c12-c15 at file `0x05DF` — vanilla green replaced with yellow (mode 3) |

### Hook layout

Five sites in bank `$00` redirect into stubs in bank `$0D` free space, plus one bank-`$00` trampoline + a portrait stub-rts inside the `UNK_00F5D0` garbage zone:

| Hook site | Original | Replaced with | Stub purpose |
|---|---|---|---|
| `$00:97E5` (file `0x017E5`) | `LDX #$0000; LDY #$3EE0` (universal fight-init) | `JSL $0D:FDD2 + 2×NOP` | Fight-init trampoline — read held buttons, decode override, opp-table fallback, write per-mode palette via 3 MVNs |
| `$00:DD43` (file `0x05D43`) | head of `CODE_00DD43` (powered-up palette writer) | `JSL $0D:FE80 + RTS` | Replaces hardcoded immediates with a per-mode lookup into the powered-up table |
| `$00:DDAA` (file `0x05DAA`) | head of `CODE_00DDAA` (knock-out-punch palette writer) | `JSL $0D:FEC0 + RTS` | Re-emits displaced bytes, then per-mode knock-out-punch lookup |
| `$00:EB1F`, `$00:EB9B`, `$00:EC68` (sep variant) | `JSR EC6E + SEP #$20` | `JSL $0D:FF68 + NOP` | Calls EC6E via bank-`$00` trampoline, frame-detects via `$0538`, applies per-mode portrait color |
| `$00:EC69` (rts variant) | `JSR EC6E + RTS` | `JSL $00:F5D5` | Same fixup as sep variant, but exits via intra-bank RTS for the bank-`$00` caller |
| `$00:EC64` | `BEQ +5 → $EC6B` | `NOP NOP` | Disables a BEQ that would land inside our 5-byte EC68 hook |

### Cross-bank-RTS trap

`CODE_00EC6E` ends with intra-bank `RTS`, not `RTL`. JSL'ing it directly from bank `$0D` corrupts both the stack (RTS pops 2, JSL pushed 3) and PBR (RTS doesn't restore PBR — it stays at `$00`, so the CPU resumes at `$00:<offset>` instead of `$0D:<offset>`).

The portrait sep-variant stubs work around this with a 5-byte trampoline at `$00:F5D0` (`JSR EC6E + RTL`). The sep stubs `JSL` the trampoline; the trampoline does an intra-bank `JSR EC6E` (RTS-balanced) and then `RTL`s back to bank `$0D`.

The portrait rts-variant stub (site `$00:EC69`) handles a different stack scenario: site `EC69` is `JSR EC6E + RTS` inside `CODE_00EC87`, which itself was JSR'd from a bank-`$00` caller. The rts stub lives in bank `$00` so its closing `RTS` runs with PBR=`$00` and pops the bank-`$00` caller's return cleanly. The stub does `JSR EC6E` directly (intra-bank, balanced), runs the per-mode fixup, then `PLA PLA PLA` to drop our own JSL frame (PCL/PCH/PBR) before falling through to `RTS` to return to EC87's caller.

### MVN clobbers DBR

Hook 1's three MVNs each set DBR to the destination bank (`$7E`). The trampoline does `PHB` early and `PLB` near the exit so the displaced `LDX #$0000; LDY #$3EE0` instructions resume with the original DBR=`$0D`.

## Patch records

17 records, ~581 bytes total (plus the SNES header checksum):

| File offset | Bytes | Effect |
|---|---|---|
| `0x017E5` | 6 | Hook 1: `JSL $0D:FDD2 + 2×NOP` (was `LDX #$0000; LDY #$3EE0`) |
| `0x05D43` | 5 | Powered-up hook: `JSL $0D:FE80 + RTS` |
| `0x05DAA` | 5 | Knock-out-punch hook: `JSL $0D:FEC0 + RTS` |
| `0x05DF` | 8 | Ending credits screen: `DATA_008547` palette 4 c12-c15 → yellow (mode 3) (static, no code) |
| `0x06B1F`, `0x06B9B`, `0x06C68` | 5 each | Portrait sep-variant hooks: `JSL $0D:FF68 + NOP` |
| `0x06C64` | 2 | NOP out the `BEQ +5` that targets inside our 5-byte EC68 hook |
| `0x06CE9` | 4 | Portrait rts-variant hook: `JSL $00:F5D5` |
| `0x075D0` | 5 | Bank-`$00` EC6E trampoline (`JSR EC6E + RTL + RTS`) |
| `0x075D5` | 69 | Bank-`$00` portrait stub-rts |
| `0x6FDAA` | 16 | Opponent → default-mode lookup table |
| `0x6FDBA` | 24 | Rest-palette color table (3 modes × 8 B) |
| `0x6FDD2` | 128 | Hook 1 fight-init trampoline (button decode → opp-table fallback → triple MVN palette write) |
| `0x6FE80` | 60 | Powered-up stub |
| `0x6FEC0` | 64 | Knock-out-punch stub |
| `0x6FF00` | 32 | Powered-up color table (4 modes × 8 B; mode 0 = vanilla) |
| `0x6FF40` | 32 | Knock-out-punch color table (4 modes × 8 B; mode 0 = vanilla) |
| `0x6FF68` | 67 | Portrait sep-variant stub (frame detection + per-mode portrait writes) |
| `0x6FFAB` | 36 | Portrait color table (3 modes × 3 frames × 4 B; mode 0 vanilla = early-exit) |

## Free space consumed

- **Bank `$0D`**: 459 bytes used within `$0D:FDAA-$0D:FFCE` (opp_table + rest_table + Hook 1 trampoline + powered-up stub + knock-out-punch stub + powered-up table + knock-out-punch table + portrait sep stub + portrait color table; with ~90 B of small unused gaps between sub-regions). Inside the disassembly's `%InsertGarbageData($0DFA69, ...)` zone.
- **Bank `$00`**: 74 bytes at `$00:F5D0-$00:F619` (EC6E trampoline + portrait stub-rts). Inside the disassembly's `UNK_00F5D0` `%InsertGarbageData` zone (~2,422 B remaining free in the zone after this patch).

## Patch in asm form

```asm
; spo_alt_glove_colors
; Runtime glove-color selector for the player. Per-opponent default colors
; (one mode per circuit by default), L/R/X/Y override per match. All player
; glove animation states (rest, powered-up, knock-out-punch, portrait HUD,
; victory/KD/get-up BG tiles) render in the chosen mode's hue.

;==============================================================================
; Hook sites (bank $00) — each redirects to a stub in bank $0D free space,
; or in the case of the rts-variant portrait hook, to a stub in bank $00.
;==============================================================================

; Hook 1 — universal fight-init (replaces displaced LDX/LDY immediates)
org $0097E5
    JSL $0DFDD2
    NOP
    NOP

; Powered-up writer (CODE_00DD43): replace head with JSL + RTS
; Original head: REP #$10; LDX #$0187 — 5 bytes. Stub re-emits these.
org $00DD43
    JSL $0DFE80
    RTS

; Knock-out-punch / rapid-punch writer (CODE_00DDAA): replace head with JSL + RTS
; Original head: LDA #$04; STA $21; LDX — 5 bytes. Stub re-emits these.
org $00DDAA
    JSL $0DFEC0
    RTS

; BEQ that targets inside our 5-byte EC68 hook — disable so it doesn't
; jump into the middle of our JSL instruction (which would execute byte
; $0D as the high byte of an opcode and crash).
org $00EC64
    db $EA, $EA     ; was: F0 05 (BEQ +5 → $EC6B)

; Portrait writer call sites (sep variant): JSR EC6E + SEP #$20 → JSL + NOP
; All three sites share a single stub at $0D:FF68.
org $00EB1F
    JSL $0DFF68
    NOP
org $00EB9B
    JSL $0DFF68
    NOP
org $00EC68
    JSL $0DFF68
    NOP

; Portrait writer call site (rts variant): JSR EC6E + RTS → JSL stub_rts
; Stub_rts lives in bank $00 so its closing RTS runs with PBR=$00 and pops
; the bank-$00 caller's return cleanly. See "Cross-bank-RTS trap" above.
org $00EC69
    JSL $00F5D5

;==============================================================================
; Bank-$00 stubs (UNK_00F5D0 garbage zone)
;==============================================================================

; EC6E trampoline (5 B) — used by the sep-variant portrait stubs to call
; CODE_00EC6E without JSL'ing it directly (EC6E ends in RTS, not RTL).
; The bare RTS at +4 is left as a no-op (legacy from an earlier exit pattern).
org $00F5D0
    JSR $EC6E       ; intra-bank, RTS-balanced
    RTL             ; pops PCL/PCH/PBR back to the JSL'er in bank $0D
    RTS             ; unused; reserved

; Portrait stub_rts (69 B) — called from site EC69 via JSL.
; EC69 is `JSR EC6E + RTS` inside CODE_00EC87 (which was JSR'd from a
; bank-$00 caller). We can't return through a bank-$0D stub here because
; the popped return PC belongs to a bank-$00 caller. So stub_rts lives
; in bank $00 and ends with intra-bank RTS, which lands correctly.
org $00F5D5
    JSR $EC6E                   ; intra-bank, A=8-bit on return
    LDA.l $7E1FE0               ; mode flag
    BEQ .end                    ; mode 0 → leave EC6E's vanilla writes intact
    DEC                         ; mode 1..3 → 0..2
    REP #$20                    ; 16-bit A
    AND #$00FF
    ASL : ASL                   ; (mode-1) * 4
    PHA
    LDA.l $7E0538               ; what EC6E just wrote
    LDX #$0000                  ; rest frame offset
    CMP #$0D4A                  ; blinkA frame?
    BNE +
    LDX #$000C                  ; blinkA frame offset
+   CMP #$14B1                  ; blinkB frame?
    BNE +
    LDX #$0018                  ; blinkB frame offset
+   TXA
    CLC
    ADC 1,S                     ; + (mode-1)*4
    TAX
    PLA
    LDA.l $0DFFAB,X             ; portrait_table[X]   → c12 word
    STA.l $7E0538
    LDA.l $0DFFAD,X             ; portrait_table[X+2] → c13 word
    STA.l $7E053A
    SEP #$20                    ; back to 8-bit A
.end:
    PLA : PLA : PLA             ; drop our JSL frame (PCL, PCH, PBR)
    RTS                         ; return to EC87's bank-$00 caller (PBR=$00)

;==============================================================================
; Bank-$0D tables and stubs
;==============================================================================

; Opponent → default-mode lookup table (16 B)
; Indexed by $7E:0600 (opponent index 0..15).
org $0DFDAA
    db 0,0,0,0      ; Minor   (opponents 0..3)  → vanilla green
    db 1,1,1,1      ; Major   (opponents 4..7)  → blue
    db 2,2,2,2      ; World   (opponents 8..11) → red
    db 3,3,3,3      ; Special (opponents 12..15) → yellow

; Rest-palette color table (24 B = 3 modes × 8 B)
; 15-bit BGR LE words, c12 c13 c14 c15. Mode 0 = no row (early-exit).
org $0DFDBA
    db $E3,$60, $E3,$69, $A8,$6A, $50,$6B   ; mode 1: blue
    db $27,$04, $2E,$04, $74,$0C, $FC,$1C   ; mode 2: red
    db $CA,$00, $F8,$05, $DD,$12, $BF,$23   ; mode 3: yellow

; Hook 1 fight-init trampoline (128 B)
; Entry env: DBR=$0D, DP=$2100, X 16-bit, A 8-bit (set by upstream SEP #$20).
; Reads $7E:0090/0091 (held buttons) and $7E:0600 (opponent index), commits
; the resulting mode to $7E:1FE0, then writes the rest palette to 3 WRAM
; destinations via MVN if the mode is non-vanilla.
org $0DFDD2
    PHP
    PHB                         ; save DBR (MVN clobbers it)
    REP #$30                    ; 16-bit A/X/Y
    SEP #$20                    ; 8-bit A for button decode
    ; ── L/R/X decode in lo byte ───────────────────────────────────────
    LDA.l $7E0090
    AND #$70                    ; mask L($20) | R($10) | X($40)
    BEQ .check_Y
    BIT #$20                    ; L?
    BEQ .try_R
    LDA #$01 : BRA .commit      ; mode 1 = blue
.try_R:
    BIT #$10
    BEQ .try_X
    LDA #$02 : BRA .commit      ; mode 2 = red
.try_X:
    LDA #$03 : BRA .commit      ; mode 3 = yellow
    ; ── Y check in hi byte ────────────────────────────────────────────
.check_Y:
    LDA.l $7E0091
    AND #$40                    ; Y bit
    BEQ .default
    LDA #$00 : BRA .commit      ; force vanilla
    ; ── Default: opp_table[opponent_index] ────────────────────────────
.default:
    LDA.l $7E0600               ; opponent index (8-bit)
    AND #$0F
    REP #$20                    ; 16-bit A for X transfer
    AND #$00FF
    TAX
    SEP #$20                    ; 8-bit A for the load
    LDA.l $0DFDAA,X             ; opp_table[X]
.commit:
    STA.l $7E1FE0               ; commit mode flag (overwrite — per-match)
    BEQ .done                   ; mode 0 → no palette write
    ; ── Compute source address: rest_table + (mode-1)*8 ──────────────
    DEC
    REP #$20
    AND #$00FF
    ASL : ASL : ASL             ; (mode-1) * 8
    CLC
    ADC #$FDBA                  ; + REST_TABLE lo16 → source addr in bank $0D
    PHA                         ; save source addr; reused via 1,S
    ; ── MVN 1: sprite OAM palette 0 c12-c15 (in-fight rest gloves) ───
    TAX
    LDY #$0518
    LDA #$0007                  ; count - 1 = 7
    MVN $7E, $0D
    ; ── MVN 2: BG palette 2 c12-c15 (victory / KD / get-up tiles) ────
    LDA 1,S
    TAX
    LDY #$0458
    LDA #$0007
    MVN $7E, $0D
    ; ── MVN 3: sprite OAM palette 3 c12-c15 (knock-out-punch base) ───
    ; Critical: at fight init the contender DMA loads vanilla green
    ; into $0578-$057F (Layer1_Contender bytes 56-63). Without this
    ; overwrite, the engine's knock-out-punch save/restore would
    ; round-trip vanilla green back over our DDAA hook's writes.
    LDA 1,S
    TAX
    LDY #$0578
    LDA #$0007
    MVN $7E, $0D
    PLA                         ; drop saved source addr
.done:
    PLB                         ; restore DBR=$0D
    PLP
    LDX #$0000                  ; displaced
    LDY #$3EE0                  ; displaced
    RTL

; Powered-up stub (60 B)
; Reads mode flag, indexes blink_table[mode*8], writes 4 words to $7E:0518-$051F.
; Mode 0 has its own row (vanilla green-blink) — no early-exit needed.
org $0DFE80
    PHP
    REP #$30
    SEP #$20
    LDA.l $7E1FE0
    REP #$20
    AND #$00FF
    ASL : ASL : ASL             ; mode * 8
    TAX
    LDA.l $0DFF00,X : STA.l $7E0518
    LDA.l $0DFF02,X : STA.l $7E051A
    LDA.l $0DFF04,X : STA.l $7E051C
    LDA.l $0DFF06,X : STA.l $7E051E
    SEP #$20
    LDA #$80
    STA.l $7E0048               ; flag VBlank palette upload
    PLP
    RTL

; Knock-out-punch / rapid-punch stub (64 B)
; Same shape as powered-up stub, plus re-emits the displaced LDA #$04; STA $21
; from the head of CODE_00DDAA.
org $0DFEC0
    PHP
    REP #$30
    SEP #$20
    LDA #$04 : STA $21          ; displaced
    LDA.l $7E1FE0
    REP #$20
    AND #$00FF
    ASL : ASL : ASL
    TAX
    LDA.l $0DFF40,X : STA.l $7E0578
    LDA.l $0DFF42,X : STA.l $7E057A
    LDA.l $0DFF44,X : STA.l $7E057C
    LDA.l $0DFF46,X : STA.l $7E057E
    SEP #$20
    LDA #$80
    STA.l $7E0048               ; flag VBlank palette upload
    PLP
    RTL

; Powered-up color table (32 B = 4 modes × 8 B)
; Mode 0 = Nintendo's original green-blink immediates.
org $0DFF00
    db $87,$01, $67,$02, $2C,$27, $F4,$47   ; mode 0: vanilla green-blink
    db $6B,$71, $AB,$7E, $C8,$7F, $50,$7F   ; mode 1: blue
    db $AF,$14, $F6,$1C, $9B,$31, $5F,$4A   ; mode 2: red
    db $52,$11, $BF,$1E, $FD,$37, $BF,$4F   ; mode 3: yellow

; Knock-out-punch color table (32 B = 4 modes × 8 B)
; Mode 0 = Nintendo's original light-blue immediates. Other modes are
; hand-picked complementary hues (see "Color reference" above).
org $0DFF40
    db $A8,$7D, $88,$7E, $4D,$7F, $F5,$7F   ; mode 0: vanilla light-blue
    db $1C,$01, $BF,$09, $7F,$12, $5F,$23   ; mode 1: blue → orange
    db $5C,$02, $FF,$02, $7F,$13, $FF,$33   ; mode 2: red → gold
    db $12,$18, $19,$20, $9F,$28, $DF,$39   ; mode 3: yellow → crimson

; Portrait sep-variant stub (67 B)
; Calls EC6E via the bank-$00 trampoline, frame-detects via $0538, then
; applies the per-mode portrait color. Exits via RTL.
org $0DFF68
    JSL $00F5D0                 ; trampoline (JSR EC6E + RTL)
    LDA.l $7E1FE0               ; mode flag
    BEQ .end                    ; mode 0 → leave vanilla writes alone
    DEC
    REP #$20
    AND #$00FF
    ASL : ASL                   ; (mode-1) * 4
    PHA
    LDA.l $7E0538               ; what EC6E just wrote
    LDX #$0000                  ; rest frame offset
    CMP #$0D4A
    BNE +
    LDX #$000C                  ; blinkA offset
+   CMP #$14B1
    BNE +
    LDX #$0018                  ; blinkB offset
+   TXA
    CLC
    ADC 1,S
    TAX
    PLA
    LDA.l $0DFFAB,X : STA.l $7E0538
    LDA.l $0DFFAD,X : STA.l $7E053A
    SEP #$20
.end:
    RTL

; Portrait color table (36 B = 3 modes × 3 frames × 4 B)
; Layout: [rest×3 modes][blinkA×3 modes][blinkB×3 modes]. Each entry is
; the c12 word followed by the c13 word (lo-hi pairs). Mode 0 is omitted —
; the stub's BEQ early-exits on mode 0 so EC6E's vanilla writes survive.
org $0DFFAB
    ; rest portraits (c12 c13)
    db $E3,$69, $A8,$6A         ; blue
    db $2E,$04, $74,$0C         ; red
    db $F8,$05, $DD,$12         ; yellow
    ; blinkA portraits
    db $A8,$6A, $50,$6B         ; blue
    db $74,$0C, $FC,$1C         ; red
    db $DD,$12, $BF,$23         ; yellow
    ; blinkB portraits
    db $92,$73, $D4,$7B         ; blue
    db $3E,$25, $7F,$2D         ; red
    db $FF,$2B, $FF,$33         ; yellow
; Ending credits screen — static palette patch, no code needed.
; DATA_008547 (ColorEndScreen.bin) palette 4 c12-c15 is the player sprite
; palette on the final stats / "THE END" screen after the credits roll.
; Vanilla green replaced with yellow (mode 3) to match the in-fight color.
org $008547 + 4*32 + 24   ; = file $05DF (SNES $00:85DF)
    db $CA,$00, $F8,$05, $DD,$12, $BF,$23   ; yellow c12-c15
```

## Compatibility

- **Apply on top of**: bare `spo.sfc` (MD5 `97fe7d7d2a1017f8480e60a365a373f0`)
- **Bundled into**: `spo_special_edition_v1.7.ips`
- **Conflicts with**: nothing in this repo
- **Cheat-code compatibility**: unaffected (no opponent palettes or fight-state machinery touched)

## See also

For the deeper investigation behind the WRAM glove-palette location, the universal fight-init hook point, the cross-bank-RTS trap that shaped the portrait-hook design, and the multi-MVN trampoline's interaction with the engine's snapshot/restore mechanism, see [doc/GLOVE_COLORS.md](../GLOVE_COLORS.md).
