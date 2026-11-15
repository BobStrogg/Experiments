# BEEBCRAFT

A side-on, Minecraft-like sandbox for the **BBC Micro Model B**, written in pure
**6502 machine code** for **MODE 7 (teletext)**. It runs on a real 32K Model B
(OS 1.20, Acorn DFS or MMFS from a MicroSD card) and in the browser through
[jsbeeb](https://github.com/mattgodbolt/jsbeeb).

```
 BEEBCRAFT  C=col T=style G=sep N=new
                             ***
                            *****
                  ***       """        "
                 *****     "%%%""     "%
                   |@+ "%%%%##%%%%%%%%%%
===============::::%%%%     ##  ##    ##
=========:::::::####C#    I     #      #
>%10 "00 #10 :00 |00 *00 C00 I00 G00 D00
 DIRT    X137 Y18 MONO ASCII     Q=QUIT
```

## What's here

| Path | What |
|---|---|
| `build/beebcraft.ssd` | Bootable DFS disc image (boot option 3, `!BOOT` = `*RUN MCRAFT`) |
| `build/BEEB.MMB` | 511-slot MMFS image: BEEBCRAFT in slot 0 (mounted in drive 0 at power-on), the 3D engines in slots 1–3 |
| `build/BEEB.MMB.zip` | The same MMB plus all four SSDs, zipped (165 KB instead of 100 MB) |
| `3d/` | **BEEBCRAFT 3D**: three first-person engines (fast raycaster, height levels, true voxel), see [3d/README.md](3d/README.md) |
| `src/beebcraft.6502` | The whole game: BeebAsm source |
| `src/loader.bas`, `src/boot.txt` | `GAME` BASIC loader and the `!BOOT` file |
| `web/` | Self-contained static site: `index.html` and a trimmed jsbeeb build that auto-boots the disc |
| `test/` | Headless jsbeeb tests (gameplay script, screenshots, timings, entry paths, browser check) |
| `screenshots/` | PNGs and text dumps of screen memory from the emulator |
| `vendor/` | jsbeeb, MMB_Utils and BeebAsm sources (build tools only, not part of the game) |

## BEEBCRAFT 3D

There are three first-person versions in [`3d/`](3d/README.md). They run on the
same Model B in MODE 7, from `3d/build/craft3d-*.ssd` or from MMB slots 1–3
(`*DIN 1`, `*DIN 2`, `*DIN 3`, then SHIFT+BREAK):

| Slot | Disc | Engine | fps (HI / LO detail) |
|---|---|---|---|
| 0 | BEEBCRAFT | the original 2D game (this README) | – |
| 1 | C3D-FAST | stage 1: fast raycaster | 14–17 / 25 |
| 2 | C3D-LEVELS | stage 2: raycaster with height levels, jumping, digging and building at heights | 7–9 / 12–14 |
| 3 | C3D-VOXEL | stage 3: true 16×16×8 voxel engine with overhangs, 3D dig and build (experimental) | 3.6–4 / 6–7 |

In the browser they are at `web/3d-fast/`, `web/3d-levels/` and `web/3d-voxel/`,
linked from `web/index.html`.

## Playing

### Controls

| Key | Action |
|---|---|
| `Z` / `X` | walk left / right (steps up one block automatically) |
| `SPACE` (or `:`) | jump, or swim up in water |
| cursor keys | aim the target cursor, up to 2 tiles from the player (it blinks) |
| `RETURN` | mine the target block (hold to keep mining) |
| `/` | place the selected block at the target |
| `1`–`9`, `0` | pick a hotbar slot: dirt, grass, stone, sand, wood, leaves, coal, iron, gold, diamond |
| `C` | colour ↔ monochrome |
| `T` | tile style: ASCII → SHADED → BLOCK → MIXED |
| `G` | contiguous ↔ separated graphics (block styles) |
| `N` | generate a new world |
| `Q` or `ESCAPE` | quit to BASIC (the BASIC program in memory is kept) |

Walking also turns the aim to face the way you walk. You can't place a block on
yourself. Bedrock and water can't be mined. Each slot holds up to 99.

### Screen

* Row 0: banner. Rows 1–22: the world viewport. Row 23: the hotbar (selected
  slot marked `>` in mono, yellow in colour). Row 24: the selected block, your
  X/Y position and the current display mode.
* The world is 256 × 40 tiles and scrolls both ways to keep you centred.
  Terrain is generated on the fly: hills, sea at row 22 with sandy beaches,
  grass/dirt/stone layers, a bedrock floor, cave tunnels, coal/iron/gold/diamond
  at increasing depths, and trees. The seed comes from the VIA timers, so
  every world (and every `N`) is different.

### Tile styles

| Tile | ASCII | BLOCK (sextant) | MIXED | Colour |
|---|---|---|---|---|
| grass | `"` | top bar | top bar | green |
| dirt | `%` | checker | checker | red |
| stone | `#` | solid | solid | white |
| sand | `:` | speckle | speckle | yellow |
| wood | `‖` | left bar | left bar | yellow |
| leaves | `*` | checker | checker | green |
| coal / iron / gold / diamond | `C I G D` | distinct sextant patterns | letters `C I G D` | white / magenta / yellow / cyan |
| water | `=` | lower two thirds | lower two thirds | blue |
| bedrock | `&` | jagged | jagged | white |
| player / cursor | `@` / `+` | `@` / `X` | `@` / `X` | inherits the colour |

**SHADED** (optional; see below) is density-shaded ASCII.

**MIXED** is the "both at once" option. Terrain uses sextant graphics, while
ores, the player and the cursor use capital letters. Teletext shows codes
&40–&5F as letters even in graphics mode ("blast-through"), so mixing them costs
no control codes.

### SHADED style (density-shaded ASCII)

This is the 4th style in the `T` cycle and works in both mono and colour. Plain
ASCII is unchanged. Each cell's glyph comes from one table lookup:
`shadeg[band*32 + exposed*16 + tile]`. That depends on:

* **exposure**: whether the cell above is open (air, the player, or the cursor
  on air), from a 16-byte `exptab`, OR'd onto the tile id;
* **depth band**: one of 4 bands by world row (rows 0–21, 22–27, 28–33,
  34–39). A per-row table (`rowband`) picks the 32-byte glyph table, patched
  into the loop once per row.

Glyphs were picked by how the SAA5050 actually draws them. It replaces
`#`, `_`, `[`, `]`, `\`, `^`, the backtick, `{`, `|`, `}` and `~` with £, #,
arrows, ½, —, ¼, ‖, ¾ and ÷, so those codes are used only where the substitute
itself looks right (‖ for wood, ÷ for deep water).

| Tile | covered (solid/water above) | exposed (open above) |
|---|---|---|
| grass | `v` | `w` (tufts) |
| dirt | `%` → `&` deeper | `o` (bare) |
| stone | `O` → `8` → `M` → `W` with depth (denser deeper) | `n` → `m` (floors) |
| sand | `:` | `.` |
| leaves | `x` (inner) | `*` (outer) |
| coal/iron/gold/diamond | `c i g d` (small, dark against dense rock) | `C I G D` (glint when exposed) |
| water | `=` → `÷` deep | `-` (surface ripple) |
| wood / bedrock | `‖` / solid block (&7F) | same |

Cost: 28 cycles a cell in mono instead of 18. In colour the loop reloads the
tile for the colour test, so about 50 cycles a cell when the colour doesn't
change. Two extra kinds of redraw keep the shading correct:

* In SHADED only, the row under the player is redrawn when the player moves.
* After mining or placing, the row under the target is redrawn.

The cursor uses a separate id on air cells so it never makes the shading
underneath flicker. The measurements below show it still fits the 25 Hz tick.

### Colour vs monochrome: the trade-off

In teletext a colour change is a control code, and that code takes up a
character cell (shown as a blank). BEEBCRAFT handles this as follows:

* **Monochrome** has no control codes in the world area. In ASCII mono all 40
  columns show world tiles. Block styles must switch to graphics mode, so they
  keep a 2-cell gutter at the left (contiguous/separated code, then
  graphics-white).
* **Colour ASCII** reserves 1 gutter column for the first colour; block styles
  keep 2 gutter columns, as above.
* Codes go in only where the colour actually changes. Air has no colour, so in a
  run like `stone, air, gold` the code goes in the air cell and nothing is lost.
  This covers most changes: sky to trees, ground, water.
* **Span merging:** if a single grass/dirt/stone/sand/wood/leaves/bedrock tile
  is sandwiched in another colour's run, it borrows that colour rather than
  costing two cells. Ores and water never do this, so they always show in their
  own colour.
* When two differently coloured solid tiles touch, the code replaces the cell
  *before* the change, which blanks the left neighbour. The player and cursor
  are never blanked. If the player's colour would have to change, it borrows the
  colour of the run it is in (blue in water, for example).
* Trees grow their leaves above the trunk rather than around it, because
  vertical neighbours never cost a code.

The visible result: in colour mode you see gaps of one cell where dirt meets
stone or where an ore sits in rock. Mono shows every tile. Press `C` to switch at
any time.

## Building

Requirements: [BeebAsm](https://github.com/stardot/beebasm) on the PATH, and
`perl` for the MMB step.

```sh
./build.sh          # -> build/beebcraft.ssd, build/listing.txt, build/BEEB.MMB
```

`build.sh` runs `beebasm -i src/beebcraft.6502 -do build/beebcraft.ssd -opt 3`.
The source `SAVE`s `MCRAFT` (load/exec &FF2000), `PUTTEXT`s `!BOOT` and
`PUTBASIC`s the `GAME` loader. `tools/make_mmb.sh` then builds `BEEB.MMB` with
MMB_Utils.

Building BeebAsm from source:

```sh
git clone https://github.com/stardot/beebasm && cd beebasm && mkdir build && cd build && cmake .. && make
# no C++ toolchain? this works too:
python3 -m venv zenv && zenv/bin/pip install ziglang
zenv/bin/python -m ziglang c++ -O2 -std=c++11 -o beebasm ../src/*.cpp
```

## Running on a real Model B

The game needs a Model B with OS 1.20 and DFS-compatible storage. It does not
need sideways RAM or a second processor. Files load into the I/O processor
(&FFxxxx addresses).

Ways to start it:

* **SHIFT+BREAK** boots it (`*OPT 4,3` → `!BOOT` → `*RUN MCRAFT`).
* `CHAIN "GAME"` runs a 3-line BASIC loader.
* `*RUN MCRAFT` or just `*MCRAFT` works too.

### MMFS (MicroSD in the user port / Turbo MMC firmware using BEEB.MMB)

**Option A: use the supplied image.** Copy `build/BEEB.MMB` to the root of the
FAT-formatted card (back up your own first). BEEBCRAFT is in disc slot 0, and
slot 0 is set as the drive-0 boot disc, so SHIFT+BREAK starts the game.

**Option B: add the SSD to your existing `BEEB.MMB`.** This uses
[MMB_Utils](https://github.com/sweharris/MMB_Utils), a copy of which is in
`vendor/MMB_Utils`:

```sh
cd vendor/MMB_Utils
./beeb dcat   -f /path/to/BEEB.MMB              # find a free slot, e.g. 123
./beeb dput_ssd -f /path/to/BEEB.MMB 123 ../../build/beebcraft.ssd
./beeb donboot  -f /path/to/BEEB.MMB 0 123       # optional: make it the drive-0 boot disc
# to update it later: ./beeb dkill -f /path/to/BEEB.MMB -y 123  then dput_ssd again
```

(`beeb <cmd>` is a wrapper around `<cmd>.pl`, e.g. `perl dput_ssd.pl ...`.)

Then, on the Beeb:

```
*DCAT                 (list discs; look for BEEBCRAFT)
*DIN 123              (mount slot 123 in drive 0; *DIN BEEBCRAFT also works)
then SHIFT+BREAK, or *EXEC !BOOT, or CHAIN "GAME", or *RUN MCRAFT
```

`*DBOOT 123` mounts the disc and boots it in one step. `*DIN 1 123` mounts it in
drive 1 instead (then `*DRIVE 1` and `*RUN MCRAFT`).

**Interfaces that read .SSD files directly** (for example some Turbo MMC /
GoMMC / Gotek-style setups): copy `build/beebcraft.ssd` to the card and select
it in that interface's menu or mount command. It's a standard 80-track,
single-sided DFS image.

## Playing in the browser

`web/` is a static site you can host as-is (GitHub Pages, Cloudflare Pages, any
web server). It holds a landing page plus a trimmed jsbeeb build (8.6 MB:
jsbeeb's own discs, tapes, econet and teletext-adaptor data were removed).

```sh
python3 -m http.server 8765 --directory web
# open http://localhost:8765/   (click the screen to give it the keyboard)
```

The emulator is opened at
`jsbeeb/index.html?model=B-DFS1.2&disc=beebcraft.ssd&autoboot&embed`, which loads
`jsbeeb/discs/beebcraft.ssd` and does SHIFT+BREAK automatically. Drop the
`&embed` for the full jsbeeb UI. After a rebuild, copy
`build/beebcraft.ssd` to `web/jsbeeb/discs/`. jsbeeb is GPL-3.0, and its licence
is included in `web/jsbeeb/`. `service-worker.js` there is a stub, since jsbeeb's
offline cache would list the files that were removed.

## Memory map (Model B, 32K)

| Address | Use |
|---|---|
| &0070–&008F | zero page (the BASIC user area; BASIC's own zero page is untouched) |
| &0220 | EVNTV hooked for the vsync event while running (restored on quit) |
| &1900 | PAGE: the `GAME` loader (only when `CHAIN`ed), which survives the game |
| &2000–&2FF2 | `MCRAFT`: code and tables, about 4.1 KB, loaded and executed at &2000 |
| &4000–&67FF | world: 40 rows × 256 columns, one page per row (row-major) |
| &6800–&6BE7 | 1000-byte off-screen frame buffer |
| &6C00 | height map (world generation) |
| &6D40–&6DB5 | key states, inventory counts, row-dirty flags |
| &6E00–&6E4F | variables |
| &7C00–&7FE7 | MODE 7 screen |

Load at &2000 (not &1900) so the BASIC loader at PAGE survives. That also
leaves room for PAGE up to &1F00 if other filing systems or ROMs raise it.

## Performance notes

* **Fixed 25 Hz game tick.** A vsync event handler counts frames, and the
  game runs input and physics every 2 frames.
* **Tear-free updates.** Everything renders into an off-screen buffer. After the
  next vsync it is copied to &7C00 with a 4-way interleaved loop
  (`LDA buf,X / STA scr,X` ×4, about 10.8K cycles). The copy starts in the
  vertical blank and runs about 20× faster than the raster, so it never shows a
  half-drawn frame.
* **Only what changed is redrawn.** Each world row has a dirty flag. Moving or
  blinking the cursor, mining or placing redraws 1–4 rows. A full redraw happens
  only when the camera scrolls or the mode changes.
* **Tight inner loops.** These use self-modifying absolute addresses instead of
  `(zp),Y`. Mono is 18 cycles per cell (`LDX world,Y : LDA glyph,X : STA buf,Y :
  DEY : BPL`). Colour is about 34 cycles per cell when the colour doesn't change.
  The player and cursor are written into the world map temporarily during the
  render, so the loops never test for them.
* **Keyboard.** Keys are scanned directly on the System VIA: auto-scan is
  turned off, each internal key number is written to port A, and bit 7 is read
  back. That takes about 1.5K cycles for all 26 keys, against about 7.5K
  through OSBYTE &81. Several keys can be held at once. The keyboard buffer is
  flushed every tick, `*FX229,1` makes ESCAPE an ordinary key, and on quit
  the game waits for Q/ESCAPE to be released.
* **Measured in jsbeeb** (`test/run-emulator.mjs` → `screenshots/perf.json`;
  2 MHz cycles including interrupts; 1 frame = 40,000 cycles, 25 Hz tick =
  80,000 cycles):

  | Redraw (per render) | full (scrolling), avg / max | partial (1–4 rows), avg |
  |---|---|---|
  | mono ASCII | 24,430 / 24,574 | 4,986 |
  | mono SHADED | 38,100 / 38,425 | 7,246 |
  | mono BLOCK / MIXED | 24,351 / 24,503 | ~5,060 |
  | colour ASCII | 42,766 / 43,113 | 6,085 |
  | colour SHADED | 59,583 / 59,997 | 8,841 |
  | colour BLOCK / MIXED | ~42,270 / 42,777 | ~6,050 |

  - Other costs per tick: screen copy 10.8K, key scan 1.5K (1.8K max), update
    0.7K (1.3K max).
  - Worst case is colour SHADED while scrolling: about 60K + 1.8K + 1.3K +
    10.8K ≈ 74K of the 80K budget.
  - While walking and scrolling in every style/colour combination, game ticks
    were exactly 2 frames apart (0 late ticks in 44 per combination), so the
    game holds 25 Hz.
  - SHADED is optional because it costs about 1.4–1.6× ASCII.

## How it was tested

`test/run-all.sh` covers all of the following:

* `test/run-emulator.mjs` boots `build/beebcraft.ssd` in a headless jsbeeb
  **Model B / DFS 1.2** with a real **SHIFT+BREAK**. It then presses keys
  through the emulated keyboard matrix and saves PNG screenshots and text
  dumps. The screenshots cover every display mode (colour/mono × ASCII/SHADED/
  BLOCK/MIXED, contiguous and separated graphics), walking, jumping, digging down,
  placing a block and generating a new world. It also writes
  `screenshots/perf.json` with full and partial render cycles for each style
  and the 25 Hz tick spacing.
* `test/test-boot.mjs` checks `CHAIN"GAME"`, `*RUN MCRAFT` and `*MCRAFT`, and
  that `Q` returns to a working BASIC prompt with the loader program intact.
* `test/browser-check.mjs` loads `web/index.html` in headless Chrome, checks
  that the game auto-boots and that keys reach it, and saves
  `screenshots/browser-*.png`.

## Known limitations / ideas

* Colour mode blanks one cell where two differently coloured solid tiles touch,
  as described above. That's a teletext limit, not a bug.
* The player is one tile tall. Physics is tile-based (no sub-tile motion).
  Water and sand are static (no flow or falling).
* There's no saving yet. A `*SAVE` of &4000–&67FF plus the variables would make
  it easy.
* No crafting, mobs, day/night or sound beyond a click on mine/place.
* The MMB image's boot behaviour was checked only by building it with
  MMB_Utils. Slot 0 is byte-identical to the SSD that was tested in the
  emulator. jsbeeb has no MMFS, so the MMFS path itself still needs a run on the
  real machine.
