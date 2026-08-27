# Gausseous

**[Live demo →](https://quixbrics.github.io/gausseous/)** (Chrome or Edge for video export)

A bespoke 3D Gaussian Splatting manipulator and animator for the browser.
Arbitrary per-Gaussian programmable manipulation via Spark's `dyno` shader-graph
system, with an eye toward parameter automation driven from outside (audio FFT,
LFOs, OSC).

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

## Control groups

The rail is one accordion, in pipeline order — what the splat *is*, then what is
done to it, then how it leaves:

**Camera** · **Transform** · **Opacity** · **Masks** · **Depth** · **Colour** ·
**Noise** · **Quantise** · **Wave** · **Smear** · **Audio** · **Output** ·
**Preset** · **Viewport**

## Effects

Most effects are a single `dyno.dynoBlock` running in one GPU pass as an
`objectModifier` on the `SplatMesh`:

- **Noise** — `sin()` field over swizzled center × per-splat random direction
- **Opacity** — per-splat cull by minimum opacity and maximum scale
- **Colour** — brightness, RGB tint, saturation
- **Masks** — a per-splat SDF edit: cut, crop or colorize inside a
  sphere / box / ellipsoid / cylinder / capsule / plane, in six slots. Any of
  Noise / Saturation / Tint / Opacity can be scoped to a slot via its own
  **Mask** control in the lane.
- **Quantise**, **Wave** — voxel snap and index-phase sweep

**Depth** and **Smear** are different: they run in Spark's own vertex shader at
rasterisation time, not in the dyno graph. Depth carries both depth fade (aerial
perspective, in the dyno pass) and depth of field (per-splat circle-of-confusion
blur). Smear stretches each splat's projected shape along a screen-space
direction using the same covariance-plus-energy-correction machinery, so the two
compose through a single alpha correction.

**Audio** drives any parameter from a track's low / mid / high content. The file is
analysed offline into per-band envelopes rather than sampled live, so scrubbing,
playback and export all agree. Click a parameter's name to focus it, then pick a band
in the lane. Audio layers on top of envelopes rather than replacing them.

**Loop** repeats a track that is shorter than the timeline, in the preview, the
envelope lookup and the exported audio alike. **Duration × N** sets the timeline
to exactly N track lengths, so a loop never lands mid-bar.

**Adjust** binds up to two parameters to the drag axes so you can change them on
the viewport itself, with the splat visible. Focus a parameter, bind X or Y in
the lane header, then drag. Each axis takes an optional sub-range and invert.
Useful on desktop, essential on a phone where the control rail covers the
subject.

Every parameter has a `↺` reset button, which brightens when the value differs
from its default, and a `◆` keyframe button.

## Animation

Each parameter carries its own envelope — a keyframe list over the normalised
0..1 timeline, with smooth or linear interpolation. Click `◆` on a row to key it
at the playhead and focus it in the curve lane; drag keys in the lane, or drag
the slider itself to auto-key. Armed sliders become live readouts of their
curve. Exports honour envelopes.

**Save JSON** writes the whole editable state — parameters, envelopes, toggles,
camera keys, gesture bindings — as a small text file. The splat is referenced by
name only, not embedded, so load your capture first and then the preset. Drop a
`.json` on the viewport to load it.

Keep the tab visible while exporting — Chrome throttles background timers to
~1s, which slows the render to roughly a frame per second.

## Assets

Splat captures are gitignored, with one deliberate exception: `3_3_2025.ply`
(~11 MB, 44,296 gaussians) is committed as the test asset, so the repo is
runnable as cloned without needing to source a capture separately.

## Mobile

The rail becomes a bottom sheet on narrow or touch screens, with larger touch
targets. A **Mobile profile** toggle (on by default there) caps the pixel ratio
and forces level-of-detail on load; it caps rather than redefines, and the OSD
reports the pixel ratio actually being rendered. Video export needs WebCodecs,
so the Render button is disabled on browsers without it (including iOS Safari).

## Debugging

Load with `?debug=1` to expose `window.__bench` — a synchronous `draw()`, an
awaitable `settled()`, and live access to the mesh, edit and evaluated
parameters. Rendering is rAF-driven and Chrome stops rAF in a hidden tab, so
this is the only reliable way to measure pixels from an automated harness.
