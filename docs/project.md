# Text Voxel

A first-person explorer where the whole world is drawn with text characters, in the spirit of Voxel Space / Comanche heightmap engines. One HTML file, no dependencies, no build step: open `index.html` in a browser (or serve the folder) and it runs.

## Status

**Phase 7 shipped (2026-09-22):** the fidelity renderer. Cells shrank to about 4×7 px (360×129 at 1440×900), every cell carries a full colour, shading is smooth with distance fog and a graded sky, the day/dusk/night palettes crossfade, lakes reflect the trees and hills on their far shore, and after dark torches flank every gate and door, a campfire burns by each stone circle and merchants carry lanterns, all lighting the ground and walls around them. The look moved from posterised blocks to the fine glyph mosaic of the reference Martin shared; a faint scanline weave keeps the text-mosaic character.

**Phase 6 shipped (2026-09-22):** a detail pass. Mottled ground with two grass tones and surface marks, wave glyphs on water, masonry joints and weathered stones on walls, plank floors, ragged three-band tree canopies with conifers, bushes and boulders, drifting clouds, and merchants drawn as proper figures (hat, face, robe with folds, belt, boots, staff) in one of four robe colours.

**Phase 5 shipped (2026-09-22):** travelling merchants. Four named merchants walk the road network back and forth; when you're within four cells the HUD shows `E talk to …`, and E opens a trade panel (greeting, four or five goods with prices, Buy buttons, your gold) that freezes the player and releases the mouse; Close or Escape resumes play and re-locks the mouse. Gold starts at 100 and sits in the HUD.

**Phase 4 shipped (2026-09-22):** a living landscape. Forests of sprite trees (depth-tested against hills and walls), flocks of birds circling above the terrain, a road network joining every landmark (with fords where a road must cross water), and two larger castle designs (a keep castle and a three-ring concentric castle) alongside the small one, one of each guaranteed per world.

**Phase 3 shipped (2026-09-22):** on foot. You now spawn walking, with gravity, a jump, a sprint, and collision against structure walls, standing stones, ceilings and steep terrain, while doorways, gates and stair steps stay passable. `F` toggles a free fly/noclip camera and the HUD shows `MODE WALK` or `MODE FLY`.

**Phase 2 shipped (2026-09-22):** landmarks and view distance. Every world now has 3 castles, 5 towers, 5 ruins and 10 standing-stone sites placed at generation time, each a small voxel building you can fly around and into (castles and towers have real interiors behind a doorway), plus `[` / `]` keys that change the render distance live with the value shown in the HUD.

**Phase 1 shipped (2026-09-21):** playable. Seeded terrain with carved rivers and a sea, free-flying WASD + mouse-look camera with Pointer Lock, five-band cel-shaded text rendering, a 4-minute day/night cycle with day/dusk/night palettes, and a monospace HUD.

Verified in Chromium, Firefox and WebKit at 1440×900 (now 360×129 cells). Per frame: ~4 ms raymarch + ~2 ms compose in Chromium at the spawn, 7–9 ms raymarch looking at a large castle at night with its torches; Firefox ~7 + 4 ms, WebKit ~5.5 + 2 ms. `−` / `+` still change the cell size (font 5–12 px) if a machine needs it. World generation including roads and trees: about a second at most on rough seeds.

## How to run

- Double-click `index.html`, or
- in the Claude desktop app, the `fps` launch config serves `~/claudecode` on port 8090, so the page is at `http://localhost:8090/text-voxel/`.
- `?seed=123` in the URL loads a specific world. The HUD's seed box does the same (press Enter; the mouse must be released first).

## Controls

| Key | Action |
|---|---|
| Click | capture the mouse (Esc releases it) |
| Mouse | look around |
| W A S D | walk (or fly) forward / left / back / right, relative to where you look |
| Space | jump (walking) · ascend (flying) |
| Shift | sprint (walking) · descend (flying) |
| F | toggle walk / fly mode |
| E | talk to the merchant named in the prompt (within 4 cells) |
| Esc (panel open) | close the trade panel |
| R | new random seed |
| T | run the clock 10× faster (toggle) |
| − / + | smaller / larger characters (more or fewer cells) |
| [ / ] | view distance −50 / +50 cells (120 to 1000, default 500), shown live as `VIEW` in the HUD |

If the browser refuses Pointer Lock (some embedded browsers do), the page falls back to click-and-drag to look and says so in the help line.

## How it works, in plain English

1. **Walking.** The player is a 1.8-unit-tall, 0.6-wide body with the eye 1.55 up (a voxel is 1 unit, doorways are 5). Each frame gravity pulls the feet down and they land on whatever is under them: the terrain (interpolated between cells so slopes feel smooth) or, inside a footprint, the highest voxel top at or below feet + one step, which is how the player stands on the correct floor of a tower. Horizontal movement is split into substeps of at most 0.3 cells and each substep checks the body's four corners against voxel spans that overlap it above step height; a blocked move retries along one axis so you slide along walls. Terrain steeper than 1.5:1 blocks uphill. Space jumps only when grounded (about 1.2 units), Shift sprints, ceilings bump your head, and small drops stick to the ground so stairs don't feel bouncy. Water is walkable at its surface (you wade). `F` switches to a free fly camera (Space/Shift up/down, no gravity or collision, a soft floor at terrain+1); switching back settles the feet on the ground below with zero velocity.
2. **Terrain.** A 512×512 grid of heights comes from value noise (a small home-made noise function; no library) summed over six octaves, plus a "ridged" layer on the high ground so mountains have spines. The world wraps at the edges, so you can fly forever.
3. **Water.** Sea level is set at the 25th percentile of heights, so every seed has water. Then two rivers are carved: from a handful of candidate peaks the walk picks the two that travel furthest before reaching the sea. Each step looks at all eight neighbours and usually takes the lowest, but sometimes the second or third lowest, which gives the channel its wobble. The channel is forced to descend a little every step so it never runs uphill, and a small brush widens it downstream and shapes the banks. This runs once at generation time, never per frame.
4. **Materials.** Water, grass, rock (steep or high), snow (the top ~3.5% of heights) and "haze" for distant land.
5. **Rendering.** For each screen column a ray walks outward across the height grid. Every sample is projected to a screen row; a per-column "lowest unpainted row" means nearer terrain always occludes farther terrain (the y-buffer form of the painter's algorithm). What's left above the terrain is sky. The screen is a grid of small cells (about 4×7 px), each holding one glyph and one colour. Glyphs are rasterised once per cell size into masks; each frame the grid is composed into a pixel buffer (glyph pixels take the cell colour, the rest a dimmed copy of it, which is the faint weave you see) and put to the canvas in one call. Block glyphs are hand-built so the weave is the same at every size.
6. **Shading.** Each cell's brightness is ambient plus diffuse × how squarely its slope faces the sun or moon, continuous, then looked up on its material's dark-to-light colour ramp (16 steps) and blended toward the horizon colour with distance (8 fog steps, starting at 15% of the view distance). The palette itself is a set of keyframes (night, dusk, day) blended smoothly by the clock, so dusk creeps in rather than switching. The sky is a zenith-to-horizon gradient with a glow around the sun. Water reflects: below every stretch of water in a column, what stands above its far shore (hills, trees, sky) is mirrored into the water, darkened and fading, with ripple cells reflecting less.
7. **Lights.** After dark, torches flank every gate and door, a campfire burns by each stone circle and merchants carry lanterns. Each is a warm point light with a radius; terrain samples and wall rows near one get its colour added with a squared falloff (lights are bucketed by 32-cell region so a sample only tests the ones that reach it), and the flames themselves are flickering teardrop sprites.
8. **Time.** One in-game day is 240 real seconds. The sun and moon move across the sky and light the terrain; the palette hard-switches between day (07–17), dusk (05–07, 17–19) and night. Stars appear when the sun is below the horizon.
9. **Landmarks.** After the rivers and the sea, a seeded pass scans the map on a coarse grid in random order and picks sites: castles need a big flat patch, towers a small one, ruins tolerate rougher ground, standing stones want a hilltop. Every structure keeps a minimum distance from every other. Castles, towers and ruins *level the ground* under their footprint (with a blended skirt), and their base is that height, so nothing floats or sinks; stones start a unit below the lowest ground in their footprint and rise from there. Each structure is its own small voxel grid built from a template: the castle has a crenellated curtain wall, four corner towers, a gatehouse with a passage through it and a roofed great hall with a door and windows; the tower has two wooden floors, a spiral stair up the inside wall, a doorway and windows; ruins are castles or towers knocked down by a noise map with one wall breached and rubble scattered; stones are single menhirs or circles with an altar. If a rugged seed offers no site flat enough, the flatness rule is relaxed in steps until every type is placed.
10. **Rendering landmarks.** The y-buffer became a per-column *painted mask*, so a doorway lintel or a ceiling can be painted with a gap below it that farther things still show through. Each column's ray is tested against the footprints of structures within the view distance (nearest first); when the terrain march reaches a footprint it hands over to a cell-by-cell walk (a DDA) through that structure, which paints every voxel span it crosses as a front face plus its top or underside, then hands back. Once the last footprint is behind the ray no structure work happens. Wall faces are cel-shaded by how squarely they face the sun or moon (sunlit / lit / shadow), roofs read light, undersides dark, and any face seen from air that is under something solid drops two bands, which is what makes interiors read as enclosed. Stone and wood have their own colour pairs in all three palettes.
11. **Roads.** Landmarks become nodes joined by a spanning tree plus each node's two nearest neighbours under 150 cells; pairs more than 220 apart get no road. Each road is an A* path on a half-resolution grid whose step cost is distance × (1 + 2.5 × slope), six times as much over water and twenty times inside another building's footprint, so roads wind up valleys, cross water only where nothing else works (those cells become fords, drawn as road) and skirt other buildings. Roads start and end just outside each building's south (door) side, are two cells wide, and have their own tan material. Heights along a road are pulled toward a running average by at most 1.5 units before trees, materials and collision are computed; buildings' levelled ground is left alone.
12. **Trees.** A broad density noise makes forests; inside them about one cell in nine grows a tree, elsewhere the odd lone one, never on water, snow, slopes over 3:1, roads (plus a cell) or footprints (plus two cells), with a cell between trunks. Trees are not voxels: each is a camera-facing sprite (a wood trunk under a leaf canopy, lit half and shaded half by the sun's side) sized by distance, stored in 32-cell buckets so a frame only considers those in range.
13. **Sprites and depth.** The raymarch now records the distance at which it painted every cell. Trees and birds are drawn afterwards, far to near, and a sprite cell only lands where it is nearer than what is already there, so hills and walls hide sprites and sprites hide what lies behind them. Nothing is drawn beyond the haze line (two thirds of view distance), where the land is already fog-coloured, so sprites never pop in the foreground.
14. **Birds.** Six flocks of six circle a point 15–35 above the local ground on 10–25-cell loops with a gentle bob, cycling three wing shapes (`v` `-` `^`, three characters wide when near). Silhouettes only, no collision.
15. **Castle variants.** Small (15×15, as before), keep castle (21×21: outer wall with seven towers and a gatehouse, an inner ward wall with turrets and its own gate, a two-storey keep with stairs) and concentric castle (27×27: three wall rings each taller than the last, eleven towers and turrets, a central keep). Every world gets one of each plus one more at random. All gates and keep doors line up on the south side, so the road leads straight in.
16. **Merchants.** Four merchants each own one of the longest roads (at least 40 cells) and start somewhere along it. Every frame they advance 3 cells/s along the road's centre line, standing on the same ground the player uses; at either end the direction flips and the overshoot is kept, so there is never a snap. They render through the same depth-tested sprite path as trees and birds: a block-character figure (round head, torso, arms, legs, three-frame walk cycle driven by distance walked) up close, a 3×3 glyph figure at mid range, `o`/`#` further, a single `i` far off. Names combine 12 first names with 10 epithets; greetings come from a pool of six; goods are 4–5 picks from a 12-item pool with prices rolled within each item's range at generation.
17. **Talking and trading.** Each frame the nearest merchant within four cells (and within four units of height) is the talk target; only one prompt ever shows. E opens the panel: keys held at that moment are cleared, movement, mouse-look, gravity and velocity are skipped entirely (not just hidden) until it closes, the merchant you're talking to stands still while the others and the clock carry on, and pointer lock is released so the buttons are clickable. Buy deducts from the gold counter, tallies `owned ×n`, and buttons disable when unaffordable. Close or Escape hides the panel and requests pointer lock again directly (both count as user gestures); if a browser refuses, the normal "Click to explore" screen appears.
18. **Texture.** Every cell carries a small texture code fixed at generation: 12% render a shade lighter, 12% a shade darker (a dither that stays put as you move, because it belongs to the world cell, not the screen), and 16% carry a surface mark within 45 cells (a darker grain cell; at large cell sizes the actual tuft, crack, pebble and sparkle glyphs). Grass comes in two tones from a patch noise. Water sits a shade brighter than flat land and ripples darken cells in moving bands. Stone wall faces get a darker joint line along the base of every voxel course and one stone in seven a shade darker; wooden floors alternate plank cells. Trees have a ragged outline, a smooth lit-crown-to-dark-underside gradient, a per-tree hue jitter, striped trunks and a ground shadow; a quarter of them and all high ones are tiered conifers; bushes grow between trees and boulders sit on rock. Ten clouds drift on one wind at 95–135 units, shaded lit-above/grey-below, depth-tested like everything else.

## Decisions worth knowing

- **Y-buffer instead of far-to-near painting.** Same image, immune to the "far terrain overwrites near terrain" bug, and columns stop early once they hit the top of the screen.
- **Lighting is relative to flat ground**, not absolute. Absolute brightness put 85% of the map in the top band; relative brightness gives flat = band 3, sun-facing = 4, shaded = 2/1/0.
- **Phase 7 replaced the five-band posterisation** with continuous shading, at Martin's request after seeing a reference world. The five-band code is gone; glyph density is now texture only, colour does the shading.
- **The frame buffer is composed in JavaScript**, not with text calls, because at 4-pixel cells a text call per run would be far too slow. The masks are still real rasterised glyphs (the block glyphs are hand-built for a consistent weave).
- **Camera pitch is a horizon shift**, the classic voxel-space trick, so you cannot look straight up. The sun and moon peak at ~49° elevation so they stay reachable.
- **One heightmap.** Road smoothing edits the same heightmap everything else reads, before trees, materials and collision are derived, so what you see, what you walk on and where trees stand always agree.
- **Tree trunks block walking** (one cell each); canopies do not. Birds have no collision.

## Known limits / next phases

- Water is walkable at its surface rather than swimmable or blocking.
- Purchases are a tally per merchant, not an inventory; goods have no effect yet.
- Up close (under three or four cells) a world cell fills a large patch of screen, so the ground turns into big flat blocks there; that is the heightmap renderer's nature, not a setting.
- Reflections are screen-space (mirrored about the far shoreline in each column), which is right for what stands on the shore and increasingly approximate for what stands well behind it.
- Roads on flat ground run dead straight or at exact diagonals (8-direction A*); they wind only where the ground makes them.
- Collision is against structures and slopes only; the player can still stand on a wall top after flying there, which is fine.
- The horizon-shift pitch limits looking up to roughly 50°.
- Rivers are 3–5 cells wide; from far away the ray sampling can skip them, so they read best from mid-distance.
- No touch controls.

## Debug handle

`window.TV` exposes `world` (including `world.structs`, `world.roads`, `world.trees`, `world.birds`, `world.merchants`), `ui` (panel state, nearest merchant, gold), `openPanel(m)` / `closePanel()`, `updateMerchants(dt)` / `updateNearest()`, `cam`, `player` (mode, feet height, vertical velocity, grounded), `keys`, `clock` (`clock.scale = 0` freezes time), `grid`, `view`, `light`, `stats` (render and draw time per frame), `regenerate(seed)`, `setMode('walk'|'fly')`, `update(dt)` (one physics step, handy for scripted tests), `groundAt(x, z, feetY)` and `setHour(h)` for poking at things from the console.
