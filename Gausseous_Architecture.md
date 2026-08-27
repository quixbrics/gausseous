# Gausseous — Architecture

Gausseous is a browser-based manipulator and animator for 3D Gaussian
Splats. It's a single static HTML file with no build step, no bundler,
and no backend — everything runs client-side, in the tab.

## Tech stack

| Layer | Choice |
|---|---|
| Language / runtime | Vanilla JavaScript (ES modules), no framework |
| Delivery | One `.html` file, ~6,400 lines. No build step, no `npm install` |
| Dependencies | Loaded at runtime via an `<script type="importmap">` from CDN — no bundler needed |
| 3D engine | [three.js](https://threejs.org) 0.180 — scene graph, camera, orbit controls, transform gizmo |
| Splat rendering | [Spark](https://sparkjs.dev) 2.1.0 — a 3D Gaussian Splatting renderer built on three.js, including its `dyno` GPU shader-graph system |
| Per-splat effects | Spark's `dyno` — a graph of GPU nodes compiled once into GLSL and driven by uniforms, used as an `objectModifier` on the splat mesh |
| Post-processing | A hand-written fullscreen `THREE.ShaderMaterial` pass |
| Video export | [mp4-muxer](https://github.com/Vanilagy/mp4-muxer) 5.x + the browser's native `WebCodecs` `VideoEncoder` |
| Audio analysis | Web Audio API, analysed offline (not sampled live) |
| Hosting | Static hosting — GitHub Pages, no server component |

Browser requirement: any modern browser renders and edits; video export
needs `WebCodecs`, so that's Chrome or Edge only (not Firefox or Safari).

## How it works

### Loading

A `.ply` / `.spz` / `.splat` / `.ksplat` / `.sog` file is handed to
Spark's `SplatMesh`, which parses it off the main thread in a Web Worker
and populates a GPU-resident buffer of Gaussians (position, scale,
rotation, colour, opacity). Local files are streamed through
`file.stream().getReader()` so the UI can show real byte progress rather
than blocking on the whole file loading into memory first.

### The effects graph

Most parameters — noise displacement, colour, opacity culling, region
masks, quantisation, wave sweep — are nodes in a single `dyno.dynoBlock`
graph attached to the mesh as an `objectModifier`. This graph is compiled
to GLSL **once** and runs entirely on the GPU, once per splat, once per
frame. The key architectural rule: every effect is *always present* in
the graph, gated by a uniform that can go to zero, rather than being
added or removed by rebuilding the graph. That means turning an effect on
or off, or animating it, is just a uniform write — cheap and immediate,
with no shader recompile on the hot path.

Six region "masks" work the same way: each is a signed-distance-field
shape (sphere, box, ellipsoid, cylinder, capsule, or plane) evaluated
per-splat inside the graph, all six always evaluated, selected by index.
A region can cut, crop, or colorize the splats inside it, and most other
effects can be scoped to run only within one region's mask.

### Rasterisation-time effects

Depth of field and depth fade run inside Spark's own vertex shader
(patched at runtime, not re-implemented) since that's where Spark handles
per-splat screen-space projection and the circle-of-confusion blur — a
defocused splat is just a wider, dimmer Gaussian, so it composites
through the existing alpha-blended sort with no special handling.

### Post-processing pass

After the raster pass, a single fullscreen shader handles everything
that needs to see the whole frame at once: vignette, halation, film
grain, a parametric film-emulsion response curve (contrast, highlight
rolloff, saturation, split toning), a colour grade (temperature/tint,
lift/gamma/gain, a second saturation stage), and an optional film-rebate
border. It reads and writes in linear light with an explicit sRGB
decode/encode, so effects composite correctly regardless of what's
already in the framebuffer.

### Parameters, animation, audio

Every adjustable parameter is declared once in a central `SPEC` table
(`[id, display precision, UI group]`), which drives everything else:
slider generation, reset-to-default, per-parameter keyframe envelopes,
audio routing, and preset serialization all read from the same list, so
adding a new parameter never means updating five different places by
hand.

- **Animation**: each parameter can carry its own keyframe envelope over
  a normalised 0–1 timeline, with linear or smooth interpolation.
- **Audio-reactive**: an audio track is analysed **offline** into
  per-band (low/mid/high) envelopes rather than sampled live, so
  scrubbing, playback, and export all agree exactly on what the audio
  was doing at any timeline position. Any parameter can be routed to a
  band with its own attack/release/gain shaping.
- **Camera**: manual orbit, a keyframed path (Catmull-Rom interpolation
  between camera poses), or a generated turntable orbit.

### Export

Rendering an export is a real-time loop through the exact same render
path used for live preview — each frame is rasterised, post-processed,
and handed to `VideoEncoder` (WebCodecs), muxed into an MP4 by
`mp4-muxer`, with the analysed audio track muxed in alongside it.

### Presets

The full editable state — parameter values, envelopes, audio routing,
camera keyframes, toggles — serializes to a small JSON file. The splat
itself is referenced by name, not embedded, so a preset is portable
across captures that share a name.

### Mobile

The control rail becomes a bottom sheet on narrow or touch screens, with
enlarged touch targets, a capped pixel ratio, and forced level-of-detail
loading for large captures. A "gesture" mode lets you bind up to two
parameters to on-canvas drag axes, since the desktop rail would otherwise
cover the subject on a phone. Keyboard shortcuts (spring-loaded
hold-to-adjust, frame stepping, mode switching) are desktop-only.
