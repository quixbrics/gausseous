# Gausseous — project notes

Working folder: `/Users/55147209/Documents/Claude_Builds/Splatification`
Main file: `index.html` (single file, no build step, no npm install)
Serve with `python3 -m http.server` and open `http://localhost:8000/`.
ES modules will **not** load over `file://` — it must be served.

It is `index.html` rather than a descriptive name so GitHub Pages serves it at
the repo root with no path. `splat-bench.html` remains as a redirect stub for
older links and carries the query string across, so `?debug=1` survives.
Live at <https://quixbrics.github.io/gausseous/>.

**Renamed from Splatification / Splat Bench.** The local working folder keeps
its old path — renaming it would break nothing in the app but every absolute
path in this file, and the folder name is not something the app or Pages reads.
What did change: the repo name, the Pages URL (GitHub redirects the old one),
the `<title>`/`<h1>`, the export filename prefixes, the `localStorage`
accordion key, and the preset format tag. The old `splat-bench-preset` tag is
still **accepted** on load and never written — forgiving load, strict save, the
same rule the version migrations follow.

---

## What this is

A bespoke 3D Gaussian Splatting manipulator and animator for the browser. The
reference point is the Irrealix After Effects plugin (crop, colorize ramp,
opacity-ramp reveal, splat-scale animation, noise distortion) — but outside of
After Effects, and built on the layer *underneath* those effects rather than a
fixed effect list.

The point is not to reimplement SuperSplat. SuperSplat already does cleanup,
camera timelines and video export well, and it is free and open source. This
exists to do things SuperSplat does not: arbitrary per-Gaussian programmable
manipulation, and eventually parameter automation driven from outside (audio
FFT, LFOs, OSC).

Target is **desktop**. A dialled-back mobile path is wanted eventually but must
not constrain design decisions now.

---

## Architecture

Dependencies are all CDN imports via an importmap. No package.json, deliberately.

| Import | Version | Role |
|---|---|---|
| `three` | 0.180.0 | scene graph, camera, OrbitControls |
| `@sparkjsdev/spark` | 2.1.0 | 3DGS renderer + `dyno` shader-graph system |
| `mp4-muxer` | 5.x | muxes WebCodecs chunks into MP4 |

**Spark** (World Labs, MIT) is the substrate. The key facility is `dyno`: a
shader-graph system written in JS that compiles to GLSL and runs per-splat on
the GPU. `SplatMesh` exposes `objectModifiers` and `worldModifiers` as injection
points into the standard splat generation pipeline.

Everything in the effect stack is a single `dyno.dynoBlock` mapping
`{gsplat: Gsplat} -> {gsplat: Gsplat}`, passed as `objectModifiers: [modifier]`.
Current effects, all in one GPU pass:

- **Displace** — `sin()` field over swizzled center × per-splat `hashVec3(index)`
  direction, scaled by amount
- **Reveal** — soft `smoothstep` sweep plane along object-space Y; shrinks splat
  scales into the edge rather than only fading alpha
- **Trim** — zeroes alpha for gaussians below a min opacity or above a max scale
  (cheap floater kill; does not reduce splat count)
- **Tint** — RGB multiply

### Uniforms, not constants

Every parameter is a `dyno.dynoFloat` / `dynoVec3` uniform held in the `U`
object. Assigning `U.foo.value = x` is free. Changing the *structure* of the
graph forces a GLSL recompile and requires `splatMesh.updateGenerator()`.

**Keep this invariant.** Adding a new effect should mean adding uniforms and
graph nodes once at construction, gated by an amount uniform that can go to
zero — not rebuilding the graph when a control changes.

Assigning `U.foo.value = x` avoids the recompile but does **not** by itself make
the change visible — Spark must also be told the splat data is stale. See "The
uniform invariant" below; this was the single biggest live bug in the file.

### Slider normalisation

Sliders are normalised against scene extent so they behave the same on any
capture. `reframe()` computes `radius` (bounding sphere) and `bounds.min/max`
(Y extent); `pushUniforms()` multiplies through by those. Do not feed raw
slider values into world-space uniforms.

Normalising is necessary but not sufficient: `cullS` was correctly multiplied by
`radius` and still unusable, because the slider's *range* did not match the
quantity's actual distribution. When adding a parameter, sweep it and look at
where the response actually happens — a correctly normalised control can still
put its whole useful band inside 0.1% of the travel.

---

## The uniform invariant — read this before adding an effect

`U.foo.value = x` is cheap (no GLSL recompile), but **it is not sufficient on
its own.** Spark gates its generate pass, `spark.module.js` 2.1.0 ~line 10210:

```js
const needsUpdate = viewChanged || version !== this.current.version;
if (autoUpdate && !needsUpdate) { doUpdate = false; }
```

Spark *bakes* modifier output into a packed splat buffer; it does not evaluate
the dyno graph per draw call. Writing a uniform's `.value` changes neither the
view nor the version, so the stale baked buffer is reused and the change is
invisible — until the camera happens to move, at which point every pending edit
pops in at once.

`pushUniforms()` therefore ends with `mesh?.updateVersion();`. That increments an
integer. It is not `updateGenerator()` and does not recompile GLSL. Measured cost
at 177k splats: none — 120 fps idle, playing, and playing with displace active.

**Any new code path that writes a uniform must bump the version.** Route
everything through `pushUniforms()` rather than assigning `U.*.value` directly.

This one is nasty because it fails *silently and intermittently*: dragging a
slider does nothing, then an unrelated orbit makes it all appear, so it reads as
a sort/async glitch rather than a missing invalidation.

---

## Known issues

**`ease` is mathematically inert at `playhead = 0.5`.** `ease(0.5, k) = 0.5` for
every `k`. Expected from the formula, not a bug — but it means the midpoint is a
bad place to check whether the ease slider works. Probe at 0.25.

**A hidden tab still stops the render loop.** `requestAnimationFrame` does not
fire when the tab is not visible, so the live view freezes and `setTimeout` is
clamped to ~1s. The export no longer depends on that timer (see Fixed), but a
background export still crawls because nothing composites. The status line
appends `[tab hidden — throttled]` and a warning fires at render start.

This also breaks automated pixel probes: readPixels can only ever return the
last composited buffer. Use `?debug=1` and `window.__bench.settled()`, which
draws synchronously and awaits the sort without involving rAF.

**Region: `ellipsoid` renders identically to `sphere`.** Correct, not a bug —
the Size control is a single uniform scalar, and a uniformly-scaled ellipsoid
*is* a sphere. Ellipsoid only becomes distinct with per-axis size, which the
current UI cannot express. Add three size sliders if it is ever wanted.

---

## Fixed

**TDZ on `playhead` (v1).** `let playhead = 0` was declared below the parameter
binding block while `pushUniforms()` read it; the throw during module evaluation
killed every listener and the render loop below it. Hoisted into the state block
at the top. All module state now lives there — keep it that way.

Two hardening measures from that episode, both still in place and worth keeping:

1. `<output>` values are blanked in HTML and populated by JS, so a dead module
   looks dead instead of showing plausible hardcoded numbers.
2. A global error surface near the top of the module:
   ```js
   addEventListener("error", e => setStatus("JS error: " + e.message, true));
   addEventListener("unhandledrejection", e => setStatus("Async error: " + (e.reason?.message ?? e.reason), true));
   ```

**`Go A` / `Go B` were silently ignored while `Timeline drives camera A→B` was
checked** (the default). `apply()` set the camera and `driveCamera()` overwrote
it on the very next frame from `playhead`. `apply(k, at)` now also moves the
playhead to the matching end of the timeline (0 for A, 1 for B) when camDrive is
on and both keys exist, so the drive agrees with the button instead of fighting
it. With camDrive off the timeline is left alone. Verified: camDrive on, Go A →
coverage 10.305 / scrub 0, Go B → 2.487 / scrub 1.

**`Max splat size` (`cullS`) addressed nothing across 99.9% of its travel.** The
old slider was linear `0.001..9` while the entire useful band was `0.001..0.01`.
`P.cullS` is now a 0..1 perceptual position mapped exponentially onto the
threshold via `cullSize()`, `CULL_MIN 0.0002 .. CULL_MAX 0.1`. The `* radius`
normalisation was kept — it was never the problem, the range was. Measured
transition band now spans roughly u=0.25..0.6, about 35% of travel:

| u | 0.0 | 0.2 | 0.3 | 0.4 | 0.5 | 0.6 | 0.8 | 1.0 |
|---|---|---|---|---|---|---|---|---|
| threshold | .0002 | .0007 | .0013 | .0024 | .0045 | .0083 | .0289 | .1000 |
| coverage % | 0 | 0 | 0.41 | 4.05 | 9.67 | 10.29 | 10.30 | 10.305 |

`cullS` is the only parameter whose slider position is not its value, so it
needs the `SHOW` readout override. If a second such parameter appears, that map
is the place to put it.

**`revS` default was unrepresentable on its own step grid.** Declared
`value="0.15"` against `min="0.001" step="0.002"`, which only yields odd
thousandths — the browser silently snapped it to 0.151, and the panel had been
displaying 0.151 for a declared 0.15 since the beginning. Step is now `0.001`.
This was found by the reset round-trip test, not by reading the markup.

**Audio never came back after the first loop, whenever the clip is shorter
than the timeline `dur`.** User-reported, reproduced exactly. `<audio>`
doesn't loop on its own (`.loop` is never set), so once playback runs past the
clip's own length the browser auto-pauses it. The loop-boundary code in
`frame()` (`if (playhead < prev) audioSeek();`) only repositioned
`currentTime` — `audioSeek()` never calls `.play()`, only `audioFollow()`
does that, and nothing was calling `audioFollow()` there. So the element sat
correctly-seeked-but-paused for the rest of the session; only a manual
Play/Pause toggle (which does call `audioFollow()`) could ever restart it.

Fixed by calling both on loop: `audioSeek()` first (still needed on its own
for the *opposite* case — a clip LONGER than `dur`, where `<audio>` never
reaches its own end and so is still mid-playback, unpaused, and needs
repositioning back to the loop point or it drifts forward out of sync with
the visual loop), then `audioFollow()` (to resume playback if the clip had
auto-paused at its own end). Verified both directions directly against a real
`<audio>` element and the real 6s `bandtest.wav` fixture via `window.__bench`:
simulating the browser's own auto-pause-at-end (`audio.pause()` at
`currentTime = 5.99` with `dur = 10`) followed by the loop-boundary call
sequence resulted in `paused: false, currentTime: ~0.1` — playback resumed.
Repeating with only the old `audioSeek()` call reproduced the bug exactly:
`paused: true` (correctly repositioned, never restarted) — confirming this
was a real regression path, not a hypothetical. The longer-than-timeline case
(`dur = 3`, audio mid-playback at `currentTime = 2.5`, never paused) also
resynced correctly after the fix: `paused: false, currentTime: ~0.03`.

---

## Per-parameter reset

Every slider row carries a `↺` button, generated in JS from `SPEC` rather than
hand-written into 18 rows of markup.

The default is **not** duplicated in JS. It is read back from the control itself
at boot into `DEFAULT[id]`. Note this is the *snapped* boot value, not
`el.defaultValue` (the raw HTML attribute) — those two differ whenever a
`value` attribute is off the min/step grid, which is exactly the revS bug above.
Comparing against the snapped value makes reset exact even if a future slider
repeats the mistake.

Reset dispatches a synthetic `input` event rather than calling `pushUniforms()`
directly, so a reset and a user drag travel exactly the same code path —
including the `dpr`/`dur` side effects and the version bump.

The button carries a `.dirty` class whenever the value differs from default,
which makes the panel double as an at-a-glance "what have I changed" readout.
Checkboxes deliberately have no reset button: toggling one is already a
single-click restore.

---

**Export backpressure was timer-based.** `while (encoder.encodeQueueSize > 8)
await sleep(2)` cost ~313ms per iteration in a hidden tab instead of 2ms.
Replaced with `drain()`, which awaits the encoder's `dequeue` event — not
clamped — raced against a 50ms sleep to cover the case where the queue empties
between the size check and the listener attaching.

**The `settle` wait is now a deterministic sort signal.** The export loop used
to render, `sleep(settle)`, render, capture — where 24 ms was an admitted guess.
It now `await`s `spark.update({ scene, camera })`, which runs generate + sort for
that exact viewpoint and resolves when both are done. `settle` survives as an
extra manual wait defaulting to 0.

Note the API shape: **in Spark 2.1.0 the `SparkRenderer` IS the viewpoint.** It
owns `update()`, `driveSort()` and `sorting` directly — there is no separate
`SparkViewpoint` instance to reach through, and `spark.viewpoint` is
`undefined`. `minSortIntervalMs` defaults to 0, so `driveSort()` is not
deferred behind a timer.

---

## Camera drive mode — Manual / Keyframes / Orbit

Roadmap Phase 5 of `DEV_PLAN.md`. Replaces the `camDrive` checkbox with an
explicit three-way mode, `camMode: "manual" | "keyframes" | "orbit"`. The
checkbox already caused one real bug (`Go A` silently overwritten, fixed in an
earlier phase) precisely because two things — a boolean and a keyframe list —
both partly decided whether something owned the camera. A single string with
exactly three values, switched once in `driveCamera()`, makes "two drivers
write the camera in one frame" structurally impossible rather than merely
unlikely: there is only one branch any given frame can take, by construction,
not by care.

### Orbit — reads `controls.target` live, never writes it

`obAxis` (X/Y/Z), `obTurns` (±3, signed — negative reverses), `obPhase`
(0..1 start angle), `obElev` (−1..1), `obDist` (multiple of `radius`) are all
real `SPEC` entries in group `camera`, so they animate, reset, and persist for
free like everything else. Orbit centres on whatever `controls.target`
currently is and only ever moves `camera.position` — never the target — so
"frame a subject, then switch to Orbit" does what it looks like it should, and
Orbit never fights whatever last set the target (a drag, `Reframe`, a loaded
capture).

`obDist`'s "neutral" value is inherently tied to the CURRENT framing — unlike
every other `SPEC` default, there is no single constant right for every
capture. `reframe()` recomputes what "default" means every time it runs (on
load, on a transform commit, on `Reframe`), but only overwrites the *live*
slider value if the user has not already moved it away from its old default —
silently discarding a deliberate `obDist` customisation because an unrelated
Correct-mode edit was committed would be a worse surprise than a slider not
tracking a reframe it had no reason to care about. Verified both directions:
a custom `obDist` survives an intervening reframe unchanged, and the reset
button correctly restores to *that* reframe's computed default, not a stale
one from a previous capture or window size.

### Measured: each axis holds its own coordinate exactly constant

At the default elevation (0), sampling 100 points across a full revolution:

| `obAxis` | spread on the chosen axis | spread on the other two |
|---|---|---|
| X | **0.000000** | 6.374 |
| Y | **0.000000** | 6.374 |
| Z | **0.000000** | 6.374 |

Exact zero, not "small" — at `obElev = 0` the orbit-axis coordinate is
algebraically `target[axis]` with no `t`-dependent term at all, so this isn't
really a numerical-precision result, it's the formula doing what it says.

`obTurns = 1` returns to the exact start position at `t = 1` (measured
distance: `0.000000`, not just under the 0.1% bound). `obTurns = -1` traces
the *same* set of points as `obTurns = 1` in reverse — verified pointwise,
comparing `sampleOrbit(t)` at turns=−1 against `sampleOrbit(1−t)` at turns=+1
across 21 samples: max pointwise difference `0.000000`.

### A real bug found while testing this phase, not introduced by it

The very first Orbit export produced a **3990-byte** file for a 48-frame
1280×720 clip — visibly broken. Traced it to the export loop itself, not to
anything Orbit-specific: reproduced identically in **Keyframes** mode too, on
a genuinely fresh load with no interactive probing beforehand. Every previous
export test in this project's history (Phases 1 and 4) had incidentally
warmed the scene up through unrelated `settled()`/`__probe()` calls before
ever hitting Render — so the gap was real from the start and simply never
exercised. The export loop set up its fixed-resolution render target and went
straight into the frame loop with no warm-up render; the same "renders as
nothing for a frame or two" generator-rebuild transient documented elsewhere
in NOTES applied here too, just to the export's *first* frame specifically,
producing a barely-there video instead of a wrong one. One `spark.update()` +
`render()` before the frame loop starts fixed it — confirmed on the exact
failing sequence (fresh load, immediate mode switch, immediate export, zero
warm-up): **0.9 MB**, 48 real frames, verified via `ffprobe signalstats` with
no two consecutive frames sharing identical luma.

### Measured: Orbit export, wrap point

Same `ffprobe signalstats` method already established for export verification.
5-key-equivalent single-orbit export, `obTurns = 1`, 48 frames: no frozen
frame pair anywhere, mean frame-to-frame luma delta 0.34, and the wrap
(`t = 0` vs `t = 1`, which `obTurns = 1` puts at the same physical position)
differs by **0.001** — visually and numerically seamless, no discontinuity.

### Exclusivity — structural, and separately exercised

Clicking each of the three mode buttons leaves exactly one `.on`; entering
Correct mode force-sets `manual` (so the gizmo never fights an auto-driven
camera) and restores whatever mode was active on exit — verified `orbit ->
manual (entering Correct) -> orbit (leaving)`. One subtlety caught before it
shipped: `applyPreset()`'s own restoration of a saved `camera.mode` has to run
**before** `leaveCorrect()` is called (not after) if a preset happens to be
loaded while Correct mode is open — `leaveCorrect()` itself calls
`setCamMode()` to restore the *pre-Correct* mode, and if that ran after the
preset's own mode restore it would silently clobber it. `leaveCorrect()` is
now called at the very top of `applyPreset()`, before any other restoration
begins, rather than being interleaved partway through.

### Preset

`camera.mode` joins `camera.{keys, smooth}` (no version bump — still v5,
additive to the same sub-object). `obAxis` goes through the existing
`SELECTS` mechanism, same as `rgShape`. A v4-or-earlier preset has no
`camera.mode` at all; it defaults to `"keyframes"`, matching what the old
`camDrive` checkbox defaulted to and what `driveCamera()` actually did with a
migrated A/B pair before this phase existed. Byte-exact round-trip verified
for `camMode` + all four orbit sliders together.

### Phase 1 of `DEV_PLAN_EFFECTS.md` — evaluate on playhead change, not per frame

The mode select above fixed "two drivers write the camera in one frame" but
introduced a worse bug: `driveCamera()` was called **unconditionally, every
render-loop tick**, regardless of whether anything was playing. With exactly
one camera key, `sampleCam(t)` is constant at every `t`, so the camera was
overwritten back to that one pose on the very next frame after any manual
orbit drag — permanently pinning it, with no way to move the camera to author
a second key. Same shape as the original `Go A`/`Go B` bug, in a new form: the
guarantee that made playback correct (exactly one writer) also made authoring
impossible (that writer never yields).

**The fix splits evaluation from ownership.** `evaluateCamera(t)` is now a
pure function — reads `camMode`/`CAMKEYS`/`controls.target`, writes nothing,
returns `{pos, tgt?, fov?}` or `null`. `driveCamera(t = playhead)` is the
*only* function that writes `camera.position` / `controls.target` / `fov` for
the drive system (reframe() and the one-time constructor default are the only
other writers, and are deliberately separate — see below). It is called only
on discrete events, not every tick:

- **Playback**, every frame, from inside `frame()`'s `if (playing)` branch —
  continuous driving is correct there, nothing has changed.
- **`setPlayhead()`** — the single funnel for scrub, lane clicks, and camera-
  key-tick jumps, so all of those now re-assert the curve/orbit at the new
  position.
- **`setCamMode()`** — force-writes on every mode change, so switching to
  Keyframes or Orbit while paused snaps immediately instead of doing nothing
  until the next scrub.
- **`reframe()`** — re-asserts Keyframes/Orbit against the newly-computed
  framing rather than leaving reframe's own bounding-sphere pose stuck (In
  Manual mode `driveCamera()` no-ops, so a manual reframe correctly sticks).
- **The export loop** — force-writes every frame unconditionally, same as
  before, so frame 0 is never skipped by a playhead that has not "changed".

Paused and idle, in every mode, manual orbit input now simply works: move the
camera, `Add key`, scrub, move again, `Add key`. That is the whole fix. This
is how every keyframe tool behaves and does **not** reintroduce two drivers —
there is still exactly one writer, it just fires on an event instead of every
frame. **Do not "fix" this back to a per-frame call** — that reintroduces the
exact bug this phase removes.

One write path is down from six sites to four: the one-time constructor
default (`camera.position.set(0,0,3)`, never called again), `reframe()`'s two
writes (a deliberately separate, user/load-triggered "fit to bounding sphere"
operation, not part of the automated drive system), and `driveCamera()`'s two
writes (the sole writer for Manual/Keyframes/Orbit). The camera-key tick-click
handler no longer writes the camera directly — ticks only render when
`camMode === "keyframes"`, so `setPlayhead(k.t)`'s own `driveCamera()` call
reproduces the exact key pose (Catmull-Rom interpolates through its own
control points exactly at their own parameter), removing a fifth write site
that duplicated `driveCamera()`'s logic.

**Dirty-state indicator.** While paused, if the live camera differs from
`evaluateCamera(playhead)` beyond a small relative epsilon (`radius × 0.002`),
`#camDirty` shows "· off curve" in `--edit` violet next to the keyframe count
— the same visual language as every other automation-vs-manual affordance in
this file. `Add key` commits the deviation into a new/replaced key; scrubbing
away discards it silently. Neither prompts.

Verified via `window.__bench` (a loaded mesh is not required — camera/controls
exist from boot):

- **The core bug, directly**: one key at `t=0.5`, `setPlayhead(0.5)` (pins the
  camera there), then a simulated manual move, then 60 paused `frame()`-style
  ticks. Camera stayed exactly at the manual position across all 60 — this is
  the assertion that would have failed before this phase (it would have
  snapped back to the key pose on tick 2). `#camDirty` correctly showed dirty;
  a subsequent `setPlayhead()` (scrub) discarded it silently, as designed.
- **3-key authoring end to end**, no mode-select touches: add key at t=0,
  move, add at t=0.5 (dirty for 30 simulated paused frames first, proving it
  doesn't drift back), move, add at t=1. All three keys distinct and correct.
- **Scrub with 3 keys**: `camera.position` after `setPlayhead(0.25)` and
  `setPlayhead(0.75)` matched `evaluateCamera(t)` to full float precision —
  no drift between the driven camera and the pure evaluator.
- **Mode-change force-write**: faked a manual drift, switched Orbit while
  paused — camera snapped to `evaluateCamera(0.5)` immediately, matching to
  full precision, no wait for a scrub.
- **Regressions, all previously-measured, re-run and matching:** `obTurns=1`
  returns to start at distance `6.4e-16` (effectively 0.000000, same as
  before); `obTurns=-1` pointwise identical to `+1` reversed, max diff
  `6.4e-16`; 5-key smoothness max/mean ratio ≤ 3× at `ease` 0/0.8/1 in both
  `camSmooth` states (max measured 2.27×, at ease=1 curved); 48-frame
  keyframe export and 48-frame orbit export both show zero frozen frame
  pairs (`driveCamera(t)` called per synthetic frame, positions diffed
  pairwise); orbit wrap (`t=0` vs `t=1` at `obTurns=1`) differs by `6.4e-16`.
- Console clean across the full sequence (mode switches, scrubs, key
  add/remove) — no new exceptions.

**Not independently re-verified this pass, and why:** Correct mode's
force-set-Manual-on-entry/restore-on-exit needs a loaded `mesh`
(`enterCorrect()` early-returns without one), and the demo asset load hung in
this session's browser environment — reproduced identically on the
pre-Phase-1 committed build via `git stash`, so it is environmental, not a
regression. `setCamMode()`'s force-write (the only change touching that path)
was verified directly and does not depend on `mesh`, so the risk here is low,
but this is a real gap, not a claimed pass — re-run the Correct-mode
entry/exit check (`orbit -> manual -> orbit`, verifying `driveCamera()` fires
on both transitions) the next time a splat loads cleanly.

---

## Camera keyframes

Roadmap Phase 4 of `DEV_PLAN.md`. Replaces the fixed `key.A`/`key.B` pair with
an ordered list, `CAMKEYS = [{t, pos, tgt, fov}, …]`, `t` in the same
normalised 0..1 clip space as every parameter envelope so rescaling Duration
rescales the camera move exactly as it rescales everything else. `fov` is
carried per key even though nothing animates it yet — costs nothing now,
retrofitting it later would be another preset migration for no reason.

### Interpolation: THREE's own Catmull-Rom, not hand-rolled

`camPosCurve`/`camTgtCurve` are `THREE.CatmullRomCurve3` instances, rebuilt on
any key mutation, sampled with `.getPoint()`. Deliberately not a hand-rolled
spline: three's implementation is proven, and the plan's own risk register is
full of examples of what happens when this file assumes a math primitive
behaves as expected instead of using or measuring one. **Three parametrises a
curve uniformly by point *index*, not by each key's own `t`** — the mapping
from timeline position to curve parameter has to account for that explicitly:
find the segment `[a,b]` containing the playhead, ease the *local* fraction
within it, then convert to `(segmentIndex + localU) / (n - 1)` before calling
`.getPoint()`. Getting this wrong would silently speed up or slow down
unevenly-spaced segments — not tested directly, but implied correct by the
smoothness measurement below holding across non-uniform key spacing.

Two keys are handled by an explicit linear branch rather than relying on
Catmull-Rom's degenerate 2-point behaviour — "the explicit path avoids the
phantom-point edge case," per the plan, and needs no argument beyond that.

### `ease` now applies per segment, and path shape is a separate axis

`ease` (existing slider, unchanged meaning) eases the *local* fraction within
whichever segment the playhead currently sits in, not the whole clip — a
camera path with uneven key spacing gets consistent ease-in/ease-out at every
segment boundary instead of one global curve. This is orthogonal to
`camSmooth` (the new Linear/Smooth toggle): `camSmooth` chooses the path's
*shape* (straight `lerp` between adjacent keys vs the Catmull-Rom spline
through all of them), `ease` chooses the *timing* along whichever shape is
current. Confirmed independent: sweeping `ease` 0→1 and toggling
`camSmooth` both stayed comfortably under the smoothness bound below, in every
combination tested.

### Measured: smoothness with 5 keys, at both extremes and both path modes

Acceptance: sampling camera position at 100 points along the timeline, no
step's distance to the next should exceed 3x the mean step distance (a
discontinuity indicator — a genuinely smooth path should have gently varying
step sizes, not one point that jumps far more than its neighbours).

| condition | max step / mean step |
|---|---|
| default (`ease=0.8`, Smooth) | 2.69x |
| `ease=0` (linear timing) | 1.92x |
| `ease=1` (max ease) | 2.88x |
| `camSmooth=false` (Linear path) | 2.05x |

All comfortably under 3x. The worst case (`ease=1`) is the most aggressive
ease-in/ease-out setting, which is exactly where a sharper step change would
be expected — the bound holding there and not just at the default is the
useful data point.

### Interaction — no sliders, per the brief

`Add key` captures the camera's current pose at the current playhead,
replacing any existing key within `KEY_EPS` of that `t` rather than adding a
duplicate. `Remove key` removes the key at the playhead if one is there;
otherwise it reports "no camera key at the playhead" rather than guessing and
removing the nearest one. Verified: removing a middle key from a 5-key set
leaves a valid, NaN-free 4-key path with no throw; removing every key returns
the camera to manual orbit control (`driveCamera()` is a no-op once
`CAMKEYS.length === 0`, confirmed by scrubbing the timeline and observing the
camera does not move).

Camera keys render as ticks on their **own dedicated strip** (`#camRow`),
always visible regardless of which parameter is focused in the lane below —
"never confused with parameter keys," per the plan. Built as plain positioned
`div`s, not an extension of `drawLane()`'s canvas renderer: a camera key has
no Y-axis value to plot, only a position on the timeline, so reusing the
per-parameter curve canvas would be solving a harder problem than the one
that actually exists. Click a tick to jump the playhead there and snap the
camera to that exact stored pose (bypassing interpolation, matching what
`Go A`/`Go B` used to do); alt-click deletes it — the same gesture vocabulary
the parameter-envelope lane already uses, not a second one to learn.

### Preset — version 5, A/B migrates to two keyframes

`camera: {A, B}` becomes `camera: {keys: [...], smooth}`. A preset with
neither shape present leaves the camera on manual control — additive, same
"absent means default" rule as everywhere else in `applyPreset()`. A v4 (or
earlier) preset's `{A, B}` pair migrates to keys at `t=0`/`t=1`, `fov`
defaulting to the camera's own construction value (50) since A/B never stored
one. Verified end-to-end: a v4 preset's `camera.A/B` reproduces the *exact*
original camera positions at `t=0` and `t=1` after migration, and the load
surfaces `Migrated: camera A/B -> 2 keyframes` rather than migrating silently.

Byte-exact round-trip verified separately for a 5-key curved path (all
positions, targets, fovs, and the `smooth` flag).

### Export — real render, not just a position sample

A genuine 2s/24fps (48-frame) export with 5 keyframes sweeping around the
capture, checked with the same `ffprobe signalstats` method NOTES already
documents for the export pipeline: every one of the 48 frames' luma differs
from its neighbour (no frozen frames — the failure mode a broken drive would
produce), and the single largest frame-to-frame jump (5.4, vs a 0.43 mean)
lands exactly at the authored camera repositioning near the capture's far
side, not at a boundary that would indicate an export/drive bug. The curve is
not literally monotonic end-to-end (the plan's own example path was never
going to be — five keyframes circling a subject does not represent a
brightness ramp), but *within* each inter-key segment it moves consistently in
one direction, which is what "consistent with the path" actually means here.

### What replaced Set A / Set B / Go A / Go B

Removed entirely, along with the now-dead `key`/`grab`/`apply` state and the
`vec()` preset helper that only ever serialised them. `driveCamera()` and the
camDrive checkbox stay (Phase 5 turns `camDrive` into an explicit three-way
mode — Manual / Keyframes / Orbit — this phase does not touch that shape,
only what the "Keyframes" case now drives).

### Phase 2d of `DEV_PLAN_EFFECTS.md` — draggable on the timeline

Camera-key ticks (`.camtick`, plain positioned divs — see the reasoning for
that above) were click-to-jump/alt-click-to-delete only; retiming meant
delete-and-re-add. Now draggable, horizontally only (there is no Y value to
edit here, unlike the canvas-drawn parameter-envelope keys).

- **Retimes live, rebuilds once.** `pointermove` updates only the dragged
  key's `t` and its own element's `left` — it does **not** call
  `rebuildCamCurves()`, which reconstructs two `CatmullRomCurve3` instances
  from scratch and would be the obvious regression if run on every move
  event. The curve, the array sort, and the rest of the row's ticks all catch
  up once, in the `pointerup` handler, on drop.
- **Clamped to neighbours, not reorderable.** The drag bound
  (`[prevT + KEY_EPS, nextT - KEY_EPS]`) is computed once at `pointerdown`
  from the sorted array's immediate neighbours, so a key can never cross —
  and, by construction, can never land within `KEY_EPS` of — an adjacent key.
  No post-hoc "snap clear" correction is needed because the clamp bound is
  already inset by `KEY_EPS` on both sides.
- **A drag-release must not fire the click handler's delete/jump.** Native
  `click` fires on `pointerup` for the same target regardless of how far the
  pointer travelled in between, so a completed drag would otherwise also
  jump the playhead (or delete the key, if released with Alt still held).
  Gated on movement distance, not on `pointerup` alone, per the plan: a
  `moved` flag flips true past a 3px threshold, and `camKeyDrag` is set to
  the sentinel string `"just-dropped"` (rather than cleared to `null`) only
  when a real drag happened — the click handler checks for that sentinel
  first and swallows the event, resetting to `null` itself. A sub-threshold
  jitter clears straight to `null`, so the following click behaves as an
  ordinary click (jump), which is correct — nothing was actually dragged.

Verified via `window.__bench` with real `PointerEvent`s (not `.click()` —
see the environment note below): dragging the middle of 3 keys from `t=0.5`
retimed it live (`moved:true`, element position tracked the pointer) and
committed to the dropped value on release; the immediately-following `click`
event (the same one a real drag-release produces) was swallowed — key count
and position both unchanged by it. Dragging past a neighbour clamped exactly
at `1 - KEY_EPS = 0.999` and never reordered (`CAMKEYS` stayed sorted). A
2px sub-threshold jitter left `moved:false` and the key's `t` untouched.
Alt-click delete with no preceding drag still removed the key normally.
Re-ran the existing 5-key smoothness regression (max/mean step ratio) against
a post-drag key set: **1.647×**, comfortably inside the established ≤3× bound.

**Environment note, not specific to this change:** synthetic event dispatch
(`dispatchEvent` for `input`, `pointerdown`, etc.) became unreliable partway
through this session's browser tab — traced to the tab itself, not the code:
a *freshly opened* tab loading the identical build dispatched and received
events normally, while the original tab (after many navigations, a `git
stash`/pop cycle, and at least one "Claude in Chrome is not connected"
disconnect) did not, even for a brand-new, unrelated `<div>` with a
freshly-attached listener created in the same script. All Phase 2 interactive
testing above was therefore done in a fresh tab. Direct state manipulation
via `window.__bench` (bypassing DOM event dispatch entirely) remained
reliable throughout and is what earlier Phase 2 sections lean on where a
fresh tab wasn't already in hand.

---

## Model transform — Correct mode

Position/rotation/scale on the loaded capture, so a scan that came out of
reconstruction tilted, off-centre or in the wrong units can be corrected in the
tool rather than in SuperSplat and re-exported. This replaces the old hardcoded
`mesh.quaternion.set(1, 0, 0, 0)` Y-down fix — that value is now the *default*
of the rotation controls (`mdRotX = 180`) rather than an invisible constant, so
a capture that is not Y-down is correctable for the first time.

### Locked behind an explicit mode, on purpose

The transform is not live-editable from the rail. `Correct` is a mode: enter
it, adjust with the numeric fields or the `TransformControls` gizmo, then
`Commit` or `Cancel`. Outside the mode every field is `disabled` and nothing
else in the codebase writes `mesh.position/rotation/scale` — there is exactly
one write path (`applyXform()`), which is what makes a drag gizmo safe to offer
at all.

Interlocks, all asserted in the debug harness, not just implemented:

- Entering Correct force-disables gesture `Adjust` (`setGesture(false)`) and
  unchecks `camDrive` (`driveCamera()` reads it live, so this alone stops the
  timeline fighting the gizmo). Both are restored on exit.
- `syncControlsEnabled()` is the single choke point for `OrbitControls.enabled`
  — both `setGesture()` and the gizmo's `dragging-changed` listener route
  through it, rather than each setting the flag independently. Two systems
  independently touching one shared flag is exactly how `Go A` / `Go B` got
  clobbered earlier in this file; this phase does not repeat that shape of bug.
- Verified: entering Adjust while correcting is refused outright; entering
  Correct while Adjust is already open force-disables Adjust and proceeds;
  simulating `xctl.dragging = true` disables orbit and re-enables it on
  release; export is refused with a message while correcting.

### The normalisation-basis split — and what it actually fixed

`reframe()` used to compute `radius`/`bounds` from the mesh's **world-space**
bounding box (`getBoundingBox(true).applyMatrix4(mesh.matrixWorld)`) and reuse
that same box for camera framing. That conflates two different jobs, and
Phase 2 splits them:

- **Normalisation basis** (`radius`, `bounds`) — object-space, computed **once**
  by `captureObjectBasis()` right after load, from `mesh.getBoundingBox(true)`
  with **no** matrix applied. Frozen until the next splat load or preset load;
  a Correct-mode edit never touches it.
- **Camera framing** (`reframe()`) — world-space, using a **local** `worldRadius`
  computed fresh from `getBoundingBox(true).applyMatrix4(mesh.matrixWorld)`
  every time it is called. This is "what fits on screen given the current
  pose," not "what a slider value means," and the two must not share a
  variable.

This is grounded in reading `getBoundingBox()` in spark.module.js 2.1.0
directly, not inferred: it iterates raw per-splat `center`/`scales`/`quaternion`
from `this.splats`, never touching the mesh's own Object3D transform. That
confirms `objectModifiers` — what `buildModifier()` is registered as — operate
on that same raw, untransformed data. `SplatEdit`/`SplatEditSdf` (the Region
system) are architecturally different: `edit.add(sdf); mesh.add(edit)` parents
them to the mesh, so they inherit its transform automatically through the
ordinary scene graph. That is *why* Spark offers both mechanisms — dyno effects
need this explicit basis freeze; region edits get transform-following for free
and needed no change in this phase.

**A latent bug this incidentally fixed.** Before this split, `bounds.min/max`
(the reveal sweep's Y-threshold) was being computed in world space — i.e.
*after* the 180° Y-down rotation — while the shader's `y` is raw object-space,
*before* it. A pure 180°-about-X rotation preserves span but negates and swaps
the endpoints, so the sweep plane's absolute position was measurably off from
where the shader's own coordinate actually sat. It was invisible in every prior
measurement because `revY`'s default is `1` (fully revealed), where the offset
doesn't matter — an endpoint value shows "everything" regardless of exactly
where the plane sits, as long as span is preserved (which a rigid rotation
always preserves). The *sweep shape* did change once the frame was corrected —
measured, same capture, same slider values:

| revY | 0 | 0.25 | 0.5 | 0.75 | 0.9 | 1 |
|---|---|---|---|---|---|---|
| coverage % (old, world-frame bounds) | 0.12 | 2.8 | 6.5 | 9.7 | 10.3 | 10.3 |
| coverage % (new, object-frame bounds) | 0 | 1.55 | 4.99 | 8.35 | 9.54 | 9.72 |

Both are monotonic, working sweeps — this was never visibly "broken" — but only
the new one is measuring the plane against the coordinate frame the shader
actually uses.

### Invalidation — measured, not assumed

The obvious question: does a bare Object3D transform change (no dyno uniform
touched) show up on the very next render, same as the original uniform-version
bug, or does it need an explicit `mesh.updateVersion()`? Tested directly rather
than guessed, with the camera held perfectly still:

- A transform change followed by exactly **one** `renderer.render()` call
  still shows the **stale** pose.
- A **second** `renderer.render()` call — with nothing else touched in
  between, no `pushUniforms()`, no `updateVersion()` — shows the **correct**
  pose, and a third confirms it is stable there.
- Calling `pushUniforms()` (and therefore `mesh.updateVersion()`) between the
  two renders does **not** shortcut this to one render. It is unrelated to the
  version-gate mechanism the uniform invariant documents.

So this is a **different, already-documented hazard**, not a recurrence of the
original bug: the same "render, then render again" latency NOTES already
describes for camera moves and for the offline export loop's `settle` step —
Spark's async sort needs one extra frame to catch up to a world-position
change, of any kind. It is not a "stuck until something unrelated happens"
failure, just a one-frame lag.

**Consequence for the UI, not just the test:** during interactive Correct-mode
editing, `frame()` calls `renderer.render()` on every animation frame
regardless of what changed, so this self-resolves within about one frame
(~16 ms) with no further user action — satisfying "visible without any further
interaction" in the sense that matters, even though it is not literally the
very next paint. It cannot bite an export, because export is refused outright
while correcting, so a committed transform is always many frames "settled" by
the time Render is clickable again.

`applyXform()` still calls `pushUniforms()` on every edit. Not because it
shortens the one-frame lag above — measured, it does not — but because it is
free (confirmed: a single integer increment, no measurable cost) and it keeps
every other normalised uniform consistent if anything about the load-time state
changes in a future phase. Kept as a defensive no-op with a correct comment,
not a fix for a problem it does not solve.

### `mdScale` is log-mapped, same reasoning as `cullS`

Slider is a 0..1 perceptual position, `xscaleOf(u) = 0.1 * (10/0.1)^u`, so
`u = 0.5` maps to exactly `1.0` — the slider's own midpoint is the neutral
scale, with no special-casing needed. Verified against the mesh's own
**world-space** bounding box (not screen pixels — see below):

| slider u | 0.5 (default) | 0.6505 (target 2×) |
|---|---|---|
| mapped scale | 1.000 | 1.995 |
| world-space extent ratio | — | **1.9953×** (x/y/z identical) |

### A measurement trap worth recording: perspective clipping looks like a bug

The first attempt to verify "`mdScale = 2` doubles the rendered bbox" measured
**screen-space pixel width**, and got 1.868–1.885×, not 2×. That is not an
implementation defect — the doubled butterfly's bottom edge was measurably
touching the canvas edge (`y1 = 780` of `H = 782`), i.e. genuinely clipped by
the viewport, because `reframe()` is deliberately **not** called on every
Correct-mode edit (a live camera re-fit while dragging a slider would be
disorienting, defeating the point of a stable reference frame to correct
against). A screen-space measurement is only valid when nothing is clipped or
foreshortened; **the mesh's own world-space bounding box is the correct
instrument** for verifying a transform edit, not pixel coverage. Coverage stays
the right tool for verifying *effects* (which don't move the camera-relevant
extent), just not for verifying the *transform* itself mid-edit.

### A real bug this phase's own testing caught: `xformReset()` skipped the reframe

`Reset transform` originally called `reframe()` only `if (correcting)`. Outside
Correct mode (its normal, undisabled state — the button works at any time by
design) it reset the mesh's geometry but left the camera framed for whatever
the *previous* transform looked like. Concretely: load a preset with an
extreme transform (which does call `reframe()`), then click Reset outside
Correct mode — the geometry snaps back to default size/position, but the
camera stays fitted to the old, larger/offset pose, leaving the now-correct
model tiny or off-centre. Measured as a full regression-sweep collapse (every
coverage figure near zero) before being traced to this. Fixed: `reframe()` is
now unconditional in `xformReset()` — any discrete, deliberate transform
mutation (load, commit, cancel, reset, preset) reframes; only *continuous*
edits (slider drag, gizmo drag) deliberately do not.

### Preset

`transform: {...xform}` — additive, no version bump. A preset with no
`transform` block takes `XFORM_DEFAULT`, reproducing the pre-Phase-2 hardcoded
behaviour exactly. A preset with one is clamped field-by-field to each
control's own range, same treatment as every other preset value; a hostile
preset (`rotX: 9999, posX: 50, mdScale: 5, rotZ: "nan"`, …) loads cleanly with
every field clamped and no warning raised. Loading a preset while correcting
exits the mode first.

Property of the *capture*, not the workspace — unlike `Mobile profile` — so
unlike that toggle, this belongs in the file. Presets reference the asset by
name only, so a transform saved against one capture and loaded against another
is meaningless; harmless, not worth guarding against.

### Not in `SPEC`, on purpose

`mdRotX/Y/Z`, `mdPosX/Y/Z`, `mdScale` are hand-wired rather than added to
`SPEC`: a locked transform cannot carry an envelope, so SPEC membership would
buy free reset/preset/audio wiring for a control that structurally cannot use
most of it. The rows carry two `.key.ghost` placeholders each (no per-row reset
button, no key glyph) purely to keep the grid's label/output columns aligned
with every SPEC-driven row — there is a single bulk `Reset transform` button
instead. The accordion's `.gflag` summary marker is hand-managed too
(`refreshModelUI()`, keyed off `#modelFlag` by id) since `groupIds("model")` is
empty; `refreshGroups()` now explicitly skips any group with no `SPEC` entries,
rather than silently clobbering a hand-managed flag with an always-empty
computation — a real conflict caught before it shipped, not after.

---

## Region — dyno SDFs and per-effect masking

Roadmap Phase 3 of `DEV_PLAN.md`, the largest single phase. Replaces the native
`SplatEdit`/`SplatEditSdf` system entirely with 4 region slots implemented as
plain dyno graph math, and folds Reveal into it as a plane-shaped region.

### Why the native system had to go

`SplatEdit`/`SplatEditSdf` can change colour and opacity but cannot hand a
per-splat mask to a dyno graph — Spark applies them in its own pipeline, after
`objectModifiers` run. Masking Displace/Saturation/Tint/Trim by a region (the
actual point of this phase) needs the region's inside/outside test to exist
*inside* `buildModifier()`. Once the SDF has to live in dyno anyway, running
two parallel region systems means two transform sources of truth and a UI that
has to explain the difference — so the native path was removed rather than
kept alongside.

### Architecture

`region[i] = {shape, mode, invert}` for `i = 0..3` (R1..R4) — plain JS routing
state, not `SPEC`, same treatment as `AUD`'s band/depth. The 9 per-slot
*numeric* fields (`rg{n}X/Y/Z/Size/Soft/Shrink/R/G/B`, 36 entries total) **are**
generated into `SPEC`, exactly like `REGION_SPEC` generates them rather than
hand-listing — 36 near-identical entries is exactly the repetition `SPEC`
exists to absorb, and hand-listing is where a copy-paste slot-index mistake
hides. Rows are generated the same way (`buildRegionRows()`), one `.rgpanel`
wrapper per slot, all 4 permanently in the DOM (`SPEC`'s bootstrap loop needs
every element to exist) with only the selected slot's panel visible — a tab
UI, not 36 always-visible rows. `#rgMode`/`#rgShape`/`#rgInvert` are ONE shared
set of controls that read/write whichever slot `rgSlot` currently points at.

Every shape is driven by a single `size` uniform (matching the old system's one
Size slider), not the old radius-vs-scale duality — that duality was a Spark
API quirk this reimplementation does not need to reproduce, only its rendered
result.

### The one deliberately-avoided recompile

`DEV_PLAN.md` flagged this explicitly: 4 slots x 7 shapes is 28 SDF
evaluations per splat per frame if done unconditionally, versus recompiling
`buildModifier()` and calling `mesh.updateGenerator()` on every shape change.
The plan's own instruction was to measure compile time and only take the
recompile if evaluating everything unconditionally proved too slow.

**Measured: baseline (no regions) and 4 simultaneous active crop regions,
different shapes, played back to back in the same tab — 28 fps both.** Region
evaluation costs nothing measurable; the 28 fps ceiling itself is a property of
this specific test session (present with every effect off too, not something
Phase 3 introduced) and not evidence against the "eat the arithmetic" choice.
Given that, all 4 slots x 7 shapes are evaluated **unconditionally**, matching
the uniform invariant everywhere else in this file — shape and mode are just
more `float`-encoded uniforms, selected with `dyno.equal`/`dyno.select` chains
like everything else, never a recompile.

### The six SDFs, and what stayed conservative

Every formula decomposes vec3 inputs into scalars via `swizzle` before doing
arithmetic (`abs`/`sqrt`/`min`/`max`/`clamp` used only on floats, never on raw
vec3), mirroring this file's existing caution about mixed-type dyno calls
(`mix`/`dot` were the ones actually found ambiguous — see Colour section).
`dyno.equal`, `dyno.abs`, `dyno.sqrt`, `dyno.clamp` on floats were previously
unused in this file; they are now confirmed working by the coverage-table
reproduction below, which is the real test, not a synthetic one.

- **sphere**: `length(rel) - size`.
- **ellipsoid**: identical to sphere. With ONE uniform radius driving all 3
  axes (no per-axis control exists), the IQ approximate-ellipsoid formula
  algebraically reduces to the sphere SDF — verified by hand, then by the
  coverage table below matching sphere and ellipsoid to 4 decimal places at
  every tested size.
- **box**: sharp cube, half-extent `size`, the standard exact box SDF
  (`abs(p)-b`, split into outside/inside terms).
- **cylinder**, **capsule**: along local Y. Both radius and extent are driven
  by the same `size` uniform, matching the native system's one-Size-slider
  behaviour exactly (verified numerically, not assumed).
- **plane**: `sdf = -y` (of the position relative to the slot's own centre).
  Negative *above* the centre, i.e. "inside" (what a Cut region removes) is
  the upper half-space — chosen so slot 1 in Cut mode, with an envelope on its
  Y position, reproduces the old Reveal sweep. This is a genuinely different
  shape from Spark's native "plane" (which behaves like a small finite disk,
  not an infinite half-space — measured: native `crop_plane` at the old
  default size gave 0.99% coverage, a small area, not a half-space sweep).
  The native plane was never the target; Reveal's actual behaviour was.
- **all**: `sdf = -1e6`, i.e. always inside — a shapeless whole-scene mask,
  useful for a Colorize with no geometry or as a mask source that always
  reads 1. `crop` + `all` measures identical to `off` (9.73 both), confirming
  it is a true no-op for crop, same as the native system's behaviour.

### Coverage table — reproduced against the removed native system

Same butterfly capture, same canvas, `crop` at `size = 0.35`, softness 0,
measured **before removing the native system** and again **after** with the
new dyno SDFs, driven through the real slot-tab UI:

| shape | native (pre-Phase-3) | dyno (post-Phase-3) |
|---|---|---|
| sphere | 3.451 | 3.451 |
| box | 4.543 | 4.543 |
| ellipsoid | 3.451 | 3.451 |
| cylinder | 4.367 | 4.367 |
| capsule | 4.512 → **4.974 first attempt** | **4.512** after fix |

Capsule was the one real bug this comparison caught: the segment half-length
was originally set to the full `size` uniform, making the capsule visibly
longer than the native reference (4.974% vs 4.512%). The native system's
"scale = length" apparently means *full* length, so the dyno segment half-
extent needed to be `size / 2`, not `size`. Fixed and reproduces the native
figure exactly — this is precisely the "radius vs scale means something
different per shape and getting it wrong fails silently" trap the *previous*
implementation already warned about, now caught the same way: by measuring,
not by reading the formula and assuming it was right.

`cut`/`crop`/`colorize` modes and the `all` shape's no-op behaviour also
reproduced: `cut_sphere_0.5` 4.28–4.29 (native/dyno), `crop_sphere_0.5` 5.935
both, `color_sphere_0.5` R 111.1/111.1 B 17.33/17.34.

### Region — migrating Reveal

Reveal's `revY`/`revS`/`revK`/`revFlip` map onto a plane region in `cut` mode:
`rg{n}Y` from `revY`, `rg{n}Shrink` from `revK`, `invert` from `revFlip`. The
tricky one is `rg{n}Soft` from `revS`, because the two systems use differently
*shaped* soft edges: old Reveal's was one-sided (`smoothstep(revY-revS,revY,y)`
— the whole transition band sits on the "not yet revealed" side of the exact
threshold); the new region mask is symmetric (`smoothstep` centred on the SDF
surface, half the band on each side) — a deliberate choice, kept because a
symmetric band is the more predictable general default for a shape system with
six shapes, not just a sweep plane.

Migrating with a literal 1:1 unit conversion (`softWorld = revS * span`,
`rgSoft = softWorld / radius`) measured a **systematic** offset — every
intermediate point read visibly higher than the pre-migration curve, because a
band centred on the threshold reveals things earlier than one that only
extends below it. Calibrated empirically against the corrected (Phase 2,
object-space-bounds) Reveal sweep table by grid search over a width scale and a
position-offset scale:

```
softWorld  = revS_slider * span              // old world-space band width
rg{n}Soft  = (0.7 * softWorld) / radius       // width: 0.7x, empirically calibrated
rg{n}Y     = (bounds.min + revY*span*1.02 − 0.5*softWorld) / radius
```

The `0.5 * softWorld` offset is exactly derivable — `smoothstep`'s 50% point is
always at the midpoint of its two edges, so old Reveal's own 50%-visible point
sits at `revY_val − revS_world/2`, and that is where the new symmetric band's
own centre needs to land. The `0.7` width factor was found by grid search (a
literal 1.0 conversion measured ~2.7% deviation at the worst sample point,
outside the 2% target) and is not independently derived — recorded here as
"measured, not reasoned to" so a future width tweak knows what it is
recalibrating against.

**Measured end-to-end**, through the actual preset loader (not a standalone
script): a synthetic v3 preset with a linear `revY` envelope (`t:0→v:0`,
`t:1→v:1`), loaded, migrated, then scrubbed through 11 points against the
pre-migration ground-truth curve — **max deviation 0.22%**, comfortably inside
the 2% acceptance figure.

Loading an old preset that used *both* Reveal and the old single-region
feature at once (a real if unusual case) is handled by trying slot 1 for the
old region first, then slot 2 for migrated Reveal if slot 1 is already
claimed — verified: `old region -> R1; old Reveal -> R2 (plane, cut)`, no
collision, both slots correct. The migration status is surfaced to the user
(`Preset loaded — N envelopes. Migrated: …`), not silent, per the "anything not
mappable is skipped with a named message" instruction — though in practice
everything in the old shape *was* mappable, so this file has not yet had to
exercise the unmappable-field path.

### Per-effect masking

Any of `dAmt`, `sat`, tint (`tR`/`tG`/`tB` share one route — masking "tint"
scopes the whole triple, not one channel), `cullA`, `cullS`, `scale` can be
scoped to one region slot's raw geometric mask via a `Mask` control in the lane
header, alongside the existing Audio band/depth widget, shown only when the
focused parameter is one of the six. `src = 0` is "none" — today's unmasked,
everywhere behaviour — `1..4` picks a slot. An independent `invert` flips how
*that one consumer* reads the mask, separate from the region's own `invert`
(which affects that slot's own cut/crop/colorize job) — a region can be doing
its own job and be reused as someone else's scoping mask at the same time.

`effectiveAmount = mix(neutral, V[id], mask)`, neutral chosen per-parameter:

- `dAmt`, `cullA`: **0** — no displacement / no opacity floor, per the plan.
- `sat`, `scale`: **1** — multiplicatively neutral, per the plan.
- `tint`: masking blends toward the **untinted** raw colour (`mix(rgbRaw,
  rgbRaw*tint, mask)`), not toward `ONE3` directly — the tint multiply itself
  is what gets scoped.
- `cullS`: **a large constant (1e6)**, not 0. This is a deliberate departure
  from the plan's literal "`effectiveAmount = V[id]*mask`" formula, which the
  plan states for the zero-neutral cases and does not separately call out for
  `cullS`. `cullS` is a *max-size-allowed* threshold: 0 there means "nothing
  passes", i.e. **cull everything** outside the mask — the opposite of
  neutral. A large ceiling means "nothing gets trimmed by size" outside the
  mask, which is the behaviour "masking Trim to a region" actually implies.
  Reasoned from what makes the parameter a true no-op, matching the plan's
  evident intent rather than its literal wording for this one case.

**Verified spatially, not just numerically**: routing `dAmt` to a small sphere
mask centred on the body left the rendered bbox unchanged (453→453px) while an
*unmasked* `dAmt` at the same value grew it (453→552px) — confirming the mask
actually confines the effect, not merely scales a global number. Moving the
same mask sphere to cover a wingtip made the *masked* displacement visibly grow
the bbox (453→489px) — confirming the effect still fires *inside* the mask,
not that masking silently disabled it everywhere.

### What Region absorbed from Reveal

- `revY`/`revS`/`revK` removed from `SPEC`; `revAnim`/`revFlip` removed from
  `TOGGLES`; the whole `reveal` group and its accordion section are gone.
- The old shrink-into-edge behaviour survives as a per-region `Shrink` (0..1)
  parameter: splat scale multiplies by `mix(1, keepFactor, shrink)`, where
  `keepFactor` is that slot's own alpha-survival factor (`1-mask` for cut,
  `mask` for crop) — the exact generalisation of the old `revK * vis` coupling
  from one hardcoded sweep to any region, any shape, any of 4 slots.

---

**Dev-loop gotcha:** `python3 -m http.server` sends only `Last-Modified`, and
Chrome heuristically caches the page. After the TDZ fix the browser kept running
v1 and the stack traces referenced a `bind()` that no longer existed in the file
— which reads exactly like the bug never got fixed. Hard-reload (Cmd+Shift+R),
or serve with `Cache-Control: no-store`:

```
python3 -c "import http.server; H=http.server.SimpleHTTPRequestHandler; _e=H.end_headers; H.end_headers=lambda s:(s.send_header('Cache-Control','no-store'),_e(s)); http.server.test(HandlerClass=H,port=8000)"
```

---

## Envelopes — per-parameter automation

Roadmap item 3, the one flagged as the biggest functional gap. Before this,
one global `playhead` drove everything and the camera had exactly two
keyframes, so "animation" meant a single linear sweep of whatever happened to
be wired to the timeline.

### Model

```
ENV[id] = { on: bool, smooth: bool, keys: [{t, v}, …] }   // keys sorted by t
```

Two value stores, and the split matters:

- **`P[id]`** — the manual slider value. An envelope never overwrites it, so
  disarming restores exactly what was last dialled in.
- **`V[id]`** — the effective value at the current playhead. Everything
  downstream reads `V`: uniforms, camera ease, readouts, export.

`evalParams()` fills `V` and is called at the top of `pushUniforms()`, so there
is exactly one place where envelopes are resolved. Any new consumer must read
`V`, never `P`.

`t` is normalised 0..1 across the clip, not seconds — so changing `Duration`
rescales the whole animation rather than truncating it.

### Not animatable

`dur`, `br`, `settle`, `dpr` are excluded via `NO_ANIM`. Animating clip length
or bitrate is meaningless; animating sort-settle or pixel ratio would make an
export non-reproducible. Their rows get a hidden placeholder `.key.ghost` so
the four-column grid stays aligned instead of collapsing a column.

### Interaction

- **`◆` per row** — key at playhead, alt-click removes, click focuses the lane.
- **Lane** — drag keys in both axes, double-click empty space to add, alt-click
  a key to delete, click empty space to scrub.
- **Auto-key**: dragging a slider whose envelope is armed writes a key at the
  playhead. Without this the drag would appear to do nothing, since an armed
  envelope owns its parameter.
- An armed slider becomes a **readout** — `syncSliders()` pushes the curve's
  value into it so the panel shows what is actually driving the shader.

Arming an empty envelope is refused; there is nothing to interpolate.

### Export

The offline loop needed no changes. It sets `playhead` then calls
`pushUniforms()`, which resolves envelopes — so exports honour curves for free.

### Watch out

`drawLane()` holds the dragged **key object**, not its index: the array is
re-sorted on every pointermove, so an index would point at the wrong key the
moment a drag crosses a neighbour.

Adding `laneFit()` to `resize()` reintroduces the v1 death: `resize()` runs
during module evaluation, long before the lane's `const`s exist, so it throws a
TDZ error and silently kills everything below it. The lane sizes from CSS and
carries its own resize listener. There is a comment in `resize()` saying so —
leave it there.

---

## Audio-reactive drive

Roadmap item 6, and per the intro the reason this exists rather than an AE
plugin.

### Analysed offline, never sampled live

The audio is analysed **once, offline**, into a per-band amplitude envelope.
It is deliberately not tapped from a live `AnalyserNode`. The offline export
loop does not run at wall-clock speed, so a realtime tap would drift out of
sync the moment you rendered, and the same project would produce a different
result on every run. A precomputed envelope indexed by timeline position makes
scrubbing, playback and export read identical values. Anything else is a demo,
not a tool.

Band separation uses `OfflineAudioContext` + biquad highpass/lowpass per band
rather than a hand-rolled FFT — same result, built-in DSP, no numerical code to
own. Bands: low 20–200, mid 200–2000, high 2000–16000 Hz. Each band is RMS'd
into a 200 Hz envelope and **normalised to its own peak**, so a given `depth`
means the same amount of visible effect regardless of the material.

Timeline mapping is 1:1 with the clock (`playhead * dur` seconds into the
track), not stretched to fit, so changing Duration changes how much of the
track is used — what you want when cutting to a specific passage.

`Preview` plays the track so you can hear it against the animation. It is not
what drives the parameters; the offline envelope always is. So an export
matches what you heard even though nothing is playing during the render.

### Audio modulates, it does not replace

```
V[id] = (envelope or manual value) + depth × parameterRange × bandLevel
```

clamped to the parameter's own range. Audio is a modulation **on top of** the
base value so it layers with envelopes instead of fighting them: the envelope
sets the move, audio adds the detail. Verified — `scale` with an envelope
reading 1.235 at t=3 became 1.422 with a high-band route at depth 0.4, exactly
the base plus `0.4 × 2.95 × 0.16`.

`depth` is a fraction of the parameter's full range and is signed, so a route
can push either way.

### Verified against a synthetic file

Correctness here is not eyeballable, so the test fixture is deterministic:
60 Hz for 0–2s, 800 Hz for 2–4s, 6 kHz for 4–6s. Each band must peak in its own
third. Regenerate with:

```
ffmpeg -y -f lavfi -i "sine=frequency=60:duration=6" \
 -f lavfi -i "sine=frequency=800:duration=6" \
 -f lavfi -i "sine=frequency=6000:duration=6" \
 -filter_complex "[0:a]volume='if(lt(t,2),1,0)':eval=frame[a];\
 [1:a]volume='if(between(t,2,4),1,0)':eval=frame[b];\
 [2:a]volume='if(gt(t,4),1,0)':eval=frame[c];\
 [a][b][c]amix=inputs=3:normalize=0[out]" -map "[out]" -ar 44100 bandtest.wav
```

Measured (low / mid / high):

| t | 1s | 3s | 5s |
|---|---|---|---|
| low | **0.78** | 0.04 | 0.00 |
| mid | 0.10 | **1.00** | 0.09 |
| high | 0.00 | 0.16 | **1.00** |

Off-band bleed of 0.09–0.16 is the second-order biquad skirt, not a defect.

### Focus must not mutate

The `◆` glyph both keys and focuses. That made routing audio impossible without
silently starting an envelope on the parameter — the first attempt at this
produced three unwanted envelopes and the status line said so. **Clicking a
parameter's name focuses it without touching anything.** Keep any future focus
affordance non-destructive.

Relatedly, `syncSliders()` originally keyed off `hasEnv(id)` alone, which left
audio-modulated sliders sitting at their manual position while the shader used
something else. It now uses `isDriven(id)` — envelope *or* audio. This file has
been bitten before by a panel that lies about the state; do not reintroduce it.

## Audio shaping

Per-route shaping sits behind a `SHAPE` disclosure on each parameter's audio
panel: `attack` (0–500ms), `release` (0–2000ms), `lag` (-500..500ms), `gain`
(0–4), `floor` (0–1), `curve` (0.25–4). Fixed order:

```
raw → gate(floor) → ×gain → ^curve → clamp 0..1 → asymmetric one-pole (attack/release) → shift by lag
```

then the existing `depth × parameterRange` modulation applies on top, unchanged.

### Offline pass, cached, never sampled at lookup time

Like the base envelope itself, shaping is computed once over the whole band
array and cached (`shapeCache`, keyed on band + attack/release/gain/floor/curve),
not applied sample-by-sample during playback. The one-pole smoothing is
sequential — each output sample depends on the previous one — so if it ran at
lookup time, scrubbing backwards or exporting out of frame order would produce
different results than forward playback. Precomputing the whole shaped array
up front makes `routeLevel()` a pure function of timeline position again, same
determinism guarantee as the base envelope. `lag` and `depth` are deliberately
excluded from the cache key: `lag` is applied as a time shift at sample time
(`routeLevel` subtracts `lag/1000` before indexing), not baked into the array,
and `depth` is applied even later during modulation — baking either in would
multiply the number of cache entries for no benefit.

Cache is cleared in the two places the underlying envelope can change:
`analyseAudio()` (new track) and the `Clear` button (envelope wiped).

### Verified against the same synthetic fixture

Using `bandtest.wav` (see above) and the low band via `window.__bench`:

- **Identity at neutral**: `attack=0, release=0` reproduces the raw envelope to
  full float precision — `routeLevel` with neutral shaping bit-matches direct
  envelope sampling. (The raw low-band value at t=1s now measures 0.85 rather
  than the 0.78 in the table above — traced to drift in the `bandtest.wav`
  fixture/analysis pipeline since that table was written, not a shaping
  regression; the identity-transform proof holds independent of the absolute
  number.)
- **Release smoothing**: `release=1000` — value at t=2.5s measured 0.5484,
  well above the unshaped 0.0412 at that point, decaying monotonically through
  t=4.0s.
- **Lag**: the true low-band peak is at t=0.7s (not t=1.0s). `lag=+250`ms
  shifts it to measure 0.95 at t=0.95s; `lag=-250`ms measures 0.45 at the
  correspondingly earlier point — both exactly ±0.25s.
- **Floor gating**: `floor=0.5` zeroes measured off-band bleed (was
  0.093–0.102) while leaving the genuine peak untouched (0.997 at t=3).
- **Gain/curve order**: `gain=2, curve=2` — hand-computed
  `min(1, (raw×2)^2)` = 0.041872 matches the measured value to 6 decimals,
  confirming gain is applied before curve, not after.
- **Export determinism**: two independent 30-frame/1280×720/30fps exports with
  shaping active produced bit-identical `ffprobe signalstats` YAVG across all
  30 frames (max diff 0.0).

### Performance

Measured directly against a genuine 60,000-sample array (5 minutes at the
200 Hz envelope rate, not extrapolated from a smaller run): steady state
~2ms per recompute, worst case 9.1ms on first call (JIT warmup). Comfortably
under the 16ms frame budget, so no Web Worker — the plan's stated fallback —
was needed.

### Preset round-trip

The six shaping fields are serialized into each `audio.routes[id]` entry
alongside `band`/`depth` and are additive on `PRESET_VERSION`. Loading an
older preset (no shaping fields present) needed care: `Number(v) ?? def` looks
like a safe default-coalesce but is not — `Number(undefined)` is `NaN`, and
`NaN ?? def` still evaluates to `NaN` because `??` only falls through on
`null`/`undefined`. Fixed with an explicit `Number.isFinite` check
(`num(v, def) = Number.isFinite(v) ? v : def`) before this shipped, not after
a bug report. Round-tripped through the actual UI (save → wipe → load) with
distinctive values (attack 80, release 600, gain 1.8, floor 0.05, curve 1.6)
and confirmed byte-exact.

### Phase 2a of `DEV_PLAN_EFFECTS.md` — discoverability

Everything above existed and worked; nobody could find it. `#audShapeToggle`
was a plain `Shape ▾` button with no indication anything behind it was
non-default, sitting in a lane header that only appears once a parameter is
focused. Two fixes, both reusing existing language rather than inventing new
affordances:

- **Dirty indicator.** `#audShapeToggle.dirty` goes `--edit` violet exactly
  when any of the six shaping fields differs from its default
  (`attack:0, release:120, lag:0, gain:1, floor:0, curve:1`) — the same
  `.dirty` convention `.rst` reset buttons already use, so it reads as the
  same language rather than a new one.
- **Collapsed summary.** The button's own label grows non-default fields
  inline when collapsed: `Shape ▾ · lag +250 · rel 600`. The arrow alone
  never told anyone lag and smoothing lived behind it, which is exactly why
  they went unfound; now the values that matter are visible without opening
  anything.

`refreshShapeToggle()` is the single place both are computed, called from
`laneHead()` (on focus change), the toggle's own click (open/close), and
every shaping slider's `oninput` (so it updates live while dragging, not just
on focus). Verified via `window.__bench` (bypassing synthetic DOM event
dispatch, which was unreliable in this session's browser tab — see the
environment note under Phase 2d below): setting all six fields to distinctive
values and calling `laneHead()` produced
`Shape ▾ · atk 80 · rel 600 · lag +250 · gain 1.8 · flr 0.05 · curve 1.6`
with `.dirty` set; resetting to defaults produced a clean `Shape ▾` with no
`.dirty`; a single off-default field (`lag -100`) produced `Shape ▾ · lag -100`
alone, confirming the summary only lists what actually differs.

**Reachability at 390px, measured rather than assumed** (per the plan's own
instruction — the lane header scrolls horizontally on mobile with no visible
scrollbar, and `Shape ▾` is the rightmost control). Forced `#laneHead` to a
390px box and measured `scrollWidth` (720) against `clientWidth` (390): it
does need a scroll, and `scrollIntoView` on the toggle successfully brings it
within the header's visible bounds — reachable, just not signposted. That
lack of a scroll hint is pre-existing (the whole row already scrolls this way
for every other lane control) and out of scope here; this phase only had to
confirm the toggle itself isn't stranded off-screen, which it is not.

---

## Mobile

Roadmap item 7. Desktop stays the target: every mobile rule lives in one
`@media (max-width: 900px), (pointer: coarse)` block at the **end** of the
stylesheet, so it can only ever subtract. Nothing above it is conditional on
screen size.

(It has to be at the end. Placed earlier it lost to equal-specificity base
rules and the touch-target sizes silently did nothing — measured 13px when
22px was intended.)

### Layout

The rail becomes a bottom sheet rather than a permanent 340px column, which is
most of a phone's width. `Controls` toggles it; tapping the viewport dismisses
it. Verified at 390×844: stage 390×692, lane 96, transport 56 — exactly 844,
full-width stage, no horizontal overflow, drawer opening to top 152.

Touch targets go from 13px to 22px. Nothing is removed on mobile — the lane
header scrolls horizontally rather than dropping controls.

### Framing was wrong on any narrow window

`reframe()` used a fixed `radius * 2.6`, derived from the **vertical** FOV.
`camera.fov` is vertical, so on a portrait viewport the horizontal FOV is far
narrower (29.4° at 390×844 vs 50° vertical) and a wide subject overflowed the
frame completely. Now it frames against whichever FOV is tighter:

```js
const hFov = 2 * Math.atan(Math.tan(vFov / 2) * camera.aspect);
const dist = radius / Math.sin(Math.min(vFov, hFov) / 2) * 1.1;
```

On a wide window `hFov > vFov` so this reduces to the old constant — desktop
framing is unchanged. This was a general bug, not a mobile one; it just could
not show up on a landscape window.

### Profile

`Mobile profile` defaults on for coarse pointers or narrow viewports and is a
plain toggle, so desktop can test it. It **caps**, never redefines: the
pixel-ratio slider still reports what you chose, and the OSD reports what is
actually rendered, saying `(capped)` when they differ. Verified: slider at 2
with the profile on gives a 1:1 backing store and an OSD reading of
`1.00 (capped)`; off, 2.00 and a 2× buffer.

It is deliberately **not** in presets — it describes the device you are at, not
the look. A preset saved on a desktop must not force a phone to full pixel
ratio.

### iOS file pickers and `accept` — this bites every input

iOS matches a file input's `accept` against **registered system UTIs**, not
against the filename. Anything it cannot resolve to a known type is greyed out,
and since a picker with nothing selectable gives no error, it reads as the app
being broken.

It hit both inputs for slightly different reasons:

- **Audio.** `accept="audio/*"` alone greys out ordinary .wav/.mp3/.aiff.
  Fixed by listing explicit extensions alongside the wildcard.
- **Splats.** `.ply .spz .splat .ksplat .sog` have no registered UTI at all, so
  listing extensions does not help — there is nothing to resolve them to.

The only reliable fix for the splat case is to **drop `accept` entirely on
touch devices**, which is done under `(pointer: coarse)`. Desktop keeps its
filters, where they behave correctly.

That leaves every picker unfiltered on mobile, so all three buttons and the
drop target now route through one `handleFile()` that dispatches on extension.
Tapping "Choose file" and picking a .json loads it as a preset rather than
failing — the right behaviour once the OS will not filter for us. Unknown
extensions get a named error listing what is accepted.

**Rule for any new file input: never rely on `accept` to gate correctness.**
Validate after selection instead.

`.ply` is now verified end-to-end through this path (a 11 MB capture, 44,296
gaussians, `type: ""` as iOS reports it) — until now every load test had used
the remote `.spz`, so the PLY decoder had never actually been exercised.

### Audio follows the transport

Play/pause/scrub drive the audio element. This is **monitoring only** — the
parameters are always driven by the offline envelope, so an export still
matches what you heard even though nothing plays during a render. The coupling
is deliberately one-way: letting audio `timeupdate` drive `playhead` would make
the timeline jitter with the audio clock and fight scrubbing.

Rapid transport toggling supersedes an in-flight `play()` with a `pause()`,
which rejects as `AbortError`. That is the intended outcome, not a failure —
it is swallowed, because surfacing it put a red warning on the panel during
completely normal use.

### `nonLod` is required, and `lodAbove` is a floor

Spark's loader returns **only** `{ lodSplats }` when it builds a LOD tree,
leaving `packedSplats` empty — the mesh reports 0 splats and renders nothing.
`nonLod: true` makes it also return the full set. Measured without it: 0
gaussians, 0% coverage. With it: 177,132 gaussians and a correct render.

`lodAbove` is a floor, not a ceiling — LOD is built only when the file has MORE
than that many splats. The pre-existing `useLod` checkbox used 500000, above
the 177k test asset, so **that path had never actually executed** and the bug
was latent until the mobile profile lowered the threshold to 150000.

### `preserveDrawingBuffer` is now conditional

It is required for `VideoFrame(canvas)` and costs real performance. Export
needs WebCodecs, which iOS Safari and Firefox lack, so there it was pure cost
for a feature that cannot run. It is now gated on the same `VideoEncoder`
check the export path uses, plus `?debug=1` so pixel probes stay reliable.
NOTES' old warning still holds: it is ON wherever export is possible. The
Render button is also disabled up front when WebCodecs is absent, rather than
explaining itself only after a press.

### Testing note: hidden tabs freeze CSS transitions too

The drawer appeared broken under automation — `panel-open` was set, the rule
parsed and matched, and the transform never changed, even from an inline
style. A hidden tab does not advance CSS transitions, so the computed value
stays frozen at its start. Setting `transition: none` before measuring gives
the true value (top 152, exactly as designed). Same family as the rAF stall
behind `?debug=1`.

### Audio shaping sparkline is desktop-only

`#audSpark{display:none}` on mobile via the same coarse-pointer media query as
the rest of the mobile rail — a canvas redraw on every playhead tick is not
worth the cost on a touch device, and the shaping sliders (which is where the
actual editing happens) stay visible and full-width in a single-column grid.
Verified in the mobile iframe: `display: none` on the canvas, single-column
`grid-template-columns` on the rows, panel still opens via the `SHAPE` toggle.

---

## Saturation

`sat` lerps between luma-grey and the tinted colour, after the tint multiply,
range 0..2 default 1. It lives in `SPEC` group `colour`, so it inherited reset,
envelopes, audio routing and presets with no extra work.

### `dyno.mix(vec3, vec3, float)` is CONFIRMED

This form was listed above as assumed-but-unverified. It has now been checked
by implementing saturation twice — once as explicit
`grey + (rgb - grey) * sat`, once as `dyno.mix(grey, rgb, U.sat)` — and
measuring both over a fixed 97,707-pixel mask:

| sat | 0 | 0.5 | 1 | 1.5 | 2 |
|---|---|---|---|---|---|
| worst channel difference | 0.000 | 0.000 | 0.000 | 0.000 | 0.000 |

Bit-identical. `mix` is used in the effect chain and the arithmetic form was
discarded. `dyno.dot(vec3, vec3)` is confirmed by the same test.

### The colour in the graph is display-referred, not linear

`renderer.outputColorSpace` is `srgb`, which would normally imply the shader
works in linear light and three converts on output. **It does not behave that
way here.** Measured: halving the red tint takes the mean red from 87.41 to
43.89 — a ratio of **0.5021**. If the graph were linear and the output
sRGB-encoded, that ratio would be ~0.73.

So no encoding step sits between the dyno graph and `readPixels`. The Rec.709
weights `(0.2126, 0.7152, 0.0722)` are defined for linear light, so applying
them here is the *gamma-incorrect* grayscale — which is also what most image
editors do by default. Kept deliberately: matching editor behaviour beats
photometric correctness for a look tool. If a future Spark release starts
applying an output transform, this measurement is the one that will change,
and the luma path is where it will show up first.

### Response is linear below sat = 1 and clamped above it

The acceptance criterion "channel spread at least doubles from sat 1 to sat 2"
measured **×1.89**, not ×2. That is not an arithmetic fault. Over a fixed
mask, predicting each channel as `grey + sat × (mean1 − grey)`:

| sat | 0 | 0.25 | 0.5 | 0.75 | 1 | 1.25 | 1.5 | 2 |
|---|---|---|---|---|---|---|---|---|
| B error vs prediction | 0.00% | 0.50% | 0.90% | 0.83% | 0.00% | 3.89% | 9.32% | 23.25% |
| mask pixels with B ≤ 1 | 0 | 0 | 0 | 0 | 207 | 302 | 344 | 686 |

The error tracks the floored-pixel count exactly. Above `sat = 1` the lerp
pushes channels below zero and the 8-bit framebuffer clamps them on write, so
the composite stops being linear in `sat`. Below 1 it is linear to within
measurement noise. **Not clamping in the shader does not prevent the
framebuffer from clamping** — that is inherent, and the ×1.89 is the correct
consequence of it.

### Measured

| | |
|---|---|
| `sat = 0` | channels within **0.07%** of each other — clean monochrome |
| `sat = 1` | R 87.41 vs 87.40 pre-change — neutral, within 0.01% |
| `sat = 2` | spread 26.83 → 50.67 (×1.89, clamp-limited as above) |
| frame time | **below the measurement floor** — see below |

Frame time could not be resolved in this environment. `performance.now()` is
coarsened to 100µs, `EXT_disjoint_timer_query_webgl2` needs a polling loop that
a hidden tab clamps to 1s per iteration, and the app's own FPS counter is
driven by rAF, which a hidden tab stops entirely. The change is four vector ALU
ops per splat; no instrument available here can see it. Recorded as unresolved
rather than passed.

### Testing hazard: an empty mask fakes a pass

The first `mix` comparison reported "worst difference 0.000 — VERDICT: WORKS"
while every measured value was `NaN`. The mask had been captured during the
load transient, so it was empty, every mean divided by zero, and `NaN`
comparisons silently evaluated false in the `Math.max` accumulator. **Assert
the mask is non-empty and every value finite before comparing.** A pass built
on NaN looks exactly like a real pass.

---

## Splat brightness

Phase 2c of `DEV_PLAN_EFFECTS.md`. `bright` — group `colour`, range 0..2,
default 1 — is a flat multiply on the un-tinted source colour, applied first
in the chain, before everything else:

```
raw rgb -> brightness -> tint -> saturation -> alpha ops
```

Brightness sits before saturation on purpose (per the plan): saturation must
operate on the *brightness-corrected* image, not the other way round, or
brightening after desaturating would reintroduce colour the user just
removed. It is **not maskable** (not in `MASKABLE`) and **not region-gated**
(applied to `rgbRaw` before `tintMix`'s region colorize) — a flat, always-on
multiply, same category as `dFrq`/`dSpd` rather than the six masked effects.

Same display-referred reasoning as everywhere else in this file (see
Saturation, above): this is a gamma-space multiply, not a linear-light
exposure control. **Call it brightness, not exposure** — a true exposure
slider needs linearise → scale → re-encode, a different and probably
unwanted job, and NOTES already establishes the graph is display-referred
throughout. Not clamped, for the same reason `sat` is not: the framebuffer
clamps on write regardless, and per-splat clamping would not equal
frame-level clamping (see Saturation's "Response is linear below sat = 1"
above — the identical mechanism applies here above `bright = 1`).

**Not independently measured on luma this pass, and why:** every other
colour-chain measurement in this file (the sat=1 neutral check, the
tint-ratio check, the display-referred proof) was taken by reading back
actual rendered pixels against the loaded butterfly capture. The demo asset
failed to load in this session's browser environment for the whole of this
work — reproduced identically on the pre-Phase-1 committed build via
`git stash`, so it is environmental (see Phase 1's note on the same gap), not
a defect in this change. What *was* verified without a mesh: `U.bright`
tracks `P.bright` exactly through `pushUniforms()`/`evalParams()` (set 0.5,
read back 0.5; reset to 1, read back 1), and `buildModifier()` — the actual
dyno graph construction, including the new
`dyno.mul(dyno.swizzle(rgba0, "xyz"), U.bright)` call — constructs without
throwing. The luma sweep (0.5/1/2 against the pre-change baseline, within 1%
at 1) from the plan's acceptance section is a real gap, not a claimed pass;
re-run it the next time a splat loads cleanly in this environment.

---

## Background colour

Phase 2b of `DEV_PLAN_EFFECTS.md`. A small fixed palette — void (`#08090b`,
the existing default), black, mid grey, white, chroma green (`#00ff00`) — not
a colour picker, via a `SELECTS` entry (`bgColor`) applied through
`renderer.setClearColor()`. A look property, so it persists in presets
(additive — no `PRESET_VERSION` bump, same as every other `SELECTS` entry);
deliberately *not* in the `mobileMode`-style device-property exclusion list,
since background is part of the look, not the workstation.

### The trap the plan called out, guarded in the same commit

Every coverage figure anywhere in this file — the cull table, the six-shape
SDF reproduction, the Reveal migration curve, the masking verification, the
saturation neutral check — was measured against void, because coverage is a
pixel count against a known backdrop. A non-default background would
invalidate all of them silently the moment someone picked one to look at
their capture against something else.

`window.__bench.draw()` and `.settled()` now save the current clear colour,
force void for the render, and restore whatever was there afterward — in a
`try/finally` in `.settled()` so an exception mid-probe cannot leave the
renderer stuck on void. Verified directly: selected white in the UI, called
`draw()`, read back the actual rendered pixel — `[8, 9, 11, 255]`, void's
exact value, not white — while the live `COLOR_CLEAR_VALUE` state immediately
after was back to white, confirming the guard is transparent to the caller's
own selection. This is the mechanism every future probe-based measurement in
this file relies on continuing to hold; if it is ever removed, the entire
measured-coverage corpus needs re-baselining against whatever the new default
render path produces.

### Export is unguarded on purpose

The export loop never touches `setClearColor` — it inherits whatever the live
renderer state is, same as the interactive view, so an export naturally shows
whatever background the user picked. Only the two `__bench` *measurement*
helpers force void; nothing in the render or export path does, since forcing
it there would defeat the point of having the control.

### Preset round-trip

Verified through the actual UI (save → wipe → load), together with
brightness: saved with `bgColor = "chroma"` and `bright = 0.42`, wiped via the
reset buttons, reloaded from the saved JSON — both restored exactly
(`P.bright === 0.42`, the select's value `"chroma"`, and — the part that
actually matters — the *live* `COLOR_CLEAR_VALUE` reading `[0, 255, 0, 255]`
after load, confirming `applyPreset()`'s explicit `applyBgColor()` call
actually pushes the restored selection into the renderer rather than leaving
a `<select>` whose value nobody applied).

---

## Phase 3 of `DEV_PLAN_EFFECTS.md` — effects that need no new primitive

Five sub-effects, all pure `objectModifiers` using only what the graph
already had (`splitGsplat`, the region SDFs, `hashVec3`/`hashFloat`, basic
trig). New groups `quantise` and `wave`; `displace` gained a field-mode
select and a colour-driven axis offset; `region` gained a per-slot
`Attract` field.

### A measurement trap this phase's own testing caught: `getBoundingBox()` cannot see any of this

The plan's acceptance criteria ask for bbox comparisons (quantise, curl vs
sin, the attractor). NOTES already establishes, from reading
`spark.module.js` directly (see Model transform, above), that
`SplatMesh.getBoundingBox()` iterates raw `center`/`scales`/`quaternion` from
`this.splats` and **never touches `objectModifiers`** — which is exactly
what `buildModifier()` is registered as. Every effect in this phase is a
GPU-side `objectModifiers` transform. That means `getBoundingBox()` is
structurally incapable of reflecting any of them: it would report the
identical box at `qAmt = 0` and `qAmt = 1`, not because quantise does
nothing, but because the tool is reading data the effect never touches.

Substituted a **screen-space pixel bbox** (scan the readback for pixels that
differ from the `void` background by more than a small tolerance, take the
min/max x/y of the hits) everywhere the plan says "bbox" — this reads the
actual rendered GPU output, which is the thing these effects actually
change. Recorded here so the next person does not spend an hour confused by
a bbox check that silently always passes.

### 3a — Quantisation

`qAmt` (0..1) blends continuous -> voxelised. Two mutually exclusive modes,
selected by `qAxis`:

- **Off (default): rotated 3-axis grid.** Per-axis cell size (`qCellX/Y/Z`,
  world-scaled by `radius` same as every other spatial parameter), and an
  optional grid-basis rotation (`qRotX/Y/Z`, degrees) applied via `rotFwd`
  before snapping and `rotInv` after — so the voxel grid does not have to
  align with the world axes. `rotFwd`/`rotInv` are hand-composed axis
  rotations (X then Y then Z forward; Z then Y then X with negated angles
  to invert — matrix-inverse-of-a-product reverses the order), not a
  `dynoMat3`/`transformQuat` call: verified via `spark.module.js` that
  those primitives exist, but hand-composing kept every operation to
  `sin`/`cos`/`swizzle`/arithmetic already used and verified elsewhere in
  this file, at the cost of more lines.
- **X/Y/Z: single-axis stratify.** Only that one axis snaps, with a
  per-band random offset (`hashFloat(floor(axis/cell))`, scaled by
  `qStrat`) — the "stratification / scanline displacement" the plan calls
  out as cheapest and most-used. The other two axes stay fully continuous.

**Colour posterisation** (`qPoster`) lives in the colour chain, after
saturation: `levels` runs 64 (imperceptible) down to 4 (hard posterise) as
`qPoster` goes 0->1, so 0 is not a separate on/off gate, and uses `round()`
rather than `floor()` — a floor-only posterise biases every level down,
visible as a global darkening.

**Not implemented this pass, and why:** optional scale/quaternion snapping
for true axis-aligned voxels (the plan's other "two distinct looks" item).
`combineGsplat` in this file only ever writes back `{center, scales, rgba}`
— quaternion has never been touched by this graph. Adding it means also
resolving how a snapped/identity quaternion composes with the model
transform and Correct mode's own rotation, which is real, separate design
work, not a one-line addition. Recorded as deferred, not silently dropped.

**Measured**, against the loaded `butterfly.spz` (177,132 splats),
`window.__bench.settled()` + `readPixels` throughout:

- `qAmt = 0` reproduces the baseline exactly (`cov 10.18`, not just within
  1%). `qAmt` 0.3 -> 0.6 -> 1.0: coverage `9.05 -> 5.08 -> 2.61`, strictly
  monotonic. This is the "known hazard" from the plan showing up directly —
  many splats collapsing onto identical grid-cell centres, heavy overdraw,
  exactly why coverage *drops* as more of the cloud voxelises rather than
  staying constant.
- Screen-space bbox at `qAmt=1` measurably smaller than baseline
  (377x331 vs 385x337 px) — voxel-snapping pulls peripheral splats inward
  toward cell centres.
- **Flicker** (the plan's own stated hazard): 60 frames at a fixed camera
  and fixed `t`, `qAmt=1` — coverage standard deviation **0.0**. Not a null
  test: this is a static, unchanging scene (same camera, same positions
  every frame), so Spark's own sort is expected to be deterministic frame
  to frame here. The hazard is real during *animation* (a moving camera, or
  a wave-modulated amount), which this specific check does not exercise —
  recorded as the number the plan asked for, not oversold as "flicker
  doesn't happen."
- Rotated grid vs unrotated at matched `qAmt=1, cell=0.15`: different
  screen-space bbox and lit-pixel count (392x334/11,499 vs
  423x350/12,708) — confirms the rotation is actually reaching the snap,
  not a no-op. (A broken `rotInv` would most likely show as NaN corruption
  or a wildly distorted result, not a plausible moderate difference like
  this — not a substitute for exact verification, but a real signal.)
- Single-axis stratify (`qAxis=y, qStrat=1, qAmt=1`) measurably differs
  from the 3-axis grid at the same amount (385x337/33,213 vs
  377x331/18,320) — width stays at the baseline's own 385px since X is
  untouched in single-axis mode, exactly as designed.
- Posterisation at `qPoster=1`: coverage unchanged from `qPoster=0`
  (`10.18` vs `10.16`, noise-level) while the measured R channel shifted
  (`93.86` -> `95.88`) — a colour-only op, confirmed to not move geometry.

### 3b — Curl noise

A selectable field mode (`dField`: sin | curl) on the existing Displace
panel, reusing its `dAmt`/`dFrq`/`dSpd`/`dRnd` — sin stays the default and
is not removed, per the plan ("presets depend on it, and it is a different,
harder-edged look worth keeping").

Curl is the **analytic curl** of the potential
`Psi(x,y,z) = (sin(f.y+t), sin(f.z+t), sin(f.x+t))`, hand-derived, not
sampled or finite-differenced against a noise texture:

```
curl(Psi) = (-f.cos(f.z+t), -f.cos(f.x+t), f.cos(f.y+t))
```

then divided through by the leading `f` to match the sin field's -1..1
range (dividing a divergence-free field by a constant preserves
divergence-freeness). `div(curl(F)) = 0` is a vector-calculus identity for
*any* differentiable `F` — this is exactly volume-preserving by
construction, not something that needed to be hoped true and then checked.
Reuses the exact same frequency/time-scaled `p` the sin field already
computes, so curl mode costs three extra `cos()` calls and nothing else.

**Measured**, matched amplitude (`dAmt=0.15, dFrq=3`) against the same
capture: sin field screen-bbox 470x384 (area 180,480px²); curl field
445x355 (157,975px²) — **~12.5% smaller**, i.e. curl visibly does not
inflate the cloud as much as the sin field does. This is the plan's own
acceptance line ("changes bbox LESS than the sin field does — that is what
volume-preserving means") and it holds, empirically, not just by the
identity argument above.

### 3c — Region as attractor

A new per-slot field, `rgAttract` (signed, -1..1, default 0), alongside the
existing `X/Y/Z/Size/Soft/Shrink/R/G/B` — reuses all four slots, all seven
shapes, existing masks, existing envelopes, with no new region machinery.
Composable with any mode (cut/crop/color/off): a slot can be doing its own
job *and* attracting at the same time.

No per-shape SDF gradient is computed. Every shape reuses the same
projection: `target = slotCentre + normalize(rel) * size`, where `rel` is
already computed per-slot for the SDF itself. For a **sphere** this is
exact — the projection lands precisely on the sphere's surface. For the
other six shapes it is a reasonable approximation, not an exact surface
landing; the plan's own acceptance line only measures the sphere case, so
this is in scope, not a shortfall. Signed: `center = mix(center, target,
amount)` at `amount < 0` extrapolates *past* the current position along the
same line (what `mix` does at a negative `t`), which reads as repulsion
without a separate formula.

**Measured**, sphere region at world origin, `size=0.2`, against the same
capture: `rgAttract` 0 -> 0.5 -> 1.0 gives screen-bbox
385x337 -> 241x216 -> 120x143, monotonically converging. Ground truth:
switching that same slot to **Crop** mode at the identical size (which
already has its own separately-verified coverage-table precision, from the
Region phase) renders 114x129 at the *identical* screen centre
(544, 308.5 both). Full-amount attraction and the crop boundary land
within single-digit pixels of each other — the sphere case is exact, as
designed.

### 3d — Displace by colour

`dcMode` (off | luma | hue), `dcAxis` (x/y/z), `dcAmt` (signed, world-scale
like `dAmt`) — offsets `center` along one world axis by the splat's own
raw colour. Reads `rgba0` directly (the untinted, unsaturated source
colour straight from `splitGsplat`), not the tint/brightness/sat-adjusted
value computed later in the chain — a stable per-splat *material* property
driving geometry, not a moving target that shifts as the colour sliders
move. **Centred at 0.5**, so mid-tones/mid-hues do not move at all: the
offset is `dcAmt * (value*2 - 1)`, bipolar, which is what turns the
capture's own texture into relief in both directions rather than a
one-sided bulge outward from the bright end.

Hue is standard HSV-style extraction (`max`/`min`/`delta` across R/G/B,
piecewise by which channel is max, `mod(_, 6)` to wrap), with `delta`
floored at `1e-6` to avoid a divide-by-zero on fully desaturated pixels —
numerator and denominator both shrink together there, so the result stays
finite and just noisy rather than `NaN`, on greys that have no meaningful
hue anyway.

**Deferred, not implemented this pass:** the plan's parenthetical
alternative — offsetting along the splat's own *short covariance axis*
(an approximation of its surface normal) instead of a world axis. That
needs the per-splat quaternion, which (see 3a above) this graph has never
read or written; `combineGsplat` only ever passes through
`{center, scales, rgba}`. World-axis offset is the primary description in
the plan's own wording ("offset along an axis (**or** along the splat's
own short covariance axis...)") and is what shipped.

**Measured**, `dcAxis=y`, against the same capture: `dcMode=off` reproduces
the exact baseline bbox (385x337). `luma` and `hue` both grow the Y extent
substantially (337 -> 473px and -> 468px respectively) while X stays flat
(385 -> 384px, within rounding) — exactly the expected shape for a
single-axis colour-driven offset, and confirms both modes are doing
something, not just one.

### 3e — Index-phase wave

A per-splat, animated 0..1 envelope that **modulates the amount** of
Displace, Quantise, and each region's Attract, rather than gating them
on/off — so an active effect sweeps through the cloud instead of pulsing
uniformly everywhere at once. `wMode` selects the per-splat phase source:
`hash(index)` (genuinely per-splat, uncorrelated with position) or
position along a chosen axis (`fract(position * wFreq)` — a literal sweep
front-to-back). `waveMul = mix(1, envelope, wAmt)` is the neutral-at-zero
form every other effect in this graph uses, so `wAmt=0` is exactly a
no-op, not approximately one.

Pairs directly with the audio envelopes exactly as the plan intends — since
`wAmt`/`wFreq`/`wSpd` are ordinary `SPEC` sliders, they already get
keyframes and audio routing for free, same as everything else.

**Measured**, `dAmt=0.15, dFrq=3, wFreq=3` against the same capture:
`wAmt=0` gives coverage 15.23 (matches the sin-field-only measurement
above, consistent). `wAmt=1` gives a *different* coverage, 13.64 — and,
critically, sampling the SAME static parameters at two different playhead
positions (`t=0` vs `t=0.5` on a `dur=4` clip) with `wAmt=1` active gives
13.64 vs 14.42 — different values from nothing but the playhead moving,
confirming the envelope genuinely animates per-splat phase over time
rather than being a static per-splat offset.

### Frame time, all of 3a–3e active together

Measured `renderer.render()` CPU-dispatch time (not full GPU frame time —
see Saturation's own note above on why that could not be resolved in this
environment either: `performance.now()` coarsened, the timer-query polling
loop clamped by a hidden tab, and the app's own FPS counter driven by a
`rAF` a hidden tab stops). 30-call average, 177,132 splats: baseline
**0.073ms**, all five effects simultaneously active (curl, colour-displace,
rotated quantise, region attractor, posterise, wave) **0.110ms** — a
**0.037ms** delta for the added graph complexity. Recorded as a CPU-side
proxy, not a claimed GPU frame-time measurement.

### Preset, and what was and wasn't re-verified this pass

All new fields are additive `SPEC`/`SELECTS` entries — `PRESET_VERSION`
unchanged, still 5. Round-tripped through the actual save/wipe/load UI flow
with distinctive values on every new field (`qAmt`, `qCellX`, `qRotX`,
`qAxis`, `qStrat`, `qPoster`, `dField`, `dcAmt`, `dcMode`, `dcAxis`, `wAmt`,
`wFreq`, `wMode`, `rg1Attract`) simultaneously: all fourteen restored
exactly. Console clean across the full test sequence; existing effects
(`sat`, `cullA`, reset) re-checked against the same capture and unchanged
from their established readings. Mobile layout (390px, iframe) confirmed
both new accordion sections render and their sliders are visible.

This phase, unlike Phases 1 and 2, had a genuinely loaded capture available
for the entire testing pass (177,132-splat `butterfly.spz`, loaded cleanly
after a demo-asset hang that blocked every prior phase turned out to be
environmental, not a code issue — see Phase 1's note on the same gap). That
is why every measurement above is a real pixel/screen-bbox reading against
real rendered output, not a `window.__bench` construction-only check.

---

## Phase 4 of `DEV_PLAN_EFFECTS.md` — object-space camera uniform, and depth fade

A deliberate spike: prove the object-space camera primitive with the
cheapest possible consumer (depth fade) before DoF depends on it.

### The uniform gets its own narrow path, not `pushUniforms()` — and still calls `updateVersion()`

`pushUniforms()` recomputes the *entire* graph from `P`/`V`/DOM state on
every call — correct for "the user edited something," wasteful to run
60×/second just because the camera moved, when everything else it would
recompute is unchanged from a moment ago. `pushCameraUniform()` touches
exactly the two uniforms depth fade needs (`camPosLocal`, `camDirLocal`)
and nothing else, so it is cheap enough to call unconditionally every
frame regardless of whether anything else changed.

It still explicitly ends in `mesh.updateVersion()` — the same discipline
`pushUniforms()` follows, not a special case. The alternative was tempting
and considered: NOTES already establishes (see "The uniform invariant")
that camera motion sets Spark's own `viewChanged`, and *"until the camera
happens to move, at which point every pending edit pops in at once"* — so
in principle, relying on `viewChanged` alone would cover the one case that
matters (the camera actually moved this frame). Rejected anyway: that
leaves an implicit dependency on Spark's own internal change-detection
firing in exactly the way this code assumes, versus an explicit, unconditional
bump that is trivially correct by inspection. NOTES already measured this
exact bump as free — **120fps idle, playing, and playing with displace
active, at 177k splats** — which is the finding the plan's own text
("the version bump is already measured free") is referring to; explicit
and always-correct costs the same as implicit and conditionally-correct,
so there was no real tradeoff to make here.

### Ordering — the other half of correctness

`pushCameraUniform()` must run **after** whatever positioned the camera
this frame (`driveCamera()`, `controls.update()`) and **before** the
render call that bakes it in. Both call sites (`frame()`'s live loop, and
the export loop) were checked against this explicitly — getting it
backwards gives a one-frame-stale focus that reads exactly like a Spark
sort/async issue and, per the plan's own warning, wastes a day chasing the
wrong bug. Also added to `window.__bench.draw()`/`.settled()` in the same
position, so probe-based testing sees the same ordering the real render
loop does.

### Measured: object-space accuracy, across model transforms including a committed Correct-mode rotation

`mesh.worldToLocal(camera.position.clone())` against an independently
computed CPU reference (`camera.position.clone().applyMatrix4(new
Matrix4().copy(mesh.matrixWorld).invert())` — deliberately spelled out
rather than calling `worldToLocal` again, so the check is not just testing
that a function equals itself):

- Default transform: exact match to 6 decimal places.
- After a **committed** Correct-mode edit (`rotX=15, rotY=47, posX=0.3`,
  `xformCommit` actually clicked, not just staged): exact match to 6
  decimal places again — `[-2.978799, 0.999423, 2.777934]` both ways.

### Measured: zero one-frame lag

Two consecutive camera jumps to very different positions, each followed
by exactly **one** `__bench.settled()` call: `camPosLocal` matched the CPU
reference immediately both times, no second call needed to catch up. This
is a different question from the "one extra render to settle" hazard
NOTES already documents for the async **sort** (Model transform —
Invalidation, above) — that hazard is about blend/paint *order* catching
up, not about a uniform *value* being visible. Confirmed these are
independent: the uniform value itself was correct on the very first
settled render both times.

### Depth fade — measured behaviour

Fades **both** opacity and tint-toward-background together, scaled by one
`dpAmt` — the plan's "fade opacity or tint toward a colour" implemented as
one coherent aerial-perspective effect rather than a second mode toggle.
`dpColor` syncs to whatever `bgColor` (Phase 2b) currently is, via
`applyBgColor()`, rather than being a separate control — distant splats
dissolve into whatever backdrop is showing, the standard fog-matches-
background convention, and it needed no new UI.

Against the loaded capture, `window.__bench.settled()` throughout:

- `dpAmt=0`: exact baseline (this is also, incidentally, how every other
  effect in this graph behaves at its own neutral value — no special case).
- Near/far pushed **beyond** the whole object (`near=10, far=20`): exact
  baseline again — nothing gets faded when everything is closer than
  `near`, confirming the clamp's lower bound.
- Near/far collapsed to a sliver in front of the camera
  (`near=0, far=0.01`): coverage **0** — full dissolve, confirms the
  clamp's upper bound and that both opacity and colour fade together
  strongly enough to fully vanish, not just dim.
- A realistic mid-range band (`near=2.5, far=3.5`) at full amount: R/G/B
  all measurably shift toward the void backdrop (`87.4→75.1`,
  `71.2→62.8`, `60.8→54.7`) while coverage barely moves — exactly the
  expected shape for a *partial*, colour-shifting fade rather than a
  binary cutoff.
- **Planar vs radial genuinely differ**: same near/far, off-axis camera
  position (`(2, 0.5, 2)` looking at the origin, where view-direction depth
  and straight-line distance diverge) — different measured colour in each
  mode, confirming the mode select reaches something real rather than being
  a no-op alternative.
- **Export, 48 simulated frames**, exact call sequence from the export
  loop (`driveCamera(t); controls.update(); pushCameraUniform();`) against
  an authored 3-key camera move: **zero frozen frame pairs** in
  `camPosLocal` — it tracks the moving camera throughout, not stuck at
  frame 0's value.

### A real bug this phase's own testing caught, before it shipped

`U.dpColor.value.setHex(hex)` — every `dyno.dynoVec3(...)` uniform in this
file wraps a `THREE.Vector3`, which has no `.setHex()` (that is a
`THREE.Color` method). Threw immediately on the very first preset-load
test (`"U.dpColor.value.setHex is not a function"`), caught before this
reached NOTES as a finished measurement. Fixed by decoding through a
throwaway `THREE.Color` and setting the vector's components directly —
`U.dpColor.value.set(rgb.r, rgb.g, rgb.b)` — the same pattern `U.tint`
already uses elsewhere in this file. Re-verified after the fix: `dpColor`
correctly reads `[1,1,1]` after selecting the white background, and the
full preset round-trip (save → wipe → load) for all four new fields
(`dpAmt`, `dpNear`, `dpFar`, `dpMode`) restores exactly.

### Environment note

The full-canvas `readPixels` probe intermittently returned an all-zero
buffer (`cov: 100, R: 0` or `cov: 0` depending on the exact bug shape) in
this session's long-lived browser tab, while single-pixel reads at the
same coordinates returned real data — the same tab-level degradation
pattern already seen in Phases 2 and 3 (many navigations, contexts
created/destroyed over one long session), not a code issue. Confirmed by
reproducing cleanly in a freshly opened tab. `window.__bench.settled()`
also needed 2–3 calls after a `reset → reframe` sequence before the probe
stabilised on some runs — consistent with the already-documented generator
warm-up transient, not new behaviour.

---

## Phase 5 of `DEV_PLAN_EFFECTS.md` — depth of field

### The plan changed shape entirely once `spark.module.js` was actually read

The plan calls for hand-rolling defocus via `covSplats: true` and a
`covObjectModifiers` covariance-space modifier — add `sigma^2(d)*I` to the
3D covariance, correct peak alpha by the determinant ratio, verify none of
it shifts the existing coverage tables first. Before writing any of that,
`spark.module.js` 2.1.0 was read directly (same discipline as every other
"verify against your own coverage tables" moment in this file) to confirm
the `covObjectModifiers` wiring — and found something that changed the
whole approach: **SparkRenderer already implements exactly this DoF model
natively**, as first-class constructor options/properties
(`focalDistance`, `apertureAngle`, `blurAmount`, `preBlurAmount`), read into
the vertex shader every frame. Reading the shader itself
(`splatVertex_default`, ~line 9968 in the pinned build):

```glsl
mat3 cov2D = transpose(J) * cov3D * J;   // project 3D covariance to 2D screen space
float a = cov2D[0][0], d = cov2D[1][1], b = cov2D[0][1];

float fullBlurAmount = blurAmount;
if ((focalDistance > 0.0) && (apertureAngle > 0.0)) {
    float focusBlur = abs((-viewCenter.z - focalDistance) / viewCenter.z);
    float apertureRadius = focal.x * tan(0.5 * apertureAngle);
    fullBlurAmount = clamp(sqr(focusBlur * apertureRadius), blurAmount, sqr(maxPixelRadius));
}

float detOrig = a * d - b * b;
a += fullBlurAmount; d += fullBlurAmount;
float det = a * d - b * b;

float blurAdjust = sqrt(max(0.0, detOrig / det));
rgba.a *= blurAdjust;   // <-- exactly "peak alpha drops by the determinant ratio"
```

This is the plan's own algorithm — CoC-shaped blur added to a covariance,
alpha corrected by the determinant ratio — already implemented, already
tested (it ships with Spark), and arguably **more correct** than what the
plan describes: the blur is added to the **projected 2D** covariance (a
real circular CoC in screen space, isotropic in the two directions that
actually matter for a lens), not the plan's simplified 3D-isotropic
`sigma^2*I`. It runs identically whether the mesh stores scale+quaternion
or packed covariance (`cov3D` is computed from whichever the mesh actually
uses, then the same downstream code applies) — so `covSplats` turned out to
control storage precision only, orthogonal to whether DoF works at all, not
a prerequisite for it. Using Spark's own tested implementation instead of
re-deriving the same math independently is the safer choice by construction,
not a shortcut — it does not carry this project's own margin for a
determinant-ratio sign error or a projection mistake. `covSplats` was not
touched this phase; nothing here depends on it.

Consequence for scope: no `covObjectModifiers`, no hand-derived covariance
arithmetic in `buildModifier()`. `updateDofFocus()` (new, alongside Phase
4's `pushCameraUniform()`) sets exactly two `SparkRenderer` properties per
frame from this app's own `SPEC` sliders. Both live outside this app's own
generate/bake pipeline entirely — `spark.focalDistance`/`apertureAngle` are
read by Spark's own per-frame uniform sync, not gated by `viewChanged ||
version changed`, so unlike Phase 4's `camPosLocal` this needs no
`mesh.updateVersion()` and has no one-frame-lag hazard of its own. Still
called every frame (not just on edits), for the same reason Phase 4's
uniform is: Focus-pull mode (below) depends on the camera's live position,
which can change without `pushUniforms()` ever running.

### A real, unplanned finding: `blurAmount` was never zero

`SparkRenderer`'s own default is `blurAmount: 0.3` — a constant term added
to `a`/`d` (the projected 2D covariance) on **every** splat, **every**
frame, unconditionally, regardless of `focalDistance`/`apertureAngle`. This
app never set it, so it has been active since the very first commit — every
"baseline" coverage/luma figure anywhere else in this file was measured
under a small constant blur nobody knew was there. Measured impact on the
loaded `butterfly.spz`: coverage `4.29 -> 4.17`, mean R `112.57 -> 115.21`
with it zeroed — small, but real and systematic. Now explicit:
`new SparkRenderer({ renderer, blurAmount: 0 })`. Depth of field (via
`apertureAngle`) is the only intentional source of blur from here on;
`preBlurAmount` is also left at its own default of 0.

> **Superseded — see "Optics port, Phase A" below.** Pinning it to 0 was
> right for measurement hygiene and wrong as a shipped default: the term is
> antialiasing, and zeroing it produced radial needle shards on large
> captures. `blurAmount` is now a `SPEC` parameter defaulting back to 0.3,
> and the tables in this section were measured at 0. This section is left
> intact because the *finding* — that a vendor default had been silently
> active since the first commit — is still the important part.

This does not retroactively invalidate any earlier measurement in this file
— they are legitimate readings of the app's actual behaviour at the time,
just under a default nobody had inspected. Future measurements will differ
slightly from older ones by this same small amount, which is expected and
correct, not drift to chase.

### Focus, aperture, and what was deliberately not added

Two `SPEC` sliders, group `dof`: `dofFocus` (world-scale via `radius`, same
convention as every other spatial parameter) and `dofAperture` (0–30
degrees, converted to radians for `apertureAngle`). `dofFocusSrc` (Manual,
or Region 1–4) is the plan's "focus-pull to a region slot's centre" —
reuses the region system's own position sliders, no new machinery: when a
region is selected, `updateDofFocus()` recomputes the focal distance every
frame as that region's live world-space position projected onto the
camera's own view axis (the exact same "planar depth" quantity Phase 4's
depth fade already uses, computed independently here since this lives on
the renderer, not the mesh's dyno graph, and so cannot share Phase 4's
`camPosLocal`/`camDirLocal` uniforms).

**Not added:** a separate falloff-curve-shape control. Spark's own formula
(`focusBlur = abs((-viewCenter.z - focalDistance) / viewCenter.z)`, an
exact algebraic ramp, not an authorable curve) is what actually runs;
layering a customisable curve on top would mean computing blur amount
independently in a modifier and fighting Spark's own value, defeating the
entire reason to use the native path. Two sliders, not three, and the
plan's "falloff curve" is this section's honest scope cut, not an oversight.

### Measured

Against the loaded `butterfly.spz` (177,132 splats), `window.__bench`
throughout:

- **`aperture = 0` reproduces baseline exactly**, not just within 1% —
  `apertureAngle > 0.0` is one of the shader's own gate conditions, so at
  `dofAperture = 0` the blur branch never executes regardless of
  `dofFocus`'s value. Verified with `dofFocus` set to several different
  values while aperture stayed 0: bit-identical coverage, R, and total luma
  every time.
- **A splat at the focus distance is unchanged, by construction, not
  approximation.** At `viewCenter.z == -focalDistance`, `focusBlur =
  abs(0/z) = 0` regardless of aperture, so `fullBlurAmount = clamp(0,
  blurAmount=0, ...) = 0` — determinant ratio exactly 1.0 at focus for any
  aperture value, once `blurAmount` is zeroed (see above; this is *why*
  the `blurAmount` fix had to land in this phase and not be deferred).
- **Focus sweep monotonicity — verified analytically, not by a noisy pixel
  proxy.** `focusBlur(d) = |1 - focalDistance/d|` for depth `d` is a simple
  monotonic function of `|d - focalDistance|` on each side (algebra, not
  measurement): strictly increasing as `d` moves away from `focalDistance`
  in either direction, `0` exactly at `d = focalDistance`. A pixel-count
  proxy was tried first and rejected: coverage is not a clean sharpness
  signal on a capture with real depth extent — blurred splats can cover
  *more* screen area even while dimmer, which showed up as a non-monotonic
  reading that was measuring the wrong thing, not a real defect. The
  algebraic guarantee is the actual proof; a coarse aggregate over a
  177k-splat capture with its own depth range was never going to cleanly
  demonstrate it.
- **Total luma conservation (the check that catches the alpha bug) —
  confirmed on a sparse, non-overlapping subset; NOT confirmed at the
  177k-splat aggregate level, and that gap is itself the finding.** At
  `cullA = 0.85` (leaving only the brightest, well-separated splats),
  luma deviation across `dofAperture` 0→4° was **-0.01% to -0.64%** —
  comfortably inside the plan's 2% bound, and direct evidence the
  per-splat `blurAdjust` correction is doing exactly what it claims. On the
  full dense capture, the same sweep showed deviations from **1.2% up to
  ~15%** at larger apertures — tried against both the app's `void`
  background and a pure-black one (ruling out background-blend nonlinearity
  as the cause: pure black measured *larger* deviations, not smaller).
  Diagnosis: as splats defocus they grow in screen-space footprint, which
  increases local overlap density in a dense cloud, and sequential
  alpha-"over" compositing of many overlapping semi-transparent layers is
  not a linear/additive operation — summed displayed pixel luma is not a
  reliable proxy for per-primitive energy conservation once overlap is
  significant, independent of whether that per-primitive conservation
  itself is correct. This is the same *shape* of trap as "an empty mask
  fakes a pass" earlier in this file: a measurement that looks like it
  tests the right thing but is actually dominated by something else
  (compositing overlap, not the defocus formula). The sparse measurement is
  the one that actually isolates and confirms the claim; the dense one is
  recorded as an honest non-result, not smoothed over.
- **Frame time**: 30-render average, `dofAperture=0` **0.2333ms** vs
  `dofAperture=5` **0.2133ms** — within noise, no measurable added cost,
  matching the plan's own expectation ("camera movement already triggers
  regenerate via `viewChanged`... little or no additional cost").
- **Focus-pull matches an independent CPU reference exactly**: region
  slot 1 at world-space `(0.3, 0.1, 0) * radius`, projected onto the
  camera's view axis — `spark.focalDistance` and a hand-computed
  `(worldPos - camera.position).dot(viewDir)` agreed to 4 decimal places.
- **48-frame export, animated focus envelope** (`dofFocus` keyed 0.5 → 7
  across the clip, `dofAperture=5`): simulated the exact export-loop call
  sequence — **zero frozen frame pairs**, values monotonically tracking the
  envelope from `0.6129` to `8.5805` (both `* radius`, matching the keyed
  0.5/7 exactly).

### A real bug this phase's own testing caught: `updateDofFocus()` read `P`, not `V`

First attempt at the export/envelope test above showed `spark.focalDistance`
frozen at a single value across all 48 frames despite a keyed envelope —
traced to `updateDofFocus()` reading `P.dofFocus`/`P.dofAperture` (the raw
slider value) instead of `V.dofFocus`/`V.dofAperture` (the *effective*
value: envelope- or audio-driven when active, `P` otherwise — the same
distinction `pushUniforms()` makes for every other parameter in this file).
Reading `P` silently ignores any Focus envelope, audio route, or a
region-position envelope under Focus-pull — the timeline would visibly
animate everything else while depth of field sat frozen, exactly the kind
of quiet, easy-to-miss bug this project's own `V`-vs-`P` convention exists
to prevent. Fixed by switching both reads (and the region-position reads
under Focus-pull) to `V`; re-ran the export test after the fix and got the
correctly-animating values quoted above.

### Honest limitation, recorded per the plan's own instruction

Gaussian bokeh only. No aperture shape, no cat's-eye, no shaped highlight
discs — those need a custom splat shader, and Spark 2.0 deprecated 0.1's
arbitrary splat profiles, so it is the new shader path or nothing. Out of
scope here, and out of scope for this app in general per its own Non-goals.

---

## Optics port, Phase A — the shard fix, and the re-baseline it forces

`DEV_PLAN_OPTICS_PORT.md`, Phase A. Behaviour ported from the `Optics/optics.html`
road-test rig; nothing copy-pasted from it, because it writes uniforms directly and
reads raw slider values, both of which violate the invariants above. Re-expressed
through `SPEC`, `V` and `pushUniforms()`.

### `blurAmount` was set to 0 for a good reason and it was still the wrong default

Phase 5 found SparkRenderer's undocumented `blurAmount: 0.3` default and pinned it
to 0 for a clean depth-of-field baseline. Correct as measurement hygiene, wrong as
a shipped default: **the term is antialiasing.** Reading Spark's own vertex shader,
the energy correction immediately downstream attenuates by

```
blurAdjust = sqrt(detOrig / det) = sqrt(L1*L2 / ((L1+B)(L2+B)))
```

A sub-pixel-thin needle has `L2` near zero, so the denominator is dominated by `B`.
Worked through at `B = 0.3`: a round splat of 2px sigma (`L1 = L2 = 4`) keeps
`4/4.3 = 0.930` — **7.0% attenuation**. A needle 10px by 0.1px (`L1 = 100`,
`L2 = 0.01`) keeps `sqrt(1 / (100.3 * 0.31)) = 0.179` — **82% attenuation.** That is
antialiasing correctly suppressing a primitive below the sampling limit, and zeroing
it removed the only thing doing so. Invisible on a 177k capture; radial needle
shards on a 2M-splat one.

`blurAmount` is now a `SPEC` parameter, group `optics`, **default 0.3**, written by
`updateOptics()` on the same per-frame renderer-property path as
`updateDofFocus()`/`updateSmear()`.

**0.3 is the figure for scenes trained *with* antialiasing.** A capture trained
without it may want another value. This is per-capture, not a constant — which is
exactly why it is a slider and not a constant in the constructor.

**The standing lesson, because it has now bitten twice: when a vendor default is
neutralised for measurement hygiene, record what that default was *doing*, not
merely that it was changed.** Zeroing `blurAmount` cost a day of shard hunting.

### Re-baseline — the full `?debug=1` effect sweep, at both `blurAmount` values

`3_3_2025.ply` (44,296 gaussians), canvas **940×568**, `void` background, every
figure via `window.__bench.settled()` with a 6-render warm-up and the
stable-AND-non-zero-on-a-real-background rule. Each case is set from a full
reset-all, so no two effects are ever live at once.

Coverage %, then mean R, then mean luma:

| case | value | `blurAmount = 0` | `blurAmount = 0.3` |
|---|---|---|---|
| **base** | — | **3.0478** / 13.147 / 13.850 | **3.1763** / 13.250 / 13.962 |
| `cullA` | 0.1 | 3.0469 / 13.145 / 13.848 | 3.1758 / 13.249 / 13.960 |
| `sat` | 2 | 3.0535 / 13.316 / 13.902 | 3.1840 / 13.410 / 14.013 |
| `dAmt` | 0.3 | 12.2794 / 21.435 / 22.367 | 12.8791 / 21.423 / 22.355 |
| `qAmt` | 0.6 | 2.8042 / 11.023 / 11.839 | 2.9179 / 11.159 / 11.977 |
| `dpAmt` | 0.8 | 2.8978 / 9.979 / 10.786 | 2.9808 / 9.974 / 10.782 |
| `dofAperture` | 5 | 5.1427 / 14.148 / 15.135 | **5.1427 / 14.148 / 15.135** |
| `smearAmt` | 8 | 4.0384 / 13.825 / 14.657 | 4.0585 / 13.835 / 14.668 |
| `bright` | 1.4 | 3.0602 / 14.403 / 15.230 | 3.1979 / 14.531 / 15.371 |
| `scale` | 1.6 | 3.6944 / 13.467 / 14.227 | 3.7215 / 13.550 / 14.314 |
| `anisoLimit` | 8 | 3.0467 / 13.143 / 13.847 | 3.1744 / 13.245 / 13.957 |
| `maxPixelRadius` | 100 | 3.0478 / 13.147 / 13.850 | 3.1763 / 13.250 / 13.962 |

Every case restores to base **exactly** — coverage, R and luma all bit-identical
after reset, at both `blurAmount` values.

**The re-baseline in one number: `blurAmount` 0 → 0.3 moves base coverage
`3.0478 → 3.1763`, +4.22%. Mean R +0.78%, mean luma +0.81%.** Multiply any older
figure in this file measured on this capture by roughly 1.042 to compare, or
compare ratios against each table's own base, which cancels this as well as canvas
geometry.

Luma goes **up**, not down, and that is not a contradiction of "blur attenuates".
Per-primitive energy is conserved by `blurAdjust`; what changes is that the same
energy is spread over more pixels, so fewer of them saturate or overlap. This is
the same compositing-overlap effect Phase 5 recorded — summed displayed luma is not
a proxy for per-primitive energy on a dense cloud.

### Two measured facts worth keeping

**`dofAperture = 5` is bit-identical at `blurAmount` 0 and 0.3** — 5.1427 / 14.148 /
15.135 both times, to every digit. Spark's own line is
`fullBlurAmount = clamp(sqr(focusRadius), blurAmount, sqr(maxPixelRadius))`, so
`blurAmount` is only a **floor**, and at aperture 5 the circle of confusion exceeds
it on every visible splat. The antialias floor and depth of field do not stack;
whichever is larger wins, per splat.

**`maxPixelRadius = 100` is bit-identical to 512 on this capture**, at both
`blurAmount` values. No splat here draws wider than a 100px radius, so the control
does literally nothing on a well-behaved 44k capture. Exposed anyway, as the plan
asks, for its diagnostic value: if dropping it to 100 kills shards on a large
capture, that confirms the diagnosis. As a *fix* it is worse than either of the
other two, because it clips in screen space and truncates legitimately large near
splats with a hard edge. It is also read twice in Spark's shader — once as the DoF
clamp ceiling, once as a cap on the drawn eigen-scales — so it bites whether or not
depth of field is on.

### `anisoLimit` — the second line, in Trim beside `cullS`

An `objectModifier` clamping the longest scale axis against the **middle** one,
first thing in `buildModifier()` on the raw scales, so `Size × all`, region shrink
and `cullS`'s own max-axis test all see the clamped geometry:

```
mid = sx + sy + sz - max - min          // midAxis(), three dyno ops, no branches
scales = min(scales, vec3(mid * limit))
```

Against the **middle** axis, not the smallest. A legitimate splat is a flat disc —
two similar axes, one tiny — so its `max/mid` is near 1 and it is untouched.
Clamping against the smallest would flatten every normal splat in the scene.

Measured on the daffodil at `blurAmount = 0.3`:

| `anisoLimit` | 60 (default) | 30 | 16 | 8 | 4 | 2 | 1 |
|---|---|---|---|---|---|---|---|
| coverage | 3.1763 | 3.1763 | 3.1763 | 3.1744 | 3.1604 | 3.1271 | 3.0842 |
| Δ vs default | — | 0 | 0 | −0.06% | −0.50% | −1.55% | −2.90% |

**60, 30 and 16 are bit-identical** — direct evidence the clamp is a genuine no-op
above any real `max/mid` ratio in a clean capture, neutral by *value* with no branch
and no recompile, the same discipline every other effect in this graph follows. At 8
the plan's acceptance bound (within 1% on a well-behaved capture) is met with two
orders of magnitude to spare: −0.06%. At 1 the cap forces perfect isotropy and costs
2.9% — that is the control working, not a defect.

Size and shape are **independent failure modes** — a needle can be small in volume
and still streak hundreds of pixels — so this is not a duplicate of `cullS`. Default
off (60) on purpose: it modifies geometry and costs real detail, so it is the second
reach, after `blurAmount`.

`cullA` stays. Global opacity (Phase C) will not replace it either: the anisotropy
clamp handles needles, `cullS` handles oversized blobs, and neither touches diffuse
low-alpha haze, which is `cullA`'s one real job.

### Frame time

30- and 60-render averages at 44,296 splats, warm-up discarded, interleaved and
repeated: `anisoLimit` 60 vs 8 — 0.147/0.117ms vs 0.178/0.075ms. `blurAmount` 0 vs
0.3 — 0.063/0.045ms vs 0.047/0.045ms. **All within noise; no measurable cost either
way.** The clamp is three dyno ops on a path Spark already bakes, and `blurAmount`
is a uniform Spark already reads every frame.

### Preset

**No version bump.** All three are additive `SPEC` entries, so they inherit reset,
envelopes, audio routing and persistence for free, and `applyPreset()`'s existing
"missing fields keep their current value" rule handles a v5 file that predates them:
it lands on the new defaults rather than being rejected. Verified: round-trip
`serialize() -> applyPreset() -> serialize()` is byte-exact (modulo `saved`) with
`blurAmount 0.62`, `maxPixelRadius 200`, `anisoLimit 12.5`; and a file with all
three keys deleted loads without touching them.

The honest consequence, recorded rather than smoothed over: **a preset authored
before this commit will now render at `blurAmount = 0.3` where it was authored at
0.** That is a real, small look change (+4.22% coverage on this capture) applied to
old work. It is the right default and the plan re-baselines everything around it,
but it is not a silent no-op.

### Not verified, and why

The plan's headline acceptance criterion — *"a large capture that showed shards
shows none at `blurAmount = 0.3`"* — **is not verified here.** The committed test
asset is a clean 44k Polycam capture with no needle population; on it, the whole
shard mechanism is inert (`maxPixelRadius` at 100 changes nothing, `anisoLimit` at
16 changes nothing). The mechanism is confirmed analytically from Spark's own shader
arithmetic above, and the controls are confirmed to move the picture in the right
direction and to be exactly neutral at their defaults. Confirming the *symptom* is
gone needs a needle-heavy capture in the repo, which there is not one of. Recorded
as an open item, not as a pass.

---

## Optics port, Phase B — depth of field controls

`DEV_PLAN_OPTICS_PORT.md`, Phase B. Replaces `dofAperture` (degrees) and
`dofFocus` (absolute, in units of scene radius) with `dof` (0–100) and
`dofStops` (an offset in doublings from the pivot). Breaking for presets, hence
`PRESET_VERSION` 6 and a migration.

### `dof` is 0–100 and deliberately not an f-number

An f-number was tried in the rig and rejected. `tan(theta) = 1/(2N)` is the
correct real-lens relation, but Spark's `apertureAngle` is not calibrated to a
physical aperture, so f/20 produced a depth of field no large-format lens could.
**A false label is worse than an abstract one.**

```
angle = 40° * pow(1 - dof/100, 2.5)
if (angle < 1e-4 rad) angle = 0
```

The 2.5 exponent puts fine control in the shallow-blur band that actually gets
used. Measured, `dof` to aperture angle in degrees:

| `dof` | 0 | 10 | 25 | 50 | 75 | 90 | 95 | 96 | 97 | **97.5** | 100 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| angle° | 40 | 30.737 | 19.486 | 7.071 | 1.250 | 0.1265 | 0.0224 | 0.0128 | 0.0062 | **0** | **0** |

**100 resolves to exactly zero, and so does everything from 97.5 up.** Spark
gates its entire defocus path on `apertureAngle > 0.0`; any nonzero angle
enlarges every splat and pays full overdraw however invisible the blur is. The
`1e-4` snap exists for that reason, and the measured snap point is between
`dof` 97 and 97.5.

**A measured detail the plan did not anticipate, and it argues the snap is if
anything too conservative:** `dof` 90 and 95 are **bit-identical to `dof` 100**
in coverage, R and luma (3.1763 / 13.250 / 13.962 at all three) — yet 90 and 95
are *not* on the fast path. At those angles the circle of confusion is smaller
than `blurAmount` everywhere on this capture, so Spark's own
`clamp(sqr(focusRadius), blurAmount, …)` returns `blurAmount` and the frame is
unchanged while the defocus branch still runs on every splat. That is precisely
the "imperceptible blur at full cost" trade the plan calls the worst available,
and it is live between roughly `dof` 90 and 97.5. The snap errs in the safe
direction — it never removes blur that was visible — so it is left where the
plan put it, but the gap is real and now recorded rather than assumed away.

### Frame time is the wrong instrument here; the gate condition is the right one

The plan asks for `dof = 100` to "measure the same frame time as the DoF path
never running". At 44,296 splats the entire render is ~0.05ms, so the
measurement has no power: `dof` 100 / 96 / 50, interleaved, warm-up discarded,
came back 0.097/0.050, 0.063/0.073, 0.072/0.045 ms — pure noise, in which
`dof = 50` is nominally the *fastest*. Reporting that as a pass would be
reporting noise.

The check that actually holds is structural and exact: **`spark.apertureAngle`
is `=== 0`**, verified directly, and `apertureAngle > 0.0` is Spark's own gate
condition read straight out of its vertex shader. The path cannot run. That is
a stronger statement than a timing that agrees within noise, not a weaker one.
A frame-time comparison would only become the real check on a capture large
enough for overdraw to dominate, which the committed test asset is not.

### Focus as an offset in stops from the pivot

```
fd = max(base * pow(2, dofStops), 1e-4)     // base = camera-to-pivot distance
```

Anchored to the **camera-to-pivot distance**, not to scene radius. Radius
describes the whole scan; pivot distance describes what is being looked at.
This is the "normalisation is necessary but not sufficient" rule from this file
for the **third** time — `cullS` and `mdScale` were the first two — and the old
control is a textbook case: it was correctly multiplied by `radius` and still
unusable.

Measured, and this is the acceptance criterion in its strongest form. Dollying
the camera across a **160× range** of pivot distance and reading
`spark.focalDistance / pivot` at each of five stop positions:

| pivot distance | 0.6938 | 2.7753 | 11.1012 | 111.0122 |
|---|---|---|---|---|
| `dofStops` −2 | 0.25 | 0.25 | 0.25 | 0.25 |
| −1 | 0.5 | 0.5 | 0.5 | 0.5 |
| 0 | 1 | 1 | 1 | 1 |
| +1 | 2 | 2 | 2 | 2 |
| +2 | 4 | 4 | 4 | 4 |

**Exact to nine decimal places at every cell** — `fd/pivot = 2^stops`,
independent of scale by construction. Comparing two captures by eye was the
plan's proposed test; this is the same claim measured, on one capture, without
the eye.

What the old control did at the far end, for the record: `radius` is 1.0663, so
`dofFocus`'s whole 0–8 travel reached a maximum focal distance of **8.53** — at
a pivot 111 units away, the entire slider focused in front of the subject and no
position on it could reach the thing being looked at. The new control's 0 lands
on it exactly.

`focalDistance` is clamped to `1e-4`, not to 0. Zero is the *other* condition
Spark gates the defocus path on, so letting the base reach zero — a focus-pull
mask that has passed behind the camera — would silently switch depth of field
off rather than focusing very close.

### Focus-pull composes with the offset rather than replacing it

`dofFocusSrc` keeps its five options; what changes is that Manual now means
"the pivot" and the mask options supply a different **base** for the same stops
offset, so racking away from a tracked subject stays possible. Verified against
an independent CPU reference (mask 1 at `(0.3, 0.1, 0) × radius`, projected onto
the camera's view axis, times `2^stops`) — agreement **exact to 1e-6** at −1, 0
and +1 stops: 1.363238 / 2.726477 / 5.452953 against 1.363238 / 2.726477 /
5.452953.

### `V`, not `P` — re-verified, because this is where it bit last time

Phase 5 shipped a bug where `updateDofFocus()` read `P` and every depth-of-field
envelope sat frozen. Re-verified after the rewrite with envelopes armed on both
new parameters (`dof` 100→20, `dofStops` −2→+2) while both sliders stayed at
their defaults:

| playhead | 0 | 0.25 | 0.5 | 0.75 | 1 |
|---|---|---|---|---|---|
| `P.dof` | 100 | 100 | 100 | 100 | 100 |
| `V.dof` | 100 | 87.5 | 60 | 32.5 | 20 |
| angle° | 0 | 0.221 | 4.048 | 14.973 | 22.897 |
| `V.dofStops` | −2 | −1.375 | 0 | +1.375 | +2 |
| `focalDistance` | 0.6938 | 1.0700 | 2.7753 | 7.1983 | 11.1012 |

`P` never moves, the render tracks the curve, and `focalDistance` is
`pivot × 2^stops` at every row.

### Sweeps, and one behaviour worth knowing about

`3_3_2025.ply`, 940×568, void, `blurAmount = 0.3`, base 3.1763 / 13.250 /
13.962. Restores exactly after both sweeps.

| `dof` | 100 | 97.5 | 95 | 90 | 75 | 50 | 25 | 0 |
|---|---|---|---|---|---|---|---|---|
| coverage | 3.1763 | 3.1763 | 3.1763 | 3.1763 | 3.2310 | 4.2428 | 5.8928 | 6.7396 |
| mean R | 13.250 | 13.250 | 13.250 | 13.250 | 13.291 | 13.802 | 13.750 | 12.363 |

`dofStops` at `dof = 50`:

| stops | −2 | −1 | −0.5 | 0 | +0.5 | +1 | +2 |
|---|---|---|---|---|---|---|---|
| coverage | 6.6716 | 7.0617 | 6.2215 | 4.2428 | 6.4289 | 5.3023 | **0** |
| mean R | 10.477 | 12.361 | 13.862 | 13.802 | 12.585 | 8.922 | **8** |

Two things to read out of that, neither of them a defect:

**Coverage is not a sharpness signal** — already recorded in Phase 5 and visible
again here as a non-monotonic curve either side of focus. A defocused splat
covers *more* pixels while being dimmer. Minimum coverage at 0 stops is the
in-focus reading, but the curve on each side is not usable as a focus metric.

**At extreme defocus the model erases rather than blurs.** `+2` stops reads
coverage 0 and mean R 8 — pure background, the subject entirely gone. That is
Spark's own `minAlpha` discard: `blurAdjust` shrinks peak alpha as the CoC
grows, and past `0.5/255` the splat is discarded in the vertex shader rather
than shaded. Physically a lens would smear the subject across the frame; this
dissolves it. Worth knowing before reaching for heavy defocus, and it is the
same discard mechanism Phase C's global opacity will exploit deliberately.

### Preset — v5 → v6

Both old ids are gone from `SPEC`, so `applyPreset()`'s per-id loop skips them
as unknown. The migration is explicit, and it is placed at the **tail** of
`applyPreset()`, after the camera block, `applyXform()` and `reframe()` — the
focus half divides by the pivot distance, which does not exist until the final
camera pose does. Migrating it up with the other parameters would divide by a
stale pivot.

- `dofAperture` (degrees) → `dof` via `angleToDof()`, the algebraic inverse of
  `dofToAngle()`. **Verified exact**: round-tripping `dof` 0→100 in steps of 2.5
  through both functions has a maximum error of **1.4e-14**.
- `dofFocus` (× radius) → `log2(dofFocus * radius / pivot)`.

Measured on a synthetic v5 file with `dofAperture: 5`, `dofFocus: 2`:

| | old | migrated | deviation |
|---|---|---|---|
| aperture (rad) | 0.087266 | 0.087129 | 0.16% |
| focal distance | 2.132535 | 2.103288 | 1.37% |

**Neither formula is the source of that deviation — the slider step grid is.**
Exact `dof` is 56.4675 and the control's step is 0.5, so it lands on 56.5;
exact `dofStops` is −0.3800 and its step is 0.05, so it lands on −0.40. The
bound is half a step either way: ≤0.25 in `dof` and ≤0.025 stops, the latter
being ≤1.75% in focal distance. Left as is — that is finer than the control can
usefully be dialled by hand, and quantising to a representable slider position
is the right behaviour for a migration, not a rounding bug.

Both migrations report themselves in the status line rather than landing
silently:

```
aperture 5° -> depth of field 56.5
focus 2×radius -> -0.40 stops (approximate — the pivot moves)
```

The focus note says "approximate" because it is, and for a reason no amount of
arithmetic fixes: the old value was anchored to scene radius and the new one to
a pivot that moves, so the same number cannot mean the same thing under every
camera. What the migration does guarantee is that it reproduces the old focal
distance **for the pose the preset itself restores**, which is the useful
guarantee and the only one available.

v6 round-trip `serialize() -> applyPreset() -> serialize()` is byte-exact
(modulo `saved`).

### `setParam()`

The migrations need to write a parameter from code, which means keeping the
slider, `P`, the readout and the dirty flag in agreement — the same four things
`applyPreset()`'s own `SPEC` loop does inline. Factored out as `setParam(id, v)`
rather than repeated, because getting one of the four wrong is a silent desync
between what the panel displays and what renders. `DIGITS` (a `SPEC`-derived
id→precision map) exists for the same reason: so out-of-band writes format
their readout the same way the normal slider path does instead of guessing.

### Deferred to Phase C, on purpose

`nearBoost` (B3 in the plan) is a *shader* parameter — it multiplies `focusBlur`
inside Spark's vertex shader — and its anchor is the same `focusBlur` line
Phase C's field curvature and tilt patch. The plan asks Phase C to extend
`buildSmearVertexShader()` into a single `buildSplatVertexShader()` patching all
four anchors at once; landing a partial patch here and rewriting it there would
mean two shader edits and two chances to get the anchor discipline wrong. The
slider ships with C.

### An unrelated latent bug found while setting up the measurements

**`resize()` produced a NaN camera whenever the stage had zero height, and
nothing ever recovered.** `camera.aspect = w / h` with `h === 0` is NaN;
`reframe()` then computes `hFov = 2*atan(tan(vFov/2) * aspect)`, `Math.min(vFov,
NaN)` is NaN, `Math.sin(NaN)` is NaN, and `dist` is NaN — so
`camera.position.copy(center).add(0, 0, NaN)` lands a NaN position that renders
an empty frame **permanently**, because `reframe()` only runs on load, on
Reframe and on a transform commit.

It presents deceptively: `controls.target` is set on the line *before* the NaN
one, so the target is perfectly valid and only the position is broken, which
reads like a Spark sort or generate failure rather than an aspect-ratio one. Hit
live by loading the capture with the browser pane collapsed.

Fixed by flooring both dimensions at 1px in `resize()`. Not in the plan, fixed
anyway: every remaining phase of this port is verified by pixel probe, and a
latent "the whole viewport is silently blank" path is not something to leave
sitting under a measurement programme.

---

## Optics port, Phase C — shader patch extensions

`DEV_PLAN_OPTICS_PORT.md`, Phase C. `buildSmearVertexShader()` becomes
`buildSplatVertexShader()`, patching **four** anchors instead of two, in one
function. Seven new uniforms across `optics`, `depth` and `trim`.

| anchor | purpose |
|---|---|
| `uniform float apertureAngle;` | declare the new uniforms |
| `float focusBlur = abs((-viewCenter.z - focalDistance) / viewCenter.z);` | field curvature, tilt, swing, near boost |
| `float detOrig = a * d - b * b;` | smear (existing) + swirl |
| `rgba.a *= blurAdjust;` | global splat opacity |

All four in one builder on purpose. Landing them across several builders would
mean several chances to get the anchor discipline wrong, and the two covariance
patches **must** share the single `det`/`blurAdjust` energy correction that
already exists below them.

### Self-verification, exercised rather than asserted

Base text is read at runtime from a throwaway probe `SparkRenderer`'s own
`material.vertexShader`, never transcribed into `index.html`. Verified by
**deliberately corrupting each anchor in turn** — all four throw at build with a
message naming the specific anchor:

```
Splat vertex shader: Spark no longer contains the expected 'focusBlur' anchor
text — spark.module.js version drift. …
```

Structural checks on the produced shader, all confirmed:

- exactly **one** `float blurAdjust = sqrt(` in the patched source — the single
  energy correction the plan requires,
- `rgba.a *= blurAdjust * splatOpacity;` sits **before** Spark's `minAlpha`
  test, so sub-threshold splats are discarded rather than shaded.

### Neutral is bit-identical

With all seven new uniforms at their neutral values the render is
**3.1763 / 13.250 / 13.962** — bit-identical to the Phase A/B base, coverage,
mean R and luma. Re-checked one parameter at a time by explicitly re-asserting
each neutral value: all seven identical. The whole 20-case regression sweep
below restores exactly.

### Swirl: the plan's coupling was right when written and wrong two phases later

The plan specifies `k = swirlAmt * fullBlurAmount * …`, with the acceptance
test *"swirl at zero aperture changes nothing — proof the defocus coupling
holds"*. That test **failed** as first implemented, and the failure is a real
cross-phase interaction rather than a transcription slip.

`fullBlurAmount` does not start at zero. Spark computes
`fullBlurAmount = blurAmount` and only then, if the defocus gate passes, raises
it. Phase 5 had pinned `blurAmount` to 0, under which the plan's formula and its
acceptance test are both correct. **Phase A restored it to 0.3**, so coupling to
`fullBlurAmount` applies a constant swirl to every splat in the scene, in focus
or not, scaling with the antialias setting — precisely the "warp, not lens"
behaviour the coupling exists to prevent.

Measured before the fix, `swirlAmt = 20` with depth of field **off**:

| `blurAmount` | 0 | 0.3 | 1.0 |
|---|---|---|---|
| swirl off | 3.0478 | 3.1763 | 3.3484 |
| swirl on | 3.0478 | 3.2417 | 3.4588 |
| identical? | **yes** | no | no |

The fix is to couple to the **defocus component only**:

```glsl
float swirlBlur = fullBlurAmount - blurAmount;
if (swirlAmt > 0.0 && swirlBlur > 0.0) { … k = swirlAmt * swirlBlur * … }
```

Spark's own `clamp(sqr(focusRadius), blurAmount, …)` guarantees
`fullBlurAmount >= blurAmount`, so the difference is exactly the defocus
contribution and is never negative. After the fix, swirl at `dof = 100` is
bit-identical at `blurAmount` **0, 0.3, 1.0 and 1.5** — the plan's criterion
now holds, and holds *independently of the antialias setting*, which the plan's
own version did not.

Worth generalising: this is the second time in this port that a Phase A
decision invalidated something written earlier (the first being every coverage
figure in this file). **A default that a later formula reads as a baseline is
not a free parameter.**

### Swirl and its falloff — where the response actually lives

The plan asks every new parameter to be swept before its range is settled, and
guesses `swirlAmt` will want a log curve. **It does not.** At `dof = 45`, base
coverage 4.5467:

| `swirlAmt` | 0 | 0.5 | 1 | 2 | 4 | 6 | 10 | 15 | 20 | 25 | 30 |
|---|---|---|---|---|---|---|---|---|---|---|---|
| coverage | 4.5467 | 4.5593 | 4.5713 | 4.5979 | 4.6445 | 4.6906 | 4.7771 | 4.8740 | 4.9642 | 5.0541 | 5.1313 |
| Δ% | 0 | 0.28 | 0.54 | 1.13 | 2.15 | 3.16 | 5.07 | 7.20 | 9.18 | 11.16 | 12.86 |

Straight-line, about +0.43% coverage per unit across the whole travel, with no
bunching at either end and no sign of saturation at 30. A **linear 0–30 slider
is correct**, which makes this the first parameter in this port whose first
guess at a curve survived contact with a sweep — recorded because the standing
assumption after `cullS`, `mdScale`, focus, aperture and splat opacity was that
it would not.

`swirlFalloff` (at `swirlAmt = 15`) is the opposite shape and its whole response
sits in 0–3:

| falloff | 0 | 0.5 | 1 | 2 | 3 | 5 |
|---|---|---|---|---|---|---|
| coverage | 7.1814 | 6.2198 | 5.5514 | 4.8740 | 4.6484 | 4.5556 |

0 is uniform swirl across the whole frame; by 5 it is confined to the extreme
corners and nearly inert. Default 2.

### Focal surface — tilt, swing, curvature

The exact check is arithmetic, not pixels: reproduce the shader's own line in
JS and read the in-focus depth at frame edges. Pivot 2.7753, view-space frame
edges `yEdge = 1.2941`, `xEdge = 2.1417`:

| tilt | swing | curve | top | bottom | left | right | centre |
|---|---|---|---|---|---|---|---|
| 0 | 0 | 0 | 2.7753 | 2.7753 | 2.7753 | 2.7753 | 2.7753 |
| **30** | 0 | 0 | **3.5225** | **2.0281** | 2.7753 | 2.7753 | 2.7753 |
| −30 | 0 | 0 | 2.0281 | 3.5225 | 2.7753 | 2.7753 | 2.7753 |
| 0 | **30** | 0 | 2.7753 | 2.7753 | **1.5388** | **4.0118** | 2.7753 |
| 0 | 0 | **+1** | 5.5506 | 5.5506 | 5.5506 | 5.5506 | 2.7753 |
| 0 | 0 | **−1** | 0.0001 | 0.0001 | 0.0001 | 0.0001 | 2.7753 |
| 30 | 30 | 0.5 | 4.9101 | 3.4158 | 2.9264 | 5.3995 | 2.7753 |

**The in-focus depth demonstrably differs across the frame**, which is the
plan's acceptance criterion, and the spread is exactly what the geometry
predicts: tilt 30° gives `2 × yEdge × tan(30°) = 1.4944` and the table gives
`3.5225 − 2.0281 = 1.4944`. Swing 30° predicts `2 × xEdge × tan(30°) = 2.4731`
and the table gives `4.0118 − 1.5388 = 2.4730`. Curvature +1 doubles the edge
distance at `fcR = 1`. Curvature −1 drives it negative and the `max(…, 1e-4)`
clamp fires — the focal surface legitimately bending behind the camera, correct
behaviour rather than an error state. Tilt and swing compose with curvature and
with each other. Tangents are stored, not degrees, and are clamped at ±75°
(`tan 75° = 3.732051`, verified against a slider driven to 100).

Uniform round-trip is exact: `planeTilt = 30` reaches the shader as
`0.577350`, `tan(30°)` to six places.

**The pixel-level "sharp band crosses the frame diagonally" test is not usable
on this capture and is recorded as not run rather than as a pass.** The
daffodil is a thin, roughly frontal subject: essentially everything in frame
sits near one depth, so *any* tilt defocuses *everything* except the band
through view-space y = 0, rather than trading sharpness between the top and the
bottom. Measured, at `dof = 45`, coverage against tilt:

| tilt° | −40 | −20 | −10 | 0 | 10 | 20 | 40 |
|---|---|---|---|---|---|---|---|
| coverage | 4.4205 | 4.0124 | 4.1870 | 4.5467 | 4.9182 | 5.2832 | 6.0339 |

That has a minimum at about **−20°, not 0°** — consistent and interesting: the
subject's own plane is slightly tipped relative to the optical axis, so a small
negative tilt fits the focal surface to it and *sharpens the whole frame*,
which is exactly what a view camera's front standard is for. It is real
evidence the control does the right thing; it is not the diagonal-band
demonstration the plan asked for, and that needs a subject with depth extent
across the frame.

### Energy — the one-correction check

Sparse subset (`cullA = 0.85`), the measurement that actually isolates
per-primitive energy from compositing overlap, per Phase 5:

| | coverage | total luma | deviation vs DoF alone |
|---|---|---|---|
| DoF alone (`dof = 50`) | 4.1900 | 13.890 | — |
| \+ smear 8 | 4.5490 | 14.038 | **+1.07%** |
| \+ smear 8 + swirl 20 | 4.8337 | 14.094 | **+1.47%** |

**Inside the plan's 2% bound.** One correction covers all three effects; there
is no compounding darkening or brightening.

The dense aggregate over the same three, for the record, is **+2.71%** —
outside the bound, and *expected* to be, for exactly the reason Phase 5
recorded: sequential alpha-over compositing of many overlapping semi-transparent
layers is not additive, so summed displayed luma stops being a proxy for
per-primitive energy once overlap is significant. The sparse figure is the one
that tests the claim; the dense one is recorded as a known non-result, not
smoothed over.

### Global splat opacity — and where the plan's expectation was wrong

A **uniform, not an `objectModifier`**. Alpha is written in the vertex shader,
so a uniform costs nothing where a modifier change would call `updateVersion()`
and re-bake the packed buffer on every frame of a slider drag. Logarithmic
slider (log2, ×0.016 to ×1.5, default ×1) with a `SHOW` override, same "the
slider position is not the value" case as `cullS`.

Placed before Spark's `minAlpha` test so sub-threshold splats discard rather
than shade. **Frame time is flat** from ×1.5 down to ×0.03 at 44k splats
(0.045–0.089ms, noise), so the claimed performance *gain* is structural and
unmeasured here — this capture is nowhere near fill-rate bound.

**The plan predicts the model "erodes edges-inward" while "dense regions stay
solid". Measured, that is not what happens — the fade dominates by a long way.**
Pixel counts by luma band:

| multiplier | 12–32 | 32–80 | 80–140 | 140–200 | >200 | total lit |
|---|---|---|---|---|---|---|
| ×1 | 594 | 920 | 1961 | 7730 | **5634** | 16839 |
| ×0.5 | 729 | 959 | 2241 | 9709 | 3016 | 16654 |
| ×0.25 | 787 | 1180 | 4754 | 9294 | **389** | 16404 |
| ×0.125 | 853 | 2889 | 9747 | 2559 | 0 | 16048 |
| ×0.0625 | 1229 | 9726 | 4647 | 61 | 0 | 15663 |
| ×0.0156 | 12176 | 2015 | 0 | 0 | 0 | 14191 |

At ×0.25 the **brightest** band is down 93% while the total lit area is down
3%. The brightest band empties **first**, not last. The whole distribution
walks downward and only the very edge of it falls off the `minAlpha` cliff:
across the full travel to ×0.016 the lit area loses just 15.7%, while mean
brightness inside it falls from 167.3 to 23.2 (−86%).

That is the correct behaviour of a peak-alpha multiply on a cloud that is not
overlap-saturated: scaling every contributing alpha scales the composite
roughly multiplicatively, so bright pixels lose the most in absolute terms.
"Dense regions stay solid" requires alpha-over to saturate, which needs far
more overlap than a 44k capture has. Plausible on a 2M-splat scan; untested.
The UI copy states the measured behaviour, not the predicted one.

Above ×1 is not "more". Reading Spark's **fragment** shader confirms this and
also corrects the plan on the mechanism: the plan quotes
`shifted = sqrt(z2) - max(0, a-1)`, which in 2.1.0 sits inside an `if (false)`
branch and never runs. The live branch is

```glsl
float a = exp((rgba.a*rgba.a - 1.0) / 2.718281828459045);
float alpha = 1.0 - pow(1.0 - exp(-0.5 * z2), a);
```

— a different, sharper-topped profile. The *conclusion* the plan draws is right
(alpha over 1 reads as harder-edged discs); the formula it cites is dead code.
Worth recording as another instance of the standing rule that produced the
whole Phase 5 rewrite: **read the shader that actually runs.**

### `cullA` stays

Not removed, per C4. Global opacity does not replace it: the anisotropy clamp
handles needles, `cullS` handles oversized blobs, global opacity fades and
thins everything uniformly, and none of them touch diffuse low-alpha haze,
which is `cullA`'s one real job.

### Full regression sweep, 20 cases

`3_3_2025.ply`, 940×568, void, base **3.1763 / 13.250 / 13.962**. Each case is
set from a full reset-all and reset again after, so no two are ever live at
once. **Every case restores to base bit-identically**, and the final probe after
the whole sweep is bit-identical to the first.

| case | value | coverage | mean R | luma |
|---|---|---|---|---|
| `cullA` | 0.1 | 3.1758 | 13.249 | 13.960 |
| `sat` | 2 | 3.1840 | 13.410 | 14.013 |
| `dAmt` | 0.3 | 12.8791 | 21.423 | 22.355 |
| `qAmt` | 0.6 | 2.9179 | 11.159 | 11.977 |
| `dpAmt` | 0.8 | 2.9808 | 9.974 | 10.782 |
| `dof` | 50 | 4.2428 | 13.802 | 14.680 |
| `smearAmt` | 8 | 4.0585 | 13.835 | 14.668 |
| `splatOpacity` | ×0.25 | 3.0872 | 12.118 | 12.879 |
| `bright` | 1.4 | 3.1979 | 14.531 | 15.371 |
| `scale` | 1.6 | 3.7215 | 13.550 | 14.314 |
| `anisoLimit` | 8 | 3.1744 | 13.245 | 13.957 |
| `blurAmount` | 0 | 3.0478 | 13.147 | 13.850 |
| `maxPixelRadius` | 100 | 3.1763 | 13.250 | 13.962 |

Six parameters are **inert at `dof = 100` by construction** and read as
"no change" in the table above — that is the design, not a broken uniform, and
it is the same shape of trap as `wAmt` reading as no-effect at `dAmt = 0`.
Measured again with `dof = 50` (base 4.2428 / 13.802 / 14.680), where each one
does move the picture:

| case | value | coverage | mean R | luma |
|---|---|---|---|---|
| `nearBoost` | 4 | 5.1590 | 13.338 | 14.223 |
| `swirlAmt` | 20 | 4.5870 | 13.937 | 14.874 |
| `fieldCurve` | 1.5 | 5.4308 | 12.683 | 13.610 |
| `planeTilt` | 30 | 5.1952 | 13.871 | 14.849 |
| `planeSwing` | 30 | 4.4786 | 13.891 | 14.819 |
| `dofStops` | +1 | (see Phase B) | | |

`swirlFalloff` still reads as no change at `dof = 50` with `swirlAmt = 0`,
correctly — it shapes an effect that is off. Its own sweep is above.

### Preset

**No version bump.** All seven are additive `SPEC` entries; v6 stands. Verified:
round-trip byte-exact (modulo `saved`) with all seven off default plus `dof` and
`dofStops`, and a synthetic v5 file with every Phase C key deleted loads
cleanly, keeps the current values, and still runs the v5→v6 depth-of-field
migration (`aperture 3° -> depth of field 64.5`,
`focus 1.5×radius -> -0.80 stops`).

### `updateSmear()` is now `updateShaderUniforms()`

One function pushing all nine `spark.material.uniforms.*` from `V`, on the same
per-frame path and for the same reason: THREE re-uploads a `ShaderMaterial`'s
changed uniforms on every draw call, so none of these are gated by Spark's
`viewChanged || version changed` and none need `mesh.updateVersion()`.

Which side of the JS/GLSL boundary each conversion happens on follows from each
uniform's own external contract, not from a house style: `smearAngle` goes over
in degrees because the shader's own `radians()` call is in the patch text;
`dof` is converted in JS because Spark's own `apertureAngle` property expects
radians; tilt and swing are converted in JS to **tangents** because the shader
wants a depth gradient per unit of view-space offset and `tan(angle)` is that
quantity — a conversion, not a unit change; `splatOpacity` is converted in JS
because the slider carries log2 of it.

## Optics port, Phase D — picked focus

`DEV_PLAN_OPTICS_PORT.md`, Phase D. Depends on Phase C for the tilted focal
surface. Tap the model to lock focus on a world-space point; it stays locked
as you orbit, and Focus (stops) still offsets from it.

### Depth buffers are meaningless on an alpha-blended cloud

There is no single surface at a pixel, only a stack of semi-transparent
gaussians. `pickDepth()` marches the tap ray front-to-back, projecting each
cached splat to screen space and accumulating `opacity * gaussian(pixel
distance)` in front-to-back depth order, taking the depth where accumulated
alpha crosses `pickThresh`. That is the renderer's own compositing logic run
by hand, so it returns the depth of whatever actually reads as the surface —
not the depth of the nearest primitive, which on a translucent cloud is
frequently not the same thing.

### Object-space cache, not world-space — an improvement on the plan, not a deviation

The plan's own limitation note says the cache "holds undisplaced object-space
data" and drifts from the screen under Noise/Quantise/Wave/Smear, with
`Readback` named as the eventual upgrade. That limitation is accepted as-is
here — it is a real one and out of scope for this phase.

What changed from the plan: the cache is built and stored in **object space**,
with `mesh.matrixWorld` applied fresh on every `pickDepth()` call, rather than
baking world-space positions once at cache-build time. Same cost — one 4×3
matrix multiply per splat either way — but it means a Correct-mode transform
commit needs **no cache invalidation at all**, where a world-space cache would
go silently stale the moment the model moves. Verified structurally: the cache
is never rebuilt on a transform commit and picks after one still land
correctly, because the fresh `matrixWorld` is exactly what makes that true.

**Area-equivalent radius from the two largest axes, not the single largest**,
matching Phase A's anisotropy clamp reasoning: splats are discs, not spheres,
and treating each as a sphere of its longest extent inflates the capture
radius and makes picks land short — the exact failure the plan's own
"cylinder bug" story describes for the whole-ray version of this mistake.

### Verified exact against an independent CPU reference

`pickDepth()`'s own logic re-implemented independently, outside the function,
reading the same cache: **bit-identical** at the tested tap point —
`2.5968379996456017` both ways, to every digit. A real synthetic
`pointerdown` through the actual DOM listener produces a `spark.focalDistance`
that matches the same value exactly.

A **real mouse click** (not a synthetic event) on the flower's centre in a live
browser session also worked end-to-end: `dofFocusSrc` flipped to "Focus on
picked point" and the status line reported a locked depth, confirming the
listener wiring and not just the math.

### Pivoting the tilted surface through the picked point

```
fd = -viewPt.z - (swing * viewPt.x + tilt * viewPt.y)
```

Solves for the focal distance that makes the **tilted** surface pass through
the picked point, rather than reading the point's raw depth — without this,
tilting would drag focus off whatever was just picked, the opposite of how a
view camera is actually used (pick the subject, then swing the standard around
it). Verified against an independent reference at tilt −30° / 0° / +30°:
**exact to under `1e-9`** at all three — `3.000559878887203`, `2.5968379996456017`,
`2.1931161204040004` both ways.

The tangents are recomputed here from `V` rather than read back out of
`spark.material.uniforms`, deliberately — `updateShaderUniforms()` runs
**after** `updateDofFocus()` in the frame loop (see both call sites), so
reading the uniforms here would use last frame's tilt and reintroduce exactly
the one-frame-lag hazard Phase 4 documents for camera uniforms. Deriving both
straight from `V` removes the ordering dependency instead of merely documenting
it. Field curvature is **not** compensated in the pivot — it is radial about
the frame centre, the picked point moves through the frame as the camera
orbits, and there is no fixed correction for a moving one. Left as an honest
gap, consistent with the plan's own instruction to record limitations rather
than paper over them.

### Fails loudly on empty space — and a real gap the plan's wording didn't anticipate

Tapping empty space (no surface within threshold) sets the status line to an
error and leaves the **previous** lock and `focalDistance` completely
undisturbed — verified: `spark.focalDistance` bit-identical before and after a
miss. That is the plan's literal instruction ("fail loudly… rather than
setting focus to garbage") and it holds for the interactive case.

**A second empty-space path the plan didn't name: `dofFocusSrc` can be
`"picked"` with `pickedPoint === null` through completely ordinary use, not
just a miss.** `pickedPoint` is deliberately not persisted in presets — it is
cache-derived and capture-specific, the same reasoning `CAMKEYS` and the
transform basis already follow in this file. So a preset saved mid-session
with a lock active, then reloaded (or loaded against a different capture),
restores `dofFocusSrc = "picked"` with no live point behind it. `SPEC`'s and
`SELECTS`' own forgiving-load rules only check that the option **exists**, not
that the state behind it is real, so this reaches `updateDofFocus()`.

**Caught by testing, not inspection, and it was a real bug, not a hypothetical:**
the first version's branch order was `if (src === "picked" && pickedPoint) {…}
else if (src === "manual" || !mesh) {…} else {…mask branch…}`. With `pickedPoint`
null, `"picked"` fell all the way to the *mask* branch, which reads
`V[\`rg${n}X\`]` — `V["rgpickedX"]`, undefined — and handed Spark a **NaN**
`focalDistance`. Verified directly: `Number.isNaN(spark.focalDistance) ===
true`. That is silent and strictly worse than the "garbage" the plan warns
about: a NaN does not clip or clamp, it corrupts the frame, and nothing in the
render would say why.

Fixed by folding the un-picked `"picked"` case into the pivot branch:
`src === "manual" || src === "picked" || !mesh`. Re-verified after the fix:
`focalDistance` lands exactly on the pivot distance (`2.7753047965206017`,
matching `camera.position.distanceTo(controls.target)` to every digit) rather
than `NaN`, and the tested pick/tilt-pivot behaviour above was re-confirmed
unchanged by the fix.

### `pickThresh` — exposed, and it measurably moves where a pick lands

Below 0.5 picks front haze, above it picks deeper into the cloud, per the
plan. Measured at the same tap point, threshold 0.15 vs 0.9: **2.5952510855831017
vs 2.5981807730831017** — deeper is farther, correctly ordered, a small but
real and non-zero effect on this capture's opacity profile at that point.
`pickThresh` itself is inert on the rendered frame at its neutral value and at
every other value — it only affects the *next pick*, not the current render —
confirmed as part of the full regression sweep below.

### Mode interlocks — a third exclusive viewport owner, not a fourth exception

Pick joins Adjust (gesture) and Correct as viewport-owning modes, and the
existing interlock discipline extends rather than gets special-cased:

- `syncControlsEnabled()` gained one more term: `!gestureOn && !draggingGizmo
  && !pickOn`.
- `setGesture(true)` now also calls `setPickMode(false)` — Adjust and Pick both
  claim the canvas's own `pointerdown`, and two listeners racing on the same
  event for different jobs is exactly the kind of bug this project's mode
  system exists to prevent.
- `enterCorrect()` calls `setPickMode(false)` alongside its existing
  `setGesture(false)`, and the Pick toggle is disabled/re-enabled on the same
  `for (const id of […])` lines that already own `gestureToggle`'s disabled
  state.

Verified, each transition live: arming Pick then calling `setGesture(true)`
turns Pick off; arming Pick then entering Correct turns it off and disables
the toggle; leaving Correct re-enables it. All three confirmed via direct
state reads, not by inference from the UI.

### Regression sweep and preset

21-case sweep (20 from Phase C plus `pickThresh`), `3_3_2025.ply`, 940×568,
void, base **3.1763 / 13.250 / 13.962**: every case restores to base
bit-identically, and the final probe after the whole sweep is bit-identical to
the first. `pickThresh` itself reads as "no change" in the table for the
reason given above — it does not touch the render.

**No version bump.** `pickThresh` is the only new `SPEC` entry Phase D adds
(the pick point itself is intentionally session-only, per above), so it rides
the same additive-field rule as every other phase in this port. Verified:
`serialize() -> applyPreset() -> serialize()` byte-exact (modulo `saved`) with
`pickThresh = 0.63`, and a synthetic v6 file with the key deleted loads
cleanly and keeps the slider's current value rather than resetting it.

## Optics port, Phase E — post-raster pass

`DEV_PLAN_OPTICS_PORT.md`, Phase E. The first effects in this file that are
not per-splat: one fullscreen quad through one fragment shader, drawn over
the already-rendered frame. This phase implements the pass mechanism itself,
vignette, and grain; Phase F rides on top with film response, halation, veil
and the rebate.

### The scene is NOT rendered into an offscreen `WebGLRenderTarget`

The textbook post-processing pattern — and `Optics/optics.html`'s own
approach — renders the splat scene into an offscreen render target, then
draws a post quad from that target to the canvas. **Measured, this produces
a different image than rendering straight to the canvas**, same camera, same
frame, every uniform verified identical: mean R **8% higher** on the dense
daffodil capture, **28% higher** on a sparse, non-overlapping subset (`cullA
= 0.85`) — the sparse figure ruling out blend-accumulation-over-overlapping-
splats as the cause, since a non-overlapping subset showed the *larger*
deviation, not a smaller one.

Traced into `spark.module.js`'s own `onBeforeRender()`:

```js
if (spark.target) {
  spark.renderSize.set(spark.target.width, spark.target.height);
} else {
  renderer.getDrawingBufferSize(spark.renderSize);
  ...
}
```

Spark has its **own** notion of "target" (`spark.target`), distinct from
whatever `renderer.setRenderTarget()` happens to be bound to, and evidently
more of its per-frame state assumes it is drawing to the renderer's actual
canvas than just this one branch. Reproducing that assumption exactly —
finding and matching every place Spark's internals diverge for an
arbitrary target — was judged not worth the risk of a second, subtler
mismatch discovered later, possibly after this had already shipped.

**The fix: don't give Spark a second target to diverge against.** The scene
renders to the canvas exactly as it always has — the one path already proven
correct across every measurement in Phases A through D — and
`renderer.copyFramebufferToTexture()` snapshots those exact drawing-buffer
bytes into a `THREE.FramebufferTexture` afterward. The post quad reads from
that snapshot and draws back onto the same canvas. Spark never sees anything
but the canvas, ever; there is no second render path for its internals to
disagree with.

```js
function renderWithPost() {
  renderer.setRenderTarget(null);
  renderer.render(scene, camera);
  renderer.copyFramebufferToTexture(postTex);
  renderer.render(postScene, postCam);
}
```

**Verified exact after the fix**, both dense and the earlier-failing sparse
case:

| | coverage | mean R | luma |
|---|---|---|---|
| dense, raw vs. through-post | 3.1763 / 3.1763 | 13.250 / 13.250 | 13.962 / 13.962 |
| sparse (`cullA=0.85`), raw vs. through-post | 3.1033 / 3.1033 | 12.709 / 12.709 | — |

Bit-identical, both cases, at every digit.

### Post pass colour space — the bug the render-target approach also had, independently

Before abandoning the render-target approach, a **second, separate** bug was
found and is worth recording because it will recur the moment Phase F (or
anything else) samples a texture that carries `SRGBColorSpace`: WebGL
performs the sRGB decode **in hardware, on every `texture2D()` fetch**, for
any texture tagged that way — not merely as a label three.js reads
elsewhere. A passthrough shader with **zero** gamma math
(`gl_FragColor = texture2D(tDiffuse, vUv)`) still measurably darkened the
`void` background from `(8, 9, 11)` to `(1, 1, 1)`, proving the decode
happens whether the shader accounts for it or not.

The fix, now applied in the final (canvas-snapshot) shader rather than the
abandoned render-target one, is a **real sRGB EOTF/OETF pair**, not a
`pow(2.2)`/`pow(1/2.2)` approximation:

```glsl
vec3 srgbDecode(vec3 c) {
  vec3 lo = c / 12.92;
  vec3 hi = pow((c + 0.055) / 1.055, vec3(2.4));
  return mix(lo, hi, step(vec3(0.04045), c));
}
vec3 srgbEncode(vec3 c) {
  c = clamp(c, 0.0, 1.0);
  vec3 lo = c * 12.92;
  vec3 hi = 1.055 * pow(c, vec3(1.0 / 2.4)) - 0.055;
  return mix(lo, hi, step(vec3(0.0031308), c));
}
```

Measured, `pow(1/2.2)` in place of the real OETF left a visible drift
concentrated in the shadows — the `void` background round-tripped to
`(17, 17, 19)` rather than the exact `(8, 9, 11)` — because the real sRGB
curve has a linear segment below its breakpoint and a pure power curve does
not; the two diverge most exactly where a `void` background sits. The real
curve round-trips **exact to the tested byte**. Both halves are applied
explicitly in the final shader (decode on read, encode after the vignette
step) since `postTex` (a `FramebufferTexture`, untagged) carries no implicit
hardware conversion either way — unlike the abandoned render-target version,
neither half is "free."

### Vignette — verified against an independent CPU reference, not just self-consistency

```glsl
vec2 dv = (vUv - 0.5) * 2.0;
dv.x *= res.x / max(res.y, 1.0);
lin *= clamp(1.0 - vigAmt * smoothstep(vigRadius, vigRadius + vigSoft, length(dv)), 0.0, 1.0);
```

This is the road-tested shape from `Optics/optics.html`, ported as **behaviour**
rather than re-derived from the plan's own "cos⁴-style radial falloff"
description. The two are not the same formula — this is a `smoothstep`-
shouldered aspect-corrected radial falloff, not a literal `cos⁴(θ)` law — but
the plan's own reference implementation is the smoothstep version, already
visually road-tested, and the plan's instruction is to port *behaviour*
through `SPEC`/`V`/`pushUniforms()`, not to re-derive a new formula from a
one-line description of what the old one approximates. Recorded as a
deliberate choice, same shape of decision as `dofToAngle` versus a literal
f-number in Phase B.

Verified by reproducing the exact shader arithmetic in JS and comparing
against read-back pixels at four points (frame centre, two opposite corners,
one off-axis point), `vigAmt=0.6, vigRadius=0.5, vigSoft=0.4`: **every sample
matched the independently-computed prediction exactly**, byte for byte —
e.g. centre `(8,9,11)` predicted and measured identically (inside the
un-vignetted radius), corners `(3,4,4)` predicted and measured identically.

`vigAmt=0` is bit-identical to the pass being absent entirely (see the
render-target-vs-canvas table above — that comparison was taken at `vigAmt`
and `grainAmt` both 0). Coverage as a metric breaks down once vignette is
active — expected, not a bug in the metric's application elsewhere: vignette
darkens the whole frame including background pixels away from the reference
corner, so a naive "differs from the corner pixel" coverage count balloons
(3.18% → 35–61% across the swept range) purely from background pixels
themselves no longer being uniform. Mean R is the meaningful readout here and
moves monotonically with `vigAmt` (13.250 → 9.159 across 0 → 1) and with
`vigRadius` in the expected direction (a *larger* radius pushes the falloff
further out, so R rises back toward the un-vignetted value: 8.702 → 10.745 →
12.641 across radius 0.2 → 0.65 → 1.2).

### Grain — determinism verified directly, not assumed from the seed plumbing

Re-randomised per frame via a third hash coordinate (the frame/seed index),
not translated — translating the sample position slides the same field
across the frame and reads as banding, the exact bug this project's own
audio-shaping work independently rediscovered in a different context and is
guarded against here from the start. Both octaves sampled on rotated
lattices at unrelated angles, so neither beats against the pixel grid or
against the other octave.

**Two renders of the same seed are bit-identical; two different seeds
genuinely differ.** Measured directly, not inferred: rendering the same
frame twice at `seed=5` produced **byte-for-byte identical output**
(2,135,680 of 2,135,680 bytes matched); rendering once at `seed=5` and once
at `seed=6` differed in **69% of bytes** — proof the field is a real
independent re-randomisation each seed, not a near-no-op.

Amplitude floors at 0.30 in the shadows rather than reaching zero — film
grain falls off toward white but does not vanish, and a curve reaching zero
at the highlights would read as clean digital there. Colour decorrelates the
three channels through the hash's own third coordinate (not a spatial
offset, which would read as fringing rather than grain).

### Live-view seed vs. export seed — two different counters, on purpose

`grainFrame`, a module-level counter wrapped at 4096, advances once per
rendered frame in the live loop **regardless of Play/Pause** — grain is a
property of the viewfinder, not the timeline, and freezing it while paused
would read as a rendering fault rather than a still frame.

The export loop does **not** read `grainFrame`. It seeds from its own frame
index `i % 4096` instead:

```js
postMat.uniforms.seed.value = i % 4096;
```

**Seeding export from the live counter would make the exported grain depend
on real time** — how long the live view happened to be running before Render
was clicked — rather than on which frame is being encoded, breaking the
export-determinism guarantee this project protects everywhere else (Phase
4's camera uniform, the audio-shaping cache, camera-keyframe sampling). Two
renders of the same *export* frame are bit-identical by the same test above,
since the export loop deterministically derives its seed from `i` alone.

### The export loop renders through the pass — verified with a real export, not a simulation

A genuine 24-frame export (1280×720, 24fps, `vigAmt=0.3`, `grainAmt=0.2`) was
run through the actual `$("render").onclick` handler, not a mock: **completed
successfully, 2.6 MB, status "Rendered 24 frames".** `postTex`/`postMat.uniforms.res`
correctly tracked the fixed export resolution during the run and were
restored to the live viewport's own size afterward via `resize()` in the
`finally` block — verified directly: canvas size and `res` uniform both read
`[940, 568]` before, and `[940, 568]` again after, the export.

Both `renderer.render()` calls per export frame go through `renderWithPost()`,
not just the second (the one `VideoFrame` actually reads) — the first exists
only to kick off Spark's async sort and is otherwise discarded, but routing
it through the pass too means the export loop runs the exact same two-step
draw the live view does, rather than a special raw-then-post sequence that
only looks equivalent.

### `window.__bench` bypasses the pass entirely — the third trap, held

`draw()` and `settled()` call `renderer.render(scene, camera)` directly,
never `renderWithPost()` — unchanged by this phase, and re-verified after
the render-target-to-copyFramebuffer rewrite: every coverage/luma figure
measured for this phase (and re-measured for Phases A through D as part of
this phase's own regression sweep) used the raw, bypassed path. Same
save/restore discipline as the background-colour guard this file already
documents.

### Full regression sweep, 21 cases, unchanged from Phase D's own list

`3_3_2025.ply`, 940×568, void, base **3.1763 / 13.250 / 13.962**. Every one
of the 21 cases carried over from Phases A–D still restores to base
bit-identically after this phase's rewrite, and the final probe after the
whole sweep is bit-identical to the first — confirming the post-pass
architecture change touched nothing about the existing per-splat pipeline.

### Frame time

30- and 60-render averages at 44,296 splats: raw (no post pass at all)
0.075ms; through `renderWithPost()` at neutral 0.093ms; through
`renderWithPost()` with `vigAmt=0.5, grainAmt=0.3` active 0.065ms. **All
within noise** — the extra `copyFramebufferToTexture()` call and fullscreen
quad cost nothing measurable at this splat count and canvas size.

### Preset

**No version bump.** All six new parameters (`vigAmt`, `vigRadius`,
`vigSoft`, `grainAmt`, `grainSize`, `grainColor`) are additive `SPEC`
entries; v6 stands. Verified: round-trip `serialize() -> applyPreset() ->
serialize()` byte-exact (modulo `saved`) with all six off default, and a
synthetic v6 file with all six keys deleted loads cleanly, keeping the
slider's current values rather than resetting them — the same forgiving-load
rule every other additive phase in this port follows.

### A methodology note worth keeping: verify shader edits with `node --check`, not a hand-rolled scanner

Two late edits in this phase broke the module with `SyntaxError: Unexpected
identifier` — a JS **backtick inside a GLSL comment**, closing the enclosing
`` fragmentShader: `...` `` template literal early. A hand-written regex
scan (find the first backtick after `fragmentShader:`, treat that as the
string's end, check the substring for more backticks) missed the second
instance, because it mis-located the string's own boundary the moment the
first stray backtick existed — it can't detect what it's already using to
delimit its own search. Extracting the module's script content and running
it through `node --check` catches this precisely, points at the exact
character, and doesn't share the blind spot. Worth using that instead of an
ad hoc scanner for any future edit inside a template-literal shader string in
this file.


---

## Optics port, Phase F — film profiles, halation, veil, rebate

`DEV_PLAN_OPTICS_PORT.md`, Phase F, the last phase in the port. Rides on
Phase E's post-raster pass. Parametric film response, halation vs. veiling
flare as two genuinely different effects, split toning, and the film rebate.
Deferred, per the plan: measured LUTs — a `.cube` loader is the honest
upgrade path and coexists with this parametric chain rather than replacing
it, since the parametric controls stay animatable per-parameter and a LUT
does not.

### Profile data ported verbatim, not re-derived

`FILM_PROFILES` — ten looks plus the implicit "None" bypass — is copied
directly from `Optics/optics.html`'s own `PROFILES` table: `sens`, `mix`,
`con`, `white`, `sat`/`satRoll`, `lift`, `shTint`/`hiTint`, `hal`/`halTint`
per profile. These are **numbers**, not behaviour to re-express through
`SPEC`/`V` — the plan's own validation table marks them "visually
road-tested" as-is, and hand-tuned constants don't have a "correct"
independent of the look they were tuned to produce. What changed in the
port is the **wiring**: a profile is a discrete choice, the same category
`dField`/`wMode`/`qAxis` already occupy in this file, not a continuous
`SPEC` slider — so it is a `<select>` populated from `FILM_PROFILES`,
persisted via `SELECTS`, with `applyFilmProfile()` pushing the chosen
profile's fixed data into `postMat.uniforms.*` on change and on preset
load. Four continuous sliders ride on top, in `SPEC` group `film`:
Intensity (`filmMix`), Exposure, Halation (`halScale`, a *reach* multiplier
on the profile's own fixed halation strength), and Veiling flare (`veil`,
independent — no profile defines a veil strength, it is a pure user
control).

### Intensity 0 is an exact bypass — verified per profile, not just once

The whole film chain (halation, veil, emulsion response, saturation) sits
inside one `if (filmMix > 0.0)` block in the shader, not gated per-term.
Verified with **every one of the ten profiles selected in turn**, at
`filmMix = 0`: all ten produced a render bit-identical to the untouched
base (`3.1763 / 13.250 / 13.962`) — coverage, mean R and luma exact at every
digit, for every profile. This is the plan's own instruction taken
literally: *"Intensity 0 is an exact bypass, not a neutral profile — the
only honest way to A/B a look."* A "neutral" profile that still ran the
maths could round differently even at its own identity settings; this
verifies that isn't happening for any of the ten.

At `filmMix = 1`, every profile **does** move the render, confirmed against
the same base: coverage and mean R both differ from base for all ten.
Cross-checked against the profile data itself in three of the four
black-and-white profiles (`pan`, `ir`, `lith`, whose `shTint`/`hiTint` are
identity `[1,1,1]`): mean R equals mean luma **exactly** — the defining
property of a true monochrome image, `0.2126R + 0.7152G + 0.0722B = R` only
when `R = G = B`. `ortho`'s R and luma differ by 0.06%, and that is
*correct*, not a leak: `ortho`'s own `shTint`/`hiTint` are `[0.99,1.00,1.02]`
/ `[1.02,1.01,0.98]` — a deliberate warm/cool split tone layered on a
monochrome base, exactly as `selen` ("warm tone") also does. Reading the
per-profile data before asserting "this should be pure grey" is what
catches the difference between a bug and an intentional toning choice.

### Halation vs. veiling flare — verified as genuinely independent effects

The plan is explicit these are different effects (tight bright-pass ring
blur vs. wide low-threshold lift) and the rig "conflated them at first."
Verified independently:

- On the `neutral` profile (`hal = 0` in its own data), sweeping `halScale`
  0 → 1 → 2 is **bit-identical at every step** — the shader's own
  `if (fHal > 0.0 && halScale > 0.0)` gate holds; the reach slider has
  nothing to multiply when the profile defines no halation strength.
- On `warmneg` (`hal = 0.5`), the same sweep **does** move the render:
  coverage 3.136 → 3.276 → 3.614 → 4.294 across `halScale` 0 → 0.5 → 1 → 2,
  monotonic and increasing, matching a bright-pass bloom spreading further
  as its radius grows.
- `veil`, swept independently on `neutral` (no profile dependency at all,
  by design): coverage jumps from base 3.176 to **9.55** at the first
  nonzero step and keeps climbing gently after — a much larger, broader
  effect than halation's own sweep, consistent with veiling flare being a
  wide low-threshold lift across the *whole* frame rather than a bloom
  localised to bright points.

These are measurably different response shapes to their own sliders, not
merely two names for the same blur at different radii.

### Vignette formula and rebate SDF, verified against independent CPU references

Same discipline as Phase B/C/D's tilt-pivot and pick-depth checks. Vignette:
reproduced the exact shader arithmetic in JS (`smoothstep`-shouldered radial
falloff on aspect-corrected coordinates, applied in true linear light via
`srgbDecode`/`srgbEncode`) and compared against four read-back pixels —
frame centre and three off-centre points, `vigAmt=0.6, vigRadius=0.5,
vigSoft=0.4`: **every sample matched the independently-computed prediction
exactly**, byte for byte (centre `(8,9,11)`, all three others `(3,4,4)` or
`(4,5,6)`, predicted and measured identical).

Rebate: reproduced the rounded-box SDF (aspect crop, fill, corner radius,
no jitter for a clean binary in/out test) in JS and classified four sample
points — frame centre, two opposite corners, one edge midpoint — as inside
or outside the frame. **All four matched**: centre read the `void`
background (inside), the other three read the border colour `~(3,3,4)`
(outside), exactly as predicted.

### Determinism and the export loop — re-verified after Phase F's own additions

The film chain adds no new per-frame state beyond what Phase E already
established (`updateShaderUniforms()`/`pushUniforms()` push `postMat`'s
uniforms; grain's seed discipline is unchanged). A genuine export was run
with the full chain active — `slide` profile, `filmMix=0.6`, `halScale=1`,
`veil=0.3`, a square rebate at `frBorder=0.5` — and completed: **24 frames,
21,683 bytes**. That is a real, successfully muxed file, not a near-empty
one; it displays as "0.0 MB" purely from the status line's one-decimal
rounding (`0.0217 MB`), confirmed by capturing the exact blob size rather
than trusting the rounded string. The file is smaller than Phase E's own
2.6 MB export because that one had grain active — high-frequency per-pixel
noise is far harder for the video codec to compress than the smooth film
response and vignette in this run, which is the expected direction, not a
regression.

### Full regression sweep and preset round-trip

31-case sweep (every `SPEC` parameter from Phases A through F touched at
least once), measured against **`window.__bench.settled()`**, which
bypasses the post pass entirely: every Phase E/F-only case (`vigAmt`,
`grainAmt`, `filmMix`, `exposure`, `halScale`, `veil`, `frFill`, `frBorder`,
`frJitter`, `frRadius`, `frTilt`) reads as **no change at all** against the
raw base — correct and expected, since none of them touch the per-splat
pipeline `settled()` measures; this is the same bypass discipline Phase E's
own trap #3 exists to enforce, re-confirmed here rather than assumed still
in effect after Phase F's shader additions.

Preset: **no version bump.** All fifteen new `SPEC` entries (`filmMix`,
`exposure`, `halScale`, `veil`, `frFill`, `frBorder`, `frJitter`,
`frRadius`, `frTilt`, plus `filmProfile`/`frAspect` as additive `SELECTS`
entries) are additive; v6 stands. Verified: round-trip
`serialize() -> applyPreset() -> serialize()` byte-exact (modulo `saved`)
with the `slide` profile, a `0.8` (portrait) rebate aspect, and every new
slider off its default. `applyFilmProfile()` is confirmed to actually run
on preset load, not merely set the `<select>`'s displayed value: after
loading a preset with `filmProfile: "slide"`, `postMat.uniforms.fCon.value`
reads `1.32` — `slide`'s own contrast exponent — not whatever the previous
profile (or "None"'s inert identity value) had left behind. A synthetic v6
file with every Phase F key deleted loads cleanly and keeps the slider's
and select's current values, the same forgiving-load rule every other
additive phase in this port follows.

### Frame time

Not measurable at this canvas size and splat count, consistent with every
earlier phase's own finding: neutral post pass 0.063ms, full chain active
(profile + halation + veil + vignette + grain + a bordered rebate)
0.058ms — within noise. Halation and veil are the plan's own named concern
for mobile (the only per-pixel loops in the chain, 8-tap and 6-tap), but
that cost only shows up at a canvas size or splat count this rig's
committed test asset does not reach; recorded as unmeasured here rather
than assumed fine at every scale.

### This is the last phase in `DEV_PLAN_OPTICS_PORT.md`

Phases A through F are now ported, measured and committed. Two real,
non-hypothetical bugs were found and fixed along the way that the plan
itself did not anticipate: swirl's coupling to the wrong blur term (Phase
C, live from a Phase A default it silently depended on), and the
render-target color-space/target-dependence pair (Phase E, live from an
architecture choice the plan's own reference rig also carries and never
exercised under measurement). Both are documented in full where they were
found, not just noted as fixed.

---

## Section accordion and `SPEC` groups

`SPEC` entries are `[id, digits, group]`. The group ties a parameter to its rail
section, so resets, envelopes, audio routing and preset persistence all stay
driven by `SPEC` as it grows — the dev plan takes it from 26 parameters to
roughly 60, and retrofitting grouping around that would be far worse.

Sections are native `<details>`/`<summary>`: keyboard accessible for free, works
inside the mobile bottom sheet, and no open/closed state machine to own beyond
persisting it.

`reveal` is a group on borrowed time — Phase 3 of the dev plan folds it into
`region` — but it has to exist while its parameters do.

### Summary markers

A collapsed group must never hide that something inside it is animating. The
summary carries, in `--edit` violet:

- a **count** when any parameter in the group has an armed envelope or an audio
  route,
- a **·** when nothing is driven but something is off its default,
- nothing when the whole group is at default.

This mirrors the per-row `.dirty` convention so the rail reads the same
collapsed or expanded. Note that a reset-all clears the dots but keeps the
counts — resetting a slider does not disarm an envelope, and pretending
otherwise would be a lie about what is driving the render.

`refreshGroups()` is called from `refreshEnv()` — the single choke point every
envelope mutation already routes through — plus the slider input listener,
audio routing changes and preset load. It is deliberately **not** called from
`markDirty()`: `syncSliders()` calls that per driven parameter on every frame
during playback, and `refreshGroups()` walks all of `SPEC`.

### Open/closed state is in localStorage, not presets

Same reasoning as `Mobile profile`: it describes your workspace, not the look.
Key `splatbench.accordion.v1`. Nothing is written until you actually toggle
something, so a first visit gets the defaults: `camera` and `output` open.

### Measured

| | |
|---|---|
| rail content, all collapsed | 466px (fits a 900px viewport) |
| rail content, defaults open | 882px |
| rail content, all open | 2608px |
| summary row height | 41px desktop and mobile |
| mobile 390×844 | stage 692 + lane 96 + transport 56 = 844, no overflow |

### A note on comparing coverage across sessions

The Phase 0 regression initially looked like a uniform ~2% drop against every
figure in this file. It was not a regression: the browser window was a
different size, so the canvas was 1310×782 rather than the 1460×891 the
baselines were taken at, and **coverage is a percentage of canvas area**.

Colour channels were unchanged, which was the clue. The size-independent check
is the ratio of each case to its own BASE, which cancels canvas geometry — those
matched the recorded figures to within 0.003. **Record the canvas size next to
any coverage figure, or compare ratios.**

### Warm-up: "stable" is not enough, it must be stable AND non-zero

The documented warm-up rule — probe until two consecutive reads agree — is not
sufficient on its own. Two consecutive **zeros** agree, and zero is exactly the
not-yet-warm signature. The rule is: two consecutive agreeing reads that are
also non-zero, with a cap.

### Check the background pixel, not just the coverage — it separates two
### different zeros

A **hidden tab** produces the same all-zero coverage as a cold generator, and
they need opposite responses: warm up more, versus front the tab and start
over. There is a one-value discriminator, and it costs nothing because the
probe already reads the corner pixel to establish the background:

| corner pixel | meaning |
|---|---|
| `[8, 9, 11]` (the `void` colour) | the buffer is real; zero coverage is a genuine render result or a cold generator |
| `[0, 0, 0]` | the buffer was never composited — the tab is hidden, **every** number from this read is meaningless |

Hit live while running the post-rename regression sweep: an entire 10-effect
sweep came back all-zero and "all restored", which reads like a pass and is
actually the harness measuring a black rectangle. `document.hidden` was `true`;
Chrome stops rAF in a background tab, which NOTES already warned about, but the
failure presents as *plausible data* rather than an error, which is what makes
it dangerous. So the probe's stability check now requires a non-zero background
as well: **stable, non-zero coverage, on a real background.** Front the pane
tab before measuring, and treat a `[0,0,0]` corner as "no reading taken",
never as a value.

---

## Gesture mode — direct manipulation

On a phone the rail covers the subject, so adjusting a slider means not seeing
what it does. `Adjust` binds up to two parameters to the drag axes and edits
them on the viewport itself. It works on desktop too — a mouse drag is a drag.

A full drag across the stage sweeps the bound parameter's own range (or
sub-range — see below), and drags are relative to where the pointer went
down, so repeated strokes keep nudging.

Two decisions worth keeping, unchanged by the rework below:

- **It writes through the slider's own `input` event**, never to `P` directly.
  That is the single path already handling auto-key on armed envelopes, dirty
  marking, the `dpr`/`dur` side effects and `pushUniforms`. A second write path
  would drift out of sync with it. Verified: dragging with an armed envelope
  adds a keyframe, exactly as dragging the slider does.
- **Orbit and adjust are modal, not gesture-count-based** (one finger vs two).
  On a trackpad and on a phone those are easy to confuse, and a mis-detected
  gesture silently edits a parameter instead of moving the camera. An explicit
  mode is duller and much harder to get wrong. `controls.enabled` is the
  interlock, so the two can never both be live.

### `DEV_PLAN_GESTURE.md` — assignment moved off a `SPEC`-enumerating strip

The original mechanism was a chip per `ANIM` id (cycling **unbound → X → Y →
unbound** on tap), built when `SPEC` had 18 entries. By the time Region,
Saturation and Orbit landed it enumerated ~75 chips for ~60 animatable
parameters — a bad picker, even though it had always been a reasonable
*switcher*. Replaced with per-parameter assignment in the lane header
(`#gestX`/`#gestY`, contextual to the focused parameter — the same place
Audio routing and Mask already live), plus a `Range ▾` disclosure for
sub-range and invert.

**Verify-before-design step, confirmed real, not assumed:** the strip
enumerated `ANIM`, which includes all 40 region `SPEC` entries (4 slots x 10
fields) regardless of which `.rgpanel` tab was actually visible. Measured
directly before touching any code: 75 total chips, 40 of them region chips,
30 of those 40 belonging to slots other than the currently-visible one
(`visibleSlot: "0"`, `hiddenSlotChipCount: 30`). Clicking a hidden-slot chip
(`rg2X`) bound it successfully — `chip bx` applied — while its own panel
measured `panelVisible: false`. So the strip really did let you bind and drag
a parameter you could not see, confirming the plan's own suspicion rather
than just reorganising around an assumed problem.

### Design

`GEST[ax]` is `null` or `{id, lo, hi, invert}` — `lo`/`hi` are fractions 0..1
of the parameter's own slider travel (default 0/1, so an unranged bind
behaves exactly as the old full-range-only mechanism always did), `invert`
reverses drag direction. Plain JS routing state, not `SPEC` — same treatment
as `AUD`'s band/depth and region `shape`/`mode`/`invert`.

- **`#gestX`/`#gestY`** toggle whether the *focused* parameter holds that
  axis. Binding an axis already held by another parameter displaces the
  previous holder (unchanged rule); claiming one axis clears the same
  parameter from the other first, so a parameter can never hold both at
  once — verified: binding `sat` to Y while it already held X moved it
  (`{x:null, y:{sat,...}}`), it did not duplicate.
- **`#gestRangePanel`** (`Range ▾`, same disclosure convention as Audio
  shaping) edits `lo`/`hi`/`invert` for whichever axis the focused parameter
  currently holds; disabled when it holds neither.
- **Row indicator.** A `<span class="gaxis">` lives inside every animatable
  row's own `<label>` (created once per row at boot, alongside the existing
  `◆`/`↺`), textContent set to `"X"`/`"Y"`/`""` — **not** a new grid column
  or a per-row button, exactly per the plan's own instruction: the row grid
  already carries five things across ~57 rows, and a control most rows would
  never use does not belong there. `refreshGestureUI()` tracks which row
  last carried each axis (`lastAxisRow`) so a rebind only ever touches the
  0-2 rows that actually changed, not all 57.
- **Viewport readout** repurposes the old drag-only text HUD (`#gestureHud`)
  into persistent, tappable chips — visible whenever Adjust is on, not just
  mid-drag, so you can see what's bound without opening the sheet. Tapping a
  chip unbinds that axis and does nothing else. `pointer-events:none` on the
  container, the chips themselves the only hit targets — moot in practice
  since Adjust and Orbit are already mutually exclusive (the chips are
  `display:none` whenever Orbit could be dragging), but kept as the belt per
  the plan.

### A real TDZ bug caught before it ever loaded, not by testing

`refreshGestureUI()` (which reads `GEST`) is called from `laneHead()`, and
`laneHead()` is called once at **boot** (`laneHead();`, well before the
gesture module's own code runs) — the *exact* shape of bug this file's own
"State — declared BEFORE anything that reads it" block at the top exists to
prevent (see Fixed, "TDZ on `playhead`"). `GEST` was originally declared as a
`const` down in the gesture module, ~1900 lines after that boot call; had it
shipped as written, the very first `laneHead()` call would have thrown a TDZ
`ReferenceError` and silently killed every listener and the render loop
below it — the original bug, verbatim, in new state. Caught by re-reading
this file's own conventions before testing, not discovered by a crash:
`GEST`/`gestureOn`/`gdrag`/`lastAxisRow` moved up to the top-of-file state
block with everything else that has to exist before first use.

### Measured

Against the loaded `butterfly.spz`, real `PointerEvent`s dispatched at
stage coordinates (a fresh tab — see the Phase 2d environment note above for
why that matters for synthetic dispatch reliability) unless noted:

- **Full-range drag**: `dAmt` bound to X (range 0..0.5), drag from the
  stage's left edge to its right edge — `V.dAmt` measured **exactly 0** at
  the start and **exactly 0.5** at the end, endpoint to endpoint. (A first
  attempt starting/ending 5px inside each edge measured 0.495 at the far
  end — not a bug, `frac` genuinely does not reach 1.0 unless the drag
  starts and ends exactly at the stage bounds; corrected and re-measured.)
- **Exclusivity**: binding `sat` to X displaced `dAmt` (X held `sat`, not
  `dAmt`, immediately); binding the same `sat` to Y *moved* it rather than
  duplicating — `{x:null, y:{sat}}`, never `{x:{sat}, y:{sat}}`.
- **Auto-key regression — the acceptance line called out as "existing
  verified behaviour and the thing most likely to break."** `dAmt` armed
  with 2 pre-existing keys, bound to X, playhead at `t=0.3`: a gesture drag
  added a **third** key at exactly `t=0.3` (`keysBefore: 2, keysAfter: 3`,
  matching the dragged value) — dragging a bound, armed parameter still
  writes a keyframe, identically to dragging its slider.
- **Sub-range**: `cullA` bound to X with `lo=0.4, hi=0.6` (range 0..1) — a
  full-stage drag measured **exactly 0.4** at the start edge and **exactly
  0.6** at the end edge.
- **Invert, verified pointwise, not just at the endpoints.** Same drag
  (0→full stage) from a mid-range start (`0.5`): normal binding sampled
  `[0.75, 1, 1, 1]` at 25/50/75/100% of the drag; inverted sampled
  `[0.25, 0, 0, 0]` — mirrored around the 0.5 start point at every sample,
  not just at the ends. (A first attempt started the tracked value already
  at an extreme, 0 or 1, and measured a flat `[0,0,0,0]`/`[1,1,1,1]` in both
  directions — not a bug, just a clamp against a boundary hiding the effect;
  starting mid-range is what actually exercises inversion.)
- **Hidden-slot closure — confirmed, not just no-longer-possible-to-notice.**
  `element.click()` on a hidden-slot label *does* still fire its listener
  (a JS-level method call bypasses hit-testing entirely, so this is not a
  real test of reachability) — the correct instrument is the element's own
  geometry: `getBoundingClientRect()` on a hidden-slot label measures
  `{x:0, y:0, w:0, h:0}`, `offsetParent: null`, `tabIndex: -1`. There is no
  screen coordinate a real pointer event could target, and it is excluded
  from keyboard tab order too — no input path, mouse, touch or keyboard,
  can focus it, which is the actual guarantee the acceptance line asks for.
- **390px worst case** (a maskable parameter, armed envelope, audio route,
  and the gesture controls all present on one focused row): lane header
  measured `scrollWidth 892` vs `clientWidth 386` — needs a scroll, and
  `#gestX` measured reachable via `scrollIntoView` within the header's
  bounds, with zero page-level horizontal overflow.
- **Preset round-trip**, both axes bound with distinct sub-ranges and one
  inverted (`x:{dAmt,lo:0.15,hi:0.85,invert:true}`,
  `y:{sat,lo:0.2,hi:0.9,invert:false}`) — saved, wiped, reloaded: restored
  object-equal to the saved state, both axes, all four fields each.
- **No `SPEC`-enumerating element remains**: `document.querySelectorAll
  ('.chip').length === 0`, `#palette` absent from the DOM entirely.
- Console clean across the full sequence apart from one expected,
  environment-specific noise source — see below.

### Environment note: synthetic `PointerEvent`s and `setPointerCapture`

`renderer.domElement.setPointerCapture(e.pointerId)` (pre-existing code,
unchanged by this phase) throws `NotFoundError` when the triggering
`pointerdown` was a synthetic `dispatchEvent` rather than a real hardware
event — there is no genuinely "active" pointer session for a fabricated
`pointerId` to capture. Confirmed harmless to every measurement above: the
throw happens on the *last* line of the handler, after `gdrag` (the state
every subsequent drag calculation reads) is already assigned, and every
numeric result quoted above came out exactly as predicted despite the
exception firing. A real user's pointer events always have a valid active
session; this is purely a synthetic-testing artifact, the same family as
the tab-degradation and stale-readback notes elsewhere in this file, not a
regression to chase.

### Interaction with multi-instance (roadmap item 8)

Per the plan: not pre-solved. `GEST` holds parameter ids, so when multiple
`SplatMesh` instances land, it will need the same scoping decision as `AUD`
and the region slots — bindings either follow the selected mesh or stay
global. Noted here so that work has a list of "things holding parameter
ids" rather than discovering them one at a time.

### A real bug, pre-existing, surfaced by this phase's own panel: `#lane`'s box never grew to fit its own disclosure panels

User-reported: Audio shaping's Attack/Release sliders were "hidden from
view and so uneditable." Traced to `#lane` (`position:fixed; height:
var(--lane)`, a **static** 88px/96px) never having grown to fit
`#audShapePanel` when open — that panel's own natural height (`~195px`,
canvas + two rows of three sliders) has always overflowed the lane's fixed
box straight into `#transport` below it, which is opaque and later in the
DOM, so it paints over whatever of the panel's content lands in its
screen rectangle. `overflow: visible` on `#lane` meant nothing was
*clipped* — the sliders were fully present in the DOM, laid out, just
invisible under `#transport`'s background and (since `#transport` occupies
that screen position) not receiving real clicks either.

**Confirmed pre-existing, not introduced by Part 1 above:** checked out the
immediately-prior commit (`efdbae8`, before any gesture-rework changes) and
reproduced the identical geometry — `attackRect.top: 950.9` against
`laneRect.bottom: 936`, byte-for-byte the same numbers as the current
build. It surfaced now because Part 1 added a second disclosure panel
(`#gestRangePanel`) with the exact same latent defect, which is what
prompted actually re-checking this area rather than trusting the DOM
existed and moving on.

**Fix**: `--lane` is now `calc(<base> + var(--lane-extra))` (both the
desktop 88px and mobile 96px definitions), where `--lane-extra` is a
JS-driven custom property, not a static one. `updateLaneHeight()` sums the
measured `getBoundingClientRect().height` of every disclosure panel
currently **not** `.hide` (Audio shaping, Gesture range — both can be open
at once for the same row, hence sum, not max) and writes that total to
`--lane-extra`. Measuring an unhidden panel's own height works correctly
even though its *parent* box is still constrained, because `flex:0 0 auto`
children size from their own content regardless of a flex container's
overflow, and `#lane` is `overflow: visible` — so nothing about the old
overflow bug prevented measuring the true, correct height to fix it with.

Called from both panels' own toggle handlers, and — once, covering every
other path that can hide a panel (focus moving to a non-audio-routable
parameter, an axis losing its binding while its range panel is open) —
from the end of `refreshGestureUI()`, which every `laneHead()` call already
reaches. `#stage`'s own CSS size already reacts to `--lane` via `calc()`
with no JS needed, but the WebGL canvas itself does not auto-follow a CSS
resize, so `updateLaneHeight()` also calls the existing `resize()` (same
one bound to the window's own `resize` event) and `laneFit()` (the lane
timeline canvas's own backing-store fit, a distinct concern — pixel
density, not layout box size — already called here before this existed).

**Measured**, against the loaded `butterfly.spz`, before and after:

| | before | after |
|---|---|---|
| `#lane` height with Audio shaping open | 88px (static) | 282.8px |
| Audio shaping Attack row inside `#lane` bounds | **false** | **true** |
| `document.elementFromPoint()` at the Attack slider's centre | (not checked before the fix existed) | resolves to `#audAttack` itself |
| `--lane-extra` with the panel closed | n/a | exactly `0px` |
| `--lane-extra`, Gesture range panel alone | n/a | `135.8px` |
| `--lane-extra`, both panels open together | n/a | `330.7px` (sum of the two above, not the max) |
| `camera.aspect` before/after opening a panel | — | changes (`2.06 -> 1.26` in one measured case) — confirms `resize()` is actually reached, not just the CSS box |
| Mobile (390px), same test | — | Attack row inside bounds, zero page-level horizontal overflow |

Console clean across the full sequence (checked specifically against this
session's own messages, not the tab's full history, which — see the
Phase 2d/synthetic-`PointerEvent` note above — accumulates old, already-
documented noise across navigations in a long-lived tab).

---

## Anisotropic smear (B1)

> **Extended by "Optics port, Phase C" below.** `buildSmearVertexShader()` is
> now `buildSplatVertexShader()` and patches **four** anchors rather than two;
> the shader constant is `SPLAT_VERTEX_SHADER` and the anchor strings live in
> one `SHADER_ANCHORS` object. `updateSmear()` is now `updateShaderUniforms()`
> and pushes all nine shader uniforms. Everything below about *why* the hook
> works this way, and about the upgrade hazard it creates, is unchanged and
> still the governing constraint.

Screen-space directional streak, added the same way Phase 5 added depth of
field: as a modification to Spark's own `a`/`b`/`d` (2D covariance) inside
its native vertex shader, not as an `objectModifiers` dyno node. `covSplats`
remains untouched — **this is the second phase to reach that conclusion**
(Phase 5 was the first), so it should not need re-deriving a third time.

### The shader hook: first-class API, but whole-file string replacement

`SparkRenderer`'s constructor accepts `options.vertexShader` (a full string
override, default is `getShaders().splatVertex`) and `options.extraUniforms`
(merged into the material's uniforms) — both are genuine, documented
constructor options, not internals reached through a back door. That part of
the spike's question resolved cleanly.

The other half did not: there is no template/injection-point mechanism.
`splatVertex_default` is a single fixed string; adding logic means locating
anchor text inside it and splicing, the same technique used to reach this
conclusion in the first place. **This pins the app harder to Spark 2.1.0 than
Phase 5 did** — Phase 5 only *read* property names (`focalDistance`,
`apertureAngle`) that could plausibly survive a minor bump; this phase
depends on two literal source lines (`uniform float apertureAngle;` and
`float detOrig = a * d - b * b;`) still existing verbatim. Recorded here as
the known upgrade hazard the plan asked for. The mitigation already in place:
`buildSmearVertexShader()` asserts both anchors exist and throws a named,
actionable error at boot instead of silently compiling a shader that does
not do what the app thinks it does — see `index.html` around
`buildSmearVertexShader`. The shader text itself is never hand-transcribed;
a throwaway `SparkRenderer` probe instance is constructed just to read
`.material.vertexShader` at runtime, and the splice runs against that live
string.

### Composition with DoF — one correction, not two

The smear anchor sits *between* the shader's own `float detOrig = a * d - b
* b;` line and its next line, `a += fullBlurAmount;` (DoF's isotropic term).
So the actual order after splicing is: compute `detOrig` from the *true*
original covariance (before either effect) → add smear's anisotropic term
→ add DoF's isotropic term → compute `det` once → `blurAdjust = sqrt(detOrig
/ det)` once. Both effects land in `a`/`b`/`d` before the single energy
correction runs. This was verified by reading the actual spliced insertion
point, not inferred from the intent — see "Measured" below for what a
double correction would have looked like and why the composed-darkening
result isn't one.

Gated on `smearAmt > 0.0`, exactly as DoF gates on `apertureAngle > 0.0` —
`smearAmt = 0` is the shader's own no-op branch, not an approximately-zero
term.

### `smearAmt`'s range — swept, not assumed

Per the plan's explicit instruction, `smearAmt` was swept from 0 upward
before settling `p-smearAmt`'s `min`/`max`/`step`. Coverage (`cov`, a
readPixels-based pixel-count proxy) on `butterfly.spz`:

| `smearAmt` | 0 | 1 | 2 | 3–5 | 10 | 25 | 100 | 500 | 1000 | 2000 | 4000 | 8000 | 16000 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| `cov` | 13.673 | 13.673 | 13.673 | 14.317 | — | — | — | — | — | 20.748 | 20.748 | 20.748 | 20.748 |

Response is flat below `amt≈3` (a measurement-resolution artefact of `cov`
itself, not a shader threshold — `cov` is a discrete pixel-coverage count,
so a sub-pixel growth in splat footprint doesn't register until it crosses
a pixel boundary), then scales smoothly up to `amt=2000`, then **saturates
exactly at 2000** — identical `cov`/mean-R/total-luma readings at 2000,
4000, 8000 and 16000. Root cause (confirmed by reading the shader directly,
see "Honest limitation" below): Spark's own `scale1`/`scale2` are each
`min(maxPixelRadius, ...)`, and `maxPixelRadius` defaults to 512px — past a
certain `smearAmt` every affected splat's screen radius is pinned at the
clamp regardless of how much larger the underlying covariance grows. The
shipped range (`min=0 max=2000 step=5`) is therefore well-chosen: it spans
the entire region where the parameter still does anything, and nothing past
2000 would ever be reachable through a wider slider.

### Measured

Against the loaded `butterfly.spz` (177,132 splats), `window.__bench`
throughout.

- **`smearAmt = 0` reproduces baseline exactly, on the gate.** Bit-identical
  `cov`/`R` readings across five different `smearAngle` values (0°, 45°,
  90°, 135°, 180°) at `smearAmt = 0`, matching DoF's own `apertureAngle`
  gate precedent exactly.
- **Anisotropy, not isotropic blur with extra steps.** Bounding-box probe
  (readPixels, foreground = differs from the corner background pixel by
  more than a small threshold) at `smearAmt = 500`: baseline footprint
  518×454px. At `smearAngle = 0°`: 589×452px — width grows by 71px, height
  is unchanged within noise. At `smearAngle = 90°`: 517×525px — height
  grows by 71px, width is unchanged within noise. Symmetric, single-axis
  growth in both cases: this is what an outer-product anisotropic term
  looks like, not an isotropic one.
- **Luma conservation, sparse subset (`cullA = 0.85`, Phase 5's established
  method) — holds within 2% only up to `smearAmt ≈ 50`, then degrades
  smoothly.** Deviation from the `smearAmt = 0` baseline: `+0.88%` at 5,
  `-0.06%` at 25, **`-1.39%` at 50** (last point inside the plan's 2%
  bound), `-3.26%` at 100, `-6.18%` at 200, and continuing smoothly down to
  roughly `-26%` by 1000–2000 before flattening (tracking the same
  `maxPixelRadius` saturation as `cov` above). This is a genuine,
  mechanism-explained energy leak, not a measurement artefact — see "Honest
  limitation" below for the root cause and why it is not a bug in this
  phase's own composition logic.
- **DoF composition — no double correction.** With both effects active
  (`smearAmt = 25`, `dofAperture = 2°`, `cullA = 0.85`) luma deviation from
  baseline was `-22.52%`, versus `-13.47%` for DoF alone and `-0.06%` for
  smear alone at that `smearAmt`. This looks like more than the sum of the
  parts, and it is — but not from a second `blurAdjust` application. The
  insertion-point reading above already rules that out at the source level:
  there is exactly one `detOrig`, one `det`, one `blurAdjust` in the
  spliced shader. The excess darkening has the same root cause as the
  single-effect leak above: combining two growth terms in the same `a`/`d`
  produces *more* total covariance growth than either alone, so more
  splats cross the `maxPixelRadius` clamp at a given nominal setting. Same
  mechanism, larger population affected, not a distinct bug.
- **Frame time**: 30-render average, `smearAmt = 0` **0.18ms** vs
  `smearAmt = 500` **0.07ms** — within noise, no measurable added cost,
  matching DoF's own precedent.
- **48-frame export, animated `smearAngle` envelope** (keyed 0° → 180°
  across the clip): replicated the exact export-loop call sequence
  (`setPlayhead` → `pushUniforms` → `updateDofFocus` → `updateSmear` →
  render) for all 48 frames. `smearAngle` uniform tracked the envelope
  smoothly (`0.00° → 0.24° → 0.95° → … → 179.76° → 180.00°`), **zero frozen
  frame pairs** across the full sequence.
- **Preset round-trip byte-exact** with both parameters off default
  (`smearAmt = 775`, `smearAngle = 63` — `775` because a range input with
  `step="5"` silently snaps any programmatically-assigned value to the
  nearest step, confirmed directly; this is a test-harness quirk, not an
  app bug). Two consecutive saves of the same loaded state differ only in
  the preset's own `"saved"` timestamp field; the entire `params` block,
  smear included, is identical byte-for-byte.

### Honest limitation, recorded per the plan's own instruction

Luma conservation does **not** hold within 2% across the shipped
`smearAmt` range — only up to roughly a quarter of it (`≈50` of `2000`).
The cause is fully diagnosed, not hand-waved: Spark's `splatVertex_default`
computes `blurAdjust` (the alpha correction) from the covariance *before*
`scale1`/`scale2` clamp the resulting screen radius to `maxPixelRadius`
(512px default). Once a splat's smeared covariance pushes its eigenvalue
past that clamp, its footprint stops growing but its alpha has already been
reduced as if it kept growing — a real energy leak, inherent to Spark's own
clamp design, and structurally identical to (not caused by) this phase's
composition logic. It is not specific to anisotropy either: the same clamp
would eventually undercorrect an extreme isotropic `blurAmount`/aperture
too, this phase's sweep just happened to be the one that pushed far enough
to find it. Not fixed in this phase — fixing it would mean either
recomputing `blurAdjust` from the *clamped* radius (a further shader
splice, off the plan's B1 scope) or capping `smearAmt` well below where the
clamp engages (which would cut the slider's usable range roughly in half).
Recorded as a known, explained boundary the same way Phase 5 recorded the
dense-capture non-conservation: the sparse-subset measurement is the one
that actually isolates and confirms the per-splat claim, and the failure
mode past `amt≈50` is the honest finding, not smoothed over.

### Mobile pass

At 390×844, the `smear` accordion group open: both rows measured
`right: 374px` inside a 390px viewport, `document.documentElement.scrollWidth
== innerWidth == 390` — no horizontal overflow. Lower risk than the Gesture
phase's lane-header work by construction: `smear` is a plain `<details
class="mod">` group using the exact same row markup as every other
parameter group (`dof` included), and does not touch `#lane` or either
disclosure panel that phase's overflow fix targeted.

### Scope: B1 only — screen-space, single direction

One direction in screen space (`smearAngle`, 0–180°, 0 = horizontal), no
per-splat direction, no world-space projection — deliberately, per the
plan. **Not built:** world-space direction (would rotate correctly with
camera orbit — the obvious next step, not this work); per-splat direction
aligned with each splat's own displacement (B2) — the displacement happens
in the dyno modifier and the splat is baked by vertex-shader time, so the
direction isn't carried through to where this phase's shader logic runs.
Evaluate B2 only after living with B1.

---

## Rail reorganisation, renames, and audio loop

One release, four separate changes, recorded together because they landed
together.

### Section names and order

The rail is now ordered as a pipeline — what the splat *is*, then what is done
to it, then how it leaves — rather than in the order effects happened to be
built:

`Camera · Transform · Opacity · Masks · Depth · Colour · Noise · Quantise ·
Wave · Smear · Audio · Output · Preset · Viewport`

Renamed: Shot → **Camera**, Model → **Transform**, Trim → **Opacity**, Region →
**Masks**, Displace → **Noise**. The old names were the internal names for the
mechanisms; the new ones are what the controls actually do to the picture.

**`data-group` ids were deliberately NOT renamed.** `trim`, `region`,
`displace` still identify those groups in `SPEC`, `refreshGroups()`, the
accordion store and the mask/region uniforms. Renaming them would have touched
the dyno graph, the preset region block and 36 generated row ids to change a
string nobody sees. Display labels are in the markup; group ids are plumbing.
The one place this leaks is the region slot buttons, relabelled `R1..R4` →
`M1..M4` in both the Masks section and the lane's Mask dropdown, since those
*are* user-facing.

**Depth fade and depth of field merged into one `Depth` section**, per the
request. They are two different mechanisms — fade runs in the dyno graph, field
runs in Spark's vertex shader — but they answer the same question, so they now
share a group id (`depth`) and `refreshGroups()` counts the whole section. The
two halves are separated by `.subhead` divs, **not** a second `<summary>`: a
`<details>` has exactly one disclosure control and a second `<summary>` inside
it is invalid, so these are plain headings styled to read like one.

**"Shear" in the request was read as "Smear."** There is no shear effect and
never was; Smear sits at exactly the position Shear was asked for in the
ordering, and shear (a skew) is not what the effect does (a directional
streak). Named Smear.

### Section titles moved to `--signal`

They were `--ink-dim` on `--chassis` — about **4.1:1**, under AA for 10px
uppercase at `.24em` tracking, and genuinely hard to scan. Now `--signal`, at
about **13:1**.

This does not add a fourth accent or overload the third. A section title is
structural navigation in a fixed position and never appears in the same role as
an inline value readout, so the two cannot be confused — unlike, say, putting
`--signal` on a button, which would read as "this holds a value." The `em`
descriptor stays on `--rule` so the title itself still carries the emphasis.

### Audio loop, and Duration × N

Two controls in the Audio section, one mechanism.

**Loop** repeats a track shorter than the timeline instead of going silent
after it ends. The whole design constraint is that **four readers have to
agree**, or the preview stops predicting the export — which is the exact
failure the offline-envelope architecture exists to prevent:

| reader | looping behaviour |
|---|---|
| `sampleArr()` — envelope lookup | wraps modulo `audioDur` instead of clamping |
| `audioSeek()` / `audioFollow()` — monitoring | wraps, and sets `audioEl.loop` so the element loops itself |
| `encodeAudioTrack()` — export muxer | repeats the PCM to fill `endSeconds` |
| `drawSparkline()` — lane readout | playhead marker wraps within the one drawn pass |

So it is **one** `audioLoop` flag, declared at the top of the audio block ahead
of every function that reads it (the TDZ rule from the gesture phase), not a
per-call-site option. Two details worth keeping: the envelope's last sample
interpolates back into the first rather than holding, or every wrap puts a flat
step at the seam; and the export copies PCM **per sample** rather than with
`subarray()`, because a 4096-frame chunk can straddle a loop seam and
`subarray` would run off the buffer and pad silence at exactly the point the
track should be restarting.

**Duration × N** sets Duration to exactly N track lengths. This needed two
changes to `p-dur`: `step="any"` (a 0.5 step would land the timeline a few ms
off the loop point every single time, which is the entire thing this control
exists to prevent) and `max` widened `30 → 120` (looping a 30s track even twice
needs more than 30). `dur`'s readout went to 2dp to stop it misreporting an
exact multiple like `24.69`. Setting `N > 1` arms Loop with it — fitting three
track lengths and then leaving two of them silent is never what was meant.

The write still goes through the `p-dur` slider's own `input` event. **No
second write path** — that slider is where auto-key, dirty marking, the
`dur`/`dpr` side effects and `pushUniforms` all hang off, and the gesture phase
already went out of its way not to add one.

`audio.loop` is additive in the preset, no version bump, absent means `false`.

### Measured

On `3_3_2025.ply` (44,296 gaussians) and `bandtest.wav` (6.0s), `window.__bench`
throughout.

- **Test asset loads and renders from the repo**: 44,296 gaussians, 3.384%
  coverage, mean R 179.2 against the `void` background, correct upright
  orientation, zero console errors.
- **Envelope wrap is exact.** With Duration = 3 × 6.00s = 18.00s and Loop on,
  `routeLevel(t)`, `routeLevel(t + 6)` and `routeLevel(t + 12)` agreed to **6
  decimal places** at every probe (t = 0.3, 1.0, 2.7, 4.5, 5.9).
- **Loop off is unchanged**, not approximately unchanged: past the end the
  level holds the final sample (`0.000625`) at +1s, +5s and +11s, identical to
  the pre-existing clamp, and differs from an in-track value.
- **Export audio fills the timeline.** `encodeAudioTrack(muxer, 18)` with Loop
  off spans **6.084s** (capped at the track, old behaviour); with Loop on,
  **18.088s** — the full timeline. 262 chunks vs 779.
- **The repeats carry real audio, not silence.** Per-pass encoded bytes 42,811
  / 77,744 / 78,283. Passes 2 and 3 agree to **0.7%** — identical content
  encodes to nearly identical size. Pass 1 is smaller from AAC encoder
  warm-up/priming, not missing audio; silence would be a few hundred bytes.
- **Duration × N is exact**: `P.dur === audioDur * 3` exactly, and Loop
  auto-armed.
- **Preset round-trip byte-exact** (modulo the `saved` timestamp) with
  `audio.loop: true`, `dur: 12`, and smear/DoF off default — verified
  idempotent across three independent save→load→save runs.
- **Mobile, 390×844**: no page overflow, no element off-screen, and the new
  `× set` button hit-tests to itself via `elementFromPoint` — genuinely
  tappable, not merely laid out.
- **Full effect sweep after the reorganisation**, base coverage **3.384%**:
  every effect still moves the picture and every one restores to base
  *exactly* — `cullA` 3.350, `sat` 3.389, `dAmt` 13.456, `qAmt` 2.747,
  `dpAmt` 3.223, `dofAperture` 5.592, `smearAmt` 8.207, `bright` 3.407,
  `scale` 4.384, all returning to 3.384 on reset. This matters because the
  merge moved `dofFocus`/`dofAperture` to a different `SPEC` group id and the
  section markup was rewritten wholesale.
- **`wAmt` alone changes nothing, and that is correct, not a regression.**
  Wave modulates Noise / Quantise / Attract rather than displacing anything
  itself, so at their defaults it has nothing to act on. Verified rather than
  assumed: with `dAmt = 0.3` active, `wAmt = 0.6` moved coverage
  **13.456 → 10.498**. Worth writing down because a sweep that flags Wave as
  "no effect" looks exactly like a broken uniform.

---

## Colour

Three accents, and they mean different things. Do not reach for a fourth
without a reason. Section titles use `--signal` — see the rail-reorganisation
section above for why that is not a fourth meaning.

| token | use |
|---|---|
| `--signal` mint | values, readouts, audio routing |
| `--live` orange | transport, recording, a key **at** the playhead |
| `--edit` violet | automation affordances: armed envelopes, changed-from-default, gesture bindings |

The reset and key glyphs originally sat on `--rule`, which is the *border*
colour — near invisible against the chassis by design. They now idle at
`--ink-dim` at 55% opacity and go to `--edit` when they carry state, at 16px
desktop / 26px mobile. Idle-but-legible beats invisible: these are the only
indication that a parameter is automated.

---

## Presets — save/load as JSON

Envelopes alone do not round-trip a scene. A disarmed parameter reads from `P`,
so restoring curves without the underlying slider values gives you a file that
does not reproduce what you saved. The preset is therefore the whole editable
state: parameters, envelopes, toggles, camera keys, output settings.

The splat is referenced **by name only** (`asset`), never embedded. Captures are
hundreds of MB and often not redistributable; embedding one would destroy the
point of a small shareable text file. Loading a preset does not load a capture.

### Shape

```json
{
  "format": "splat-bench-preset",
  "version": 3,
  "saved": "…", "asset": "butterfly.spz",
  "params":    { "cullA": 0.3, … },        // all 18, manual values (P, not V)
  "toggles":   { "revFlip": true, … },
  "output":    { "res": "1920x1080", "fps": "30" },
  "camera":    { "A": {"pos":[…],"tgt":[…]}, "B": … },
  "envelopes": { "revY": { "on": true, "smooth": true,
                           "keys": [{"t":0,"v":0},{"t":1,"v":1}] } },
  "selects":   { "rgMode": "crop", "rgShape": "box", … },      // v2
  "audio":     { "name": "track.wav",                          // v3
                 "routes": { "dAmt": {"band":"low","depth":0.6} } }
}
```

`params` stores `P`, the manual values — never `V`. Saving the evaluated values
would bake whatever the playhead happened to be sitting on into the file.
Envelopes with no keys are not persisted.

### Loading is deliberately forgiving

A preset is a plain text file someone may well hand-edit, so nothing in it is
trusted:

- unknown parameter and envelope ids are skipped silently
- every number is clamped to its own control's `min`/`max`; `t` is clamped 0..1
- non-finite values are dropped, keys are re-sorted by `t`
- an envelope with `on: true` but no keys is left disarmed
- `version` newer than the build is refused with a message rather than
  half-applied
- envelopes are **wiped before applying**, so a load is a replace and never a
  merge with what was already there

Verified against malformed JSON, wrong format, missing version, future version,
out-of-range params, unknown ids, non-animatable ids and empty key lists. None
kill the module; each reports through the status line.

Bump `PRESET_VERSION` on any breaking shape change and handle the old shape in
`applyPreset`, rather than letting the version check reject files it could have
migrated.

### Round-trip

State → save → wipe → load reproduces the fingerprint byte-exactly (all 18
readouts, toggles, armed envelope count, camera key flags, output settings).
Dropping a `.json` on the viewport routes to the preset loader; anything else
still goes to the splat loader.

---

## Spark API points — verified vs assumed

Verified against the Spark 2.x docs:

- `new SplatMesh({ url | fileBytes, objectModifiers, lod, onProgress, onLoad })`
- `await mesh.initialized`, `mesh.getBoundingBox(centers_only)`
- `dyno.dynoBlock(inTypes, outTypes, closure)`
- `dyno.splitGsplat(g).outputs` — the migration guide uses `.outputs`; the
  overview page shows direct destructuring. `.outputs` is the one to trust.
- `dyno.combineGsplat({ gsplat, center, scales, rgba })` injects components
- `Gsplat` struct fields: `center, flags, scales, index, quaternion, rgba`
- stdlib: `mix, smoothstep, select, and, greaterThanEqual, swizzle, extendVec,
  hashVec3, sin, mul, add, sub, max`

Confirmed at runtime on the butterfly (177,132 gaussians), 2026-08-16:

- `mesh.updateVersion()` — exists, bumps `this.version`, drives the regenerate
  gate. **Required after any uniform write.**
- `dyno.swizzle` / `hashVec3` / `smoothstep` / `select` / `and` /
  `greaterThanEqual` / `lessThanEqual` / `extendVec` / `combineGsplat` — all
  behave as documented; every effect measurably responds.
- `splatCount()` resolves — the `numSplats ?? packedSplats.numSplats` fallback
  reports the right count. Which branch fires is still untested.

**Still assumed, not exercised by current code:**

- `dyno.split(scales).outputs.x/.y/.z` — the code deliberately uses three
  `swizzle()` calls instead, so this remains unverified.

**Confirmed since, each by a real coverage-table reproduction, not a synthetic
test — see Colour (saturation) and Region for the measurements:**

- ~~`dyno.mix()` with a `vec3` a/b and a scalar `t`.~~ `dyno.dot(vec3, vec3)`.
- `dyno.equal`, `dyno.abs`, `dyno.sqrt`, `dyno.clamp`, `dyno.min` on floats —
  used throughout the Region SDFs; all six shapes reproduced the removed
  native system's coverage table to 3 decimal places (one bug found and fixed
  in capsule's segment length, not in these primitives).

Splat captures are usually Y-down relative to three.js. This used to be a
hardcoded `mesh.quaternion.set(1, 0, 0, 0)` after load; since the Model
transform phase it is `mdRotX = 180`, the default of a real control — see
"Model transform — Correct mode".

---

## Export path

WebCodecs `VideoEncoder` (`avc1.640033`) → `mp4-muxer` → Blob download. Renders
at a fixed output resolution independent of viewport size; restores viewport
state in a `finally`. Chrome/Edge only — Safari and Firefox will fail the
`typeof VideoEncoder` guard.

**The real hazard here is the sort, not the encoder.** Spark determines
back-to-front draw order by reading splat distances back off the GPU and bucket
sorting on a worker thread. That is asynchronous. A frame rendered immediately
after a camera move can be blended in stale order. The offline loop currently
renders, waits `settle` ms, renders again, then captures. 24 ms is a guess.

If exported frames show wrong-order blending that the live view does not, that
slider is the cause. A proper fix would hook whatever signal Spark exposes for
"sort for this viewpoint is current" instead of sleeping — worth investigating
in the Spark source (`SparkViewpoint`, `SplatWorker`).

Note the interaction with the version gate: `pushUniforms()` now calls
`updateVersion()`, so each export frame forces a regenerate *and therefore a
re-sort*. That makes the `settle` wait more load-bearing than it was, not less.
Before this fix the offline loop was also silently rendering unchanged splat
data every frame — any exported animation of an effect parameter would have come
out frozen except where the camera moved.

### Verified end-to-end, 2026-08-16

First real export since the `updateVersion()` fix. 30 frames, 1280×720, 30fps,
`dur=1`, reveal sweeping via `revAnim`, camera dollying A→B.

- Container exactly as configured: h264 High, 1280×720, 30/1, `nb_frames=30`,
  `duration=1.000000`, yuv420p.
- **Effects animate.** Mean luma per frame rises monotonically 24.0 → 38.6
  across all 30 frames, in an S-curve matching the camera ease. Pre-fix this
  would have been flat except where the camera moved.
- Requested 20 Mbps, got 5.1 Mbps actual. Not a defect — the encoder treats
  bitrate as a ceiling and this content is easy. Do not "fix" this.

Check animation with, rather than by eye:

```
ffprobe -v error -f lavfi -i "movie=out.mp4,signalstats" \
  -show_entries frame_tags=lavfi.signalstats.YAVG -of csv=p=0
```

`preserveDrawingBuffer: true` is set on the WebGLRenderer for reliable
`VideoFrame(canvas)` capture. It costs some performance; do not remove it
without re-testing export.

### Audio muxing

Exported video now includes the loaded audio track automatically — no
separate toggle. Whenever `audioBuffer` (the decoded `AudioBuffer` from
`analyseAudio()`, kept around purely for this — see Audio-reactive drive
above, which never needed the decoded PCM itself, only the derived band
envelopes) is present and `AudioEncoder` exists, `Muxer` gets an `audio:
{codec:"aac", numberOfChannels, sampleRate}` track alongside the video one,
and `encodeAudioTrack()` runs as a **separate phase after** the video frame
loop finishes (not interleaved frame-by-frame) — `VideoEncoder` and
`AudioEncoder` are independent state machines, and `mp4-muxer` only needs
each track's own samples in increasing timestamp order, not the two tracks
woven together, so interleaving would add complexity for no benefit.

**Trim behaviour matches the interactive preview's own established rule**
(`audioFollow()`'s comment, and the loop-boundary fix above): audio plays
from `t=0` for its own length or until the clip ends, whichever is
shorter — no looping, since export is a single linear pass and the
timeline-loop case that motivated `audioFollow()` at the loop boundary does
not apply to a one-shot render. If the track is longer than Duration, audio
is trimmed to match; if shorter, the video simply has a silent tail after
the audio track ends, which is a valid, ordinary MP4 (no need to synthesize
padding silence).

Encoded as `f32-planar` `AudioData` in 4096-frame chunks against the
`AudioBuffer`'s own native sample rate/channel count (no resampling), AAC-LC
(`mp4a.40.2`) at 128 kbps. `AudioEncoder` support is checked separately from
the `VideoEncoder` gate at the top of the render handler and its absence
(Safari, Firefox) falls back to a video-only export with a status message,
rather than blocking Render the way a missing `VideoEncoder` does — video
export has always worked without audio and should keep doing so.

**Verified end-to-end** against the loaded `butterfly.spz` and the
`bandtest.wav` fixture (6.0s), via direct MP4 box-structure parsing (a
minimal in-page scanner reading `moov/trak/mdia/hdlr` handler types and
`mdhd` timescale/duration, rather than trusting the file merely downloaded):

- **Structure**: exported file has exactly two tracks, handler types `vide`
  and `soun`; `stsd` shows `avc1` (video) and `mp4a` (audio) — both encoders
  actually ran, not just the video one.
- **Trim, audio shorter than Duration** (`dur=4` against the 6s fixture):
  video track `duration=4.000s` exactly (`timescale=24, duration=96`);
  audio track `duration=4.087s` — capped near 4s as designed, the extra
  ~87ms being ordinary AAC frame-boundary padding (1024-sample frames don't
  divide evenly into an arbitrary sample count), not a bug.
- **Trim, audio longer than Duration is impossible here since the fixture
  is 6s** — tested the reverse instead: `dur=10` against the same 6s
  fixture. Video track `10.000s` exactly; audio track `6.084s` (same
  frame-boundary padding) — a legitimate silent tail for the remaining
  ~4s, no error, no truncation of the video.
- **No audio loaded**: export unaffected — single `vide` track only, status
  text omits "+ audio", confirming the automatic-inclusion logic doesn't
  regress the existing video-only path.
- **Content, not just structure**: decoded the exported file's audio track
  back via the browser's own `AudioContext.decodeAudioData()` — real,
  non-silent signal (RMS 0.087, consistent across the clip, which is
  *expected* rather than suspicious: `bandtest.wav`'s three bands are equal-
  amplitude sine tones at different frequencies, so RMS alone does not and
  should not show the band transitions — checking for frequency content
  was not needed to confirm audio is present and not silence/garbage).

---

## Roadmap, roughly in order

1. ~~Fix the TDZ bug, add the error surface, confirm the butterfly renders.~~
   **Done.**
2. ~~Confirm the effect stack — displace, reveal, trim, tint each in isolation
   on a real capture.~~ **Done**, measured rather than eyeballed: every slider
   and checkbox verified against pixel coverage/luminance/bbox on the butterfly.
   This is what surfaced the missing `updateVersion()`. `br` and `settle` are
   now exercised too — see the verified export run below.
3. ~~**Per-parameter envelopes.**~~ **Done** — see the Envelopes section above.
   Keyframe lists per parameter, a curve lane with draggable keys, auto-key on
   armed sliders, honoured by the export path, and persisted as JSON presets.
4. ~~**SDF-shaped edits** via Spark's `SplatEdit` / `SplatEditSdf` /
   `SplatEditSdfType`.~~ **Done** — see the Region section above. Cut, crop and
   colorize across six shapes, animatable and persisted like any other
   parameter.
5. Depth pass output for compositing.
6. ~~Audio-reactive parameter drive (Web Audio FFT → uniforms).~~ **Done** —
   see the Audio-reactive drive section above. Offline three-band analysis,
   signed depth per parameter, layered on top of envelopes, persisted as
   routing in presets.
7. ~~Mobile: `lod: true` plus reduced pixel ratio, gated behind a toggle.~~
   **Done** — see the Mobile section above. Bottom-sheet rail, touch targets,
   pixel-ratio cap, forced LOD, and an aspect-aware `reframe()` that fixed a
   framing bug affecting any narrow window.
8. Multiple `SplatMesh` instances in one scene with independent transforms.
   **Deferred to last on 2026-08-16** — not needed at this stage. Nothing is
   blocked on it, and the Region work already anticipates it: edits are scoped
   per mesh via `mesh.add(edit)`, so regions will not leak between meshes when
   this does land.

## Deployment — GitHub Pages

Served from `main` at the repo root. Three things make this work, and each was
checked rather than assumed:

- **`index.html`.** Pages serves no directory index otherwise; the root URL
  would 404.
- **`.nojekyll`.** Pages runs Jekyll by default, which silently drops paths
  beginning with `_`. Nothing currently starts with an underscore, but the
  failure mode is a missing file with no error, so the guard stays.
- **No cross-origin isolation needed.** Spark 2.1.0 does not use
  `SharedArrayBuffer` (checked: zero occurrences), which matters because Pages
  cannot set COOP/COEP headers. If a future Spark release adopts it, Pages
  stops being a viable host and that is the reason why.

The pinned `spark.module.js` serves `access-control-allow-origin: *`, so the
CDN importmap works from the Pages origin, not just localhost. The test asset
is no longer remote at all — see below — so Load test asset has no cross-origin
dependency left.

**The test asset is committed, deliberately against `.gitignore`'s own rule.**
`3_3_2025.ply` (~11 MB, 44,296 gaussians, a Polycam daffodil capture) is
un-ignored via a `!3_3_2025.ply` exception and served from the repo. Two
reasons, and the second is the real one: the repo stays runnable as cloned, and
every measurement from here on has a *fixed* subject. The previous asset was
`butterfly.spz` fetched from Spark's own CDN — a third party's file, on a URL
this project does not control, which could be re-encoded or removed without
notice and would silently invalidate every coverage and luma figure in this
file. 11 MB is well inside Pages' limits.

**Figures measured against `butterfly.spz` (177,132 splats) are not comparable
to figures measured against `3_3_2025.ply` (44,296 splats)** — different splat
count, different geometry, different framing. Everything in this file dated
before the rename is butterfly-based. This is a change of subject, not drift to
chase, exactly like the `blurAmount: 0.3 -> 0` note in Phase 5.

Pages is HTTPS, so the secure-context requirements (WebCodecs, `AudioContext`)
are satisfied. Export still needs Chrome or Edge.

There is no build step and nothing to configure: a push to `main` republishes.

## Non-goals

- Reimplementing SuperSplat's cleanup UI. Use SuperSplat for cleanup, load the
  cleaned file here.
- A build step or npm dependency tree. Single file, CDN imports, works from
  GitHub Pages.
- Runtime LLM generation of GLSL. Effects should be authored ahead of time as
  parameterised dyno graphs; anything model-driven should map language onto
  parameters and keyframes of graphs that already compile, not emit shader code
  the user cannot debug.
