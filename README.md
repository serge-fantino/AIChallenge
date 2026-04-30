# Slice Demo

A vibe-coding challenge: a single-page Canvas demo where random cut lines slice
geometric shapes into pieces, with elastic / jelly impact animations,
collision-aware separation, color fading, a glowing cut line, and three
optional camera modes including a recursive infinite-zoom mode.

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

## Settings

| Group       | Parameter         | Notes                                                  |
| ----------- | ----------------- | ------------------------------------------------------ |
| Setup       | Formes            | initial shape count (apply via *Recommencer*)          |
| Setup       | Mode caméra       | Fixe / Pan / Zoom infini                               |
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

## Run it

Open `index.html` in any modern browser. No build step, no server.

- Click the canvas, or press `space` / `r`, to reset.
- Click ⚙ (top right) to open the settings panel.
- *Recommencer* regenerates the scene; *Défauts* restores the default settings.

## Notes

The implementation grew incrementally through a vibe-coding session, each
feature added on top of the previous one. The code is intentionally a single
file — the goal was a self-contained demo, not a framework.
