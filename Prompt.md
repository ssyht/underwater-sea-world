# Build: "The Wreck" — a procedural 3D underwater world in one HTML file

You are a senior creative technologist and an art director for underwater films. Build a 3D underwater world in a single `index.html` using Three.js. Follow this spec exactly. Work in phases. After each phase, stop, give me a short summary (what you built, file size, any known problems), and wait for me to say "next".

---

## 0. Hard constraints (apply to every phase)

- **One file:** `index.html`. All JS, GLSL and CSS go inline. Load Three.js from a CDN through an import map, pinned to one version (e.g. `https://cdn.jsdelivr.net/npm/three@0.170.0/build/three.module.js`). Use no other libraries and no addons unless a phase says so.
- **No asset files:** no models, images, textures or sound files. Every shape, texture and sound is made in code (BufferGeometry, procedural shaders, DataTextures, Web Audio).
- **Phone-first performance:** must hold 60 fps on a 2021 mid-range phone and a laptop iGPU.
  - Budget: under 120 draw calls and under 350k triangles on high quality; under 60 draw calls and under 150k triangles on low.
  - No shadow maps and no post-processing composer. Fake everything in shaders.
  - Pixel ratio = `min(devicePixelRatio, 1.5)` on desktop and `1.0` on low quality.
  - Use `InstancedMesh` for anything repeated (kelp, coral, rocks, fish, vertebrae, planks, bubbles).
  - Camera far plane sits right at the point where fog is fully opaque (~90 m). Nothing past it renders.
- **File size target:** under 150 KB unminified. Report the size at every checkpoint.
- **Additive phases:** each phase keeps everything already built exactly the same. Don't refactor, re-tune or restyle earlier systems unless you're fixing a bug. If you must touch earlier code, list each change and why at the checkpoint.
- **Code layout inside the file:** clearly commented sections: `CONFIG`, `QUALITY`, `NOISE/UTILS`, `SHADER CHUNKS`, `ENVIRONMENT`, `LOCATIONS`, `CREATURES`, `CONTROLS`, `TOUR`, `AUDIO`, `UI`, `LOOP`. Put every color and tunable number in `CONFIG` at the top.
- **Debug helpers:**
  - `?debug=1` shows FPS, draw calls, triangle count (`renderer.info`), quality tier and camera position.
  - `?shot=<name>` jumps to a fixed camera preset for repeatable screenshots: `kelp`, `reef`, `clearing`, `ship`, `skeleton`, `surface`.
  - `?q=low|high` forces a quality tier.

---

## 1. Art bible (the plan)

### 1.1 World and layout
Units are meters. The water surface is at **y = 0**. The world is a 200 × 200 m disc (soft boundary at radius 90 m). The seabed is a heightfield at about **y = −20** that varies by zone.

| Zone | Center (x, z) | Size | Seabed y | Content |
|---|---|---|---|---|
| **Kelp forest** | (−45, −10) | radius 28 | −18 ± 2, gentle rolling | 60 kelp stalks (35 on low), 14–18 m tall, rounded boulders at the bases |
| **Rocky reef** | (25, −35) | 45 × 25 arc, NE | rises to −11 in ridges | Rock clusters, coral heads, sea fans, urchins. Reef fish and jellyfish live here |
| **Sand clearing** | (0, 5) | radius 18 | flat −21 | Rippled sand, garden eels, sea stars. A bright caustic stage |
| **Whale skeleton** | (15, 22) | ~18 m long | −21 | Half-buried, sits on the glow path |
| **Sunken ship** | (38, 32) | ~24 m hull | −22, sand mounded around it | Hero location. Tilted, half buried, covered in coral |
| **Glow path** | clearing edge (8, 12) → ship bow (30, 27) | 2 m wide | — | Soft bioluminescent trail leading to the wreck |

Blend the zones with noise so no straight edges show.

### 1.2 Palette (hex)
| Role | Hex |
|---|---|
| Background top (looking up) | `#6CC3D5` |
| Fog / horizon | `#1A5F78` |
| Background bottom / deep | `#08263A` |
| Surface underside / Snell's window | `#BFEFF5` |
| Sun rays | `#CFF6F2` (additive, about 0.10 alpha) |
| Sand / sand shade / ripple crest | `#C9B98F` / `#8E8466` / `#E4D8B0` |
| Rock / rock dark / moss | `#4A5A5C` / `#2C3638` / `#5E7A4A` |
| Kelp stem / blade / backlit tip | `#5F6B24` / `#7E8B33` / `#A9B04A` |
| Coral pink / orange / purple / yellow / sea fan | `#E07A8F` / `#E8894A` / `#8A5BA8` / `#E3C35A` / `#B5443A` |
| Whale bone / stain / algae | `#D9D2BE` / `#A9A48C` / `#6F8A5A` |
| Ship wood / dark wood / waterlogged / brass | `#5A4330` / `#3A2A1E` / `#4B4A3C` / `#8C7A3E` |
| Glow path | `#7FE3D6` |
| Particles / bubbles | `#DDF3F5` / `#E6FAFF` |

Colors are the albedo before water. The water shader then removes red first with distance and depth, so true color shows only up close.

### 1.3 Light and water
- **Lights:** hemisphere light (sky `#8FD6E3`, ground `#2A3A30`) plus one directional "sun" light (`#D8F4F0`), angled about 15° off vertical. No shadows.
- **Water color model:** inject into every material through `onBeforeCompile`, or build it into the custom shaders.
  - Distance absorption per channel: `color *= exp(-dist * vec3(0.060, 0.022, 0.016))`, then mix toward the fog color with `FogExp2`-style density (~0.026).
  - Depth darkening: the deeper a fragment is (more negative y), the darker and bluer it gets.
  - Far objects must melt into blue-teal, never gray.
- **Caustics:** a procedural, animated pattern projected from above in world XZ onto every surface, weighted by `max(normal.y, 0)`. Two layers of distorted voronoi or sine-grid moving at different speeds, combined with `min`/`max` and sharpened with `pow`. The pattern is brighter on shallow surfaces and fades with depth and distance. High tier gets a slight RGB split. This is what makes the sand look like real sunlight through water.
- **Sun rays (god rays):** 10–14 tall additive planes (crossed pairs), angled with the sun. The shader fades them at the bottom and edges and scrolls noise along them. They sway slowly and get brighter near the surface. `depthWrite: false`.
- **Surface from below:** a plane at y = 0 showing animated ripple brightness, plus a brighter Snell's-window disc that follows the camera's x/z.
- **Floating particles:** `THREE.Points` (1500 on high, 600 on low) drifting in a box that wraps around the camera. Size scales with depth. Slow Brownian drift plus the current. Soft round sprites drawn in the shader.
- **Current:** one global `current(t, pos)` function (direction plus noise gusts). Kelp, particles, bubbles and small fish all use it so the whole scene moves together.

### 1.4 Ten kinds of sea life
Build every animal from code shapes. None may move in a stiff loop. Use per-instance phase, speed and size variation; noise-driven wander; capped turn rates; banking into turns; and speed tied to fin or tail frequency.

| # | Animal | Count (high/low) | Where | Motion |
|---|---|---|---|---|
| 1 | **Silver schooling fish** (sardine-like, `#C8D3D8` belly, `#4E6A78` back, fresnel flash) | 2 schools × 120 / 2 × 60 | Kelp edge and over the clearing | CPU boids (separation, alignment, cohesion, goal seek). They **scatter and turn away when the camera comes within 7 m**, then regroup. Tail wiggle in the vertex shader |
| 2 | **Manta rays** (top `#2B3138`, belly `#E8E6E0`) | 2 | Glide loops between the reef and the clearing, y ≈ −8 | Wing flap as a wave traveling from body to tip, slow (period ~4 s), long glides. They follow smooth closed curves with bank, gentle pitch and a slight height drift |
| 3 | **Sea turtle** (shell `#6B5A3A`, scute pattern `#8C7448`, skin `#7D8160`) | 1 | Slow wander over the reef and clearing | "Flying" front-flipper stroke: 2 strokes, then a 2–4 s glide. Rear flippers steer. The head turns slightly |
| 4 | **Yellow reef fish** (`#F2C230`, butterflyfish-like, tall body) | 25 / 12 | Around coral heads | Hover and dart near home coral, flip direction |
| 5 | **Blue reef fish** (`#2E6FD8`, yellow tail `#F4D03F`) | 20 / 10 | Reef | Loose groups, pecking at coral |
| 6 | **Moon jellyfish** (`#CFE9F2` at 0.35 alpha, rim `#E7B3D0`) | 8 / 4 | Above the reef edge | Bell pulse (contract fast, relax slow) that pushes them up, then they drift down. Tentacles trail with lag |
| 7 | **Garden eels** (pale `#D8CFA8`, speckled) | 30 / 16 | Sand clearing | Sway upright in the current. **Pull down into the sand when the camera comes within 6 m**, then slowly re-emerge |
| 8 | **Grouper** (`#6E5B47` mottled) | 1 | Hovers at the ship's broken hull opening | Slow fin sculling, turns to face the camera when near |
| 9 | **Sea stars** (`#D9673A`, `#B54A6A`) | 20 / 10 | Clearing and reef base | Static, with color and size variation |
| 10 | **Sea urchins** (`#2A1F33` with violet spikes) | 40 / 20 | Rock crevices | Static, with a subtle spine sway |

### 1.5 Hero location: the wreck
- **Ship:** wooden sailing hull about 24 m long. Build it by sweeping a U-shaped cross-section along the keel with a bow/stern taper. Roll it about 18° and pitch it 6°, and sink about 40% of it into a raised sand mound.
  - One side has a missing hull section that shows the ribs.
  - The deck is instanced planks with gaps and a collapsed middle section.
  - A broken mast lies in the sand beside it. Add a brass bell or fittings (`#8C7A3E`).
  - Coral, sea fans and algae (instanced) grow on the deck rail, the hull and the mast.
- **Light through the broken deck:** 3–5 narrow additive light shafts angled through the deck gaps into the hold, plus a faint warm-teal interior glow, so the hold reads as lit from above.
- **Bubbles:** about 40 instanced bubbles (one draw call) rising from a hatch and a hull crack. They wobble, speed up as they rise, grow slightly, and respawn at the bottom.
- **Whale skeleton:** about 18 m long and half-buried.
  - Skull about 4 m, flattened and elongated.
  - About 40 instanced vertebrae along a gently curving spine.
  - 13 curved rib pairs (low-segment tubes), some broken or fallen, some sunk into the sand.
  - Two separated jaw bones lying flat.
  - Algae stains from shader noise. Sea stars and urchins sit on the bones.
- **Glow path:** small bioluminescent spots and a soft ground glow along a curve from the clearing to the ship. They pulse gently in a slow wave traveling toward the ship, drawing the eye there.
- **Level of detail:**
  - `THREE.LOD` for the ship, skeleton and rock clusters (high/low geometry).
  - Creatures beyond 45 m skip animation updates. Small decor beyond 50 m is hidden.
  - Instance counts and segment counts drop on the low tier.
  - Detail is concentrated near the camera and sparse far away.

### 1.6 Camera
- **Free swim:**
  - WASD moves in the look direction (swim where you look). Space = up, Shift/C = down.
  - Water feel: acceleration plus drag, max 4 m/s, gentle bob.
  - Drag the mouse to look, with smoothing and pitch clamped to ±80°.
  - **Touch:** a virtual joystick on the left half and drag-to-look on the right half.
  - **Collision:** stay at least 1.2 m above the heightfield, below y = −0.5, inside radius 90, and outside the ship and skeleton bounding volumes (soft push-out).
- **30-second "Start dive" tour** (Phase 5): one smooth `CatmullRomCurve3` (centripetal), sampled by arc length with `getPointAt`.
  - Timing is eased with smootherstep on each segment so there are no jerks.
  - The look target is a point further along the curve, damped with a quaternion slerp. No sudden turns, roll no more than 5°.

| Time | Position (x, y, z) | Beat |
|---|---|---|
| 0 s | (−55, −3, −30) | Just under the surface in the kelp canopy, looking down |
| 6 s | (−40, −12, −15) | Sinking down through the kelp stalks |
| 11 s | (−15, −10, −25) | Leaving the kelp; **both mantas glide across the frame** (re-time their curves at tour start so they're there at 10–14 s) |
| 15 s | (5, −13, 0) | Over the sand clearing, eels, caustics |
| 19 s | (20, −16, 18) | Along the glow path, past the whale skeleton |
| 22–26 s | orbit (46, −14, 24) → (44, −12, 42) | Slow arc around the ship, looking at the lit deck and bubbles |
| 30 s | (24, −2, 34) | Rising, pitching up toward the sun and surface light |

---

## 2. Build phases

### Phase 1: Foundation and environment
1. Renderer, quality tiers (pick the starting tier from `pointer: coarse` + `hardwareConcurrency` + `?q=`). Then adapt at runtime: if the average frame time is over 22 ms for 2 s, drop pixel ratio, then particles, then instance counts.
2. Background gradient, fog, the water-color shader chunk, caustics chunk, lights.
3. Seabed heightfield with zones, vertex-color blending (sand/rock/moss), and sand ripples. Make the ripples as a normal perturbation in the fragment shader, masked to sand areas.
4. Rocks (displaced icosahedrons, instanced), coral heads, sea fans and anemone clusters (instanced, swaying), reef layout.
5. Kelp. Each stalk is a segmented ribbon with blades, all in one InstancedMesh. Sway in the vertex shader grows with height² and is driven by the global current, with per-stalk phase. Blade tips are backlit.
6. Sun rays, surface plane with Snell's window, floating particles.
7. Free-swim controls (keyboard, mouse, touch) and collision.
8. Debug overlay and `?shot=` presets.

**Checkpoint A:** stop. Report the file size and the draw calls/triangles at `?shot=kelp` and `?shot=clearing`.

### Phase 2: Sea life
Add all 10 animals from 1.4. Leave everything from Phase 1 exactly the same.
- **Fish geometry:** a low-poly custom BufferGeometry, about 40–80 triangles. The tail bends in the vertex shader: `offset.x = sin(t*freq + phase - z*k) * amp * smoothstep(head, tail, z)²`. Tail frequency follows swim speed.
- **Boids:** spatial hash for neighbor lookup. Update at 30 Hz if needed and interpolate positions.

**Checkpoint B:** stop and tell me which animal looks the least real and exactly how you would fix it (geometry, motion, shading). Don't apply the fix until I say so.

### Phase 3: Hero location
Add the ship, light shafts through the deck, bubbles, whale skeleton, glow path and LOD from 1.5. Leave everything else the same. Sculpt the sand mound into the heightfield under the ship only.

**Checkpoint C:** stop. Report draw calls/triangles at `?shot=ship` on both tiers.

### Phase 4: Filmmaker realism pass
1. **Take screenshots.** If Playwright is available (or you can install it with `npx playwright install chromium`), capture `?shot=kelp`, `reef`, `clearing`, `ship` and `skeleton` at 1280×720 in headless Chromium (`--use-angle=swiftshader` or `--use-gl=swiftshader`), and look at them yourself. If you can't, ask me for screenshots and wait.
2. Look at them as an underwater cinematographer would. **List the 5 things that look most fake** (color, light, fog, movement, scale), worst first, with the reason for each.
3. **Fix only those 5.** Leave everything else the same. Also check these two must-haves and include them in the 5 if they're weak:
   - The light must ripple on the sand like real sunlight through water: sharp, bright filaments that move and warp, not a sliding texture.
   - Far things must fade into blue-teal with red lost first, never gray.
4. Re-capture the same shots and compare before and after.

**Checkpoint D:** stop. List what you'd fix next, in priority order (top 5).

### Phase 5: Sound and "Start dive"
1. **Audio:** Web Audio API only, no files. A master gain with a 3 s fade-in feeds a generated-impulse convolver reverb (decaying noise, about 4 s).
   - **Low hum:** brown noise through a lowpass (about 180 Hz), plus two detuned sine oscillators (about 48 and 55 Hz). A slow LFO moves volume and cutoff.
   - **Soft bubbles:** random clusters every 2–7 s. Short sine blips (40–90 ms) with an upward pitch sweep (400→1200 Hz), through a bandpass, panned randomly. A little louder when near the ship.
   - **Distant whale call:** every 25–45 s. A sine/triangle pair gliding 180→110→140 Hz over 4–6 s with slow vibrato, a lowpass at about 600 Hz, heavy reverb send, low volume, random panning.
2. **UI:** a minimal overlay with the title, a **"Start dive"** button and a "Free swim" button, plus small control hints. Translucent dark glass style, system font.
   - "Start dive" unlocks or resumes the AudioContext (browsers block sound until a click), hides all UI and runs the 30 s tour from 1.6.
   - Tap, click or Esc skips the tour. When the tour ends, hand control to free swim smoothly from the current pose and fade the UI back in, with a small mute toggle.

**Checkpoint E:** stop.

### Phase 6: Final pass
- Run a performance check on both tiers and confirm the budgets from section 0.
- Confirm there are no console errors and that the window resizes and rotates correctly on mobile.
- Report:
  - final file size
  - draw calls and triangles per `?shot`
  - known limitations
  - your top 3 next upgrades

---

## 3. Quality bar
This should look like a frame from a nature documentary, not a game demo. Favor soft, layered depth (foreground, mid, background), restrained color, slow motion and strong light direction from above. When unsure, add less but do it better.