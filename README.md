# Super Punch-Out!! Special Edition (Hack)

**Repository:** [github.com/gallaux/super-punch-out-special-edition](https://github.com/gallaux/super-punch-out-special-edition)

**Base ROM:** Super Punch-Out!! (USA, SNES) — MD5 `97fe7d7d2a1017f8480e60a365a373f0`
**Apply with:** Lunar IPS, flips, or any standard IPS patcher. Always patch the **original, unmodified ROM**.

<img width="256" height="224" alt="Super Punch-Out!! Special Edition (USA)_000" src="https://github.com/user-attachments/assets/f671137e-def2-47c2-846f-20748be57fa1" />
<img width="256" height="224" alt="Super Punch-Out!! Special Edition (USA)_001" src="https://github.com/user-attachments/assets/20bb0c6e-2d73-4ca2-ba18-6bb9584b7f50" />
<img width="256" height="224" alt="Super Punch-Out!! Special Edition (USA)_002" src="https://github.com/user-attachments/assets/ccadbeb0-6891-46d3-9da6-fbd5541e1196" />
<img width="256" height="224" alt="Super Punch-Out!! Special Edition (USA)_003" src="https://github.com/user-attachments/assets/8bf624b7-9f18-40fa-9d36-f0ab3e8127a1" />
<img width="256" height="224" alt="Super Punch-Out!! Special Edition (USA)_004" src="https://github.com/user-attachments/assets/c2388040-58ed-4175-8ff9-864da235d250" />
<img width="256" height="224" alt="Super Punch-Out!! Special Edition (USA)_005" src="https://github.com/user-attachments/assets/f664f19a-4299-464d-9408-713bb404f71d" />
<img width="256" height="224" alt="Super Punch-Out!! Special Edition (USA)_006" src="https://github.com/user-attachments/assets/018cabe6-a0c2-4446-978e-6d45ed95af7b" />
<img width="256" height="224" alt="Super Punch-Out!! Special Edition (USA)_007" src="https://github.com/user-attachments/assets/7df047bd-4246-4d27-b960-3dd721e94d85" />
<img width="256" height="224" alt="Super Punch-Out!! Special Edition (USA)_008" src="https://github.com/user-attachments/assets/176f3744-5709-44dc-a071-429c87b88828" />

---

## Background

Super Punch-Out!! (1994, SNES) is one of the best arcade-style boxing games ever made. You play as an up-and-coming boxer fighting your way through four circuits of increasingly absurd opponents, each with their own patterns, tells, and personality. The gameplay is fast, stylish, and deeply satisfying to master — a perfect arcade-like experience that holds up completely.

Super Punch-Out!! contains a hidden 2-player mode that was never exposed through the normal menu. In this mode, holding a specific button combination causes Controller 2 to control the opponent — effectively turning a single-player game into a local versus match. This secret was first publicly discovered in 2022.

This hack makes that experience a first-class feature. Instead of having to input secret button combinations, you select **VERSUS MODE** from the game's Mode Select menu just like any other mode. No combination, no guesswork.

As it stands, this is the definitive version of the game. The Versus mode is inherently unbalanced — but it's the version I wish Nintendo had shipped back in the '90s. They eventually added a proper VS mode in the Wii sequel many years later. The "Special Edition" name is a nod to the game's ultimate hidden circuit!

---

## Patch files

### [`spo_versus_hack.ips`](patches/standalone/spo_versus_hack.ips) — Versus Hack (base)

The core hack. Adds Versus Mode to the menu and fixes Special Circuit security checksum lock. Apply to the original ROM.

### [`spo_special_edition_v1.1.ips`](patches/spo_special_edition_v1.1.ips) — Special Edition (recommended)

The full experience: `spo_versus_hack.ips` stacked with the first five standalone patches below. Apply directly to the original ROM — **not** on top of `spo_versus_hack.ips`.

### Standalone patches

These are independent fixes that can be applied alone or mixed and matched, on top of the original ROM or `spo_versus_hack.ips`:

| File | What it does |
|---|---|
| [`spo_sandman_stats_fix.ips`](patches/standalone/spo_sandman_stats_fix.ips) | Fixes Mr. Sandman's profile screen, which copies Super Macho Man's stats verbatim in US/EU. Restores the correct values from the Japanese version (age 30, weight 270 lbs, record 28-4). |
| [`spo_jp_charset_enabled.ips`](patches/standalone/spo_jp_charset_enabled.ips) | Makes the Japanese character set L/R-cycling always active in name entry, starting on the Western set. The hidden button combo is no longer needed. |
| [`spo_special_title_screen.ips`](patches/standalone/spo_special_title_screen.ips) | Cosmetic: replaces the title-screen ring logo and color palette with the Special Circuit variants for some extra flair. |
| [`spo_disable_security_checksum.ips`](patches/standalone/spo_disable_security_checksum.ips) | The World Circuit completion checksum prevents the Special Circuit from unlocking when using save states, emulators (SNES Classic, Switch NSO), or patched ROMs. This patch disables that checksum check. |
| [`spo_credits.ips`](patches/standalone/spo_credits.ips) | Adds a **CREDITS** entry to the Records View select screen. Selecting CREDITS launches the game's ending-cutscene credits roll. After the credits finish the game stays on the final screen and requires reset — that matches the original cutscene's behavior, not specific to this patch. |
| [`spo_sound_mode_ui_incomplete.ips`](patches/incomplete/spo_sound_mode_ui_incomplete.ips) | Proof-of-concept patch that adds a SOUND MODE entry to the title-screen menu and shifts the full menu UI to make room for it. **Known bugs: pressing A on SOUND MODE triggers the DATA CLEAR menu instead; selecting NEW GAME or CONTINUE then moving the cursor in the name entry screen causes a CPU deadlock.** Wiring the actual Sound Library launch requires further investigation. Not included in the Special Edition and not recommended for general use. |

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

### 4. Special Circuit always unlockable

The World Circuit completion checksum prevents the Special Circuit from unlocking when using save states, emulators, or patched ROMs. This patch disables that checksum check.

---

## How to play Versus mode

1. From the title screen, choose **NEW GAME** or **CONTINUE**.
2. At the **Mode Select** screen, choose **VERSUS MODE**.
3. Select your circuit and opponent (Controller 1 still controls the menu interactions).
4. Once the match starts, **Controller 1 controls the left boxer** and **Controller 2 controls the right boxer**.

---

## Nice to have / todos

- **"BEST TIME / YOUR BEST" table**: carries over from the Time Attack screen layout and appears during Versus opponent select. Cosmetic issue only; does not affect gameplay. It would be nice if we could hide or replace it altogether for a seamless UI.
- **"SPECIAL EDITION" on the title screen**: adding text or a sprite underneath the main game logo, that fades in and out with the rest of the title screen. Not feasible with current tooling — the title screen is entirely baked into a proprietary compressed tileset and tilemap (no runtime text rendering). Would require either a custom recompressor for the SPO compression format, ~19 KB of free ROM space for an uncompressed tilemap, or new letter tiles in the tileset.
- **"SOUND MODE" menu implementation**: adding SOUND MODE to the title-screen menu is a known-problematic approach — adding a 4th item to the title screen cursor system causes a CPU deadlock in name entry due to the highlight renderer iterating beyond the valid layout table bounds. A proof-of-concept UI patch exists (`patches/incomplete/spo_sound_mode_ui_incomplete.ips`) but it has two unresolved bugs (A-button goes to DATA CLEAR; name entry deadlocks). Note that the Mode Select screen has a fully separate cursor system that reinitializes item counts on every entry — the same interference issue would not apply there.

---

## Technical reference

For the full reverse-engineering notes — every patch address, the SNES memory map, the Mode Select state machine architecture, and the free-space inventory — see [doc/TECHNICAL.md](doc/TECHNICAL.md).
Key facts for anyone continuing this work:

- **ROM mapping:** LoROM. File offset = bank × 0x8000 + (SNES_addr − 0x8000).
- **Free space remaining:** ~1,240 bytes in the confirmed garbage zone at bank $0D (file 0x6FB0C–0x6FFE3). Bank $01 is fully consumed.

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
