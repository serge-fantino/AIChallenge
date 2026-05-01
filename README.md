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

- **Initial scene**: ~10 non-overlapping random shapes (circles approximated as
  28-segment polygons, triangles, parallelograms) placed via rejection sampling
  with bounding-circle separation.
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
  iterative relaxation loop using the Separating Axis Theorem. The result is
  guaranteed to have no overlapping polygons — neighbors absorb the impact and
  shift to make room.
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
  more (configurable) of the new pieces inherit the cut line's color.
- **Color fade-back**: colored pieces gradually return to neutral grey over a
  random window. When a colored piece is itself recut, both halves inherit and
  continue the same fade; the newly chosen piece on the next cut starts a
  fresh one.
- **Three camera modes**:
  - *Fixe* — no camera animation, the whole canvas stays in view.
  - *Pan* — for each cycle, pan + zoom into a cluster anchored to one of the
    initial shapes, do N cuts inside that zone, pan back out, pick another.
    Framing follows the current bounds of the zone's descendants.
  - *Zoom infini* — recursive drilling: do N cuts in the current view, then
    zoom further into the largest visible piece and repeat. Stops when the
    visible area runs out of cuttable pieces or the cumulative scale gets out
    of hand, then resets to the overview.
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
| Setup       | Mode caméra       | Fixe / Pan / Zoom infini                               |
| Setup       | Interactif        | toggle DEMO ↔ PLAY (top-left button mirrors this)      |
| Caméra      | Padding           | extra margin around the framed cluster (screen px)     |
| Caméra      | Zoom max          | zoom cap (used in Pan mode framing)                    |
| Caméra      | Vitesse pan       | pan-in / pan-out duration                              |
| Caméra      | Coupes/zone       | number of successive cuts per zoom level               |
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
| Apparence   | Style             | rivets / solid / dashed / dotted                       |
| Son         | Activé            | master mute toggle                                     |
| Son         | Volume            | master volume (0–100%)                                 |

## Performance

Cuts run in `O(N²)` for the SAT collision relaxation and rendering touches
every visible polygon vertex every frame, so a few targeted optimizations
keep things smooth as the piece count grows. A small HUD in the bottom
right shows the current count and frame-rate so you can watch the
degradation in real time.

- **SAT broad-phase**: each item gets an axis-aligned bounding box (`Float64Array`s,
  no objects) before the relaxation loop; pairs whose AABBs don't overlap
  skip the expensive `mtv()` call. Bounding boxes are updated incrementally
  after each push (uniform translation = same shift on the box). With ~100
  pieces this typically replaces ~5000 SAT tests per iteration with 50–200,
  giving ~20–50× speedup on the cut step.
- **Cached centroid / bounding radius** per shape (`s._centroid`,
  `s._boundingRadius`), lazily computed and reused by all the camera /
  zone / framing helpers. Cleared/updated in place when the animation bakes
  (translation only — bounding radius is invariant).
- **Viewport culling**: each frame, shapes whose bounding circle (centroid
  + radius + the magnitude of any in-flight animation translation) lies
  entirely outside `visibleRect()` are skipped at render. Big win in zoom
  mode, where most pieces are off-screen.
- **Allocation-free per-point animation**: the per-mode functions
  (`rigidPointInto`, `jellyInto`, `waveInto`, `squashInto`) write into a
  caller-provided `out` object instead of returning a new `{x, y}`. The
  drawing pipeline reuses a single shared buffer (`visBuf`), so animating
  N shapes × P points per frame allocates nothing.
- **Batched rivets**: the rivets along a polygon's perimeter are now drawn
  in a single `beginPath` / `fill` per shape, with `moveTo` between each
  arc to keep them disjoint. Cuts ~10× the number of fill draw calls.
- **Performance HUD** (bottom right): pieces count, smoothed FPS + frame
  time over the last 60 frames, and a "Charge" ratio of current FPS vs the
  best FPS observed in the session. Goes orange below 80% of peak and red
  below 50%.

## Run it

Open `index.html` in any modern browser. No build step, no server.

- Click the canvas, or press `space` / `r`, to reset (DEMO mode).
- In PLAY mode, each click on the canvas triggers a cut at that point.
- Two-finger drag (touch) or trackpad scroll = pan; pinch / `ctrl`+wheel = zoom.
- Click the top-left **DEMO / PLAY** button to switch modes.
- Click ⚙ (top right) to open the settings panel.
- *Recommencer* regenerates the scene; *Défauts* restores the default settings.

## Notes

The implementation grew incrementally through a vibe-coding session, each
feature added on top of the previous one. The code is intentionally a single
file — the goal was a self-contained demo, not a framework.
