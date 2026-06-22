# `spo_iron_circuit.ips` — Summary

Adds a fifth entry, **IRON CIRCUIT**, to the Championship Mode → Circuit Select screen, below SPECIAL CIRCUIT. IRON CIRCUIT is a 16-opponent gauntlet that runs every fighter in order, from Gabby Jay to Nick Bruiser, without HP recovery between fights. Player stamina and cumulative knockdown count carry across every match.

## What it does

- **IRON CIRCUIT entry on Circuit Select** — always selectable, regardless of save state. The Circuit Select layout shifts up 4 rows so all five circuits fit; all decorations (brackets, subtitle, per-circuit W/L digits) move in lockstep.
- **16-opponent gauntlet** — fights proceed in opponent order from Gabby Jay through Nick Bruiser. The game routes directly to the next opponent after each win.
- **HP carry-over with +16 recovery** — player stamina carries from one fight to the next, capped at `$50` (max). The HP bar at fight start renders at the carry-over value immediately, with no tween.
- **Cumulative knockdown tracking** — a per-run knockdown counter accumulates across all fights. Stamina recovery on get-up degrades with each cumulative knockdown (1st: full recovery; 2nd: zero recovery; 3rd: forced KO).
- **Continue resets the run** — spending a continue resets the cumulative KD counter and the HP stash. The player gets a fresh 3-KD allowance and full HP for the next fight; the run's loss counter is incremented for the SRAM record.
- **REST counter frozen at 3** — the player's REST count does not increase during iron. The visible tile stays at 3 for the entire run.
- **Iron-specific presentation**:
  - Deep red (rust) BG2 palette on all iron fights and the pre-fight title screen.
  - `IRON⋆CIRCUIT` pre-fight title, centered, using the shared `⋆CIRCUIT` suffix already in ROM.
  - Iron rank display counting down from `#15` (Gabby Jay) to `#1` (Rick Bruiser), then `CHAMP` banner for Nick Bruiser (fight 16).
  - `IRON CIRCUIT TOTAL` label on the per-fight score-tally screen.
  - Stamina bonus omitted from the score-tally total (carry-over is the reward instead). The tally screen runs at accelerated speed on iron.
- **Celebration animation on the final fight** — the champion-defeated jump-and-turn animation fires after beating Nick Bruiser, matching the behavior of every other circuit's final fight.
- **Championship belt screen at iron-complete**:
  - `IRON CIRCUIT CLEAR` big-font title (replaces the SPECIAL CIRCUIT CLEAR text that would otherwise render).
  - Rust BG2 palette + steel-blue belt graphic.
  - Opponent records list and TOTAL SCORE line suppressed.
  - W/L line reads `16 Wins {N} Loss`.
  - Vanilla championship belt screens (MINOR/MAJOR/WORLD/SPECIAL) are unaffected.
- **Per-slot iron W/L persistence** — after completing iron for the first time on a save slot, the Circuit Select screen shows `16 WIN N LOSS` next to the IRON CIRCUIT entry on subsequent visits. `N` is the best (lowest) loss count achieved on that slot. Slot-independent. Never-completed slots show nothing.
- **No contamination of vanilla circuits** — all iron-specific hooks are gated on the iron flag. MINOR/MAJOR/WORLD/SPECIAL W/L digits, max-circuit records, and best-circuit scores are not affected by iron runs.
- **SPECIAL CIRCUIT always visible** — the SPECIAL CIRCUIT slot is always rendered on the Championship Mode and Time Attack circuit-select screens, even before the vanilla unlock condition is met. The slot remains greyed-out and unselectable until earned; only visibility changes.

## Gauntlet flow

1. From Championship Mode → Circuit Select, select **IRON CIRCUIT**.
2. The game enters a full 16-fight gauntlet: Gabby Jay (fight 1) through Nick Bruiser (fight 16).
3. Each fight ends with a BEST TIME screen (fights 1–15). Fight 16 plays the celebration animation, then renders the BEST TIME for Nick Bruiser, then the IRON CIRCUIT CLEAR belt screen, then credits.
4. A loss and continue resets cumulative KD and HP stash — the gauntlet continues from the next fight, not from fight 1.

## Technical architecture

Iron mode is implemented as a single bit at WRAM `$7E:1FE1` (the **iron flag**), set when the player presses A on the IRON CIRCUIT slot, cleared at iron-complete or on a non-iron circuit selection. Every iron-specific hook checks this flag and falls through to vanilla behavior when it is clear.

The patch hijacks Special Circuit's circuit identity (`$0601 = $03`) at iron entry, so all downstream circuit-themed asset loads (BG, ring, palette, announcer voice) reuse Special's data without per-asset hooks. Two additional flag-gated stubs override the BG2 palette (`$00:9C18`) and the pre-fight title (`$08:D19D`) to give iron its own deep-red look while leaving Special's purple intact.

The 16-opponent chain is driven by **POST_MATCH** at `$00:B250`, which on iron writes `$0602 = $0600 EOR $0F` between fights — nonzero for fights 1–15 (so the natural "circuit complete" branch never fires), zero for the transition into fight 16 (so the engine's celebration animation fires after the win). When `$0600` reaches `$10` (one past the 16th fighter), POST_MATCH clears the iron flag and routes through the **DISPATCH** stub at `$00:B270` to **CHAMP_SCREEN** at `$00:F9B6`, which restores `$0600 = $0F` and `$0601 = $03` (mirroring vanilla `CODE_00BD2F`'s `LDY #$030F; STY $0600` 16-bit store), renders the BEST TIME (`JSR $C045`) and championship belt screen (`JSR $BEE8`), saves the iron W/L record to SRAM, and JMLs to credits at `$00:B29C`.

### WRAM map

| Address | Role |
|---|---|
| `$7E:1FE1` | Iron flag (1 = inside gauntlet) |
| `$7E:1FE5–$1FE9` | Runtime-staged `IRON\0` letter-byte string for the pre-fight title |
| `$7E:1FEA` | Cumulative KD count for the current run |
| `$7E:1FEB` | HP stash (sentinel `$FF` = no stash, else carry-over stamina value) |
| `$7E:1FEC`/`$1FED` | Iron belt-screen flag (set by CHAMP_SCREEN_STUB before `JSR $BEE8`, cleared after; M=16 read requires both bytes clear) |
| `$7E:1FEE` | Per-run loss counter (cleared on iron entry, incremented on each continue) |

### Hook sites

Bank `$00` unless otherwise noted. Each hook is gated on the iron flag — vanilla circuits take the original code path.

| Hook site | Purpose |
|---|---|
| `$01:B94D` | Descriptor render trampoline — replaces `JSR $D1FC` with `JSL` to a bank-`$00` stub that inlines the renderer and writes the IRON tile row |
| `$01:B9CD` | Confirm hook — sets the iron flag, hijacks `$0601 = $03` (Special) on iron entry, resets cumulative KD / HP stash / loss counter |
| `$00:B250` (POST_MATCH) | 16-opponent chain driver; iron-complete trigger when `$0600 == $10`; celebration encoding (`$0602 = $0600 EOR $0F`) for fight 16 |
| `$00:B270` (DISPATCH) | Iron-final-fight path: clears iron flag, JMLs to CHAMP_SCREEN (which renders BEST TIME + championship belt + credits) |
| `$00:B231` (B231_INIT) | Clears `$7E:1FEC`+`$1FED` after every fight for all circuits (defensive cleanup of belt-screen flag) |
| `$00:B226` / `$00:B24A` | Stamina-stash hooks — capture `$089F` to `$7E:1FEB` before bonus screen drains it, on both KO and TKO post-match paths |
| `$00:96A5` | Stamina-init override — reads HP stash, applies +16 recovery (cap `$50`), writes to `$089F` at fight start |
| `$00:E9F7` | HP-bar tween fix — pre-fills tile positions 1–9 with empty-bar tile `$38`; sets `$0B14` to carry value so the bar snaps without animating |
| `$00:9C18` | BG2 palette override — MVNs the iron rust palette when iron flag `$7E:1FE1` OR belt-screen flag `$7E:1FEC` is set, original Special purple on vanilla |
| `$00:BF81` | Belt-screen palette override — MVNs rust BG2 + steel-blue belt + sprite palette into the championship screen on iron |
| `$00:C6A6` | KD-increment hook — INCs cumulative `$7E:1FEA`; forces `$089D = 3` (instant TKO) when cumulative reaches 3 |
| `$08:C0D8` | Belt-screen title hook — stages "IRON" letter bytes into low WRAM, points the title renderer at them, centers `+1` col |
| `$08:C09B`, `$08:C0AE`, `$08:C0C7` | Belt-screen TEXT_SKIP hooks — suppress opponent records, TOTAL SCORE label, and TOTAL SCORE numerals on iron |
| `$08:D19D` | Pre-fight title override — substitutes `IRON⋆CIRCUIT` for `SPECIAL⋆CIRCUIT` at the LARGE-FONT renderer |
| `$08:D1C5` | Pre-fight rank renderer — draws `#15..#1` for fights 1–15 (BG1 hash + BG3 small-font digits), CHAMP banner for fight 16 |
| `$08:D2AB` | Pre-fight portrait shift — replaces `LDX #$38A8; STX $7A` (5 B); handles iron+opp2 (Piston Hurricane +4px) and inlines `spo_super_macho_man_fix`'s opp11 portrait shift |
| `$08:D74E`, `$08:D755` | Belt-screen W/L digit override — write "16" instead of "4" in the championship "X Wins Y Loss" line |
| `$01:9EBB` | Continue hook — resets cumulative KD `$7E:1FEA`, sentinel `$7E:1FEB`, INCs loss counter `$7E:1FEE` |
| `$01:9EEC` | Score-tally circuit-name override — substitutes `IRON CIRCUIT TOTAL` for `SPECIAL CIRCUIT TOTAL` on iron |
| `$01:9F72` | Phase-bypass hook — skips the stamina drain/recovery animation phase entirely on iron, regardless of A-press fast-forward |
| `$01:A0C7` | Fast-cascade hook — score-tally per-iter delay shrinks from 8 frames to 1 frame on iron |
| `$01:A27E` | Recovery formula override — feeds cumulative KD `$7E:1FEA` to the engine's stamina recovery math instead of per-match `$089D` |
| `$01:A416` | Aggregate BEST TIME suppression — defeats the gate that would render a SPECIAL CIRCUIT BEST TIME aggregate after every 4th iron fight |
| `$01:A99C` | Stamina-bonus zero — returns `A=0` on iron so the stamina-bonus contribution to score is 0 (carry-over is the reward instead) |
| `$01:AAF2` | REST +1 award suppression — freezes `$0606` and the visible REST tile on iron |
| `$01:A2EC`, `$01:A381` | Champion-bonus drain suppression — defeats the `$0600 mod 4 == 3` gate so iron's champion-position fights skip the drain |
| `$01:B93C`, `$01:BCDB` | Show-special hooks — force the SPECIAL CIRCUIT slot to render on Championship Mode and Time Attack circuit-select even when not unlocked (vanilla disable rendering preserved) |
| `$01:B959`, `$01:BD29` | Iron W/L draw trampolines — call DRAW_STUB after the vanilla circuit-select renderer to overlay `16 WIN N LOSS` on iron-completed slots |
| `$01:B7D6` | Slot-kill hook — fires inside the per-profile FILE KILL routine after the engine zeroes the slot record. Calls SLOT_KILL stub to also zero the iron W/L bytes at `$700FE8 + slot` and `$701FE8 + slot`. ALL CLEAR is unaffected (it already zeros all SRAM). |

### Stub layout

All stubs live in free-space regions documented in [TECHNICAL.md § 7](../TECHNICAL.md#7-free-space-map).

| Stub | Location | Purpose |
|---|---|---|
| `$00:F61A` | bank `$00` | Inlined `$D1FC` body + IRON tile-row writer |
| `$00:F702` | bank `$00` | CONFIRM stub — sets iron flag, hijacks `$0601` |
| `$00:F724` | bank `$00` | POST_MATCH stub — 16-opp chain driver, celebration encoding, iron-complete trigger |
| `$00:F747` | bank `$00` | DISPATCH stub — iron-final-fight credits path |
| `$00:F75A` | bank `$00` | BG2 palette swap (gated on iron flag `$7E:1FE1` OR belt-screen flag `$7E:1FEC`) |
| `$00:F781` | bank `$00` | 16-color iron rust palette (BGR15 LE) |
| `$00:F7A1` | bank `$00` | Pre-fight title stub |
| `$00:F7D6` | bank `$00` | Iron rank renderer (BG1 `#` + BG3 small-font digits) |
| `$00:F8B9` | bank `$00` | Portrait shift — iron+opp2 → `$38AC`; opp11 (SMM, iron or vanilla) → `$38AC`; else → `$38A8` |
| `$00:F8E0` | bank `$00` | Cumulative KD + force-KO on 3rd cumulative |
| `$00:F903` | bank `$00` | CONFIRM_RESET wrapper — clears cumulative KD / HP stash / loss counter on iron entry |
| `$00:F917` | bank `$00` | Stamina init override (carry-over restore + `+$10` cap-`$50`) |
| `$00:F948` | bank `$00` | Stamina stash (KO + TKO paths share this stub) |
| `$00:F959` | bank `$00` | HP-bar tween fix — empty-tile pre-fill |
| `$00:F9B6` | bank `$00` | CHAMP_SCREEN — sets belt-screen flag, sets `$0600 = $0F` and `$0601 = $03`, calls `JSR $C045` + `JSR $BEE8` + SAVE_STUB, JMLs to credits |
| `$00:F9E1` | bank `$00` | CHAMP_PAL — iron belt-screen palette MVNs |
| `$00:FA53` | bank `$00` | B231_INIT — clears `$7E:1FEC`+`$1FED` every post-fight |
| `$00:FA61` | bank `$00` | CHAMP_TITLE — `SPECIAL` → `IRON` in belt-screen big-font title |
| `$00:FA96` | bank `$00` | TEXT_SKIP — three call sites, suppresses opponent records on iron belt screen |
| `$00:FAA5` | bank `$00` | W/L digit override — writes "1" + "6" instead of "4" |
| `$00:FAF5` | bank `$00` | SAVE_STUB — writes iron W/L to SRAM `$700FE8 + slot` (primary + backup) at iron-complete |
| `$00:FB2A` | bank `$00` | DRAW_STUB — renders `16 WIN N LOSS` on Circuit Select for iron-completed slots |
| `$01:F784` | bank `$01` | REST_NO_GROW — suppresses `$0606` INC + visible REST tile bumps |
| `$01:F7A5` | bank `$01` | DRAIN_SKIP — shared by champion-bonus drain and aggregate-screen suppression |
| `$01:F7B2` | bank `$01` | TALLY_NAME — score-tally circuit-name override |
| `$01:F7D0` | bank `$01` | RECOVERY_OVERRIDE — feeds cumulative KD to recovery math |
| `$01:F7E0` | bank `$01` | CONTINUE_RESET — clears cumulative + sentinel, INCs loss counter |
| `$01:FED4` | bank `$01` | PHASE_BYPASS — skips drain/recovery animation phase |
| `$01:FEF5` | bank `$01` | BONUS_ZERO — zeroes stamina-bonus contribution to score |
| `$01:FF02` | bank `$01` | FAST_CASCADE — 1-frame score-tally cascade |
| `$01:FF20` | bank `$01` | Two 8-byte trampolines bridging the iron W/L draw hooks |
| `$01:FF30` | bank `$01` | SLOT_KILL — zeros iron W/L bytes on per-profile FILE KILL (15 B) |
| `$0D:FA69` | bank `$0D` | IRON tally descriptor (`   IRON` text for the score-tally screen) |

## SRAM persistence

Iron W/L data is stored at `$700FE8 + slot` (primary) and `$701FE8 + slot` (backup), 1 byte per slot, 8 slots. These addresses sit past the engine's per-slot record table (`$700100..$700F7F`, ~30 records × `$80` stride) and outside the engine's checksum sweep. The backup at `$701FE8 + slot` survives the engine's corruption-recovery `MVN` path at `CODE_01ACCA`.

Encoding (offset-by-1, with `$00` reserved for "never completed"):

| Encoded value | Meaning |
|---|---|
| `$00` | Never completed (matches both fresh battery SRAM and post–ALL CLEAR zeroed SRAM — no init code required) |
| `$01` | 16-0 (perfect) |
| `$02` | 16-1 |
| `$03` | 16-2 |

Only runs with 2 or fewer losses are saved. A run that ends via a 3rd cumulative KD does not update the record. The W/L line on the Circuit Select screen renders only when the encoded byte is non-zero.

**Clearing iron W/L data:**

- **ALL CLEAR** — zeros the entire SRAM (including iron's bytes). Iron records disappear for every slot. No iron-specific code needed; the engine's existing ALL CLEAR routine already covers this.
- **FILE KILL (per-profile delete)** — the engine's per-slot delete routine only zeros the slot record at `$700100 + slot*$80`, which is outside the iron W/L address range. A small hook at `$01:B7D6` (inside `CODE_01B7A1`) extends this to also zero `$700FE8 + slot` and `$701FE8 + slot`, so deleting a profile cleanly removes its iron record. This is iron-only — no behavior change for other circuits.

## Free space consumed

- **Bank `$00` `UNK_00F5D0`** (`$00:F61A–$00:FCD9`): ~1,728 bytes for all iron stubs, the inlined descriptor renderer, and the 32-byte rust BG2 palette. The first 74 bytes of `UNK_00F5D0` (`$00:F5D0–$00:F619`) belong to `spo_alt_glove_colors`.
- **Bank `$01` `UNK_01F784`** (`$01:F784–$01:F7FE`): ~123 bytes for REST_NO_GROW, DRAIN_SKIP, TALLY_NAME, RECOVERY_OVERRIDE, and CONTINUE_RESET. The hard ceiling at `$01:F800` is real game code (`JMP $F815`) — none of these stubs cross it.
- **Bank `$01` `UNK_01FEC2`** (`$01:FED4–$01:FF3E`): ~107 bytes for PHASE_BYPASS, BONUS_ZERO, FAST_CASCADE, the two W/L draw trampolines, and the SLOT_KILL stub. The first 18 bytes of this region (`$01:FEC2–$01:FED3`) belong to `spo_end_credits`'s no-record artifact fix.
- **Bank `$0D`**: 13 bytes for the IRON tally descriptor at `$0D:FA69`.
- **Bank `$08`**: no stub bytes — six in-place hooks only. Iron-staged data lives in low WRAM and is read via the bank-`$08` low-WRAM mirror.

## Patch in asm form

For reference, the full patch expressed as an asm overlay. Records are grouped by phase. The full reasoning for each block lives in [TECHNICAL.md](../TECHNICAL.md). Body bytes for the larger stubs (CHAMP_SCREEN, RANK, DRAW_STUB, PHASE_BYPASS, etc.) are summarized rather than enumerated — those bodies are several hundred bytes each.

```asm
;==============================================================================
; spo_iron_circuit — full overlay
;------------------------------------------------------------------------------
; Adds IRON CIRCUIT (16-opponent gauntlet) to Championship Mode -> Circuit Select.
; Implements HP carry-over, cumulative KD, iron-themed presentation, championship
; belt screen with iron palette + "16 Wins" override, and SRAM W/L persistence.
; ~2,000 bytes of edits across banks $00, $01, $08, and $0D.
;==============================================================================

;------------------------------------------------------------------------------
; Circuit Select layout — descriptor address shifts (file 0x06ADA2-0x06ADC7)
; Move MINOR/MAJOR/WORLD/SPECIAL up 4 BG3 rows so IRON fits at row 22.
; The "<<CIRCUIT SELECT>>" subtitle on BG2 shifts up 4 rows in lockstep.
; Hide CHAMPIONSHIP MODE header by zeroing its descriptor count byte.
;------------------------------------------------------------------------------
org $0DADA2 : db $54        ; SPECIAL top         (was $55) — row 22 -> 18
org $0DADA6 : db $54        ; SPECIAL CIRCUIT     (was $55)
org $0DADAA : db $53        ; WORLD top           (was $54) — row 18 -> 14
org $0DADAE : db $53        ; WORLD CIRCUIT       (was $54)
org $0DADB2 : db $52        ; MAJOR top           (was $53) — row 14 -> 10
org $0DADB6 : db $52        ; MAJOR CIRCUIT       (was $53)
org $0DADBA : db $51        ; MINOR top           (was $52) — row 10 -> 6
org $0DADBE : db $51        ; MINOR CIRCUIT       (was $52)
org $0DADC2 : db $58        ; <<CIRCUIT SELECT>> pt1  (was $59) — BG2 row 6 -> 2
org $0DADC6 : db $58        ; <<CIRCUIT SELECT>> pt2  (was $59)
org $0DADC7 : db $00        ; CHAMPIONSHIP MODE descriptor count byte (was $0D)
                            ; — zero it so the renderer treats it as a terminator

;------------------------------------------------------------------------------
; In-row constants — highlight base, brackets, cursor sprite Y
;------------------------------------------------------------------------------
org $01B8DD : db $01     ; highlight base high byte
org $01B96A : db $00     ; '<' bracket X arg high byte
org $01B96D : db $00     ; '<' bracket Y arg high byte
org $01B999 : db $2F     ; cursor sprite Y constant ($53 = $2F post-shift)

;------------------------------------------------------------------------------
; WIN/LOSS digit base — 26 single-byte rewrites inside CODE_01E5C5
; Each STA.l $7E5Axx,X has its middle byte changed from $5A to $59
; (BG2 row 10 -> row 6, in lockstep with the circuit row shift). The 26
; affected file offsets are 0x0E5ED, 0x0E5F3, 0x0E5F8, 0x0E5FE, 0x0E604,
; 0x0E609, 0x0E60E, 0x0E613, 0x0E618, 0x0E628, 0x0E632, 0x0E636, 0x0E63C,
; 0x0E640, 0x0E64D, 0x0E651, 0x0E65B, 0x0E65F, 0x0E665, 0x0E669, 0x0E66E,
; 0x0E672, 0x0E676, 0x0E67A, 0x0E67E, 0x0E682.
;------------------------------------------------------------------------------

;------------------------------------------------------------------------------
; Descriptor render trampoline (file 0x00B94D)
;------------------------------------------------------------------------------
org $01B94D
    JSL $00F61A           ; was: JSR $D1FC; LDX #$0000
    NOP : NOP              ; orphan bytes from the displaced LDX

;------------------------------------------------------------------------------
; Confirm hook (file 0x00B9CD)
;------------------------------------------------------------------------------
org $01B9CD
    JSL $00F903           ; CONFIRM_RESET wrapper -> CONFIRM stub
    NOP : NOP : NOP : NOP : NOP : NOP : NOP

;------------------------------------------------------------------------------
; POST_MATCH override (file 0x003250)
;------------------------------------------------------------------------------
org $00B250
    JML $00F724           ; replaces DEC $0602 + first byte of JMP $88B1

;------------------------------------------------------------------------------
; DISPATCH redirect (file 0x003270)
;------------------------------------------------------------------------------
org $00B270
    JML $00F747           ; replaces JMP indirect; eats 1 B of UNK_00B273

;------------------------------------------------------------------------------
; BG2 palette hook (file 0x001C18)
;------------------------------------------------------------------------------
org $009C18
    JSL $00F75A           ; was: PHB; LDA #$001F; MVN $00,$10; PLB
    NOP : NOP : NOP : NOP

;------------------------------------------------------------------------------
; Stamina-init override (file 0x0016A5)
;------------------------------------------------------------------------------
org $0096A5
    JSL $00F90D           ; was: LDA #$50; STA $089F
    NOP

;------------------------------------------------------------------------------
; HP-bar tween fix (file 0x0069F7)
;------------------------------------------------------------------------------
org $00E9F7
    JML $00F959           ; was: LDA #$50; STA dp $14 (entered via JMP, must JML)

;------------------------------------------------------------------------------
; Stamina-stash hooks (KO + TKO post-match paths)
;------------------------------------------------------------------------------
org $00B226 : JSR $F948    ; was: JSR $C045  (KO post-match)
org $00B24A : JSR $F948    ; was: JSR $C045  (TKO post-match)

;------------------------------------------------------------------------------
; B231_INIT, BF81 belt-screen palette, KD-increment
;------------------------------------------------------------------------------
org $00B231 : JSR $FA53    ; was: STZ $0603
org $00BF81 : JSR $F9E1    ; was: MVN $00,$10
org $00C6A6 : JSR $F8E0    ; was: LDA dp $9D; TAY; INC; STA dp $9D
              NOP : NOP : NOP

;------------------------------------------------------------------------------
; Bank-$08 hooks (pre-fight title, rank, portrait, belt-screen title/skip/digits)
;------------------------------------------------------------------------------
org $08D19D : JSL $00F7A1                              ; pre-fight title
org $08D1C5 : JSL $00F7D6                              ; pre-fight rank
org $08D2AB : JSL $00F8B9 : NOP                        ; portrait shift (iron+opp2 + inlined SMM; overlaps spo_super_macho_man_fix — iron wins last)
org $08C0D8 : JSL $00FA61                              ; CHAMP_TITLE
org $08C09B : JSL $00FA96                              ; TEXT_SKIP (opponent records)
org $08C0AE : JSL $00FA96                              ; TEXT_SKIP (TOTAL SCORE label)
org $08C0C7 : JSL $00FA96                              ; TEXT_SKIP (TOTAL SCORE numerals)
org $08D74E : JSL $00FAA5                              ; W/L digit-top
org $08D755 : JSL $00FACE                              ; W/L digit-bottom

;------------------------------------------------------------------------------
; Bank-$01 hooks (REST freeze, drain skip, recovery, continue, score-tally,
; phase-bypass, bonus-zero, fast-cascade, aggregate-suppression, show-special,
; iron W/L draw trampolines)
;------------------------------------------------------------------------------
org $01AAF2 : JSR $F784    ; REST +1 award suppression
              NOP : NOP : NOP : NOP : NOP : NOP : NOP : NOP : NOP
              NOP : NOP : NOP : NOP : NOP : NOP : NOP : NOP
org $01A2EC : JSR $F7A5    ; champion-drain skip
org $01A381 : JSR $F7A5    ; champion-drain skip
org $019EEC : JSR $F7B2    ; score-tally circuit-name
              NOP : NOP : NOP
org $01A27E : JSR $F7D0    ; recovery formula override
org $019EBB : JSR $F7E0    ; continue-reset
org $019F72 : JSL $01FED4  ; phase-bypass
              NOP : NOP : NOP : NOP
org $01A99C : JSR $FEF5    ; stamina-bonus zero
org $01A0C7 : JSR $FF02    ; fast-cascade
              NOP
org $01A416 : JSR $F7A5    ; aggregate-screen suppression (reuses DRAIN_SKIP)
org $01B93C : LDA #$03 : NOP        ; show-special: Championship Mode
org $01BCDB : LDA #$03              ; show-special: Time Attack
org $01B959 : JSR $FF20             ; iron W/L draw trampoline (post JSR E5C5)
org $01BD29 : JSR $FF26             ; iron W/L draw trampoline (post JSR CB57)
org $01B7D6 : JSR $FF30 : NOP       ; slot-kill hook (clear iron W/L on FILE KILL)

;------------------------------------------------------------------------------
; Stub bodies (bank $00, UNK_00F5D0 free space)
;------------------------------------------------------------------------------
org $00F61A
    ; Inlined CODE_01D1FC body + POST_RENDER (sentinel-fill rows 22-23, write
    ; IRON CIRCUIT tile bytes, force $14 = 4, STZ $24, LDX #$0000, RTL).
    ; ~226 bytes including the 22-byte tile table.

org $00F702
    ; CONFIRM stub: writes $7E:1FE1 = (cursor>>2). Iron path forces $0600 = 0,
    ; $0601 = 3 (Special) so all circuit-themed asset loads inherit Special's
    ; data. Non-iron path is the original 11-byte sequence.

org $00F724
    ; POST_MATCH stub: replays DEC $0602; iron path drives the 16-opp chain
    ; ($0602 = $0600 EOR $0F so $0602 == 0 only on $0600 == $0F, firing the
    ; engine's CODE_00D6C5 celebration gate on fight 16); on $0600 >= $10,
    ; clears the iron flag and JMLs to credits. 35 bytes.

org $00F747
    ; DISPATCH stub: on iron, clears flag + JML credits; on vanilla, JMP
    ; indirect (replicates the original $00:B270 dispatch). 19 bytes.

org $00F75A
    ; BG2 palette swap: gated on iron flag $7E:1FE1 OR iron belt-screen flag
    ; $7E:1FEC. On iron path MVN rust palette from bank $00; on vanilla MVN
    ; from bank $10 with the caller-set X. The $7E:1FEC gate is what makes
    ; the BEST TIME inside CHAMP_SCREEN_STUB render the rust palette (the
    ; iron flag is cleared by the iron-complete path before CHAMP_SCREEN
    ; runs). 39 bytes.

org $00F781
    ; Iron rust BG2 palette: 16 BGR15-LE colors, 32 bytes.

org $00F7A1
    ; Pre-fight title stub: on iron writes "IRON\0" letter bytes to $7E:1FE5
    ; and points Y at the runtime-staged string; on vanilla replicates the
    ; eaten LDA $0000,y; TAY. 53 bytes.

org $00F7D6
    ; Iron rank renderer: iron_rank = 15 - $0600. Renders BG1 hash + BG3
    ; 1-or-2 small-font digit tiles, with conditional col shifts for PISTON
    ; HURRICANE / champ-pos 2-digit ranks. ~227 bytes.

org $00F8B9
    ; Portrait shift: on iron+opp2 -> $38AC (+PISTON_PORTRAIT_X_PX); on opp11
    ; (SUPER MACHO MAN, vanilla or iron) -> $38AC (inlines spo_super_macho_man_fix
    ; logic for standalone safety); else -> $38A8 (default). 39 bytes.

org $00F8E0
    ; KD_HOOK: replicates eaten LDA dp $9D; TAY; INC; STA dp $9D; on iron
    ; INCs $7E:1FEA and forces $089D = 3 when cumulative reaches 3.
    ; ~35 bytes.

org $00F903
    ; CONFIRM_RESET wrapper: on iron entry clears $7E:1FEA, $7E:1FEC,
    ; $7E:1FEE, and inits $7E:1FEB to $FF; chains to CONFIRM at $00:F702.
    ; 20 bytes.

org $00F917
    ; Stamina init override: vanilla -> write $50; first-iron -> write $50,
    ; init sentinel; carry-over -> A = stash, ADC #$10, cap $50, write $089F.
    ; All paths exit with A = $50 for the caller's STA $099F.
    ; ~49 bytes.

org $00F948
    ; Stamina stash: on iron, copies $089F -> $7E:1FEB; replicates the eaten
    ; JSR $00:C045. Shared by KO and TKO post-match paths. ~17 bytes.

org $00F959
    ; HP-bar tween fix: on iron + carry-over seeds $0B14 to the carry value,
    ; pre-fills tile positions 1-9 in both WRAM rows ($50CC..$50DC and
    ; $510C..$511C) with empty-bar tile $38, then JMLs to $00:E9FB.
    ; ~93 bytes.

org $00F9B6
    ; CHAMP_SCREEN: JSR SAVE_STUB; sets $0602 = $80, $0603 = 1, $7E:1FEC = 1,
    ; $0600 = $0F (Nick Bruiser) and $0601 = $03 (Special), mirroring vanilla
    ; CODE_00BD2F's `LDY #$030F; STY $0600` (X=16 writes both bytes). The
    ; $0601 restore is required because the post-celebration state machine
    ; can drift it off iron's hijacked $03 — without this, BEE8's BG graphics
    ; lookup at $00:BF66 (DATA_128000+$0601*2) reads the wrong entry and the
    ; belt screen renders Minor's BG under the rust palette. JSR $C045 (BEST
    ; TIME); JSR $BEE8 (belt screen); clears $7E:1FEC; JML $00:B29C (credits).
    ; 43 bytes.

org $00F9E1
    ; CHAMP_PAL: replaces MVN $00,$10 inside CODE_00BF66 with iron rust BG2
    ; MVN to $0460, blue-belt MVN to $0440, blue-belt sprite MVN to $05C0.
    ; M=16 throughout; vanilla path restores A = $001F before original MVN.
    ; ~50 bytes.

org $00FA53
    ; B231_INIT: replicates STZ $0603; clears $7E:1FEC and $7E:1FED via
    ; LDA #$00; STA.l (M=16 read at BF81 needs both bytes zero). 14 bytes.

org $00FA61
    ; CHAMP_TITLE: stages "IRON\0" letter bytes into $7E:1FE5..$1FE9, points
    ; Y at $1FE5, INX INX (1-col centering for the 4-letter "IRON" name).
    ; Vanilla replicates eaten LDA $0000,y; TAY. ~53 bytes.

org $00FA96
    ; TEXT_SKIP: on iron + $7E:1FEC, returns immediately (skips the displaced
    ; JSL CODE_08D80A); on vanilla replicates the JSL. 14 bytes; reused at
    ; three call sites.

org $00FAA5
    ; W/L digit override: top + bottom hooks combined. Writes "1" at col 10
    ; and "6" at col 11 on iron belt screen, replacing vanilla's "4" at col
    ; 11. CLC before RTL is mandatory (caller's next ADC #$0010 reads C).
    ; 80 bytes total (top entry $00:FAA5, bottom entry $00:FACE).

org $00FAF5
    ; SAVE_STUB: reads $7E:1FEE, caps to 0..2, INCs to encoded form, compares
    ; to stored $700FE8 + slot, dual-writes primary AND backup if improvement.
    ; 53 bytes.

org $00FB2A
    ; DRAW_STUB: reads $700FE8 + $0608; if zero (never completed), RTL.
    ; Else DEC A, dispatches via BNE+BRL to one of three loss-block bodies
    ; (0L / 1L / 2L) each writing 22 tile/attr pairs to BG3 row 22 ($5DA8..).
    ; Tile pattern: "1 6 W I N {N} L O S S [E S]" (the trailing "ES" only
    ; renders for N != 0 to read as "LOSSES"). 432 bytes.

;------------------------------------------------------------------------------
; Stub bodies (bank $01, UNK_01F784)
;------------------------------------------------------------------------------
org $01F784
    ; REST_NO_GROW: on iron skips both $0606 INC and visible REST tile
    ; bumps at $7E:567C / $7E:56BC; vanilla replicates the eaten 20-byte
    ; block. 33 bytes.

org $01F7A5
    ; DRAIN_SKIP: on iron returns A = 0 to defeat $0600 mod 4 == 3 gates;
    ; on vanilla replicates LDA $0600. Shared by champion-bonus drain hooks
    ; and aggregate-screen suppression hook. 13 bytes.

org $01F7B2
    ; TALLY_NAME: on iron points $00D0 (absolute, NOT DP) at IRON tally
    ; descriptor at $0D:FA69 and chains to CODE_01A7BD; on vanilla replicates
    ; LDA $9F13,X; JSR CODE_01A7A6. 30 bytes.

org $01F7D0
    ; RECOVERY_OVERRIDE: on iron returns A = $7E:1FEA (cumulative KD); on
    ; vanilla returns A = $089D. Caller's EOR/DEC/multiply runs on the
    ; returned value. 15 bytes.

org $01F7E0
    ; CONTINUE_RESET: replicates DEC $0606; on iron clears $7E:1FEA, sets
    ; $7E:1FEB to $FF (sentinel), INCs $7E:1FEE (loss counter). 31 bytes;
    ; hard ceiling at $01:F800 (real game code follows).

;------------------------------------------------------------------------------
; Stub bodies (bank $01, UNK_01FEC2)
;------------------------------------------------------------------------------
org $01FED4
    ; PHASE_BYPASS: on iron sets dp $78 = 1 (1-frame idle), drops the JSL
    ; frame (PLA x 3), JMPs to $01:C752 (advance state past the drain phase
    ; entirely). On vanilla replicates eaten JSR $A859; BEQ; JMP $A298.
    ; Catches BOTH dispatcher and A-press skip paths. 33 bytes.

org $01FEF5
    ; BONUS_ZERO: on iron returns A = 0 (downstream multiply produces 0 ->
    ; stamina-bonus contribution to score is 0); vanilla replicates LDA
    ; $00D0. 13 bytes.

org $01FF02
    ; FAST_CASCADE: on iron writes dp $78 = 1 (cascade runs on consecutive
    ; frames); vanilla writes dp $78 = 8. 16 bytes.

org $01FF20
    ; IRON_WL_TRAMP pair: 8-byte trampoline for JSR E5C5 + 8-byte trampoline
    ; for JSR CB57. Each does "JSR <original>; JSL DRAW_STUB; RTS". 16 bytes.

org $01FF30
    ; SLOT_KILL: called from $01:B7D6 inside the per-profile FILE KILL routine.
    ; X = slot index (0..7) on entry. Writes $00 to $700FE8,X (primary) and
    ; $701FE8,X (backup), then replicates the eaten LDA.l $700005 instruction
    ; before returning. Only fires on per-slot FILE KILL — ALL CLEAR uses a
    ; different routine (CODE_01ABAD) that already zeros all SRAM. 15 bytes.

;------------------------------------------------------------------------------
; Data (bank $0D, UNK_0DFA69)
;------------------------------------------------------------------------------
org $0DFA69
    ; IRON tally descriptor: dw $55C6, db $2C $07, db "   IRON" (small-font
    ; tile bytes), dw $0000 terminator. Renders "   IRON CIRCUIT TOTAL" on
    ; the post-match score-tally screen (the existing "CIRCUIT TOTAL"
    ; descriptor follows naturally). 13 bytes.
```

The stub-body comments above describe what each block does at a high level. Larger stubs (RANK ~227 B, DRAW ~433 B, CHAMP_PAL ~50 B, CHAMP_TITLE ~53 B) and the tightly-coupled bank-`$00` v19 block (`$00:F61A–$00:F758`, ~320 B covering the inlined renderer + CONFIRM + POST_MATCH + DISPATCH) are summarized rather than enumerated — the IPS file itself is the authoritative byte-level source.

## Compatibility

- **Apply on top of**: bare `spo.sfc` (MD5 `97fe7d7d2a1017f8480e60a365a373f0`).
- **Bundled into**: [`spo_special_edition_v1.7.ips`](../../patches/spo_special_edition_v1.7.ips) — the recommended distribution for this hack.
- **Conflicts with**: nothing that breaks functionality. One address overlap exists at file `0x0452AB` (`$08:D2AB`) with `spo_super_macho_man_fix.ips` and `spo_alt_glove_colors.ips` — both write `JSL $0D:FD87` there while iron writes `JSL $00:F8B9`. IPS last-write wins. Iron's stub handles both the Piston Hurricane portrait shift (iron+opp2) and the Super Macho Man portrait shift (opp11, inlined), so all three patches work correctly regardless of apply order. The bundled `spo_special_edition_v1.7.ips` applies iron last.
- **Cheat-code compatibility**: all known Action Replay / Game Genie codes for the original ROM remain unaffected — iron hooks do not modify opponent palettes, fight-state machinery, or any ROM location targeted by existing cheat codes.
- **SNES header checksum**: the standalone patch updates the header checksum field at file `0x7FDC..0x7FDF` so a vanilla ROM patched with iron alone reads as self-consistent. The bundled `spo_special_edition_v1.7.ips` likewise stamps a correct checksum for the combined SE 1.7 ROM. SNES emulators do not enforce this field.

## See also

- [TECHNICAL.md](../TECHNICAL.md) — full reverse-engineering notes, stub bodies at byte level, free-space inventory, and the Mode Select / Championship Mode state machine architecture.
- [VERSUS_HACK.md](VERSUS_HACK.md) — the core hack iron builds on indirectly. Iron's confirm hook and SPECIAL CIRCUIT show-always logic both cooperate with VERSUS_HACK's Special-Circuit lock bypass at file `0x003C23`.
