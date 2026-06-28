# Opponent Palettes

Complete reference for the sprite palette data of all 16 opponent slots in Super Punch-Out!! — what's in ROM, where it lives, how it maps to on-screen sprites, and what's known about each sub-palette's role.

This is a **reference document**. If you're writing a patch that recolors opponents (full swap, per-circuit reskin, themed overlay, etc.), start here.

---

## TL;DR

- Each fighter has a small palette block at a fixed SNES bus address in bank `$0D`, containing **3, 4, or 6 sub-palettes** of 16 colors each (32 bytes per sub-palette).
- All 16 fighter slots additionally have a 16-color palette inside `Sprite_LargePortraits.bin` (file `0x087E00`), indexed by fighter ID.
- For each fighter, three sub-palettes are confirmed to drive on-screen sprites:
  - **`Sprite_<Fighter>.bin` Pal 0** → small portrait (in-match HUD)
  - **`Sprite_<Fighter>.bin` Pal 2** → in-fight body sprite (the opponent you punch)
  - **`Sprite_LargePortraits.bin` slot N** → large pre-fight portrait
- Sub-palettes 1, 3, 4, 5 (when they exist) have **unknown roles**. Pal 1 has been empirically confirmed to not appear during normal play for Gabby Jay. The other extras are uninvestigated.

---

## ROM mapping

SPO is **LoROM**. SNES bus address → ROM file offset:
```
file = ((bank & 0x7F) << 15) | (offset & 0x7FFF)
```

Worked examples:
- `$0DB9DA` → file `0x06B9DA`
- `$10FE00` → file `0x087E00`

All addresses in this doc give both representations when relevant.

---

## Color format: BGR555

Each color is **2 bytes, little-endian**, encoded as **BGR555**: 5 bits red, 5 bits green, 5 bits blue, top bit unused. This is the SNES's native CGRAM format.

### Bit layout (16-bit word)

```
bit 15  14 13 12 11 10   9  8  7  6  5    4  3  2  1  0
  [-]   [ B (5 bits) ]   [ G (5 bits) ]   [ R (5 bits) ]
```

The top bit is unused (always 0 in palette data). When stored in ROM the word is little-endian — the **low byte is `gggRRRRR`**, the **high byte is `0bbbbbgg`**.

### Worked example

Take BGR555 word `$5B3F` (file bytes `3F 5B`):
```
word        = 0x5B3F                = 0101 1011 0011 1111
              [-] BBBBB GGGGG RRRRR =    0 10110 11001 11111
R = word        & 0x1F = 0x1F = 31
G = (word >> 5) & 0x1F = 0x19 = 25
B = (word >> 10) & 0x1F = 0x16 = 22
```

To convert each 5-bit channel to an 8-bit (`#RRGGBB`) value, use the **bit-replicating expansion** `(n << 3) | (n >> 2)` so the maximum 5-bit value `0x1F` maps to `0xFF` (and not `0xF8`):

```
R8 = (31 << 3) | (31 >> 2) = 0xF8 | 0x07 = 0xFF
G8 = (25 << 3) | (25 >> 2) = 0xC8 | 0x06 = 0xCE
B8 = (22 << 3) | (22 >> 2) = 0xB0 | 0x05 = 0xB5
```

→ Display color **`#FFCEB5`** (a light skin tone — this is Gabby Jay's Pal 0 slot C6).

### Round-trip caveat (5-bit truncation)

Going from `#RRGGBB` → BGR555 truncates each channel from 8 bits to 5, losing the low 3 bits. The reverse conversion expands back via bit-replication. The two operations are **not exact inverses**:

```
#FFCEB5  →  truncate (>> 3)  →  R=31 G=25 B=22  →  BGR555 0x5B3F
0x5B3F   →  expand   (<<3|>>2) →  R=0xFF G=0xCE B=0xB5  →  #FFCEB5  (round-trip OK in this case)

#743D31  →  truncate (>> 3)  →  R=14 G= 7 B= 6  →  BGR555 0x18EE
0x18EE   →  expand   (<<3|>>2) →  R=0x73 G=0x38 B=0x31 →  #733831  (off by 1-2 bits per channel)
```

The "off by 1-2 bits per channel" outcome is normal and visually indistinguishable. When authoring alt palettes, pick the BGR555 result you want and **don't expect bit-exact `#RRGGBB` round-trips**.

### Reference conversions (Python)

```python
def hex_to_bgr555(hex_str):
    """ '#RRGGBB' -> BGR555 word (int, 0x0000-0x7FFF) """
    h = hex_str.lstrip('#')
    r = int(h[0:2], 16) >> 3
    g = int(h[2:4], 16) >> 3
    b = int(h[4:6], 16) >> 3
    return (b << 10) | (g << 5) | r

def bgr555_to_hex(word):
    """ BGR555 word -> '#RRGGBB' (8-bit per channel) """
    r5 =  word        & 0x1F
    g5 = (word >> 5)  & 0x1F
    b5 = (word >> 10) & 0x1F
    r = (r5 << 3) | (r5 >> 2)
    g = (g5 << 3) | (g5 >> 2)
    b = (b5 << 3) | (b5 >> 2)
    return f"#{r:02X}{g:02X}{b:02X}"

def bgr555_to_bytes(word):
    """ BGR555 word -> 2 bytes (little-endian, as stored in ROM) """
    return bytes([word & 0xFF, (word >> 8) & 0xFF])

def bytes_to_bgr555(lo, hi):
    """ 2 little-endian ROM bytes -> BGR555 word """
    return lo | (hi << 8)
```

### Special-case slot: C0 (SNES backdrop)

CGRAM slot 0 of every palette is the **SNES backdrop color** (used wherever a sprite has a "transparent" pixel). All sprite palettes in SPO use `$3800` at C0 (= R=0, G=0, B=14 → #000073, a dark blue). When authoring alt palettes, **leave C0 untouched** unless you specifically want to change the backdrop everywhere.

### Variants and naming

- The SNES community sometimes calls this format **"15-bit color"** or **"SNES BGR"**. They all refer to the same encoding.
- Watch out for **5-bit BGR with the top bit interpreted as a priority/transparency flag** in some documentation — that's not true for palette data; only OBJ tile data uses the top bits for those purposes.
- **Not to be confused with RGB555** (red-green-blue ordering), used by some other systems. SNES uses BGR ordering.

---

## Per-fighter palette source addresses

Fighter index order matches `DATA_108000` (the large-portrait GFX pointer table) for fighters 0-13. Rick & Nick Bruiser are fighters 14 & 15 (twin fight, treated as a single match in vanilla but with separate palette blocks).

| Idx | Fighter           | `Sprite_<F>.bin` SNES  | File range          | Size    | Sub-pals | Large portrait SNES | Large portrait file |
|-----|-------------------|------------------------|---------------------|---------|----------|---------------------|---------------------|
|  0  | Gabby Jay         | `$0DB9DA-$0DBA39`      | `0x06B9DA-0x06BA39` | 0x60    | 3        | `$10FE00-$10FE1F`   | `0x087E00-0x087E1F` |
|  1  | Bear Hugger       | `$0DBC3C-$0DBC9B`      | `0x06BC3C-0x06BC9B` | 0x60    | 3        | `$10FE20-$10FE3F`   | `0x087E20-0x087E3F` |
|  2  | Piston Hurricane  | `$0DBE9E-$0DBEFD`      | `0x06BE9E-0x06BEFD` | 0x60    | 3        | `$10FE40-$10FE5F`   | `0x087E40-0x087E5F` |
|  3  | Bald Bull         | `$0DC100-$0DC17F`      | `0x06C100-0x06C17F` | **0x80**| **4**    | `$10FE60-$10FE7F`   | `0x087E60-0x087E7F` |
|  4  | Bob Charlie       | `$0DC382-$0DC3E1`      | `0x06C382-0x06C3E1` | 0x60    | 3        | `$10FE80-$10FE9F`   | `0x087E80-0x087E9F` |
|  5  | Dragon Chan       | `$0DC5E4-$0DC643`      | `0x06C5E4-0x06C643` | 0x60    | 3        | `$10FEA0-$10FEBF`   | `0x087EA0-0x087EBF` |
|  6  | Masked Muscle     | `$0DC846-$0DC8C5`      | `0x06C846-0x06C8C5` | **0x80**| **4**    | `$10FEC0-$10FEDF`   | `0x087EC0-0x087EDF` |
|  7  | Mr Sandman        | `$0DCAC8-$0DCB27`      | `0x06CAC8-0x06CB27` | 0x60    | 3        | `$10FEE0-$10FEFF`   | `0x087EE0-0x087EFF` |
|  8  | Aran Ryan         | `$0DCD2A-$0DCD89`      | `0x06CD2A-0x06CD89` | 0x60    | 3        | `$10FF00-$10FF1F`   | `0x087F00-0x087F1F` |
|  9  | Heike Kagero      | `$0DCF8C-$0DCFEB`      | `0x06CF8C-0x06CFEB` | 0x60    | 3        | `$10FF20-$10FF3F`   | `0x087F20-0x087F3F` |
| 10  | Mad Clown         | `$0DD1EE-$0DD26D`      | `0x06D1EE-0x06D26D` | **0x80**| **4**    | `$10FF40-$10FF5F`   | `0x087F40-0x087F5F` |
| 11  | Super Macho Man   | `$0DD470-$0DD4CF`      | `0x06D470-0x06D4CF` | 0x60    | 3        | `$10FF60-$10FF7F`   | `0x087F60-0x087F7F` |
| 12  | Narcis Prince     | `$0DD6D2-$0DD731`      | `0x06D6D2-0x06D731` | 0x60    | 3        | `$10FF80-$10FF9F`   | `0x087F80-0x087F9F` |
| 13  | Hoy Quarlow       | `$0DD934-$0DD993`      | `0x06D934-0x06D993` | 0x60    | 3        | `$10FFA0-$10FFBF`   | `0x087FA0-0x087FBF` |
| 14  | Rick Bruiser      | `$0DDB96-$0DDC55`      | `0x06DB96-0x06DC55` | **0xC0**| **6**    | `$10FFC0-$10FFDF`   | `0x087FC0-0x087FDF` |
| 15  | Nick Bruiser      | `$0DDE58-$0DDF17`      | `0x06DE58-0x06DF17` | **0xC0**| **6**    | `$10FFE0-$10FFFF`   | `0x087FE0-0x087FFF` |

Source of truth: the SPO disassembly's `AssetPointersAndFiles.asm` lines 536-551 (per-fighter blocks) and 556 (LargePortraits block).

---

## Sub-palette role assignment

### For 3-sub-palette fighters (12 of 16)

| Sub-pal | Role                                       | Verified for                        |
|---------|--------------------------------------------|-------------------------------------|
| Pal 0   | **Small portrait** (in-match HUD)          | Gabby Jay (via locator test ROM)    |
| Pal 1   | Unknown — does not appear during normal play | Gabby Jay (not seen on any screen) |
| Pal 2   | **In-fight body sprite** (the opponent you punch) | Gabby Jay (via PoC palette swap)   |

Status for the other 11 three-sub-palette fighters: **untested but strongly assumed to follow the same Pal 0 / Pal 1 / Pal 2 mapping** — the asset table layout is uniform and the engine presumably uses one load path. Run a per-fighter locator test (see "Verification methodology" below) to confirm before relying on this for production patches.

### For 4-sub-palette fighters (Bald Bull, Masked Muscle, Mad Clown)

**Role of the extra sub-palette is unknown.** Hypotheses worth testing:
- Knockdown / hurt-state recolor (these three are mid-game heavy hitters with distinct knockdown animations)
- Stage-specific variant (Bald Bull and Mad Clown have memorable special moves with palette flashes)
- Powered-up / rage state
- Unused / leftover from development

Sub-pals 0 and 2 are **probably still small-portrait and body**, but this is unverified. Sub-pals 1 and 3 are open questions.

### For 6-sub-palette fighters (Rick Bruiser, Nick Bruiser)

The Bruisers fight back-to-back as a "twin" match. The 6 sub-palettes likely encode both twin states (alive/defeated, or before/after Bald Bull's intro, or similar). **Entirely unverified.**

### `Sprite_LargePortraits.bin`

| Property         | Value                                        |
|------------------|----------------------------------------------|
| File offset      | `0x087E00`                                   |
| SNES bus address | `$10FE00`                                    |
| Total size       | `0x200` bytes (512 B)                        |
| Slot count       | 16 (one per fighter, fighter-index ordered)  |
| Slot size        | `0x20` bytes (16 colors × 2 bytes BGR555)    |

Gabby Jay = slot 0 (verified). Other fighter slots follow `DATA_108000` ordering (see source-address table above).

---

## Verification methodology

How sub-palette roles are confirmed:

1. **Source-bomb diagnostic ROM.** Paint each candidate sub-palette in `Sprite_<Fighter>.bin` and the corresponding `Sprite_LargePortraits.bin` slot with a distinct, high-contrast color (cyan / yellow / magenta). Apply to a original `Super Punch-Out!! (USA).sfc` ROM, fight the fighter, observe which on-screen objects change to which color across all screens (circuit-select, opponent-select, pre-fight profile, fight-start, mid-fight, knockdown, victory).

2. **Cross-reference with CGRAM viewer.** Mesen's CGRAM viewer shows which sprite-palette slot holds the marker color, confirming the *destination* palette (e.g., sprite-palette 4 = in-fight body).

Example: for Gabby Jay (idx 0), a diagnostic test ROM was built that paints:

- Gabby's Pal 0 → **cyan** (#00FFFF)
- Gabby's Pal 1 → **yellow** (#FFFF00)
- LargePortraits slot 0 → **magenta** (#FF00FF)

Observed in-game (this is the basis for the role assignments above):

| Screen                       | Cyan (Pal 0)             | Yellow (Pal 1)  | Magenta (LP slot 0)      |
|------------------------------|--------------------------|-----------------|--------------------------|
| Pre-fight profile screen     | —                        | —               | Large portrait sprite    |
| In-fight HUD                 | Small portrait sprite    | —               | —                        |
| Mid-fight body sprite        | — (still vanilla; Pal 2 holds the body) | — | —              |
| All other screens            | —                        | —               | —                        |

Pal 1 didn't appear on any screen — it's either unused or only loaded in a context not encountered during a normal playthrough.

The in-fight body palette (Pal 2) is confirmed independently by `output/spo_alt_opp_palette_gabby_v1.sfc` (PoC that modifies 9 of 16 colors in Pal 2 and observes them on Gabby's body in-fight).

---

## Gabby Jay palette decode

Gabby Jay's palettes as they ship in vanilla SPO (BGR555 hex + #RRGGBB approximate display):

### Pal 0 — Small portrait
File `0x06B9DA-0x06B9F9` · SNES `$0DB9DA-$0DB9F9`

| Slot | BGR555    | #RRGGBB     |  | Slot | BGR555    | #RRGGBB     |
|------|-----------|-------------|--|------|-----------|-------------|
| C0   | `$3800`   | `#000073`   |  | C8   | `$4189`   | `#4A6384`   |
| C1   | `$4210`   | `#848484`   |  | C9   | `$5E70`   | `#849CBD`   |
| C2   | `$6318`   | `#C6C6C6`   |  | CA   | `$7F99`   | `#CEE7FF`   |
| C3   | `$0CAE`   | `#732918`   |  | CB   | `$0013`   | `#9C0000`   |
| C4   | `$29B8`   | `#C66B52`   |  | CC   | `$001B`   | `#DE0000`   |
| C5   | `$427B`   | `#DE9C84`   |  | CD   | `$011F`   | `#FF4200`   |
| C6   | `$5B3F`   | `#FFCEB5`   |  | CE   | `$01DF`   | `#FF7300`   |
| C7   | `$24C6`   | `#31314A`   |  | CF   | `$7FFF`   | `#FFFFFF`   |

Color reading: C0 = SNES backdrop / transparent. C3-C6 = skin ramp (dark to light). C7-CA = blue suit / shadow. CB-CE = red glove ramp. CF = white highlight.

### Pal 1 — Unknown role (not observed on screen)
File `0x06B9FA-0x06BA19` · SNES `$0DB9FA-$0DBA19`

| Slot | BGR555    | #RRGGBB     |  | Slot | BGR555    | #RRGGBB     |
|------|-----------|-------------|--|------|-----------|-------------|
| C0   | `$3800`   | `#000073`   |  | C8   | `$12B8`   | `#C6AD21`   |
| C1   | `$0046`   | `#311000`   |  | C9   | `$16F9`   | `#CEBD29`   |
| C2   | `$012F`   | `#7B4A00`   |  | CA   | `$1B3B`   | `#DECE31`   |
| C3   | `$0570`   | `#845A08`   |  | CB   | `$1B7C`   | `#E7D731`   |
| C4   | `$05B2`   | `#946B08`   |  | CC   | `$00E2`   | `#103810`   |
| C5   | `$09F3`   | `#9C7B10`   |  | CD   | `$01C2`   | `#107308`   |
| C6   | `$0E35`   | `#AD8C18`   |  | CE   | `$1287`   | `#39A521`   |
| C7   | `$1276`   | `#B59C21`   |  | CF   | `$338F`   | `#7BE763`   |

Looks like a yellow/gold ramp + green accents. Possibly a "powered-up flash" or "victory sparkle" variant — but unverified.

### Pal 2 — In-fight body sprite
File `0x06BA1A-0x06BA39` · SNES `$0DBA1A-$0DBA39`

| Slot | BGR555    | #RRGGBB     |  | Slot | BGR555    | #RRGGBB     |
|------|-----------|-------------|--|------|-----------|-------------|
| C0   | `$3800`   | `#000073`   |  | C8   | `$4189`   | `#4A6384`   |
| C1   | `$4210`   | `#848484`   |  | C9   | `$5E70`   | `#849CBD`   |
| C2   | `$6318`   | `#C6C6C6`   |  | CA   | `$7F99`   | `#CEE7FF`   |
| C3   | `$0CAE`   | `#732918`   |  | CB   | `$1D67`   | `#3963397B` |
| C4   | `$29B8`   | `#C66B52`   |  | CC   | `$3E6F`   | `#7B9C7B`   |
| C5   | `$427B`   | `#DE9C84`   |  | CD   | `$5F77`   | `#BDD7BD`   |
| C6   | `$5B3F`   | `#FFCEB5`   |  | CE   | `$001F`   | `#FF0000`   |
| C7   | `$24C6`   | `#31314A`   |  | CF   | `$7FFF`   | `#FFFFFF`   |

Note that **C0-CA are byte-identical to Pal 0** (same skin ramp, same blue suit / shadow). The difference is **CB-CD: green ring/background tones instead of red gloves**, and **CE: pure red** (likely the actual glove peak for the body sprite). This is why a portrait-style sub-palette and a body-style sub-palette can coexist on the same fighter — the body needs different secondary colors than the portrait.

### Large portrait — slot 0 of Sprite_LargePortraits.bin
File `0x087E00-0x087E1F` · SNES `$10FE00-$10FE1F`

| Slot | BGR555    | #RRGGBB     |  | Slot | BGR555    | #RRGGBB     |
|------|-----------|-------------|--|------|-----------|-------------|
| C0   | `$3800`   | `#000073`   |  | C8   | `$6F39`   | `#CECEDE`   |
| C1   | `$24A2`   | `#10294A`   |  | C9   | `$4E31`   | `#8C8C9C`   |
| C2   | `$4589`   | `#4A638C`   |  | CA   | `$398C`   | `#636373`   |
| C3   | `$664E`   | `#7394CE`   |  | CB   | `$20C6`   | `#313142`   |
| C4   | `$106D`   | `#6B1821`   |  | CC   | `$61E3`   | `#187BC6`   |
| C5   | `$39FA`   | `#D67B73`   |  | CD   | `$7AA9`   | `#4AADF7`   |
| C6   | `$4A9E`   | `#F7A594`   |  | CE   | `$2933`   | `#9C4A52`   |
| C7   | `$633F`   | `#FFCEC6`   |  | CF   | `$7FFF`   | `#FFFFFF`   |

A higher-color-count portrait than the small-portrait sprite, with skin tones (C4-C7), gloves (C4-C6 again, ramped), blue uniform / shadow (C1-C3, C8-CB), and bright blue headband accents (CC-CD). Used on the pre-fight profile screen ("CHAMP. GABBY JAY") and any other screen that displays the large portrait sprite.

---

## Other fighters (palettes not yet decoded)

The 15 other fighters' palettes can be decoded with the same approach: read the 32-byte per-sub-palette block at the SNES bus address from the per-fighter source-address table above, walk each 2-byte BGR555 word, and decode using the bit layout in the "Color format: BGR555" section.

Until per-fighter verification is done, **assume the role assignment for 3-sub-palette fighters generalizes from Gabby** (Pal 0 = small portrait, Pal 2 = body), and **don't assume anything about 4- and 6-sub-palette fighters' extras**.

---

## What's NOT documented here (open questions)

1. **Load sites in code.** This doc covers *where palettes live in ROM*, not *where the engine reads them*. For runtime patching, you also need to find the MVN/DMA/loop instructions in the engine that copy these bytes into CGRAM. Not investigated here.

2. **CGRAM destination addresses.** Only the in-fight body destination is confirmed (CGRAM sprite-palette 4 = byte addresses `$100..$11F`). Small portrait and large portrait destinations are unverified — the test ROM proved *which source feeds which on-screen object* but not *which CGRAM slot the engine writes to*.

3. **Role of extra sub-palettes** (Pal 1 universally; Pal 3 on 4-pal fighters; Pals 1, 3, 4, 5 on the Bruisers). All open. Pal 1 has been confirmed *not to appear on screen* during normal play for Gabby Jay — same likely holds for the other 11 three-sub-palette fighters but not verified.

4. **Whether the Bruisers' twin layout maps Pal 0-2 to one twin and Pal 3-5 to the other**, or some other arrangement.

5. **Layer1_Contender.bin.** Per the alt-glove docs, this is loaded at fight-init and contains player-glove colors. The role of any opponent-related bytes inside it (if any) is unclear and not covered by this doc.

---

## How this doc gets used

- **Authoring per-opponent alt palettes**: each fighter needs alt palettes hand-authored. The source addresses above tell you where to read vanilla for reference; the role table tells you which sub-palettes matter.
- **Full opponent reskin patches**: same data, different patch shape.
- **Identifying what an unknown sub-palette is for**: extend the locator test methodology to other fighters and contexts (e.g., test the Bruisers across both twin states, test Bald Bull during his charge animation).
- **Sanity-checking palette modifications**: the verified C0 = SNES backdrop pattern means slot 0 of any of these palettes should normally be `$3800` (= #000073). Other shared structural patterns (skin ramps at C3-C6, etc.) help spot accidental corruptions.

---

## See also

- [`doc/standalone/ALT_GLOVE_COLORS.md`](standalone/ALT_GLOVE_COLORS.md) — sibling system for *player* glove palettes
- [`doc/standalone/SUPER_MACHO_MAN_FIX.md`](standalone/SUPER_MACHO_MAN_FIX.md) — touches the pre-fight profile screen (same screen that displays large portraits)
- SPO disassembly's `AssetPointersAndFiles.asm` and `Palettes/` directory — source of truth for ROM addresses and the extracted per-fighter `.bin` palette files
