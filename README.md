# Gausseous

**[Live demo →](https://quixbrics.github.io/gausseous/)** (Chrome or Edge for video export)

A bespoke 3D Gaussian Splatting manipulator and animator for the browser.
Arbitrary per-Gaussian programmable manipulation via Spark's `dyno` shader-graph
system, driven by keyframes and by an audio track.

Single file, no build step, no npm install. Dependencies are CDN imports via an
importmap: [three](https://threejs.org) 0.180,
[Spark](https://sparkjs.dev) 2.1.0, mp4-muxer 5.x.

> Formerly *Splatification / Splat Bench*. Presets written under the old name
> still load — the old `splat-bench-preset` format tag is accepted alongside the
> current one.

## Run

ES modules will not load over `file://` — it must be served.

```bash
python3 -m http.server
```

Then open <http://localhost:8000/> and click **Load test asset**.

Chrome or Edge is required for video export (WebCodecs `VideoEncoder`).

Because the whole app is one cached HTML file, use a hard reload (Cmd+Shift+R)
after editing, or serve with no-store headers:

```bash
python3 -c "import http.server; H=http.server.SimpleHTTPRequestHandler; _e=H.end_headers; H.end_headers=lambda s:(s.send_header('Cache-Control','no-store'),_e(s)); http.server.test(HandlerClass=H,port=8000)"
```

## First session

1. Load a splat (drop it on the viewport, **Choose file**, or **Load test asset**).
   To swap it later, use **Load splat…** in Scene → Splat, or drop another file.
2. A **Next** card offers audio (**Test audio** loads `bandtest.wav`). Loading
   audio sets the timeline to the track's length and loops it.
3. Press **Play**. To make more things move with the sound, arm a band under
   **Map** in the transport bar and click any slider.

Dismiss the card with ×; the same things live in the transport bar.

## Layout

- **Transport** (bottom): the scrub bar with the track's waveform behind it;
  Play, the audio file, the **Map** band chips, the clock, **Length** and Render.
- **Lane** (above the transport): the focused parameter's keyframe curve, its
  audio mapping, mask and gesture binding. The waveform is drawn faintly behind
  the curve.
- **Rail** (right): five tabs.

| Tab | Sections |
|---|---|
| **Scene** | Splat · Camera · Transform · Opacity · Masks |
| **Look** | Background · Points · Lights · Colour · Depth · Optics · Film · Frame · Grain |
| **Motion** | Noise · Fracture · Quantise · Wave · Smear |
| **Audio** | Meters · Mappings |
| **Export** | Output · Preset · Viewport |

A tab shows a count when parameters inside it are driven by keyframes or
audio, and a dot when something is changed from its default.

## Timeline length

**Length** in the transport bar replaces the old Duration slider (Camera) and
the Loop / “× set” controls (Audio):

- **Seconds** — set the length directly.
- **Track ×** — a whole number of track lengths, so the audio loops cleanly.
  This is what a first audio load picks.
- **Bars** — bars × beats at a BPM (typed or tapped), with a beat grid on the
  lane and scrub bar and the bar.beat position on the clock. Tempo is entered
  rather than detected: detection on field recordings is unreliable.

**Loop** repeats a track that is shorter than the timeline, in the preview, the
envelope lookup and the exported audio alike. Camera Orbit turns and the light
rig's Orbit both count per timeline loop, so whole numbers are seamless.

Preview playback uses Web Audio: the decoded track loops sample-accurately and
each timeline pass is scheduled on the audio clock ahead of time, so loops are
gapless in the preview as well as the export. While audio is playing, the
playhead follows the audio clock, so the picture cannot drift from the sound.

## Audio

The file is analysed offline into band envelopes rather than sampled live, so
scrubbing, playback and export all agree. Four sources:

- **Low / Mid / High** — loudness in 20–200 Hz, 200 Hz–2 kHz, 2–16 kHz.
- **Hit** — a short pulse on each detected onset (drum, pluck, transient),
  whatever its frequency. Material without clear transients gives few hits.

**Mapping.** Arm a band under **Map** (transport bar), then click any slider in
the rail — the slider does not move, it gains a coloured band tag. Click a
mapped slider again to unmap; **Esc** finishes. Every mapping is listed under
**Audio → Mappings** with its depth, smoothing and a remove button; clicking a
name jumps to the control. Depth is a fraction of the control's whole range
(negative pushes it down). Audio adds on top of the slider and any keyframes.

**Smoothing presets** — Punchy, Smooth, Swell, Gate — set attack, release,
gain, floor and curve together. The lane's **Shape ▾** still exposes every
field, plus lag (negative lag lands an effect on the beat rather than after it).

The old way in still works: click a parameter's name to focus it in the lane
and pick a band there.

## Effects

Most effects are a single `dyno.dynoBlock` running in one GPU pass as an
`objectModifier` on the `SplatMesh`:

- **Points** — the point-cloud morph (from Fractured / FX Bench): blends every
  splat toward one uniform dot. **Point size** is log-mapped (FX Bench's
  measured range); **Size jitter** adds stable per-splat variation; **Antialias
  floor** is Spark's `blurAmount`, lowered to keep sub-pixel dots alive.
  Maskable. The dots are still Gaussians, so small ones thin and darken the
  image — compensate with Opacity × all, deliberately not automatically.
- **Mesh** (in Points) — a coarse wire mesh through a thinned set of splat
  centres, meant to sit over the point cloud. **Coarseness** sets the grid the
  centres are thinned into (one node per cell, the most opaque splat in it).
  Topology: **Nearest neighbours** (organic, roughly triangulated), **Skeleton**
  (minimum spanning tree — branching, no loops) or **Lattice** (neighbouring grid
  cells). Built once (tens of ms on the test asset), drawn in one pass, so it costs
  almost nothing per frame; every look control is a shader value. Colour: white,
  mono, **two-tone** (hue or its complement per link), capture colour, or black
  (Normal blend). **Taper** (needle / spindle), **Dotted** with **Dot length** and
  **Dot travel** (per loop), node dots, and a **Depth cue** (fog, or x-ray when
  negative). **Reveal** grows the mesh outward from the pivot, top, bottom or at
  random — each link draws on as it arrives — and is deterministic, so it can be
  keyed or mapped to audio and exports as previewed. The mesh sits at the splats'
  rest positions: Noise, Quantise and Wave move the points, not the mesh.
- **Fracture** (Motion tab) — Voronoi-style cells from FX Bench: the capture is
  cut into chunks on a grid (keyed to rest positions, so a chunk stays a chunk)
  and each chunk moves as a whole along its own direction and phase. Maskable,
  and swept by the Wave like Noise.
- **Noise** — `sin()` or curl field over swizzled center × per-splat random direction
- **Opacity** — per-splat cull by minimum opacity and maximum scale
- **Colour** — brightness, RGB tint, saturation
- **Masks** — a per-splat SDF edit: cut, crop or colorize inside a
  sphere / box / ellipsoid / cylinder / capsule / plane, in six slots. Noise /
  Saturation / Tint / Opacity / Point cloud can be scoped to a slot via the
  lane's **Mask** control.
- **Quantise**, **Wave** — voxel snap and index-phase sweep

**Lights** (from FX Bench, phases Q–T) run in Spark's splat vertex shader, not
the dyno graph, so they move every frame without re-baking. Three point lights
are *added* to the capture: **Darken capture** pulls the baked light down,
**Glow** blends between lighting the splats and glowing regardless of what they
fall on. Each light has a band chip in its header (its intensity follows that
band), **Place** (then click the model), and a draggable numbered marker on the
viewport while the section is open. **Shading** turns on facing-dependent light
using surface directions estimated from neighbouring splat centres in a
background worker (a few seconds on large captures); **Rim** adds edge light.
**Orbit** turns the whole rig per timeline loop. Limits: it cannot remove light
baked into a capture, so bright overcast scans respond weakly; shading only
helps on flat-ish surfaces; masks cannot scope lights; strong lights clip
before the Film pass. Amount 0 is an exact bypass.

**Depth** and **Smear** also run in Spark's vertex shader. Depth carries depth
fade (in the dyno pass) and depth of field (per-splat circle-of-confusion
blur). Smear stretches each splat's projected shape along a screen-space
direction using the same covariance-plus-energy-correction machinery, so the
two compose through a single alpha correction.

**Background** (top of the Look tab) — Void, Black, White, Mid grey, Chroma
green, or **Hue**, whose hue / saturation / brightness can be keyed or mapped to
audio. Depth fade fades toward it. **On hits**: *step hue* (each detected hit
moves the hue on by **Hit hue shift** and holds it until the next), *black ↔
white* (alternates on every hit), or *flash* (jumps toward a shifted hue and fades
over **Flash fade**). Hits are counted from the analysed track, so a held colour
scrubs and exports exactly. **CRT** (in Frame) — scanlines counted per
frame height, so exports match the preview at any resolution, plus a colour
fringe that grows toward the edges.

**Adjust** binds up to two parameters to the drag axes so you can change them on
the viewport itself, with the splat visible. Focus a parameter, bind X or Y in
the lane header, then drag. Each axis takes an optional sub-range and invert.

Every parameter has a `↺` reset button, which brightens when the value differs
from its default, and a `◆` keyframe button.

## Animation

Each parameter carries its own envelope — a keyframe list over the normalised
0..1 timeline, with smooth or linear interpolation. Click `◆` on a row to key it
at the playhead and focus it in the curve lane; drag keys in the lane, or drag
the slider itself to auto-key. Armed sliders become live readouts of their
curve. Exports honour envelopes.

**Camera keys** (Scene → Camera, Keyframes) are a separate row of diamond
ticks above the lane: **Add key** stores the current view at the playhead, and
ticks drag to retime. Between keys:

- **Linear / Smooth / Orbit** sets the path shape. *Orbit* swings around
  the target instead of cutting across, so the distance to it eases from
  key to key rather than dipping between them. It turns the short way
  round; to go the long way, add a key in between.
- **Ease / Flow** sets the timing. *Ease* settles at every key, by the
  **Ease** slider. *Flow* keeps the camera's speed through keys, so a
  multi-key move plays as one glide.
- **Arrival** shapes how the camera lands on the key at the playhead. Use
  `◀ key` / `key ▶` to step between keys, drag the two handles, or pick
  Smooth, Fast, Linear, Ease in or Ease out. **Auto** goes back to the path's
  timing. Shaped keys show a ring on their tick. A shaped key is still a
  landing in Flow, and the camera eases into and out of it.
- **Snap** (Hits, Beats, Both) pulls a dragged tick onto the nearest audio
  hit or beat within 8 px. Hold Shift to place it freely. Hits show as faint
  marks on the row. Beats need Length in Bars mode. `Alt + ,` / `Alt + .`
  (⌥ on a Mac) jump the playhead to the previous / next hit, for "go to the
  hit, Add key". Snap is remembered in the browser, not saved in presets.

**Save JSON** writes the whole editable state — parameters, envelopes, toggles,
camera keys, gesture bindings, audio mappings, timeline length — as a small
text file. The splat and the audio are referenced by name only, not embedded,
so load your capture first and then the preset. Drop a `.json` on the viewport
to load it. Preset format v11; older presets still load, with any features they predate (Points, Lights, Mesh, Fracture, CRT, background hue, camera arrival curves and Flow timing) reset.

Keep the tab visible while exporting — Chrome throttles background timers to
~1s, which slows the render to roughly a frame per second.

## Assets

Splat captures are gitignored, with one deliberate exception: `3_3_2025.ply`
(~11 MB, 44,296 gaussians) is committed as the test asset, so the repo is
runnable as cloned without needing to source a capture separately.
`bandtest.wav` is the test audio.

## Mobile

The rail becomes a bottom sheet on narrow or touch screens, with larger touch
targets. A **Mobile profile** toggle (on by default there) caps the pixel ratio
and forces level-of-detail on load; it caps rather than redefines, and the OSD
reports the pixel ratio actually being rendered. Arming a Map band opens the
sheet. Video export needs WebCodecs, so the Render button is disabled on
browsers without it (including iOS Safari).

## Debugging

Load with `?debug=1` to expose `window.__bench` — a synchronous `draw()`, an
awaitable `settled()`, and live access to the mesh, edit and evaluated
parameters. Rendering is rAF-driven and Chrome stops rAF in a hidden tab, so
this is the only reliable way to measure pixels from an automated harness.
