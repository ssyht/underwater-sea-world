# The Wreck — checkpoint log

Measured in headless Chromium 1234 on ANGLE/Metal (Apple M5), 1280×720, pixel ratio 1.0,
`?debug=1`, numbers read from `renderer.info` after a full frame (caustic pass + scene).
Headless FPS is capped by the browser and is **not** a phone benchmark.

## Checkpoint A — Phase 1: foundation and environment

**Built:** renderer + quality tiers (pointer/cores/`?q=`) with runtime adaptation (22 ms for 2 s →
pixel ratio → particles → instance counts); background gradient, water-color chunk (per-channel
absorption, depth darkening, exp2 fog that melts into the same gradient as the background);
caustics (two distorted voronoi layers drawn each frame into a tileable texture, min/max
combine, pow sharpen, RGB split on high); hemisphere + directional sun; heightfield seabed
with noise-blended zones, vertex-weighted sand/rock/moss and fragment-shader sand ripples;
instanced displaced-icosahedron rock clusters, brain + branching coral, lattice sea fans,
swaying anemone clusters; 60/35 instanced kelp stalks (stipe + blades, height² sway driven by
the global current, backlit blade tips, near-camera dither so the camera never clips a blade);
crossed additive sun-ray planes; surface plane with Snell's window; 1500/600 drifting particles;
free-swim controls (WASD/Space/Shift-C, mouse look, touch joystick + drag-look) with collision;
`?debug=1`, `?shot=`, `?q=`.

**File size:** 57.5 KB

| Shot | Tier | Draw calls | Triangles |
|---|---|---|---|
| `?shot=kelp` | high | 11 | 110.7k |
| `?shot=clearing` | high | 10 | 105.3k |
| `?shot=kelp` | low | 10 | 33.7k |
| `?shot=clearing` | low | 9 | 32.2k |

Interaction check: holding W for 1.5 s swam 3.8 m along the look direction; a mouse drag
turned the view; holding Shift descended until the 1.2 m floor clearance stopped it.

**Known problems:** purple coral reads saturated blue at range (red absorption on a
blue-heavy albedo); caustics are visible but soft at grazing angles. Both are left for the
Phase 4 realism pass. One favicon 404 in the console → fixed with an inline empty icon.

## Checkpoint B — Phase 2: sea life

**Built (all 10, one instanced draw call each):** 2×120 / 2×60 silver sardines (CPU boids —
separation, alignment, cohesion, wandering goal — on a hashed grid, stepped at 30 Hz and
interpolated; inside 7 m of the camera they scatter with a higher turn cap and weakened cohesion,
then regroup; banked by turn rate; tail bend in the vertex shader, tail frequency from speed,
fresnel/sun flash on the flanks); 2 manta rays (body→tip flap wave, ~4 s period, flap/glide cycle,
banked closed centripetal Catmull-Rom loop with height drift, re-timeable for the tour); 1 sea
turtle (2 strokes then a 2–4 s glide, rear flippers steer, head turns, voronoi scutes);
25/12 yellow butterflyfish (hover/dart/flip at a home coral, eye band + false eye spot);
20/10 blue tangs with yellow tails (loose groups roaming between corals, pecking);
8/4 moon jellyfish (fast contract / slow relax pulse drives a vertical impulse, sink between
pulses, gonad rings, tentacles trail with phase lag and velocity drag); 30/16 garden eels (sway
in the current, retract inside 6 m, re-emerge slowly, buried part discarded under the sand);
1 grouper at the hull opening (pectoral sculling, turns to face a near camera); 20/10 sea stars;
40/20 urchins in rock crevices with spine sway. Creatures beyond 45 m skip updates; stars and
urchins shrink away between 45–50 m.

**File size:** 97.3 KB (includes the debug-only `?focus=<animal>&fd=&fo=` follow camera added to inspect each species)

**Least real animal: the manta ray.** Seen side-on (and the tour pushes both across the frame
at 10–14 s) it reads as a thin paper sheet with a stick for a tail.
How I would fix it — not applied, per the brief:
- *Geometry:* raise a real central disc (thickness peaking ~0.35 m over |x| < 0.25, falling off
  quickly into the wings), sweep the leading edge into falcate, pointed wing tips, model the
  cephalic lobes as rolled scoops either side of a mouth gap, add the pelvic fins and a shorter,
  thicker tail root.
- *Motion:* add a chordwise phase lag (trailing edge lags the leading edge by ~0.6 rad so the wing
  twists like a real stroke), heave the body opposite to the wing tips, add a slight pitch
  oscillation, and ease the glide pose into a gentle upward wing curve instead of flat.
- *Shading:* countershaded top with the pale shoulder chevrons placed on the front third only,
  belly spots and five gill slits on the underside, a soft wet-skin sheen (fresnel + broad sun
  specular), and wing-margin darkening seen from below.

Runner-up: the grouper's faceted 6-sided body and painted-on eye up close.

Bugs fixed during the phase: reef fish rotation went NaN when the first frame had dt = 0
(roll computed as turn/dt); manta uniform `uTop` collided with the shared background uniform.

## Checkpoint C — Phase 3: hero location

**Built:** 24 m wooden hull swept from a U section along the keel with bow taper, rounded transom
and sheer; rolled 18°, pitched 6° bow-down and bedded ~40% into a sand mound sculpted into the
heightfield under the ship only. Jagged missing section on the raised (port, local +x) side shows
instanced ribs; instanced deck planks with gaps, a collapsed middle section with planks fallen
into the hold, deck beams and a hatch coaming; broken mast stump plus the fallen mast and yard
half buried beside the hull; brass bell on a yoke and brass portholes; instanced coral heads,
branching coral, sea fans and algae tufts on the rail, hull, deck and mast. Five (three on low)
narrow additive shafts angle through the deck gaps into the hold, and an additive back-face
volume gives the hold a faint warm-teal glow. 40/24 bubbles (one draw call, fully in the vertex
shader) rise from the bow hatch and a hull crack, wobbling, accelerating, growing and respawning.
Whale skeleton (~18 m): flattened 4 m skull, 40/28 instanced vertebrae on a gently curving spine,
13 rib pairs (intact, broken, fallen flat, sunk), two separated jaw bones lying flat, noise algae
stains; sea stars and urchins sit on the bones. Glow path: a soft additive ground ribbon plus
140/70 bioluminescent points from the clearing edge to the bow, pulsing in a wave toward the ship.
LOD: `THREE.LOD` for the ship (42 m), the skeleton (34 m) and every rock cluster (28 m), each with
low-detail twins sharing instance buffers. Soft push-out colliders keep the swimmer outside the
hull box and the skeleton capsule.

**File size:** 124.2 KB

| Shot | High: calls / tris | Low: calls / tris |
|---|---|---|
| `?shot=ship` | **35 / 174.8k** | **35 / 60.4k** |
| `?shot=skeleton` | 32 / 117.7k | 32 / 48.8k |
| `?shot=kelp` | 17 / 134.9k | 16 / 46.6k |
| `?shot=reef` | 26 / 141.0k | 26 / 49.0k |
| `?shot=clearing` | 28 / 153.9k | 27 / 53.3k |
| `?shot=surface` | 21 / 186.3k | 20 / 56.0k |

Budgets hold everywhere (high < 120 calls / 350k tris, low < 60 / 150k).

**Changes to earlier code (and why):**
1. `extraHeight` now returns the ship mound — the brief asks for the mound in the heightfield.
2. `DECOR_V` moved from CREATURES to SHADER CHUNKS and applied to brain coral, branching coral
   and anemones — "small decor beyond 50 m is hidden" (they shrink to nothing 45–50 m).
3. Coral and sea-fan geometry/materials exported as `CORAL_KIT` / `FAN_KIT` so the wreck's
   growth reuses them (no visual change to the reef).
4. `SHIP_M` / `shipToWorld` moved from CREATURES to LOCATIONS (needed before the creatures run).
5. `CONFIG.ship.opening` moved onto the real hull hole and the grouper's offset shortened so it
   hovers at the opening instead of 2.5 m outside it.
6. Phase 2 star/urchin instance buffers grew by the bone spots so bone-dwelling animals cost no
   extra draw calls.
7. Debug helpers: `?cam=x,y,z,tx,ty,tz` pose and `__wreck.height` for inspection.

**Known problems:** at the 42 m LOD switch the low hull silhouette can read as sitting on the
mound rather than in it from bow-on angles; barnacle flecks were too bright (toned down).

## Checkpoint D — Phase 4: filmmaker realism pass

**Screenshots:** `?shot=kelp|reef|clearing|ship|skeleton` captured at 1280×720 in headless Chromium
(ANGLE/Metal GPU rather than SwiftShader — same pixels, much faster), reviewed, fixed, re-captured.

**The five things that looked most fake (worst first) and what changed:**

| # | Problem (category) | Why it read as fake | Fix (before → after) |
|---|---|---|---|
| 1 | Soft, dim caustics (light) — *must-have* | Faint blurred lines with no dark cells; at grazing angles the mip chain smeared them into a sliding wash | Thinner, harder filaments (line ×3.1, pow 7), intensity 1.25 → 2.3 with the cell interiors pulled below ambient, −0.8 LOD bias plus anisotropy, a slow per-surface warp of the lookup so filaments wobble locally. Now a crisp, moving net of bright filaments on the sand |
| 2 | Far objects went navy/slate (fog) — *must-have* | Absorption darkened objects well before the exp2 fog reached them, so the ship, rocks and school became dark silhouettes | Absorbed light is now replaced by in-scattered water colour per channel (`col·T + fog·(1−T)`, `T = exp(−d·absorb)`) before the exp2 mix. Red still goes first; distant things dissolve into blue-teal and are never darker than the water |
| 3 | Over-saturated plastic colour up close (colour) | Hemisphere sky (linear red 0.27) filtered red before the water did, double-counting the loss: purple coral went electric blue, sand flat mint | Sun 1.0 → 1.45, hemisphere 1.05 → 0.78, depth absorption 0.55 → 0.4; coral albedo desaturated 22%. Near colour reads closer to the palette; distance still strips red |
| 4 | Nothing grounded (light) | No contact shading under rocks, hull, bones or boulders; caustics shone straight through the hull overhang; the hull's raised bilge hovered over the sand | A 256² top-down occlusion map splatted once from every rock, the hull, the bones, coral heads and the kelp canopy (offset along the sun) darkens the sand, blocks caustics and adds contact AO. Sand mound shifted 1.8 m toward the raised side and raised to 3.2 m; bow forefoot flattened so the stem buries |
| 5 | Marine snow read as a starfield (scale) | Every point was a similar hard-edged disc | Size range skewed small and faint, gaussian sprites, ~7% large soft near-lens bokeh that fade out beyond 6 m |

Draw calls and triangles are unchanged by the pass (e.g. ship 35 / 174.8k high).
**File size:** 127.1 KB

**What I'd fix next (priority order):**
1. Rebuild the manta rays per the Checkpoint B plan (raised disc, falcate tips, cephalic lobes,
   chordwise flap lag, belly markings).
2. Hull/sand transition: blend a sand tint and noise onto hull vertices just above the mound and
   sculpt a shallow scour trench at the bow so the buried line stops reading as a clean cut.
3. Kelp density and canopy: a second stipe per holdfast and blades lying flat on the surface
   canopy so the forest reads layered (dark silhouettes against bright water) like the reference.
4. Coral heads: displaced brain-groove geometry, plate/table corals and encrusting sponge so the
   reef stops reading as smooth domes.
5. Garden eels and reef fish shading: eels read as dark sticks at range — add backlight and paler
   counter-shading; smooth the faceted reef-fish normals.

## Checkpoint E — Phase 5: sound and "Start dive"

**Audio (Web Audio only, no files):** master gain with a 3 s fade-in feeding a convolver whose
impulse is generated decaying stereo noise (~4 s). Low hum = looping brown noise → 180 Hz lowpass
plus detuned 48/55 Hz sines, with a 0.06 Hz LFO on volume and cutoff. Bubble clusters every 2–7 s:
3–8 sine blips of 40–90 ms sweeping 400→1200 Hz through a bandpass, randomly panned, up to 2.6×
louder near the wreck. Whale call every 25–45 s: sine + triangle gliding 180→110→140 Hz over
4–6 s, 4.2 Hz ±3 Hz vibrato, 600 Hz lowpass, heavy reverb send, low and randomly panned. Audio
suspends with the tab.

**UI:** translucent dark glass card (system font) with the title, "Start dive", "Free swim" and
control hints (touch hints on coarse pointers). "Start dive" unlocks/resumes the AudioContext,
hides all UI and runs the tour; tap, click or Esc skips it. On finish or skip, free swim takes
over from the current pose and velocity and a compact HUD fades in (title, hints, "Start dive",
Mute). Buttons blur after use so Space keeps swimming instead of re-pressing them.

**Tour:** one centripetal `CatmullRomCurve3` through the beat table, sampled with `getPointAt`;
per-segment smootherstep timing blended 50/50 with a C1 monotone cubic so the camera eases through
beats without stopping dead; the look target is 2.4 s ahead on the curve, blended to look down in
the canopy (0–5 s), at the lit deck (21–27 s) and up at the sun (27–30 s), reached with a damped
quaternion slerp; bank limited to 5°. Mantas are re-timed at tour start.

**Verified in headless Chromium:** after a real click on "Start dive" the AudioContext reports
`running`; camera at 0 s (−55,−3,−30), 15 s (5.1,−13,0.1), 19 s (20,−16,18), 29.7 s
(24.1,−2.1,34.1); both mantas cross the frame at 11.5–13 s; at 30 s mode returns to free swim and
the HUD is visible; Esc at 3 s skips; Mute toggles. No console errors.

**File size:** 139.5 KB

## Phase 6 — final pass

**Final file size:** 139.6 KB (`index.html`, unminified; budget 150 KB). Three.js r170 from jsDelivr
via an import map; no other libraries, no addons, no asset files.

**Draw calls / triangles per `?shot`** (1280×720, pixel ratio 1.0, after a full frame):

| Shot | High | Low |
|---|---|---|
| `kelp` | 17 / 134.9k | 16 / 46.6k |
| `reef` | 26 / 141.0k | 26 / 49.0k |
| `clearing` | 28 / 153.9k | 27 / 53.3k |
| `ship` | 35 / 174.8k | 35 / 60.4k |
| `skeleton` | 32 / 117.7k | 32 / 48.8k |
| `surface` | 21 / 186.3k | 20 / 56.0k |
| full 30 s tour (worst frame) | 47 / 228.9k | 47 / 74.4k |

Budgets confirmed: high stays under 120 calls / 350k triangles, low under 60 / 150k, on every
preset and on every frame of the tour.

**Performance check:** headless Chromium on ANGLE/Metal (Apple M5), full tour per tier —
frame interval p50 8.3 ms / p95 10.1 ms (high) and 8.3 / 9.2 ms (low), capped by the 120 Hz
headless clock; JS CPU per frame p50 1.3 ms (high) / 0.84 ms (low). The adaptive tier never
triggered. **Not run:** a real 2021 mid-range phone or a laptop iGPU — headless numbers on a
desktop-class GPU cannot prove the phone target; the runtime adaptation (pixel ratio → particles →
instance counts after 2 s over 22 ms) is the safety net there.

**Console:** no errors or warnings across every shot, both tiers, the tour, skip, mute and the
phone run. One `setPointerCapture` exception surfaced with synthetic touch events → now guarded.

**Mobile:** emulated 390×844 @3× touch device picks the low tier at pixel ratio 1.0; canvas and
camera follow resize and rotation (aspect 0.462 → 2.164 on rotate to 844×390); the left-half
joystick swam 3.7 m in 1.5 s; the card and HUD fit portrait and landscape with no horizontal
scroll.

**Known limitations:**
- Phone frame rate is untested on real hardware (see above).
- Manta rays still read as thin sheets side-on; the grouper is faceted up close (Checkpoint B plan).
- The hull meets the sand on a clean line, and the bow slightly overhangs its mound.
- Garden eels read as dark sticks at range; purple coral turns blue-violet with distance (physically
  plausible, visually loud).
- Safari < 15.4 lacks some WebGL2 paths three r170 relies on; there is a clear WebGL-failure message.
- Audio starts only after a click/tap (browser policy); "Free swim" also unlocks it.

**Top 3 next upgrades:**
1. Rebuild the mantas (raised disc, falcate tips, cephalic lobes, chordwise flap lag, belly
   markings) — they own the tour's 10–14 s beat.
2. Sand-to-hull blending and a scour trench around the bow, plus sand settling on the deck, so
   the wreck reads as buried rather than placed.
3. A layered kelp canopy (second stipe per holdfast, surface-floating blades) with stronger
   backlit translucency, matching the reference's dark-silhouette-against-bright-water look.
