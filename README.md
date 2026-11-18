# The Wreck

A procedural 3D underwater world in a single `index.html`: a kelp forest, a rocky reef, a sand
clearing, a half-buried whale skeleton and a sunken sailing ship at the end of a glowing trail,
with ten kinds of sea life, Web Audio ambience and a 30-second guided dive.

Everything — geometry, textures, caustics, sound — is generated in code with Three.js (r170,
loaded from a CDN). There are no asset files.

## Run it

Serve the folder with any static server and open `index.html`, for example:

```sh
python3 -m http.server 8000
# then open http://localhost:8000
```

## Controls

| Input | Action |
|---|---|
| W A S D | Swim where you look |
| Space / Shift or C | Rise / sink |
| Mouse drag | Look |
| Touch | Left half: joystick · right half: drag to look |
| Tap, click or Esc | Skip the tour |

## URL parameters

| Parameter | Effect |
|---|---|
| `?debug=1` | FPS, draw calls, triangles, quality tier, camera position |
| `?shot=kelp\|reef\|clearing\|ship\|skeleton\|surface` | Fixed camera preset for screenshots |
| `?q=low\|high` | Force a quality tier |
| `?tour=1` | Start the guided dive immediately (no audio) |
| `?cam=x,y,z,tx,ty,tz` | Place the camera (debug) |
| `?focus=<animal>&fd=<distance>` | Follow a creature (debug): `school`, `manta`, `turtle`, `yellow`, `blue`, `jelly`, `eel`, `grouper`, `star`, `urchin` |

## Files

- `index.html` — the whole experience (sections: CONFIG, QUALITY, NOISE/UTILS, SHADER CHUNKS,
  ENVIRONMENT, LOCATIONS, CREATURES, CONTROLS, TOUR, AUDIO, UI, LOOP)
- `Prompt.md` — the build brief
- `CHECKPOINTS.md` — phase-by-phase build log with measurements
