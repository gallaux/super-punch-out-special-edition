# Super Punch-Out!! Special Edition (Hack)

**Repository:** [github.com/gallaux/super-punch-out-special-edition](https://github.com/gallaux/super-punch-out-special-edition)

**Base ROM:** Super Punch-Out!! (USA, SNES) — MD5 `97fe7d7d2a1017f8480e60a365a373f0`
**Apply with:** Lunar IPS, flips, or any standard IPS patcher. Always patch the **original, unmodified ROM**.

<img width="256" height="224" alt="Super Punch-Out!! Special Edition (USA)_000" src="https://github.com/user-attachments/assets/dcd44da3-e7a0-441d-967d-4e4ea2cb5210" />
<img width="256" height="224" alt="Super Punch-Out!! Special Edition (USA)_001" src="https://github.com/user-attachments/assets/20bb0c6e-2d73-4ca2-ba18-6bb9584b7f50" />
<img width="256" height="224" alt="Super Punch-Out!! Special Edition (USA)_002" src="https://github.com/user-attachments/assets/69c1a244-62a4-4a1e-bbcf-620b26aabe52" />
<img width="256" height="224" alt="Super Punch-Out!! Special Edition (USA)_003" src="https://github.com/user-attachments/assets/182e23f3-831f-46ce-90be-f352ccc8ce56" />
<img width="256" height="224" alt="Super Punch-Out!! Special Edition (USA)_004" src="https://github.com/user-attachments/assets/c2388040-58ed-4175-8ff9-864da235d250" />
<img width="256" height="224" alt="Super Punch-Out!! Special Edition (USA)_005" src="https://github.com/user-attachments/assets/01d79c97-2847-481e-bd9b-c9563f87dd55" />
<img width="256" height="224" alt="Super Punch-Out!! Special Edition (USA)_006" src="https://github.com/user-attachments/assets/018cabe6-a0c2-4446-978e-6d45ed95af7b" />
<img width="256" height="224" alt="Super Punch-Out!! Special Edition (USA)_007" src="https://github.com/user-attachments/assets/7df047bd-4246-4d27-b960-3dd721e94d85" />
<img width="256" height="224" alt="Super Punch-Out!! Special Edition (USA)_008" src="https://github.com/user-attachments/assets/25c4f5ad-a370-4fcb-8f48-7fdcc08e0615" />
<img width="256" height="224" alt="Super Punch-Out!! Special Edition (USA)_009" src="https://github.com/user-attachments/assets/091938c6-af05-47b2-9419-089a988e32f8" />
<img width="256" height="224" alt="Super Punch-Out!! Special Edition (USA)_010" src="https://github.com/user-attachments/assets/e15bbe5a-7198-4bd3-b4d2-f7daabd249a1" />
<img width="256" height="224" alt="Super Punch-Out!! Special Edition (USA)_011" src="https://github.com/user-attachments/assets/f183750a-60e4-46ab-9976-93d8e9635875" />

---

## Background

Super Punch-Out!! (1994, SNES) is one of the best arcade-style boxing games ever made. You play as an up-and-coming boxer fighting your way through four circuits of increasingly absurd opponents, each with their own patterns, tells, and personality. The gameplay is fast, stylish, and deeply satisfying to master — a perfect arcade-like experience that holds up completely.

Super Punch-Out!! contains a hidden 2-player mode that was never exposed through the normal menu. In this mode, holding a specific button combination causes Controller 2 to control the opponent — effectively turning a single-player game into a local versus match. This secret was first publicly discovered in 2022.

This hack makes that experience a first-class feature. Instead of having to input secret button combinations, you select **VERSUS MODE** from the game's Mode Select menu just like any other mode. No combination, no guesswork.

As it stands, this is the definitive version of the game. The Versus mode is inherently unbalanced — but it's the version I wish Nintendo had shipped back in the '90s. They eventually added a proper VS mode in the Wii sequel many years later. The "Special Edition" name is a nod to the game's ultimate hidden circuit!

---

## What's in the Special Edition

- **VERSUS MODE** added to the Mode Select menu — 2-player matches launch directly, no secret button combo required
  - **Either controller can pick on the opponent-select screen** — P2 can choose their own fighter without handing the controller back
- **Alternate glove colors**
  — each circuit has its own default glove color (mirroring Punch-Out!! Wii's per-opponent glove tones)
  - per-match override: hold **L** (blue), **R** (red), **X** (yellow), or **Y** (green) before the fight starts — on the opponent-select screen in Time Attack / Versus, or on the pre-fight opponent screen in Championship — and confirm with A or Start to enter the fight in the chosen color. The override applies only to that match; the next fight reseeds from the opponent default.
- **Typo fixes:**
  - Mr. Sandman's profile stats (originally a verbatim copy of Super Macho Man's; restored to age 30, weight 270 lbs, record 28-4 — the correct values from the manual and Japanese version)
  - Mad Clown's weight (390 lbs → 370 lbs, matching the manual)
  - "SUPER MACHOMAN" → "SUPER MACHO MAN" across every screen in the game
  - "devestating" → "devastating" in the in-game tutorial demo
- **Cosmetic title-screen tweaks** — Special Circuit ring artwork and palette, plus a "SPECIAL EDITION" sub-title
- **Japanese name-entry character set** always available via L/R cycling (no hidden button combo needed)
- **CREDITS entry** added to the Records View screen — replays the ending cutscene's credits roll
- **Special Circuit always unlocks correctly** — the original ROM's security checksum that locks Special on save states / emulators / patched ROMs is bypassed

---

## Patch files

### [`spo_special_edition_v1.6.ips`](patches/spo_special_edition_v1.6.ips) — Special Edition (recommended)

The full experience: all the standalone patches below bundled into a single IPS. Apply directly to the original ROM.

### Standalone patches

These are independent fixes that can be applied alone or mixed and matched, on top of the original ROM. **All standalone patches in this repo are byte-level compatible** — they can be applied in any order on top of a fresh `spo.sfc` and the result is identical to the bundled `spo_special_edition_v1.6.ips`.

| File | What it does | Details |
|---|---|---|
| [`spo_versus_hack.ips`](patches/standalone/spo_versus_hack.ips) **(core hack)** | The core hack of this repo. Adds Versus Mode to the menu, lets either controller pick on the opponent-select screen, and fixes the Special Circuit security checksum lock. | [doc](doc/standalone/VERSUS_HACK.md) |
| [`spo_alt_glove_colors.ips`](patches/standalone/spo_alt_glove_colors.ips) | Adds a per-circuit glove-color selector. Each circuit gets a different default color (mirroring how Punch-Out!! Wii dressed Little Mac across opponents), and the player can override per-match by holding L/R/X/Y at fight start. | [doc](doc/standalone/ALT_GLOVE_COLORS.md) |
| [`spo_profile_stats_fix.ips`](patches/standalone/spo_profile_stats_fix.ips) | Fixes two profile-screen stat errors present in the US/EUR ROM: Mr. Sandman's stats are a verbatim copy of Super Macho Man's (restored to age 30, weight 270 lbs, record 28-4); Mad Clown's weight reads 390 lbs instead of 370 lbs. Both correct values match the US manual and the later-released Japanese version of the game. | [doc](doc/standalone/PROFILE_STATS_FIX.md) |
| [`spo_super_macho_man_fix.ips`](patches/standalone/spo_super_macho_man_fix.ips) | Fixes the "SUPER MACHOMAN" → "SUPER MACHO MAN" typo across all screens in the game. | [doc](doc/standalone/SUPER_MACHO_MAN_FIX.md) |
| [`spo_how_to_typo_fix.ips`](patches/standalone/spo_how_to_typo_fix.ips) | Fixes a single-letter typo in the in-game tutorial demo: "devestating" → "devastating". | [doc](doc/standalone/HOW_TO_TYPO_FIX.md) |
| [`spo_title_screen_special_ring.ips`](patches/standalone/spo_title_screen_special_ring.ips) | Cosmetic: replaces the title-screen ring logo and color palette with the Special Circuit variants for some extra flair. | [doc](doc/standalone/TITLE_SCREEN_SPECIAL_RING.md) |
| [`spo_title_screen_special_logo.ips`](patches/standalone/spo_title_screen_special_logo.ips) | Cosmetic: adds a SPECIAL EDITION text line to the title screen, below the main game logo. | [doc](doc/standalone/TITLE_SCREEN_SPECIAL_LOGO.md) |
| [`spo_jp_charset_enabled.ips`](patches/standalone/spo_jp_charset_enabled.ips) | Makes the Japanese character set L/R-cycling always active in name entry, starting on the Western set. The hidden button combo is no longer needed. | [doc](doc/standalone/JP_CHARSET_ENABLED.md) |
| [`spo_end_credits.ips`](patches/standalone/spo_end_credits.ips) | Adds a **CREDITS** entry to the Records View select screen. Selecting CREDITS launches the game's ending-cutscene credits roll. After the credits finish the game stays on the final screen and requires reset — that matches the original cutscene's behavior, not specific to this patch. | [doc](doc/standalone/END_CREDITS.md) |
| [`spo_disable_security_checksum.ips`](patches/standalone/spo_disable_security_checksum.ips) | The World Circuit completion checksum prevents the Special Circuit from unlocking when using save states, emulators (SNES Classic, Switch NSO), or patched ROMs. This patch disables that checksum check. | [doc](doc/standalone/DISABLE_SECURITY_CHECKSUM.md) |

### Incomplete / experimental patches

These are proof-of-concept patches with known bugs. Not included in the **Special Edition** and not recommended for general use:

| File | What it does |
|---|---|
| [`spo_sound_mode_incomplete.ips`](patches/incomplete/spo_sound_mode_incomplete.ips) | Proof-of-concept patch that adds a SOUND MODE entry to the title-screen menu. **Known bugs: pressing A on SOUND MODE triggers DATA CLEAR instead; name-entry screen behaves incorrectly after selecting NEW GAME or CONTINUE.** Wiring the actual Sound Library launch requires further investigation. |

---

## What the Versus Hack adds

### 1. VERSUS MODE on the Mode Select screen

The Mode Select screen gains a new **VERSUS MODE** entry between Championship and Time Attack. Selecting it launches a 2-player match directly, without needing any specific input combination.

- Unlock logic is preserved — Time Attack and Records View require Championship progress; Versus is always available.
- Backing out of opponent select returns to Mode Select cleanly.

### 2. Player 2 controls the opponent

In Versus matches, **Controller 2 controls the opponent**. This is the same mechanic as the 2022-discovered secret mode, now wired in permanently whenever you enter from Versus.

### 3. Correct in-game screen labels

The opponent-select screen shows **"VERSUS MODE"** as its header and **"< PLAYER 2 SELECT >"** as the prompt when entering from Versus. Time Attack is completely unaffected — it still shows "TIME ATTACK MODE" and "< OPPONENT SELECT >".

### 4. Player 2 picks their own fighter

On the VERSUS opponent-select screen, **Controller 2's D-pad, A, and Start mirror Controller 1's** — so Player 2 can pick their own character without handing the controller back.

### 5. Special Circuit security checksum disabled

The World Circuit completion checksum prevents the Special Circuit from unlocking when using save states, emulators, or patched ROMs. This patch disables that checksum check so the Special Circuit is always accessible.

---

## How to play Versus mode

1. From the title screen, choose **NEW GAME** or **CONTINUE**.
2. At the **Mode Select** screen, choose **VERSUS MODE**.
3. Select your circuit and opponent. Either controller can drive the opponent-select screen — Player 2 can navigate and confirm their own character.
4. Once the match starts, **Controller 1 controls the left boxer** and **Controller 2 controls the right boxer**.

---

## Incomplete implementation

- **"SOUND MODE" menu implementation**: a proof-of-concept UI patch exists (`patches/incomplete/spo_sound_mode_incomplete.ips`) but has three unresolved issues: (1) the cursor snap system uses tilemap-derived position values — the correct cursor value for the SOUND MODE row has not been empirically determined; (2) the `$14` item-count variable bleeds into name-entry and downstream screens; (3) the Sound Library (`CODE_00913B`) lives outside the normal state machine and can only be launched safely from a cold boot state. Full investigation documented in [doc/incomplete/AUDIO_MODE_INCOMPLETE.md](doc/incomplete/AUDIO_MODE_INCOMPLETE.md).

---

## Technical reference

For the full reverse-engineering notes — every patch address, the SNES memory map, the Mode Select state machine architecture, and the free-space inventory — see [doc/TECHNICAL.md](doc/TECHNICAL.md).
Key facts for anyone continuing this work:

- **ROM mapping:** LoROM. File offset = bank × 0x8000 + (SNES_addr − 0x8000).
- **Free space remaining:** 170 bytes free in bank `$0D` (within the `$0DFA69–$0DFFE3` `%InsertGarbageData` zone), 4 bytes at `$01:FFE0–$FFE3`, and **2,422 bytes free in bank `$00`** (within the newly-mapped `UNK_00F5D0` zone, mostly untouched after applying the complete Special Edition patch — **2,596 bytes** total).

---

## Cheat code compatibility

All known Action Replay / cheat codes for the original ROM remain fully compatible with every patch in this repository. The libretro cheat database for this title ([Super Punch-Out!! (World) (Action Replay).cht](https://github.com/libretro/libretro-database/blob/master/cht/Nintendo%20-%20Super%20Nintendo%20Entertainment%20System/Super%20Punch-Out!!%20(World)%20(Action%20Replay).cht)) contains two categories:

- **ROM-intercepting cheats** (PRO ACTION REPLAY codes that patch ROM reads) target addresses that none of our patches modify, so they apply cleanly on top of any combination of patches.
- **RAM cheats** (codes that write directly to WRAM at runtime) operate entirely independently of ROM content and are unaffected by any ROM patch.

No conflicts exist or are expected.

---

## Links and resources

- [**GitHub Repository**](https://github.com/gallaux/super-punch-out-special-edition) — Source, patch files, and documentation
- [**ROMhacking.net**](https://www.romhacking.net/hacks/9705/) — This rom hack's entry
- [**Super Punch-Out!! Disassembly**](https://github.com/Yoshifanatic1/Super-Punch-Out-Disassembly) — Full disassembly of the game; invaluable reference for this hack
- [**2-Player Mode Discovery**](https://x.com/new_cheats_news/status/1556727895778856960) — The tweet that revealed the hidden VS mode (Aug 8, 2022)
- [**The Cutting Room Floor**](https://tcrf.net/Super_Punch-Out!!_(SNES)) — Unused content and hidden features documentation
- [**GameHacking.org**](https://gamehacking.org/game/44921) — Game Genie / Pro Action Replay codes
- [**Punch-Out!! Wiki**](https://punchout.fandom.com/wiki/Super_Punch-Out!!_(SNES)) — Fan wiki with character, gameplay, and misc info
- [**SPU Sound Library**](https://github.com/RealVMMR/SPU-Sound_Library) — Lua script for manipulating SPC sound effects and music. Not related to this hack, but a cool project
