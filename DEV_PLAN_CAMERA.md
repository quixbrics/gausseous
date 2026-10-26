# DEV_PLAN_CAMERA.md — Gausseous camera keyframe dynamics

Oct 26, 2026 · @Quixbrics

Written to be handed to a fresh Claude Code session with no memory of the
conversation that produced it. Fork point: `cfdff86` ("Remove the starting
looks"). Line numbers below are approximate as of that commit; search by the
names given, not the numbers.

Source of the ideas: GS_Bed v2 (`../GS_Bed`), specifically
`core/ui/editor.js` (curve editor), `core/camera/planner.js` (`bezier`,
`CURVES`, the monotone Hermite pass in `plan()` and `interpolate()`) and
`core/camera/object-rig.js` (rig-space interpolation). **Rewrite, don't
import.** GS_Bed is ES modules with a different document model; Gausseous is
one file. Copy the maths, not the code structure.

---

## §0 — Read this before touching anything

**Scope rule.** This plan improves how the camera moves between keyframes.
It does not add camera modes, a move palette, a planner, a cue engine, bounds
or audio-driven camera moves. If a step seems to need any of those, stop and
write it up in NOTES instead.

**Behaviour rule.** With default settings, an existing preset must play,
scrub and export **exactly** as it does at `cfdff86`. Every new option is
off by default, or "Auto", which means today's behaviour. Phase 0 builds the
check that enforces this. Do not skip it.

Invariants already in the file that this work must keep (read the comments
at each site before editing):

- `driveCamera()` is the **only** writer of `camera.position`,
  `controls.target` and `camera.fov` outside OrbitControls and `reframe()`.
  It runs every frame **while playing** (the playback branch of `frame()`),
  and on paused frames only on a playhead change, mode change, Reframe,
  preset load, or in the export loop. Nothing in this plan adds a second
  writer or a per-frame call while paused.
- `drawCamRow()` also runs every frame while playing, and it removes and
  recreates every `.camtick` div each time. Anything this plan adds to
  `#camRow` must survive that (don't give it the `camtick` class) and must
  not do expensive work inside `drawCamRow()`.
- `evaluateCamera()` / `sampleCam()` are **pure**. They read state and write
  nothing. Scrub, playback, the dirty indicator (`updateCamDirty`) and the
  export loop all depend on that. Keep every new sampling function pure.
- Camera key `t` is in normalised 0..1 timeline space, like every envelope.
  Do not introduce seconds into stored keys.
- Tick dragging does not rebuild curves on `pointermove`; it catches up on
  drop. Keep that: any cached path data rebuilds in `rebuildCamCurves()`,
  which is called on drop, add, remove, preset load.
- `pushCameraUniform()` ordering (after the camera is positioned, before
  render) is unchanged by this work.

Append a `## Camera dynamics` section to `NOTES.md` per phase in the
existing format: what was built, gate results, traps caught. Update
`README.md` (Animation section) and `Gausseous_Architecture.md` (the Camera
bullet) once at the end.

---

## §1 — What changes, in one page

Today, Keyframes mode samples the path in `sampleCam(playheadT)`: it finds
the two keys around the playhead, eases the local fraction with the global
`V.ease` slider (default 0.8, a smoothstep blend), then reads either a
straight line or a `THREE.CatmullRomCurve3` parametrised by key **index**.
Two consequences:

1. **The camera nearly stops at every key.** Easing is applied per segment,
   so a path through six keys plays as six separate glides.
2. **Every segment has the same character.** One slider for the whole path.

This plan adds, in order:

| Phase | What | Default | Size |
|---|---|---|---|
| 0 | Regression harness: legacy sampler + fixtures | — | small |
| 1 | Per-key **arrival curve** (cubic Bézier) + editor in the Camera section | Auto (= global Ease) | small |
| 2 | **Flow** timing: velocity carries through keys | Ease (= today) | small–moderate |
| 3 | **Snap** tick drags to hits / beats; hit marks on the camera row | Off | small |
| 4 *(optional)* | **Orbit** path shape: interpolate around the target | Off | moderate |

Phase 4 is only done if Phases 1–3, used on real material, don't feel like
enough. Decide after Phase 3, not before.

---

## §2 — Phase 0: the regression harness

Before changing `sampleCam`, copy it **verbatim** (with `ease` and the
Catmull-Rom construction) into a self-contained `sampleCamLegacy(keys,
smooth, easeK, t)` that takes its inputs as arguments instead of reading
globals. It lives in the check harness only, never in the app's camera path.

Add `window.__bench.camCheck()` (next to the existing `__bench` object,
~line 8051) that:

- builds four fixture key sets: 2 keys; 3 keys evenly spaced; 6 keys with
  uneven timing (gaps of 0.02 and 0.4 next to each other); 6 keys circling a
  target at constant distance;
- for each fixture, each `smooth` value, and `easeK` in {0, 0.5, 0.8, 1},
  samples 400 evenly spaced `t` plus every key `t` and `t ± 1e-4`;
- compares the live sampler against `sampleCamLegacy` and returns the max
  absolute difference in pos, tgt and fov.

The live `sampleCam` reads globals (`CAMKEYS`, `camSmooth`, `V.ease`,
`camPosCurve`, `camTgtCurve`, and later the Flow cache and timing/shape
state). So `camCheck()` saves all of them, swaps in each fixture, calls
`rebuildCamCurves()`, samples, and restores everything in a `finally`
(then calls `rebuildCamCurves()` once more). It is synchronous, so nothing
else runs in between. Keep Phase 0 to this swap. Don't refactor `sampleCam`
into a pure helper here: Phase 0 must not change the live code path at all.

`window.__bench` only exists when the `DEBUG` flag is set (see the `if
(DEBUG)` guard around it), so load the app with that flag to run the check.

**Gate 0:** `camCheck()` reports 0 (or ≤ 1e-12) on the unchanged code.
This gate is rerun at the end of every later phase with all new options at
their defaults, and must still pass.

---

## §3 — Phase 1: per-key arrival curves

### Data

A key gains `curve: [x1, y1, x2, y2] | null`. It shapes the segment that
**arrives at** that key (previous key → this key). Arrival rather than
departure, because the musical point of a key is usually where the camera
lands (and Phase 3 snaps keys onto hits). The first key's `curve` is unused.

`null` = Auto = today's `ease(localRaw, V.ease)`. `V.ease` keeps its current
meaning, envelopes and audio routing for every key without a curve.

**Re-adding a key must keep its curve.** `addCamKey()` replaces the whole
record when a key already exists at the playhead (`CAMKEYS[i] = rec`). Carry
the old key's `curve` into `rec` so "move the camera, Add key again" only
updates the pose.

### Maths

Port `bezier()` and `CURVES` from `GS_Bed/core/camera/planner.js`
unchanged: cubic Bézier from (0,0) to (1,1), x solved by 40-step
bisection, y in 0..1 so a curve **cannot overshoot** its end key. Presets:
Smooth `[0.65,0,0.35,1]`, Fast `[0.2,0,0.08,1]`, Linear
`[0.25,0.25,0.75,0.75]`, Ease in `[0.6,0,0.9,0.6]`, Ease out
`[0.1,0.4,0.4,1]`. Clamp handle y to 0..1 on input and on preset load.

In `sampleCam`, replace the one line computing `localU`:

```js
const localU = b.curve ? bezier(b.curve, localRaw) : ease(localRaw, V.ease);
```

Nothing else in the sampler changes in this phase. fov uses the same `localU`.

### UI

- **Selected key = the key at the playhead** (`camKeyIndexAt(playhead)`).
  That is already how Add/Remove key work and how `.onkey` highlights a
  tick; clicking a tick already jumps to it. No new selection state.
- In the Camera section (Scene tab, under the Add/Remove/Smooth row, ~line
  768) add an **Arrival** block: a 200×112 CSS-px canvas curve editor
  (port the drawing and two-handle drag from `GS_Bed/core/ui/editor.js`:
  grid, dashed grey curve for Auto, solid blue for custom, amber handles,
  `setPointerCapture`, devicePixelRatio scaling), preset buttons and
  **Auto**. Dragging a handle on an Auto curve starts from Smooth.
- Disabled with a one-line hint when no key is at the playhead ("Scrub to a
  key to shape how the camera arrives there") or on the first key ("The
  first key has nothing arriving at it").
- Mark ticks whose key has a custom curve (e.g. a small dot or different
  fill), so shaped keys are visible on the camera row.
- **Previous / next key buttons** (◀ key, key ▶) in the Arrival block,
  calling the existing `jumpCamKey(-1 / 1)`. On desktop this saves reaching
  for ⇧ , / ⇧ . or the row. On mobile they are the only practical way to
  reach a key: `#camRow` is hidden at phone width (the `#camRow{
  display:none }` media rule), and scrubbing to within `KEY_EPS` (1e-3) of a
  key by hand is not realistic.
- During a curve drag, do **not** call `driveCamera()`. The selected key is
  the one at the playhead, and at a key the camera is exactly at that key's
  pose whatever its arrival curve is, so the paused view would not change.
  It would only snap away any manual orbit and clear the "off curve"
  marker. The editor's own drawing is the feedback. Do not rebuild anything
  on pointermove either. (If a live preview turns out to be wanted, make it
  an explicit "Preview" button that plays the arriving segment once, and
  write that up in NOTES first.)
- The editor's enabled state and the curve it shows follow the playhead.
  Refresh them wherever the `.onkey` highlight is refreshed (scrub, tick
  click, key jump, add/remove, preset load). During playback, skip the
  canvas redraw unless the key at the playhead has actually changed.
- Mobile: the canvas must work with touch (pointer events) and sit inside
  the bottom sheet without horizontal scroll. Handles get a ≥ 24 px hit
  radius on touch.

### Presets

Bump `PRESET_VERSION` to 11. Save writes `curve` only when non-null. Load
accepts a 4-number array of finite values, clamps y to 0..1 and x to 0..1,
otherwise `null`. A v10-or-earlier preset loads with every curve `null`.

**Gate 1:** Gate 0 still passes. A key set with one custom Fast curve
differs from legacy only inside that one segment (check every sample outside
it is identical). Save → load round-trips curves exactly. Export of a
shaped path matches preview frame for frame (spot-check 3 frames with the
existing export path).

---

## §4 — Phase 2: Flow timing

### What it does

A new path-level **Timing** control beside Smooth/Linear: **Ease** (today)
and **Flow**. In Flow, a segment whose destination key has no custom curve
plays at the raw time fraction, and the camera's velocity is continuous
across keys between such segments. A key with a custom curve keeps its
curve: it is a deliberate landing.

### Separate shape from timing

Flow cannot be had by remapping `u` on the existing `CatmullRomCurve3`,
because that curve is parametrised by key index, not time: the camera's speed
jumps at every key whose neighbouring segments differ in length or duration.
The ease is what currently hides that jump.

So in Flow, sample position, target and fov from **one cubic Hermite per
segment** built in `rebuildCamCurves()`:

- Points `p_i` (and separately `tgt_i`, `fov_i`), key times `t_i`.
- Tangents (per unit of t), **time-scaled Catmull-Rom**:
  `m_i = (p_{i+1} − p_{i−1}) / (t_{i+1} − t_{i−1})`; zero at the first and
  last key (the move starts and ends at rest).
- **Tangent cap** against overshoot when key timing is uneven (a short
  segment beside a long one is the case that loops): for each key, scale
  `m_i` down so that `|m_i| · h ≤ 2 · |chord|` for both the incoming and
  outgoing segment (h = that segment's duration, chord = that segment's
  `p_end − p_start`). Use the smaller scale.
- **Shape mode `Linear` + Flow:** straight segments at constant speed per
  segment (no ease). Speed still steps at keys. Document it; don't try to fix
  it, as that is what Linear means.
- Segment evaluation: `u = localRaw` if the destination key has no curve,
  else `u = bezier(curve, localRaw)`. Then Hermite at `u` with the
  segment's endpoint tangents × h. A curved segment keeps the path's shape
  but is retimed by its curve. That is the GS_Bed rule (custom-curve segments
  don't flow) without flattening the path.
- Tangents at a key shared with a curved segment stay as computed (don't
  zero them). The curve's own end slope decides how the camera arrives and
  leaves. Presets with flat ends (Smooth, Fast) still stop on the key.

Do **not** port GS_Bed's per-axis monotone (harmonic-mean) tangents here.
They suit rig-space axes, but in world XYZ an orbit reverses x and z
constantly, and per-axis monotonicity would zero the velocity at each
reversal: a wobble, the opposite of Flow. They return in Phase 4 where they
fit.

**Exception: fov.** fov is one scalar channel, not a world axis, so that
argument doesn't apply to it. A Catmull-Rom tangent at a key where fov peaks
or dips is non-zero and overshoots past the key's value. Use the monotone
(harmonic-mean, zero at a sign change) tangents for fov only. pos and tgt
keep the time-scaled Catmull-Rom tangents with the cap above.

In Ease timing the existing THREE curves and code path are untouched, so
Gate 0 still holds by construction.

### UI and presets

A two-way segmented control (Ease | Flow) next to Smooth. The Ease slider
row dims (not hides) in Flow with the hint "used by keys set to Auto in
Ease timing only". Preset: `camera.timing: "ease" | "flow"`, absent =
`"ease"`. No further `PRESET_VERSION` bump: Phase 1 already moved it to 11,
and an absent field already means today's behaviour.

**Reset timing everywhere `camSmooth` is reset.** Loading a new capture
clears the keys and sets `camSmooth = true` (in the file-load path, beside
`CAMKEYS = []`). Set timing back to `"ease"` there too. Do the same in all
three camera branches of preset load: the `keys` branch reads
`camera.timing`, while the old A/B branch and the no-camera branch both
reset it to `"ease"`. Update the Ease | Flow buttons wherever the
Smooth/Linear button label is updated.

**Gate 2:** Gate 0 passes in Ease. In Flow on the uneven 6-key fixture:
(a) at each interior key between two Auto segments, left and right
finite-difference velocity agree within 2% of their magnitude, or within
1e-6 of the key set's diagonal per unit t when both are near zero (a capped
tangent can legitimately be 0); (b) every sample lies within the
key set's bounding box expanded by 10% of its diagonal (no loops from the
tangent cap); (c) on the circling fixture, the distance to target never
drops below 85% of the smallest key distance. Record (c)'s actual minimum in
NOTES: it measures how much world-space interpolation cuts the corner, which
is the evidence for or against Phase 4.

---

## §5 — Phase 3: snapping to hits and beats

Snapping is an **editing aid**, not content. It is not saved in presets.
Remember the choice in `localStorage` (wrapped in try/catch).

- A **Snap** selector in the Camera section: Off | Hits | Beats | Both.
  Default Off. Beats is only offered when Length is in Bars mode (the only
  mode with a grid); otherwise it is disabled with a tooltip.
- **Targets, in normalised t:**
  - Hits: `audioEnv.hit.hitTimes` (seconds, ascending). Timeline seconds are
    `playhead * P.dur`. When `audioLoop` is on, repeat the hit list every
    `audioDur` up to `P.dur`, the same rule `hitCountAt()` uses (~line
    1561). Divide by `P.dur`. Cache the target array; rebuild on audio
    load, loop toggle and length change. Use the raw analysed hit times,
    **ignoring any per-band lag**, exactly as `hitCountAt()` does, so a key
    snapped to a hit lines up with the background-on-hits colour change.
    This is deliberate. Say so in NOTES, since a user with a lag set on a
    hit route will see that route react slightly off the snapped key.
  - Beats: `i / (TL.bars * TL.beats)` for every beat; bar lines included.
- **Tick drag** (`pointermove` handler, ~line 6409): after computing the
  clamped `t`, if snap is on and Shift is not held, move to the nearest target
  within 8 px (`8 / row width` in t), still inside the neighbour clamp.
  Show the snapped position live; nothing else changes about the drag.
- **Hit marks:** when Snap includes Hits, draw faint 1 px marks for the
  targets on `#camRow` (a single absolutely-positioned canvas behind the
  ticks, not one div per hit). Skip drawing if there are more than ~600 in
  view; snapping still works. The canvas is not a `.camtick`, so
  `drawCamRow()` leaves it alone. Redraw it only when the targets, the Snap
  mode or the row width change (resize), **never** from `drawCamRow()`,
  which runs every playback frame.
- **Playhead to hit:** a key pair to jump the playhead to the previous/next
  hit, for "go to the hit, Add key". The obvious choice, ⌥ , / ⌥ . next to
  the existing ⇧ , / ⇧ . (camera key jump), **does not work as is**: the
  keydown handler returns early on `e.ctrlKey || e.metaKey || e.altKey`
  before the `NAV_KEYS` switch. Either add a narrow exception above that
  line for Alt + `Comma` / `Period` only (match on `e.code`, since ⌥ changes
  `e.key` on macOS, e.g. ⌥ , gives "≤"), or pick unmodified keys not already
  in `NAV_KEYS`, `HOLD_TARGET` or the table. Add whichever you choose to the
  `#kbdOverlay` table.

**Gate 3:** With Snap Off, tick drags behave exactly as before. With Hits on
`bandtest.wav` looped ×2, every snapped key lands within 1e-6 of a hit
target, including second-pass hits. Shift-drag ignores snapping. Gate 0 still
passes.

---

## §6 — Phase 4 (optional): Orbit path shape

Only if Gate 2(c) and real use show corner-cutting is a problem.

The path-shape toggle becomes three-way: **Linear | Smooth | Orbit**.
Preset `camera.shape`, with the old `smooth` boolean still written for older
builds and read when `shape` is absent. No `PRESET_VERSION` bump, for the
same reason as `timing`. Shape is reset wherever `camSmooth` and timing are
reset: the new-capture path and all three preset-load branches.

In Orbit:

- Target interpolates as in Smooth (THREE curve in Ease, Hermite in Flow).
- Each key's camera offset `pos − tgt` becomes `(az, el, logR)`. el is
  clamped to ±89°. az is **unwrapped** key to key along the shortest
  delta, so a 350° → 10° pair turns 20°, not 340°. To go the long way, the
  user adds a key in between. Document that rather than building a control.
- Interpolate az, el and logR with the segment's `u` (eased in Ease timing).
  In Flow, use GS_Bed's **monotone cubic Hermite** for these three channels
  (`plan()` tangent pass: harmonic mean of adjacent slopes, zero where the
  slope changes sign). Here it fits: no overshoot past a key's angle or
  distance.
- Camera = interpolated target + spherical(az, el, exp(logR)). log space
  makes push-ins and pull-outs feel even.

**Gate 4:** Gate 0 passes in Linear and Smooth. On the circling fixture in
Orbit, distance to target stays within 1% of the key distance throughout. A
350° → 10° pair turns 20°. Flow + Orbit passes Gate 2(a) on az.

---

## §7 — Non-goals

- GS_Bed's move palette, drift planner, coverage cone or travel volume.
- Cue suggestion, beat tracking or section detection. Snapping uses only the
  hit analysis Gausseous already has; tempo stays entered or tapped.
- Audio driving camera position directly, beyond what `SPEC` already allows
  (`ease`, the Orbit-mode parameters).
- Per-key fov curves separate from position, roll, or camera shake.
- Undo for camera edits, unless the file gains general undo first.
- Any new dependency, module file or build step. Everything lands in
  `index.html`.

## §8 — Files touched

- `index.html`: the camera keyframe block (`CAMKEYS`, `ease`,
  `rebuildCamCurves`, `addCamKey`, `sampleCam`, ~4975–5080); the
  new-capture reset of `CAMKEYS`/`camSmooth` (~4831); `jumpCamKey` (~8246,
  reused by the Arrival block's key buttons); `drawCamRow` and the tick
  drag handlers (~6362–6430); `refreshCamUI`; Camera section HTML/CSS
  (~747–800, `#camRow` CSS ~105–140); preset save (~6798) and load
  (~7069) with `PRESET_VERSION`; the shortcut table and keydown handler
  (Phase 3); `window.__bench` (Phase 0).
- `NOTES.md`: new `## Camera dynamics` section, appended per phase.
- `README.md`, `Gausseous_Architecture.md`: once, at the end.
