# `spo_alt_colors.ips` — Per-Opponent Alt Palettes

**Status:** in development. Two throwaway proof-of-concept ROMs built (Gabby Jay only, ungated). Architecture decided, full feature not yet implemented.

**Target shipping bundle:** SE 1.7 ships with `spo_iron_circuit.ips` plus the ten existing standalone patches (versus, alt-glove-colors, profile fixes, typo fixes, title-screen tweaks, JP charset, end-credits, security-checksum). This `spo_alt_colors.ips` patch is intended as a follow-up addition to a future SE bundle once the per-fighter palette authoring is complete.

---

## What this patch does

Provides per-opponent alternate sprite palettes for all 14 fighters across three sprite contexts:

1. **In-fight body sprite** — the opponent you're punching (CGRAM sprite-palette 4)
2. **Small portrait** — used in the in-fight HUD / opponent-select / circuit-select context (load destination not yet confirmed)
3. **Large portrait** — the big pre-fight profile screen ("CHAMP. GABBY JAY", same screen `SUPER_MACHO_MAN_FIX` shifts Super Macho Man's portrait on)

Each opponent gets one alt palette per context (3 alt palettes × 14 fighters = 42 alt palettes total).

### Gating behavior

The alt palettes activate under two conditions, applied independently per context:

| Context              | Iron-circuit auto-trigger | SELECT-held override |
|----------------------|---------------------------|----------------------|
| In-fight body        | yes                       | yes                  |
| Small portrait       | yes                       | no                   |
| Large portrait       | yes                       | no                   |

- **Iron-circuit auto-trigger:** when `$7E:1FE1 == 1` (set by `spo_iron_circuit.ips` on the iron confirm-hook), all three alt palettes activate automatically. Default-on, no user opt-in.
- **SELECT-held override:** the in-fight body alt palette can be force-activated by holding SELECT during the pre-fight button-decode window. This is a *match-only* override and does not affect the portraits.
- **Compatibility with alt-glove modes:** the SELECT-held opponent override is orthogonal to the existing alt-glove L/R/X/Y/L+R glove-mode chord. Holding L+SELECT enters the fight with an alt opponent palette **and** the L-mode (blue) gloves; SELECT alone enters with alt opponent + vanilla-default gloves.

---

## Architecture: independent patches, shared contract

This patch and `spo_iron_circuit.ips` are designed as **independent IPS patches that cooperate via a single shared WRAM byte**:

- **`$7E:1FE1`** — iron-circuit-active flag. Written by `spo_iron_circuit.ips` (set on iron-circuit confirm-hook, cleared on post-match / DISPATCH safety net). Read by this patch (gates all three alt palettes default-on).

Either patch can be installed without the other:

| Stack                              | Behavior                                                                          |
|------------------------------------|-----------------------------------------------------------------------------------|
| iron alone                         | iron flag toggles, no palette reaction (vanilla iron look)                        |
| alt-colors alone                   | iron flag never set, iron auto-trigger never fires, SELECT-held override still works |
| both                               | full feature                                                                      |

Neither patch references the other's code. The contract is the byte address `$7E:1FE1` and its 0/1 semantics.

A future SE bundle will fold both patches into a single user-facing IPS, but they continue to be developed and tested as independent build artifacts.

### Implementation pattern: sibling hook, not extension

Rather than extending `alt_glove_colors v1`'s fight-init trampoline (`$0D:FDD2`), this patch installs a **sibling hook** at a nearby vanilla site, with its own trampoline in free space. Reasons:

- alt-glove v1 ships in SE 1.7 — modifying its bytes would require coordinating with the SE baseline.
- Sibling hooks let each patch own its bytes cleanly. Diffing for collisions is mechanical.
- Cost is ~tens of bytes of extra trampoline code; well within free-space budget.

(An earlier `alt_glove_colors_v2` design was sketched but abandoned. Only `alt_glove_colors v1` ships — this patch builds against v1's hook layout.)

---

## What's verified empirically

### In-fight body palette (Gabby Jay)

- **Source:** `Sprite_GabbyJay.bin` sub-palette 2 (file offset `0x06BA1A..0x06BA39`, 16 colors / 32 bytes)
- **Destination:** CGRAM sprite-palette 4 (`$80..$8F` words, displayed as rows "C0..CF" in Mesen's CGRAM viewer during a Gabby Jay fight)
- **Verification method:** built `output/spo_alt_opp_palette_gabby_v1.sfc` (build script: `work/scripts/build_alt_opp_palette_gabby_v1.py`) which overwrites 9 of the 16 slots in Pal 2 with user-chosen alt colors. User confirmed the alt colors appear on Gabby's in-fight sprite.
- **Slots changed in PoC:** C1 (#500000), C2 (#702810), C7 (#000000), C8 (#340101), C9 (#702818), CA (#743D31), CB (#585868), CC (#8888AA), CD (#FFFFFF). Slots C0 (transparent backdrop), C3-C6 (skin tones), CE (red glove peak), CF (white) preserved from vanilla.

### Large pre-fight portrait palette (Gabby Jay)

- **Source:** `Sprite_LargePortraits.bin` slot 0 (file offset `0x087E00..0x087E1F`, 16 colors / 32 bytes)
- **Asset structure:** the `.bin` is 0x200 bytes total = **16 fighter slots × 32 bytes each**, in fighter-index order. Gabby Jay = index 0 (confirmed by cross-referencing `DATA_108000` GFX pointer table in the disassembly, which lists `GFX_Sprite_GabbyJayLargePortrait` first).
- **Destination CGRAM region:** not yet pinned down (loads on entering the pre-fight profile screen, not at fight-init).
- **Verification method:** built `output/spo_alt_opp_portrait_gabby_v1.sfc` (build script: `work/scripts/build_alt_opp_portrait_gabby_v1.py`) which converts all 15 non-backdrop slots to luminance-preserving grayscale. User confirmed Gabby's pre-fight large portrait appears in B&W.

### LoROM mapping verified

- SPO is **LoROM**. Asset-table addresses in `work/disassembly/SPO/AsarScripts/AssetPointersAndFiles.asm` are SNES bus addresses; ROM file offsets are computed via `((bank & 0x7F) << 15) | (offset & 0x7FFF)`.
- Worked examples:
  - `$0DB9DA` (PAL_Sprite_GabbyJay) → file `0x06B9DA`
  - `$10FE00` (Sprite_LargePortraits) → file `0x087E00`
- This was a real bug in v1 of the body-palette PoC (initially patched HiROM offset `0x0DB9DA`, which is garbage data in the `.bin` zone). Caught by byte-comparing `Sprite_GabbyJay.bin` against `work/spo.sfc`.

---

## What's known but not verified end-to-end

### Small portrait palette

- **Suspected source:** `Sprite_GabbyJay.bin` sub-palette 0 (file offset `0x06B9DA..0x06B9F9`).
- **Reasoning:** the `.bin` has 3 sub-palettes (Pal 0, Pal 1, Pal 2). Pal 2 is the in-fight body (verified). Pal 1 has a yellow/gold ramp consistent with a "stunned/star" or "powered-up" variant. Pal 0 has skin tones + gray accents that match a portrait-type sprite.
- **What's not verified:** whether Pal 0 actually drives an on-screen sprite, and which screen it appears on (in-fight HUD? circuit-select? opponent-select?). The ungated v1 PoC initially patched Pal 0 thinking it was the body palette; the user reported it changed "the portrait" — so something in the portrait class, but the exact context wasn't pinned down before we pivoted to Pal 2.

### Pal 1 of `Sprite_<Opp>.bin`

- **Status:** identity unknown. Yellow/gold ramp suggests a powered/stunned/special-effect variant. Not on the spec for this patch (only 3 contexts: body + small portrait + large portrait).

---

## Open questions / next investigation steps

1. **Confirm small portrait source = Pal 0 and identify its on-screen context.** Strategy: source-bomb test ROM (`output/spo_test_gabby_palette_locator.sfc`, build script `build_test_gabby_palette_locator.py`) paints Pal 0 cyan, Pal 1 yellow, Large Portrait magenta. Run, observe each game screen, record which on-screen object turns which color and which CGRAM palette slot holds it.

2. **Find the load site for the in-fight body palette.** The alt-glove docs note the engine DMAs the contender palette at fight-init (alt-glove Hook 1 specifically overwrites this). Need to find that DMA instruction or the immediately-following read site to know where to install our sibling hook. Mesen CGRAM-write breakpoint is the fast path.

3. **Find the load site for the large portrait palette.** Disassembly references three `MVN $00,DATA_10FE00>>16` instructions. Need to identify which one runs on the pre-fight profile screen.

4. **Find the load site for the small portrait palette.** Unknown until question 1 is answered.

5. **Determine SELECT bit position in input read.** The alt-glove input decoder reads `$7E:0090/0091` (held buttons). SELECT is bit 5 of `$7E:0091`. Need to verify this is read at the right moment (the alt-glove input decode runs at fight-init; SELECT-held override needs to be in the same window).

6. **Confirm alt-glove L+SELECT compatibility.** The current alt-glove decoder masks `$7E:0091 & $70` to detect L/R/X. SELECT (bit 5 = `$20`) is currently unused there — adding our SELECT-detect should not collide, but verify in the assembly.

---

## Free-space budget

Per the post-stack audit (SE 1.7 + alt_glove_colors v1 stacked):

| Zone                          | Total   | Used     | Free    | Max contiguous |
|-------------------------------|---------|----------|---------|----------------|
| Bank `$00` `UNK_00F5D0`       | 2,608 B | 899 B    | 1,709 B | ~352 B         |
| Bank `$0D` `UNK_0DFA69`       | 1,431 B | 472 B    | 959 B   | 113 B          |
| Bank `$01` `UNK_01F784`       | 2,172 B | 185 B    | 1,987 B | 2,210 B (best) |
| **TOTAL**                     | **6,211 B** | **1,556 B** | **4,655 B** |             |

### Allocation strategy

- **Alt palette tables (1,344 B = 14 × 3 × 32 B):** bank `$01` `UNK_01F784` starting at `$01:F812`. Single contiguous block, no fragmentation pain.
- **Sibling hook trampoline (~200-400 B estimated, exact size pending load-site discovery):** bank `$0D` `UNK_0DFA69`'s 113 B contiguous gap, with spillover into bank `$00` `UNK_00F5D0` if needed via JSL.
- **Headroom after this patch:** ~3,000 B remaining across all three zones.

### Constraints

- **No byte changes outside this patch's claimed regions.** Build verification step: build SE 1.7 alone, build SE 1.7 + alt-colors, diff. Every byte that differs must be inside an alt-colors-claimed range. Anything else = collision = abort.
- **No mutation of alt-glove v1 bytes.** Sibling hook only.
- **No mutation of spo_iron_circuit.ips bytes.** iron is stable; alt-colors works around it, never through it.

---

## Patches built so far (proofs of concept, ungated, throwaway)

| Build script                                    | IPS                                              | What it does |
|-------------------------------------------------|--------------------------------------------------|---|
| `build_alt_opp_palette_gabby_v1.py`             | `output/spo_alt_opp_palette_gabby_v1.ips`        | Gabby in-fight body Pal 2: 9 user-specified alt colors |
| `build_alt_opp_portrait_gabby_v1.py`            | `output/spo_alt_opp_portrait_gabby_v1.ips`       | Gabby large portrait: all 15 non-backdrop slots → luminance-preserving grayscale |
| `build_test_gabby_palette_locator.py`           | `output/spo_test_gabby_palette_locator.ips`      | Diagnostic: paints `Sprite_GabbyJay.bin` Pal 0 cyan, Pal 1 yellow, Large Portrait Gabby slot magenta. Used to identify load destinations. |

These are all ROM-byte edits, not runtime hooks. They prove the *source bytes* are correct but tell us nothing about the load path or how to gate behavior. They will be retired once the gated implementation lands.

---

## Roadmap

### Phase 1: gating prototype (Gabby Jay only)
1. Run `spo_test_gabby_palette_locator.sfc`, observe each game screen, record which on-screen objects use which source palette and which CGRAM destination.
2. Identify the three load sites (body, small portrait, large portrait).
3. Implement the sibling hook with full gating: iron auto-trigger (default-on, all 3 contexts) + SELECT-held override (in-fight body only) + alt-glove compatibility (L+SELECT chord works for both).
4. Live-test in Mesen: vanilla play unchanged, iron-circuit shows alt palettes automatically, SELECT-held in regular fights shows alt body palette only, L+SELECT shows alt body + L-mode gloves.
5. Verify byte-diff against the SE 1.7 baseline: zero collateral changes.

### Phase 2: full per-fighter authoring
- Author 13 more fighters' palettes (3 per fighter = 39 alt palettes).
- Pure mechanical extension once Phase 1 architecture proves out.

### Phase 3: bundle into a future SE release
- Bundle this patch + all standalone patches into a single follow-up SE IPS.
- Update `README.md` with the new feature list.
- Tag release; future work limited to bugfixes unless a major version jump is warranted.

---

## File map

```
work/scripts/
  build_alt_opp_palette_gabby_v1.py        # body, ungated PoC
  build_alt_opp_portrait_gabby_v1.py       # large portrait, ungated PoC (B&W)
  build_test_gabby_palette_locator.py      # diagnostic source-bomb

output/
  spo_alt_opp_palette_gabby_v1.{ips,sfc}
  spo_alt_opp_portrait_gabby_v1.{ips,sfc}
  spo_test_gabby_palette_locator.{ips,sfc}

doc/standalone/
  ALT_COLORS.md                            # this file
  ALT_GLOVE_COLORS.md                      # alt-glove v1 (shipped in SE 1.7)
  IRON_CIRCUIT.md                          # iron — owner of the $7E:1FE1 contract
  SUPER_MACHO_MAN_FIX.md                   # references the same large-portrait asset class
```

---

## See also

- `doc/standalone/IRON_CIRCUIT.md` — defines the `$7E:1FE1` iron-active flag this patch reads.
- `doc/standalone/ALT_GLOVE_COLORS.md` — alt-glove v1; uses the same fight-init hook neighborhood and the same `$7E:0090/0091` input read. Reference for the existing input-decode pattern.
- `doc/standalone/SUPER_MACHO_MAN_FIX.md` — also touches the pre-fight profile screen (shifts Super Macho Man's large portrait +4 px right). Same screen this patch's large-portrait palettes apply to.
