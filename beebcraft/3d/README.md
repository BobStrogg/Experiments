# BEEBCRAFT 3D

First-person 3D versions of BEEBCRAFT for a real **BBC Micro Model B**: pure
6502, **MODE 7**, 32K, OS 1.20, DFS or MMFS. They need no sideways RAM and no
second processor. There are three engines, built one after the other. Each
has its own source, disc, MMB slot and web page. The 2D game in `../src` is
unchanged and still sits in MMB slot 0.

| Stage | Engine | Disc (title) | MMB slot | Web page | Speed (hi / lo detail) |
|---|---|---|---|---|---|
| 1 | Fast raycaster (flat world, Wolfenstein-style) | `build/craft3d-fast.ssd` (C3D-FAST) | 1 | `web/3d-fast/` | 14–17 / 25 fps |
| 2 | Raycaster with height levels (hills, steps, holes, pillars, jumping) | `build/craft3d-levels.ssd` (C3D-LEVELS) | 2 | `web/3d-levels/` | 7–9 / 12–14 fps |
| 3 | True voxel engine, 16×16×8 (experimental) | `build/craft3d-voxel.ssd` (C3D-VOXEL) | 3 | `web/3d-voxel/` | 3.6–4 / 6–7 fps |

All fps figures were measured in jsbeeb (cycle-exact Model B, 2 MHz 6502)
while turning on the spot, from the game's own frame counter over 250 vsyncs
(5 s). The world is random each boot, so expect ±10%.

## Files

```
3d/build.sh                 build all three + MMB + web pages
3d/fast/main.6502           stage 1 source (BeebAsm); tables.6502 is generated
3d/levels/main.6502         stage 2
3d/voxel/main.6502          stage 3
3d/*/boot.txt, loader.bas   !BOOT (*RUN CRAFT3D) and the GAME BASIC loader
3d/tools/gen_tables_*.py    trig / reciprocal / quarter-square / projection / texture tables
3d/tools/make_web.py        writes web/3d-<engine>/index.html and the links on web/index.html
3d/test/run3d.mjs <engine>  headless jsbeeb: screenshots in every render mode, fps, gameplay asserts
3d/test/boot3d.mjs <ssd>    SHIFT+BREAK, CHAIN"GAME", *RUN CRAFT3D, *CRAFT3D, quit back to BASIC
3d/test/browser3d.mjs <e>   headless Chrome against the web page (needs web/ served on :8765)
3d/screenshots/<engine>/    PNG + text dumps of screen memory, perf.json (fps, cycles, results)
3d/build/                   craft3d-fast.ssd, craft3d-levels.ssd, craft3d-voxel.ssd, listings
```

## Running on a real Model B

Each disc boots with **SHIFT+BREAK** (`*OPT 4,3` → `!BOOT` → `*RUN CRAFT3D`).
`CHAIN "GAME"`, `*RUN CRAFT3D` or `*CRAFT3D` also work. `Q` or `ESCAPE`
quits to BASIC.

With MMFS and the supplied `build/BEEB.MMB` (also in `build/BEEB.MMB.zip`):

```
*DIN 1      then SHIFT+BREAK     stage 1, fast raycaster
*DIN 2      then SHIFT+BREAK     stage 2, height levels
*DIN 3      then SHIFT+BREAK     stage 3, voxel engine
*DIN 0      then SHIFT+BREAK     the original 2D BEEBCRAFT (also the power-on default)
```

(`*DBOOT 2` mounts and boots in one go.) To add them to your own MMB:
`./beeb dput_ssd -f BEEB.MMB <slot> 3d/build/craft3d-levels.ssd`, etc. (see the
main README). Interfaces that read SSDs directly can use the `.ssd` files as they are.

## Controls

Common to all three:

| Key | Action |
|---|---|
| cursor up / down (or `W` / `S`) | walk forward / back (with collision) |
| cursor left / right | turn |
| `Z` / `X` (or `A` / `D`) | strafe left / right |
| `1`–`9`, `0` | select block |
| `C` | colour ↔ monochrome |
| `T` | sextant blocks ↔ ASCII density shading |
| `G` | contiguous ↔ separated graphics |
| `H` | HI detail (38 rays, 2 px per ray) ↔ LO detail (19 rays, faster) |
| `N` | new random world |
| `Q` / `ESCAPE` | quit to BASIC |

| | Stage 1 fast | Stage 2 levels | Stage 3 voxel |
|---|---|---|---|
| `RETURN` | mine the block in front | dig: lower the column under the crosshair (top) or in front (face) | dig the exact voxel under the crosshair |
| `/` | place a block in front (`SPACE` also works) | build: raise that column by one | build onto the face you are looking at (side, top or underside) |
| `SPACE` | place | jump | jump |
| `R` / `F` | – | look up / down | look up / down |
| steps | – | walking into a one-level step climbs it | walking into a one-block step climbs it |

The screen has 20 rows of 3D view with a crosshair. Under it are the title and
fps, the hotbar with counts, the selected block and target, the position and
compass heading, the display mode with a key reminder, and a minimap
(bottom-right, in sextant graphics, with your position blinking).

## Render modes (all engines)

* **Colour blocks**: sextant graphics, one SAA5050 colour per 2×3 cell. Colour
  changes need a control-code cell, so the renderer replays per-row colour runs
  built during the cast.
* **Mono blocks**: white sextants. Textures are dither patterns per material
  and per face (lit / shaded), so blocks read clearly without colour.
* **Separated** (`G`): either of the above with separated graphics.
* **ASCII density** (`T`): characters by brightness (`#%&o:-.` and
  per-material glyphs), in mono or colour (alphanumeric colours).
* **HI / LO** (`H`): LO casts half the rays and doubles them.

Measured fps (jsbeeb, Model B):

| Mode | Stage 1 fast | Stage 2 levels | Stage 3 voxel |
|---|---|---|---|
| colour blocks HI | 14.4 | 7.2 | 3.6 |
| colour blocks LO | 24.8 | 11.6 | 6.4 |
| mono blocks HI | 16.6 | 8.2 | 3.8 |
| mono blocks LO | 25.0 | 13.6 | 7.2 |
| mono separated HI | 16.6 | 8.2 | 3.8 |
| colour separated HI | 14.8 | 7.4 | 3.6 |
| colour ASCII HI | 16.6 | 7.8 | 3.8 |
| mono ASCII HI | 16.6 | 8.8 | 4.0 |

Per-frame cycle costs (HI, colour) from `perf.json`:

| | Stage 1 | Stage 2 | Stage 3 |
|---|---|---|---|
| raycast | ~60K | ~215K | ~490K |
| render (colour conversion) | ~43K | ~15K | ~16K |
| copy back buffer → &7C00 | 10.9K | 10.9K | 10.9K |

## How each engine works

**Stage 1, fast raycaster.** A 32×32 map with one block id per cell. The ray
directions are 4.12 fixed point, using camera-plane DDA. 1/|dir| comes from a
reciprocal table, and multiplies use quarter squares (8×16 with a cached
multiplier). Only even screen columns are cast. Each odd column is
interpolated when both neighbours hit the same cell face, since wall height
is linear in screen x. Rays stop after 20 cells and draw a fog block there.
Walls are textured by material and side. Floors and sky are flat, and colour
floor rows use per-row colour runs. The frame is drawn to a back buffer and
copied after vsync, because MODE 7 can't double-buffer.

**Stage 2, raycaster with height levels.** Each map byte holds
`block*16 + height` (0–15, eighths of a block on screen). Rendering is a
voxel-space column march. At each cell boundary the engine draws the top of
the cell it leaves (if that is below the eye) and the face of the cell it
enters (if it is higher). Each column has a 60-pixel material buffer and a
"lowest drawn" horizon. Projection is `3/d` in 3.5 fixed point, patched once
per boundary into a quarter-square multiply with the shift baked in.
Identical neighbouring cells are merged, and the ray exits early once nothing
further can rise above the drawn horizon. Physics: feet height in 8.8
eighths, gravity, jump and auto step-up. `R`/`F` move the horizon to look
up and down.

**Stage 3, true voxel engine.** The world is 16×16×8 voxels: eight 256-byte
layers of block ids plus a 256-byte column bitmask (bit z = solid). It holds
hills, ores, a planks hut with a doorway and an overhanging roof, a stone arch
with a gold lintel, and trees with canopies. Each ray walks the 16×16 grid.
In each column cell, every run of solid voxels is projected:

* each voxel's side face at the entry distance (so each block keeps its own
  texture);
* the run's top between the entry and exit distances (if below the eye);
* its underside (if above the eye).

These go into a 60-pixel column with per-pixel occlusion, which handles
overhangs, arches, roofs and canopies. The column keeps an "uncovered window"
`[wtop, wend)`. It is used to:

* clip fills;
* skip whole cells whose highest projected point is already covered;
* end the ray once the window lies above the projected world ceiling.

Dig and build work on the exact voxel and face under the crosshair (side, top
or underside). Collision and gravity are fully 3D: you can walk under the
roof and the arch, and you bump your head on ceilings.

## Testing

```sh
./3d/build.sh                                  # all three discs, BEEB.MMB (+zip), web pages
node 3d/test/run3d.mjs fast|levels|voxel       # screenshots per mode, fps, gameplay asserts
node 3d/test/boot3d.mjs 3d/build/craft3d-voxel.ssd
python3 -m http.server 8765 --directory web & node 3d/test/browser3d.mjs voxel
```

What the gameplay scripts check:

* **fast**: walking never ends inside a solid cell. A target is found, placing
  adds one block and removes it from the inventory, and mining reverses that.
* **levels**: a jump peaks about 1.1 levels up, walking climbs the staircase
  to level 8, and dig and build change the height sum and inventory by ±1.
* **voxel**: a jump peaks 1.13 blocks up. Digging the arch pillar clears that
  exact voxel in VOX (count −1, inventory +1), and placing adds it back.
  Looking down and digging removes the top voxel underfoot (target kind =
  top).

The MMB slots 1–3 were checked by extracting each with `dget_ssd` and
comparing with the built `.ssd`. Each extracted disc then passed all four
boot paths.

## Limitations

* Stage 1: flat world. The sky is plain black. Colour is per column (MODE 7
  allows one colour change per character cell), so block edges in colour get
  a one-cell control-code gap.
* Stage 2: one height per column, so no overhangs or tunnels (that is what
  stage 3 is for). Up to 16 boundaries per ray, then haze. No texture
  u-coordinates (textures vary by height and face only).
* Stage 3: 3.6–4 fps at HI and 6–7 at LO, so it's playable but sluggish.
  * The world is small (16×16×8) and there is no fog beyond the edge.
  * Side faces use the voxel's entry distance only, so faces seen at a
    grazing angle look flat-stepped.
  * The colour cell rule makes small features in colour look blocky.
  * Movement per frame is capped at 12 vsyncs, so the game slows slightly
    below 4 fps.
* All: worlds are random per boot, and there is no save/load in 3D.
