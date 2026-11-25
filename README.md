
# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

**사계 지휘자 (Four Seasons Conductor)** — a media-studio final project. The player conducts Vivaldi's Four Seasons in front of a webcam: the right hand beats time, the left hand's height sets the volume. Everything lives in **one file, `index.threejs.html`** (the Three.js version, ~1500 lines of inline JS) plus the GLB models in `assets/`. There is no build step, package manager, or test suite. The game UI and its comments are in Korean.

- `index.backup.html` is the original, unmodified version from before the cleanup and refactor. Use it only for reference or to diff against. Never edit it.
- `index.threejs.html` was named `index.html` until the Three.js switch; open it as `http://localhost:8000/index.threejs.html` (the bare `/` URL now shows a file list).
- `index.webgl.html` is the last version with the old hand-written WebGL renderer, kept as a fallback after the Three.js switch. Don't edit it.
- External dependencies:
  - MediaPipe Pose, pinned to `@mediapipe/pose@0.5.1675469404` in both the `<script src>` and `locateFile`. Keep the two versions in sync.
  - Three.js `0.170.0` from jsdelivr, through the `<script type="importmap">` (`three` and `three/addons/`) and the DRACO decoder path in the GLB loader. Keep all three in sync. The main script is a `<script type="module">` (strict mode) wrapping the same IIFE.

## Running and testing

The webcam needs a secure origin, so open the file over http instead of `file://`.

- `python3 -m http.server` run inside this Desktop folder fails with a `PermissionError` (os.getcwd). Copy the file to a scratch or tmp directory and serve it from there:
  ```bash
  python3 -m http.server 8765 --bind 127.0.0.1
  ```
- **Debug API:** `index.threejs.html#debug` exposes `window.dbg`, defined on the line near the end of the file:
  - `x(code)` evals code in the closure.
  - `st()` returns `state`; `s()` returns `{pos, state, bpm}`.
  - `g()` returns gesture stats.
  - `pose(results)` injects fake MediaPipe results; it also sets `camOn`.
  - `forceCalib()` opens calibration.
  - `jump(beat)`, `look(deg)` and `calibStep()` are also available.
- **Simulation:** to drive whole songs without a camera, use `dbg.pose(...)` with synthetic landmarks, or the keyboard fallback:
  - Enter or J = beat
  - F = accent
  - C = cut-off
  - B = both hands up
  - Space = pause / resume (also the ⏸ button at the top right)
- **Headless caveat:** if the Browser pane is hidden, rAF is throttled and songs stall. Call `finish()` via `dbg.x` to check the end-of-song logic. To keep a song moving, tick the loop by hand: `dbg.x('loop(performance.now())')` every ~30 ms.
- **Unverifiable here:** there is no node, so syntax is checked by loading the page and reading the console. Real webcam behavior and actual sound quality can't be verified; say so when reporting.

## Architecture (all inside one IIFE in `index.threejs.html`)

- **Rendering:** Three.js, kept behind the old immediate-mode call style.
  - `Geo` still builds the meshes with `M4` matrices; `upload()` returns a registered `THREE.Mesh` (MeshStandardMaterial, vertex colours converted to linear). Spheres, `seg` tubes and cylinder sides get smooth normals (`Geo.smooth`, one group per call); boxes stay flat. `hex()` accepts `#333` short colours.
  - Every frame the loop hides all meshes in `ALL`; each `drawMesh(m, model, {mul, tint, unlit, glow, point})` shows one, sets its matrix, `mul` → material colour, `tint` → emissive. `unlit≥.5` swaps in a MeshBasicMaterial; `glow>1` pushes it past white so `UnrealBloomPass` (threshold 1.35) makes it shine (sun, bolts, rings, stars).
  - Lights: hemisphere + key/fill directionals + three stage `spots` (`SPOT` strengths, the centre one casts shadows), all scaled by `dimNow`; `flashL` is the lightning flash. Fog is `scene.fog`.
  - Dynamic buffers (`dyn('pts'|'lines'|'mesh')` + `fillDyn`) carry petals/snow/stars (round `DOT` sprite), rain lines and the mittens. Mittens live in `ovScene`/`ovCam`, drawn by a second RenderPass that clears depth.
  - `composer`: RenderPass → mitten pass → bloom → OutputPass (ACES tone mapping). `setQuality(q)` drops shadows (q 1) then bloom (q 0) when the real frame rate stays under 28 for 3 s; a throttled hidden pane can trip it, so call `dbg.x('setQuality(2)')` when testing.
  - `PV` for `project()` (DOM labels) is built from the Three camera each frame.
  - Optional Blender models in `assets/` (Draco GLB, metres, Y-up; Blender -Y is the game's +Z/front). `loadGLB` tries each file once; a missing file or object keeps the `Geo` mesh. `glbGroup` clones (SkeletonUtils) and registers it as a `THREE.Group`, which `drawMesh` handles by multiplying `mul` into each material's saved `base` colour and setting `tint` as emissive.
    - `musicians.glb`: one top-level object per kind (`violin`, `flute`, `horn`, `harp`, `timpani`, `cello`), origin on the floor, placed and rotated by the game. Style: simple avatar look (egg head, flat face features) in black concert suits. Materials named `skin`/`hair`/`suit` are recoloured per seed from `SKIN_GLB` (light→dark, mixed ethnicities), `HAIR_GLB` and `SUIT` (near-blacks).
    - Variants: objects named `<kind>_<group>_v<n>` (n = 1–2 digits) are alternatives; `pickVariants` keeps one per group, chosen by `hsh(seed, group)` so groups mix independently. Groups: `eyes` 6, `brows` 6, `nose` 4, `mouth` 6, `hair` 10 (one bald empty), `extra` 10 (v0 glasses, v1 mustache, v2 beard, v3 blush, v4 goatee, v5–9 empty, so glasses stay rare), `collar` 3 (bow tie, necktie, none). Empties are fine as "nothing". Give mesh data the object's name: a `.001` data name becomes a child `…_v0001`. Per player the game also scales the body (width .9–1.1, height .94–1.06) and the `<kind>_head` bone (.9–1.08); don't name a mesh object the same as a bone.
    - The Blender source for all four GLBs is `assets/musicians.blend` (collections `musicians`, `hall`, `audience`, `props`); its text block `lib` holds the builders (`concert_body(kind, c, m, armpose, standing)`, `eyes_v`…, `keyclip`, `sector`, `emat`, `G()` = game→Blender coords `(x, -z, y)`). Before exporting musicians, non-variant meshes on the same bone are joined into `<kind>_part_<bone>` (234 → 42 meshes, about 625 draw calls in game); keep variants separate. A frame costs about 5 ms with shadows and bloom on the dev Mac.
    - Emissive materials (hall sconces/chandeliers, exit signs, baton tip) keep their glow: `glbGroup` saves `baseE` and `drawMesh` adds `tint` on top; meshes with emissive intensity > 1 don't cast shadows. In Blender scripts, iterate `list(collection.all_objects)`: changing `hide_render` mid-loop cuts the live iterator short. Clips: `play_<kind>` is exactly one beat long and is scrubbed to `pos % 1` (weight .35 + .65·part pulse while playing); `idle_<kind>` or `idle` runs freely otherwise. With a mixer the old bob/sway is halved.
    - `hall.glb`: object `hall`, in world position, matching `buildHall`/`TIER` sizes (tiers y 0/.32/.64, radii 1.6–5/5–7.2/7.2–9.3, wall panels r 10.2, ceiling y 8.2, side walls x ±11, podium (0,.12,.35)); leave the proscenium (`buildFrame`, `ST`) area free.
    - `audience.glb`: `audience_row0`…`audience_row6` (row r at z 3.4+1.15r, y .32r, facing -Z) and `audience_wall`, in world position; rows bob separately when clapping.
    - `props.glb`: `baton`, 1 unit long along +Y from the grip; `drawMitten` stretches it from the grip to the tip in `ovScene`.
    - `backdrops.glb`: pop-up-book season backdrops `bd_spring`/`bd_summer`/`bd_autumn`/`bd_winter` (rehearsal keeps the code backdrop), flat paper cut-outs layered between z −10.03 (sky) and −9.66 (front), in world position. Ground layers are lifted by `DY`=.9 because the orchestra hides the lowest metre. Static cut-outs are joined into `<pre>_static`; moving ones are separate objects with custom properties (exported as glTF extras → `userData`): `anim` (drift, fly, run, bob, hop, sway, slide, flow, rise, dance, bloom), `speed`, `amp`, optional `phase`, and `idx` for blooms. `animBackdrop` moves them each frame (dance follows the beat `pos`; sway grows with `windNow`); spring blooms (`anim:'bloom'`) open with `health` in `idx` order and replace the Geo `BLOOMS`. Sun, bolts and the health tint stay in code. Export this collection with `use_active_collection` (selection export leaked other collections' objects in).
  - First-person arms (`drawMitten`): black tailcoat sleeve, white cuff and a round skin-tone hand; the left cuff turns warm/cool with the volume.
    - Test GLBs (boxes + a node-rotation clip) can be generated with a small Python glTF writer to check these hooks without Blender.
  - `buildHall` and `musician` build the concert hall and the players.
  - `buildScene(o)` builds the per-season backdrop and supports `bend` (leaning trees) and `clouds2`. `buildBolts()` makes the summer lightning.
  - A 2D canvas HUD (the "보면대", or music stand) is drawn by `drawStand`.
    - Left side: the left-hand volume meter. The allowed range is one pair of green dashed lines at target ±`TOL` (0.18) with the dynamic letter (p, f…) beside it, "▲크게/▼작게" labels, and a small "곧 ⬆/⬇" note before the next dynamic.
    - Right side: the beat highway.
- **Music:** a WebAudio synth with no audio files.
  - Building blocks: `osc`, `noiseHit`, convolver reverb, and an HRTF panner per part.
  - Note data: `MIDI_NOTES[season].ev` is a flat array of `[beatMs, track, midi, durMs]` quads. Tracks are 0 solo, 1 vln I, 2 vln II, 3 viola, 4 cello.
  - `midiEvents()` turns the notes into scheduled events.
    - It dedupes unison notes by `beat|midi`, folding the doubled note's volume into its twin instead. Without this, autumn sounded off.
    - Each track's instrument comes from the default `VOICE` mapping, unless the current section supplies its own `map`.
  - `twinkleMusic` is the rehearsal tune (반짝반짝 작은 별).
  - Season themes: in every season each orchestra part stands for one piece of nature (`SEASONS.x.names`) and plays that sound from its seat, for example summer `sun`=폭염 plays `cicada`. The full table is in the comment above `SEASONS.spring.sfx`.
  - Season sound effects: each season has `SEASONS.x.sfx(push)`, mixed into the events in beats by `midiMusic` (and by `springMusic` for spring). `accEv` swaps the accent sound (autumn `gun`, winter `stomp`). The effect kinds live in `play()` and are built from `tick` (a filtered noise click), `noiseHit` and `osc`. An event with `fx:n` (optional `col`) calls `burst()` when it sounds; `crackle` warms the backdrop through `heat`.
  - Ambience: one looping noise (`ambGain`/`ambFilter`, formerly `rainGain`), set each frame from `SEASONS.x.amb(sectionType)`.
- **Season config:** each season is a `SEASONS[key]` entry, plus an `Object.assign(SEASONS.x, {...})` block placed after `MIDI_NOTES`.
  - Fields: `sections`, `dyn`, `holds`, `accents`, `both`, `focus`, `bpm`, `total` and `mv`.
  - `sc(fromBeat, toBeat, type, label, {bpm, map})` takes **beats**; it stores bars internally, with `BPB=4`.
  - `applySeason` sets `CUT = total` and `TOTAL_BARS = total/BPB`.
  - Winter's MIDI data was hand-reordered: movement 1 runs to beat 80, then the movement 2 Largo, then an added cadence, for a total of 100.
- **Pose → gestures:**
  - `onPose` is the entry point and `processRight` handles the right hand. Landmarks used: 0 nose, 11/12 shoulders, 15/16 wrists. The display is mirrored; the subject's left wrist is 15.
  - Everything is normalized by `shoulderW`. Left-hand height is `leftH=(shoulderY-wrist.y)/shoulderW`, then `vol=clamp((leftH-LC.lo)/(LC.hi-LC.lo))`.
  - A beat is the turning point of a down-then-up wrist motion with amplitude above `TUNE.beatAmp`. A much larger amplitude counts as an accent.
  - `beat()` judges the timing: Perfect <.16 and Good <.3 beat off the nearest integer beat. Before judging, it subtracts the measured camera lag (`TUNE.lag`).
  - The face is only required to be visible. It is used for the final bow and the 🙌 both-hands-up gesture.
  - Head turn: `headYawRaw = asin(noseOffset / .3)`, where `noseOffset` is the nose's sideways distance from the shoulder midpoint in shoulder widths (sign flipped for the mirror). `updateBow` turns to the audience when it passes 40°.
  - `camLoop` switches to `modelComplexity:0` (lite mode) once if fps < 18 after 5 seconds.
- **Calibration** (`openCalib` / `setStep` / `updateCalib` / `closeCalib`):
  - Step 1: face, both shoulders and both wrists must be visible.
  - Step 2: left-hand range (hand up high, then all the way down), which goes to `LC.hi` / `LC.lo`.
  - Step 3: left-hand practice: hold still for ◠ (fermata), then two quick drops for ✕ (cut-off).
  - Step 4: the player strokes 8 times to a 0.75s metronome (`startMetro`). The median delay goes to `TUNE.lag` and the median amplitude to `baseAmp` and `beatAmp`.
  - The rehearsal path (`alignOnly`) is step 1 → step 4 → rehearsal.
  - Persistence: `localStorage` keys `leftCal` hold `LC`, and `gestureTune` holds `beatAmp`, `lag` and `baseAmp`.
- **Rehearsal:** `REH_STEPS`, `rehTick` and `rehGesture` run it, and `drawRehGuide` draws the guide hands. The guide hands use **body-relative** positions with the same `LC` math as the meter, so the guide and the stand agree. Keep those two in sync if you change either. While a lesson is paused, the stand's meter target comes from `meterTarget`, which returns that step's height (top 1, bottom 0, hold .5, cut .6, both 1) so the meter and the card point at the same place.
- **Weather:** `SEASONS.x.fx.rate()` gives particles per frame (it can be above 1: winter snow is 2.5–6). About 30% spawn right in front of the camera as `big` and are drawn larger (`nearBuf`). `SEASONS.x.wind(sectionType)` blows particles and rain sideways (`windNow`). Rain lines (`drops`) cover the whole stage in summer and in spring's thunderstorm.
- **Spring rule, "서두르지 마":** in `beat()`, striking early (`fr < -.16`) costs health and drops brown petals. The blossoms on the spring backdrop are separate meshes (`BLOOMS`), and how many are open follows `health`. `SEASONS.x.meter` renames the health bar (꽃 / 여름 / 나뭇잎 / 눈).
- **Flow:** `start` → play loop (`loop`, `schedule`, `onGesture`) → `finish`.
  - Pause (`togglePause`): sets `paused` and calls `ac.suspend()`, so the scheduled notes and the audio clock freeze together. The loop's play block and `onGesture` skip while paused. Resuming resets `lastBeatAt`/`beats` so the pause doesn't count as a stop.
  - `backToSeasons` (the 🍂 계절 선택 button, also in the pause card) mutes the buses, closes calibration and rehearsal, and calls `showSeasonPick`. There is no recalibrate button any more; calibration runs when the webcam is first turned on.
  - `finish` computes the result tier from `health`. Summer also requires a cut-off; without one, the tier drops.
  - After `finish` comes the bow ending: `beginBow`, `doBow`, `applause`.

## Conventions

- The code style is dense and minified-like: short names and one-line functions. Match it rather than reformatting.
- Large mechanical edits, such as reordering MIDI data or removing dead code, were done with small Python scripts in the scratchpad rather than with hand edits.
