# `spo_versus_hack.ips` — Summary

The core hack of this repo. Adds VERSUS MODE to the Mode Select menu, lets either controller pick on the opponent-select screen, and disables the Special Circuit security checksum lock.

<img width="256" height="224" alt="Super Punch-Out!! Special Edition (USA)_001" src="https://github.com/user-attachments/assets/11d6251a-c56d-46ea-a4e4-e26b297e24a0" />
<img width="256" height="224" alt="Super Punch-Out!! Special Edition (USA)_002" src="https://github.com/user-attachments/assets/195a7883-e233-4066-ad3d-49f829872569" />

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

## Patch in asm form

For reference, the full patch expressed as an asm overlay. Records are grouped by bank and listed in file-offset order, matching records [1] through [34] in [doc/TECHNICAL.md § 5](../TECHNICAL.md#5-versus-hack--patch-by-patch-breakdown). The full reasoning for each block lives there.

```asm
;==============================================================================
; spo_versus_hack — full overlay
;------------------------------------------------------------------------------
; Adds VERSUS MODE to the Mode Select menu, lets either controller pick on
; the opponent-select screen, and disables the Special Circuit security
; checksum lock. ~700 bytes of edits across banks $00, $01, and $0D.
;==============================================================================

;------------------------------------------------------------------------------
; [1] Special Circuit lock bypass (file 0x003C23)
;------------------------------------------------------------------------------
org $00BC23
    LDA #$00              ; was: ORA $D5 (05 D5) — force unlocked

;------------------------------------------------------------------------------
; [3] P2 control hook: JSL to FC4C dispatch stub (file 0x080CA)
;------------------------------------------------------------------------------
org $0180CA
    JSL $0DFC4C           ; dispatch stub (record [31])
    BNE +
    JMP $80DE             ; flag was set: continue normally
+   JMP $806F             ; not set: skip

;------------------------------------------------------------------------------
; [2] Conditional VERSUS/TIME ATTACK header stub (file 0x008018)
;     Sits in dead-loop jump-table slots at $8018/$801B/$801E plus 9 bytes
;     of free space at $8021-$8029.
;------------------------------------------------------------------------------
org $018018
    PHA                   ; save Y-arg (A = TYA result = $0C or $0E)
    LDA $0607             ; read mode flag
    CMP #$03              ; VERSUS?
    BEQ +                 ; branch BEFORE PLA (PLA would clobber Z flag)
    PLA                   ; non-VERSUS: restore Y-arg
    JMP $D1FC             ; tail-call descriptor renderer
+   PLA                   ; VERSUS: restore Y-arg
    LDA #$A4              ; override descriptor index to VERSUS descriptor
    JMP $D1FC

;------------------------------------------------------------------------------
; [4] Trampoline 1: conditional TA disable (file 0x008457)
;------------------------------------------------------------------------------
org $018457
    BNE +                 ; if $700010,X != 0 (has progress): skip to RTS
    LDA $0607
    CMP #$03
    BEQ +                 ; if VERSUS: skip to RTS (leave TA enabled)
    INC $22               ; TA + no progress: disable TIME ATTACK
+   RTS

;------------------------------------------------------------------------------
; [5] Back-out clear stub (file 0x008463)
;------------------------------------------------------------------------------
org $018463
    STZ $0607             ; clear mode flag on back-out
    JMP $C766             ; continue to original epilog

;------------------------------------------------------------------------------
; [6]-[10] Mode Select menu rewrite (5-item layout)
;------------------------------------------------------------------------------
org $01BB93 : db $04          ; [6]  item count: 4 → 5
org $01BB99 : db $01          ; [7]  highlight base $024C → $014C (row 9 → row 5)

; [8] Disable flag init (4 bytes across $01BBA3, $01BBA7, $01BBB3-BBB4)
;     Net effect: VERSUS always enabled, TIME ATTACK gated on progress,
;     RECORDS VIEW gated on progress, BUTTON SETTINGS always enabled.

; [9] Progress gate rewrite (file 0x00BBE2-BBF2)
org $01BBE2
    JSR $8457             ; trampoline 1: conditional TA disable
    NOP
    STZ $24               ; BUTTON SETTINGS always enabled
    LDA $700010,X
    BEQ +
    STZ $23               ; RECORDS VIEW: enabled with progress
+   NOP : NOP : NOP

; [9b] Decoration tilemap + arrow positions for 5-item layout
org $01BBC7 : dw $5854        ; MENU/SELECT decoration   (was $5994)
org $01BBD0 : dw $0050        ; left arrow position      (was $0110)
org $01BBD3 : dw $006E        ; right arrow position     (was $012E)

org $01BC1D : db $27          ; [10] OAM cursor Y base $47 → $27

;------------------------------------------------------------------------------
; [11]-[13] Trampoline + redirect hooks
;------------------------------------------------------------------------------
org $01BCAA+1 : dw $FFD3      ; [11] JSR target → trampoline 2 (circuit count)
org $01BD0F+1 : dw $8018      ; [12] JSR target → conditional header stub
org $01BEEF+1 : dw $8463      ; [13] JMP target → back-out clear stub

;------------------------------------------------------------------------------
; [14] Mode Select confirm: jump to dispatch stub (file 0x00C90E)
;------------------------------------------------------------------------------
org $01C90E
    JMP $FFB0             ; was: CMP #$01; BEQ ... (top of original 4-item chain)

;------------------------------------------------------------------------------
; [27] Init hook for Versus opponent-select table-hide (file 0x0BE7A)
;------------------------------------------------------------------------------
org $01BE7A
    JSL $0DFB62           ; was: JSL $0EF3DD

;------------------------------------------------------------------------------
; [28] Char-switch hook for Versus opponent-select table-hide (file 0x0BEC8)
;------------------------------------------------------------------------------
org $01BEC8
    JSL $0DFB98           ; was: LDX #$000F / JSR $D7BC
    NOP : NOP

;------------------------------------------------------------------------------
; [34a] P2-mirrors-P1 hook on Versus opponent-select (file 0x00BECE)
;------------------------------------------------------------------------------
org $01BECE
    JSL $0DFC91           ; was: JSR $D8F7; CMP #$09
    NOP                   ; padding to keep downstream BEQ offset

;------------------------------------------------------------------------------
; [32] Post-match epilog hook (file 0x00C936)
;------------------------------------------------------------------------------
org $01C936
    JML $0DFC72           ; was: STZ $01; LDA #$0C; STA $03
    NOP : NOP

;------------------------------------------------------------------------------
; [15]-[18] 5-item Mode Select dispatch stub + handlers (bank $01 free space)
;------------------------------------------------------------------------------
org $01FFB0
                          ; [15] dispatch stub
    CMP #$01
    BEQ versus_handler    ; → $FFC9
    CMP #$02
    BEQ ta_trampoline     ; → $FFC3
    CMP #$03
    BEQ rv_trampoline     ; → $FFC6
    LDA #$06              ; BUTTON SETTINGS fall-through
    STA $03
    JMP $C951

ta_trampoline:            ; [16] $01:FFC3 — TIME ATTACK
    JMP $C926

rv_trampoline:            ; [16] $01:FFC6 — RECORDS VIEW
    JMP $C930

versus_handler:           ; [17] $01:FFC9 — VERSUS launcher
    LDA #$03
    STA $0607             ; mode flag = VERSUS
    STZ $05
    JMP $C74C

                          ; [18] $01:FFD3 — trampoline 2: conditional circuit count
    LDA $0607
    CMP #$03
    BEQ +                 ; VERSUS: force 4 circuits
    JMP $D641             ; TA: tail-call SRAM circuit-count reader
+   LDA #$04
    RTS

;------------------------------------------------------------------------------
; [33b] Mode-flag rewrite hook (file 0x031D9 / SNES $00:B1D9)
;------------------------------------------------------------------------------
org $00B1D9
    JSL $0DFD99           ; was: JSL $01EC00

;------------------------------------------------------------------------------
; [19]-[26] DA332 / DA3DA pointer redirects + new tile/layout data (bank $0D)
;------------------------------------------------------------------------------
org $0DA38C : dw $FB03        ; [19] DA332[$2D] → "PLAYER 2" tile data
org $0DA3A0 : dw $FAC3        ; [20] DA332[$37] → "VERSUS"   tile data
org $0DA3DE : dw $FA86        ; [21] DA3DA entry 2 → new Mode Select layout
org $0DA47E : dw $FACA        ; [22] DA3DA byte-offset $A4 → VERSUS opponent-select descriptor

; [23] New Mode Select layout sub-table (61 bytes)
org $0DFA86
    db $0D,$1C,$4E,$51    ; CHAMPIONSHIP  row 5  col 7
    db $0C,$1C,$6A,$51    ; MODE          row 5  col 21
    db $37,$1C,$54,$52    ; VERSUS        row 9  col 10
    db $0C,$1C,$64,$52    ; MODE          row 9  col 18
    db $0F,$1C,$50,$53    ; TIME          row 13 col 8
    db $12,$1C,$5A,$53    ; ATTACK        row 13 col 13
    db $0C,$1C,$68,$53    ; MODE          row 13 col 20
    db $10,$1C,$4E,$54    ; RECORDS       row 17 col 7
    db $0E,$1C,$5E,$54    ; VIEW          row 17 col 15
    db $0C,$1C,$6A,$54    ; MODE          row 17 col 21
    db $11,$1C,$4C,$55    ; BUTTON        row 21 col 6
    db $13,$1C,$5A,$55    ; SETTING       row 21 col 13
    db $0C,$20,$54,$58    ; MODE          header row
    db $05,$20,$5C,$58    ; <<SELECT      header row
    db $00                ; terminator

; [24] "VERSUS" tile data (length-prefixed)
org $0DFAC3
    db $06, $1F,$0E,$1B,$1C,$1E,$1C    ; V,E,R,S,U,S

; [25] 12-record VERSUS opponent-select descriptor (49 bytes)
;      Full circuit rows + VERSUS MODE header + < PLAYER 2 SELECT >
org $0DFACA
    ; (full record set — see TECHNICAL.md record [25] for details)
    db $0A,$1C,$84,$53    ; SPECIAL   row 14 col 2
    db $0B,$1C,$94,$53    ; CIRCUIT   row 14 col 10
    ; …
    db $37,$0C,$94,$50    ; VERSUS    row 2  col 10
    db $0C,$0C,$A2,$50    ; MODE      row 2  col 17
    db $05,$20,$60,$59    ; <<SELECT  row 37 col 16
    db $2D,$20,$50,$59    ; PLAYER 2  row 37 col 8
    db $00                ; terminator

; [26] "PLAYER 2" tile data
org $0DFB03
    db $08, $19,$15,$0A,$22,$0E,$1B,$EF,$02   ; P,L,A,Y,E,R,space,2

;------------------------------------------------------------------------------
; [29]-[30] Versus opponent-select table-hide stubs (bank $0D)
;------------------------------------------------------------------------------
; [29] $0D:FB62 — init blanking stub (54 bytes)
;      Calls JSL $0E:F3DD (original work), then if $0607 == $03 (Versus),
;      blanks a 30-col × 7-row rectangle on BG1 ($5482) and BG3 ($5C82)
;      by writing $0000 (tile 0, palette 0).
org $0DFB62
    ; (full body in TECHNICAL.md record [29])

; [30] $0D:FB98 — char-switch blanking stub (71 bytes)
;      Inlines CODE_01D7BC's body (cross-bank LDA.l of bank-$01 lookup table),
;      then if $0607 == $03, blanks BG3 only.
org $0DFB98
    ; (full body in TECHNICAL.md record [30])

;------------------------------------------------------------------------------
; [31] $0D:FC4C — P2 control dispatch stub (36 bytes)
;------------------------------------------------------------------------------
org $0DFC4C
    LDA $0607             ; read mode flag
    CMP #$02              ; secret VS button-combo mode?
    BEQ secret_path
    CMP #$03              ; VERSUS mode?
    BEQ force_p2
    LDA #$01
    RTL                   ; not VS-related, return A != 0
secret_path:
    LDA $00AB             ; read held-button flags
    AND $00AF
    BMI force_p2          ; combo held: set flag
    LDA #$01
    RTL                   ; combo not held, return A != 0
force_p2:
    LDA #$08
    STA $30               ; P2 boss-control flag = $08
    LDA #$80
    STA $3D               ; supplemental flag = $80
    LDA #$00
    RTL                   ; return A = 0 (flag set)

;------------------------------------------------------------------------------
; [33] $0D:FC72 — post-match epilog stub (31 bytes)
;------------------------------------------------------------------------------
org $0DFC72
    LDA $0607
    CMP #$03              ; VERSUS?
    BEQ versus_branch
    ; non-VERSUS: replicate original epilog
    STZ $01
    LDA #$0C
    STA $03               ; circuit-select state
    JML $01C93C
versus_branch:
    STZ $01
    STZ $03
    STZ $05
    JML $01C951           ; PLB; RTL — return to top-level loop
    NOP : NOP : NOP : NOP ; padding

;------------------------------------------------------------------------------
; [34b] $0D:FC91 — P2-mirrors-P1 merge + poll stub (216 bytes)
;------------------------------------------------------------------------------
; Gates on $0607 == $03 (Versus opponent-select). Edge-detects P2's newly-
; pressed buttons via held XOR prev-held AND held (using $00A6/$00A7 as
; shadow). Merges P2's A / Start / D-pad into P1's pressed-this-frame
; globals ($0095 / $00A1 / $0092-$0093) so the menu's input dispatch sees
; either controller. Then PEA + JMP into a 123-byte verbatim copy of
; $01:D8F7's body so its RTS lands inside the stub. Finally re-issues the
; displaced CMP #$09 before RTL.
org $0DFC91
    ; (full body in TECHNICAL.md record [34])

;------------------------------------------------------------------------------
; [33b cont.] $0D:FD99 — mode-flag rewrite stub (17 bytes)
;------------------------------------------------------------------------------
org $0DFD99
    LDA $0607
    CMP #$03
    BNE +                 ; not VERSUS: skip rewrite
    LDA #$02
    STA $0607             ; impersonate secret-mode flag for shared dispatcher
+   JSL $01EC00           ; tail-call displaced original
    RTL
```

The `; (full body in TECHNICAL.md record [N])` placeholders cover three large stubs (records [29], [30], [34b]) where the byte-level body is already enumerated in TECHNICAL.md and would just duplicate it here.

## Compatibility

- **Apply on top of**: bare `spo.sfc` (MD5 `97fe7d7d2a1017f8480e60a365a373f0`)
- **Bundled into**: `spo_special_edition_v1.7.ips`
- **Conflicts with**: `spo_disable_security_checksum.ips` writes the same value to file `0x003C23` (the Special Circuit lock-bypass byte) — applying both is a harmless no-op double-write.
- **Cheat-code compatibility**: unaffected

## See also

For the full reverse-engineering reference — every record, the Mode Select state machine architecture, the VERSUS opponent-select reuse strategy, the P2-mirrors-P1 input merge stub, the post-match routing, free-space inventory, and key-address quick reference — see [doc/TECHNICAL.md](../TECHNICAL.md), particularly:

- Section 2: The secret 2-player mode
- Section 3: Title screen state machine
- Section 4: Mode Select architecture
- Section 5: Versus Hack — patch-by-patch breakdown (records [1]–[34])
- Section 7: Free space map
