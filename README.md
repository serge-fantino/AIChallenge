# Slice Demo

A vibe-coding challenge: a single-page Canvas demo where random cut lines slice
geometric shapes into pieces, with elastic / jelly impact animations,
collision-aware separation, color fading, a glowing cut line, three optional
camera modes including a recursive infinite-zoom mode, and a fully synthesized
Web Audio soundtrack (laser, shatter, wind).

Inspired from https://www.instagram.com/reel/DXw9aQ3p6Tl/?igsh=MW44MXN1cXRyY28zdQ==

Demo here: https://serge-fantino.github.io/AIChallenge/

## The challenge

> On a blank page, show a few simple geometric shapes (circles, triangles,
> parallelograms). Then animate a straight colored line that crosses the screen
> from edge to edge in a random direction. If it intersects a shape, split that
> shape into two pieces along the line and physically separate them by a
> random distance perpendicular to the cut, while keeping each piece's
> geometric coherence. The line then disappears, leaving a more complex set of
> shapes. Repeat the process — each new line can slice more pieces. Continue
> until everything is reduced to small fragments.

## What's implemented

A single self-contained `index.html` (vanilla JS + Canvas, no dependencies)
that builds on the original challenge with several layered features:

- **Initial scene**: a configurable number of random shapes (default 500,
  slider 1-1000) — circles, triangles, parallelograms,
  pentagons, hexagons, heptagons, octagons — slightly jittered per vertex so
  they don't look perfectly regular) placed by a **phyllotaxis spiral**
  starting at the canvas centre and growing outward at the golden angle —
  hex-packing density, organic look, deterministic centre-out fill. Once the
  spiral has placed everything (or hit the edge of the central band), the
  same SAT relaxation used during cuts runs in-place to push apart any
  pairs whose vertex jitter caused a residual overlap. Shape size scales
  with the **full** canvas area and shape count (not the central band), so
  changing the centred margin doesn't shrink the pieces — it just leaves
  fewer slots, and the spiral places fewer shapes if it can't fit them all.
- **Convex polygon clipping**: each cut line splits any intersecting convex
  polygon into two convex halves by walking edges, classifying vertices by
  signed distance to the line, and inserting interpolated intersection points.
- **Four cut animation modes**:
  - *Rigide* — original behavior: rigid translate + counter-rotating spin with
    `ease-out-elastic`.
  - *Jelly* — per-point delay based on distance to the cut line. Points on the
    cut edge fly out first, far points lag behind, the shape stretches and
    snaps back into shape.
  - *Vague* — per-point delay based on signed projection along the cut tangent;
    one end of the cut leaves first, the other follows like a wave.
  - *Squash* — non-uniform scale pulse that elongates the piece along the
    motion direction and compresses it perpendicular to it.
- **Collision-aware separation (SAT)**: after a cut, all pieces (the new
  halves *and* any other shape they intrude into) are pushed apart by an
  iterative relaxation loop using the Separating Axis Theorem. The
  separation tolerance is in screen pixels (the world-unit threshold scales
  with `camera.scale`), so the same "essentially touching" threshold applies
  at any zoom. A final cleanup sweep then drops the smaller of any residual
  overlapping pair the relaxation couldn't fully resolve at very high N.
  Combined with the bg-filled / newest-under rendering pass below, any
  remaining sub-pixel residual is masked visually.
- **Marker / rivet rendering**: shapes are drawn as outlines (no fill) in a
  felt-tip style with small grey "rivets" along the perimeter. Stroke style is
  switchable: rivets, solid, dashed, dotted.
- **Cut line never misses**: a cut line is always routed through the centroid
  of an existing cuttable shape, then extended to the edges of the visible
  region. The target is biased toward shapes whose centroid lies inside the
  central 20% of the visible viewport, so the line consistently passes near the
  center of the screen.
- **Glowing cut line**: the line is wrapped in a softer, lighter, semi-
  transparent halo (two layers for a smooth falloff) that fades faster than
  the line itself for a quick "spark of impact" feel. Glow size, decay and
  intensity are configurable.
- **Single / multi-piece colorization**: across the whole cut path, one or
  more (configurable) of the new pieces inherit the cut line's color. In
  Pan and Zoom-infini modes, the first colored piece is forced to be the
  visible new piece **closest to the viewport centre** so the colorization
  always lands where the user is looking; the remaining picks stay
  uniformly random across all new pieces.
- **Color fade-back**: colored pieces gradually return to neutral grey over a
  random window. When a colored piece is itself recut, only **one** of the
  two halves (the first returned by the polygon split) keeps the parent's
  colour and continues the same fade; the other resets to neutral and only
  becomes coloured if picked by the colorization step on this same cut.
- **Initial-shape palette**: each starting shape can be assigned a base fill
  colour drawn from a built-in palette — five soft monochrome families
  (pink / blue / green / warm / purple) plus a saturated rainbow. The
  fill colour persists through every cut: both halves of any split inherit
  the parent's palette colour, so the initial palette assignment carries
  all the way through every generation. Selectable from the Settings
  panel; defaults to *Aucune* (neutral bg fill, original look).
- **Three camera modes**:
  - *Fixe* — no camera animation, the whole canvas stays in view.
  - *Pan* — for each cycle, pan + zoom into a cluster anchored to one of the
    initial shapes, do N cuts inside that zone, pan back out, pick another.
    Framing follows the current bounds of the zone's descendants.
  - *Zoom infini* — recursive drilling: do N cuts in the current view, then
    zoom further into a specific piece and repeat. The drill target is the
    largest cuttable visible piece such that, framed at **50% screen
    occupancy** (bounding diameter = half the smaller viewport dimension)
    with the piece exactly centred, the new viewport stays fully **inside**
    the current one — every step zooms only into already-visible territory.
    Stops when no piece satisfies the constraint or the cumulative scale
    gets silly, then resets to the overview.
- **Zoom-aware rendering & physics**: at any zoom level, stroke width, rivet
  spacing/radius, dash patterns, and the cut separation distance are scaled by
  the inverse of `camera.scale` so they all keep a constant on-screen size.
  The cuttable threshold (`MIN_AREA`) is interpreted in screen-space too,
  letting the recursive zoom continue cutting tiny world-coord pieces that
  still look big on screen.
- **Translucent settings panel**: a slide-in panel (⚙ button, top right)
  exposes all animation parameters with live updates and a "Recommencer" /
  "Défauts" pair of buttons.
- **Persisted settings**: every change is auto-saved to `localStorage` (key
  `sliceDemo.settings.v1`), so reloads keep your tweaks. The *Défauts* button
  restores the built-in defaults.
- **DEMO / PLAY toggle** (top-left button, also a checkbox in the panel):
  - *DEMO* — the autonomous cycle described above runs on its own.
  - *PLAY* — every cut is click-driven. The first click acknowledges with
    a cut in the current view, then the camera pans in (in Pan / Zoom
    modes) and the user clicks again for each subsequent cut. After
    `Coupes/zone` clicks, the camera advances (pan-out in Pan mode, zoom
    deeper into the largest visible piece in Zoom mode), and the next
    click in the new view continues the cycle.
- **Speed multiplier** (top-left button next to DEMO / PLAY, cycles 0.5× /
  1× / 1.5× / 2× / 3×): scales the simulation `dt`. Every time-based value
  (line draw, animation duration, pan duration, glow decay, fade, gesture
  inactivity, …) speeds up proportionally; HUD frame-time stays honest
  (recorded with real `dt`).
- **Camera mode toggle** (top-left button next to the speed cycler, cycles
  *FIXE* / *PAN* / *ZOOM*): mirrors the *Mode caméra* select in the panel
  for one-click switching without opening settings. State is shared with
  the panel — changing one keeps the other in sync.
- **Vignette**: a CSS overlay (`backdrop-filter: blur` masked by a radial
  gradient) that softens the edges of the canvas without affecting the
  rendered pixels — gives a focus-on-center look while letting pieces
  extending past the central band fade out gracefully. Configurable size,
  0 = off. Not included in PNG exports.
- **Centred placement & spacing controls**: `Marge centrale` (% of each
  axis on each side — central rect where shape centres can go) and
  `Espacement` (px of guaranteed clearance between bounding circles of
  any two adjacent shapes).
- **Randomised cuts per zone**: the camera cycle uses `Coupes min` /
  `Coupes max` (default 3 / 6) — a uniform random pick per cluster — so
  each zone gets a different rhythm before the camera advances.
- **PNG export** (camera icon, top right): pauses the simulation and shows a
  full-screen overlay. Drag a region to crop, or use **Tout cadrer** to
  auto-fit every shape into the frame. The export re-renders into an
  offscreen canvas at `exportScale×` the live resolution (default 3×, so
  ~6K from a 1920 viewport) — strokes scale up too, no upscaling artefacts,
  no vignette baked in.
- **Free-look gestures** (work in both DEMO and PLAY):
  - Two-finger drag on touch devices = pan; pinch = zoom.
  - Trackpad two-finger scroll = pan; pinch (or `ctrl`/`cmd`+wheel) = zoom
    around the cursor.
  - In DEMO mode, gestures suspend the auto cycle for ~3 s of inactivity
    so the camera doesn't immediately yank itself back to a cluster.
- **Synthesized audio (Web Audio API, no external files)**:
  - *Laser* on every cut line: sawtooth oscillator sweeping 1.8 kHz → 180 Hz
    with an animated low-pass filter, ~180 ms.
  - *Shatter* when the cut applies: bandpass-filtered noise impact, a
    high-pass noise tail, and a handful of high-pitched triangle "tings" at
    random frequencies — like broken dishware.
  - *Wind* during camera pan / zoom: looped white noise filtered by a
    low-pass + bandpass chain, with smooth fade-in / fade-out. Auto-starts
    while the camera is animating, stops when it settles.
  - Audio context is lazily created on the first user interaction (browser
    gesture requirement).

## Settings

| Group       | Parameter         | Notes                                                  |
| ----------- | ----------------- | ------------------------------------------------------ |
| Setup       | Formes            | initial shape count (apply via *Recommencer*)          |
| Setup       | Marge centrale    | % of each axis as centred-band inset for placement     |
| Setup       | Espacement        | min clearance between adjacent shapes' bounding circles|
| Setup       | Mode caméra       | Fixe / Pan / Zoom infini                               |
| Setup       | Interactif        | toggle DEMO ↔ PLAY (top-left button mirrors this)      |
| Caméra      | Padding           | extra margin around the framed cluster (screen px)     |
| Caméra      | Zoom max          | zoom cap (used in Pan mode framing)                    |
| Caméra      | Vitesse pan       | pan-in / pan-out duration                              |
| Caméra      | Coupes min / max  | random cut count per cluster, uniform in the range     |
| Caméra      | Pause vue         | hold time at the overview between cycles               |
| Coupe       | Vitesse           | piece animation total duration                         |
| Coupe       | Mode              | Rigide / Jelly / Vague / Squash                        |
| Coupe       | Élasticité        | elastic decay (lower = bouncier)                       |
| Coupe       | Spread jelly      | per-point delay spread for Jelly / Vague               |
| Coupe       | Écartement        | max separation distance per cut (screen px)            |
| Coupe       | Pièces colorées   | how many new pieces inherit the cut color              |
| Ligne       | Vitesse           | line draw + fade duration                              |
| Ligne       | Maintien          | how long the line stays at full opacity after the cut  |
| Ligne       | Pause             | pause between cuts                                     |
| Ligne       | Glow              | halo width                                             |
| Ligne       | Glow decay        | how fast the halo fades after the line is drawn        |
| Ligne       | Glow intensité    | multiplier on halo opacity                             |
| Couleur     | Fade min / max    | range for the fade-to-grey duration                    |
| Apparence   | Fond              | background color                                       |
| Apparence   | Trait             | base stroke color (= fade target)                      |
| Apparence   | Épaisseur         | stroke width (in screen px, constant under zoom)       |
| Apparence   | Trait proportionnel | stroke thickness scales with each piece's on-screen radius |
| Apparence   | Réf. trait        | reference radius for the proportional stroke           |
| Apparence   | Trait max         | absolute cap on stroke thickness (proportional mode)   |
| Apparence   | Vignette          | radial blur halo width at the edges (0 = off)          |
| Apparence   | Style             | rivets / solid / dashed / dotted                       |
| Apparence   | Palette           | initial-shape fill: Aucune / mono pastel ×5 / Arc-en-ciel |
| Son         | Activé            | master mute toggle                                     |
| Son         | Volume            | master volume (0–100%)                                 |

## Performance

Cuts run in `O(N²)` for the SAT collision relaxation and rendering touches
every visible polygon vertex every frame, so a few targeted optimizations
keep things smooth as the piece count grows. A small HUD in the bottom
right shows the current count and frame-rate so you can watch the
degradation in real time.

- **SAT broad-phase via spatial grid**: a hash grid keyed by `(cellX, cellY)`
  buckets the AABBs at each relaxation iteration; only intra-bucket pairs
  are tested. Turns the `O(N²)` sweep into roughly `O(N × k)` where `k` is
  the average bucket size. AABB scratch arrays (`Float64Array`s) and the
  grid `Map` are module-level and reused across cuts, so no per-call
  allocations. With ~5000 pieces this drops from millions of pair tests
  per iteration to a few thousand.
- **Cached per-shape SAT axes**: edge normals are translation-invariant, so
  they're computed once per shape at the start of each relaxation and
  reused across all iterations (instead of `getAxes()` being called twice
  inside every `mtv()` call).
- **Allocation-free MTV**: `projectInto(pts, axx, axy, out)` writes the
  min/max projection into a shared scratch object instead of returning a
  fresh `[mn, mx]` array per axis — avoids hundreds of millions of tiny
  array allocations per cut at high N.
- **Cached centroid / bounding radius** per shape (`s._centroid`,
  `s._boundingRadius`), lazily computed and reused by all the camera /
  zone / framing helpers. Cleared/updated in place when the animation bakes
  (translation only — bounding radius is invariant).
- **Viewport culling + tiny-shape culling**: each frame, shapes whose
  bounding circle lies entirely outside `visibleRect()` *or* whose
  on-screen radius is smaller than ~1.5 px are skipped at render — the
  former wins big in zoom mode, the latter wins big at scenes with tens
  of thousands of fragments.
- **`Path2D` cache for settled shapes**: non-animating shapes build their
  polygon once into a `Path2D` and stroke it via `ctx.stroke(s._path2D)`
  thereafter — no `beginPath` / 28× `lineTo` / `closePath` per frame.
  The cache is invalidated when the animation bakes (the only time the
  shape's points change). At ~30k shapes this is the single biggest
  per-frame win.
- **Allocation-free per-point animation**: the per-mode functions
  (`rigidPointInto`, `jellyInto`, `waveInto`, `squashInto`) write into a
  caller-provided `out` object instead of returning a new `{x, y}`. The
  drawing pipeline reuses a single shared buffer (`visBuf`), so animating
  N shapes × P points per frame allocates nothing.
- **Batched rivets**: the rivets along a polygon's perimeter are now drawn
  in a single `beginPath` / `fill` per shape, with `moveTo` between each
  arc to keep them disjoint. Cuts ~10× the number of fill draw calls.
- **Idle throttling**: when `scheduleNextCut()` finishes without launching
  anything (no cuttable shape, no camera move queued), it backs off for
  500 ms instead of being polled at 60 Hz. Saves Mops/sec when the demo
  has finally exhausted everything cuttable, leaving CPU for the
  in-flight animations to settle gracefully.
- **Two-phase cut for perceived smoothness**: the heavy work (`prepareCut`
  — convex splits, SAT relaxation, cleanup, building the next shape array)
  runs **eagerly** when a cut starts, before the line begins drawing. The
  result sits in `cutState.prepared` until the line completes its draw
  animation, at which point `commitPreparedCut` swaps the new shapes in
  (cheap: array assignment + shatter sound). The total wall time is
  unchanged, but the freeze is hidden under click latency rather than
  interrupting an animation in flight — much smoother at 100k+ pieces.
- **SAT cleanup pass + scale-aware tolerance**: the relaxation runs up to
  60 iterations, then a final broad-phase sweep catches any residual
  overlapping pair and drops the **smaller** (by area) of the two — rather
  than pushing further and risking a chain reaction in dense neighbourhoods.
  The "treat as separated" tolerance (`_mtvEps`) is set to `0.5 / camera.scale`
  at each cut so it stays sub-pixel on screen at any zoom (previously a
  fixed 0.5 world-unit value translated to dozens of screen pixels of
  tolerated overlap in zoom-infini mode).
- **Bg-filled / newest-under rendering**: each piece is filled with the
  background colour before being stroked, and freshly-cut pieces are
  inserted at the **front** of the shape array so older survivors render
  on top. Combined, this masks any sub-pixel SAT residual that the
  relaxation/cleanup couldn't fully resolve — the older, larger piece's
  fill covers the newer, smaller piece underneath. ~50% extra rendering
  cost on visible pieces, kept in check by the viewport culling.
- **Lower `MIN_AREA` + larger `ZONE_TOLERANCE`**: the cuttable threshold
  is 50 px² (≈ radius 4) and zone-membership tolerance is 250 px, so
  thousands of cumulative cuts can keep happening past the previous
  ~10k-piece exhaustion limit (now reaches 100k+ pieces with the
  prepare/commit split absorbing the per-cut spike, and the rendering
  bottleneck only kicking in well past that on a typical laptop).
- **Performance HUD** (bottom right): pieces count, smoothed FPS + frame
  time over the last 60 frames, and a "Charge" ratio of current FPS vs the
  best FPS observed in the session. Goes orange below 80% of peak and red
  below 50%.

## Run it

Open `index.html` in any modern browser. No build step, no server.

- Click the canvas, or press `space` / `r`, to reset (DEMO mode).
- In PLAY mode, each click on the canvas triggers a cut at that point.
- Two-finger drag (touch) or trackpad scroll = pan; pinch / `ctrl`+wheel = zoom.
- Top-left: **DEMO / PLAY** toggle, the **speed** cycler (0.5×–3×), and the
  **camera mode** cycler (*FIXE* / *PAN* / *ZOOM*).
- Top-right: the camera icon exports a PNG (drag a region or *Tout cadrer*)
  and ⚙ opens the settings.
- *Recommencer* regenerates the scene; *Défauts* restores the default settings.

## Notes

The implementation grew incrementally through a vibe-coding session, each
feature added on top of the previous one. The code is intentionally a single
file — the goal was a self-contained demo, not a framework.
