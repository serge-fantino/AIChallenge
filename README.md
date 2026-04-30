# Slice Demo

A vibe-coding challenge: a single-page Canvas demo where random cut lines slice
geometric shapes into pieces, with elastic impact animations, collision-aware
separation, color fading, and an optional cinematic camera that zooms onto
clusters of shapes.

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
- **Elastic impact animation**: the two halves fly apart along the cut normal
  with an `ease-out-elastic` overshoot, plus a small counter-rotating spin —
  like real impact debris. The transform is applied at draw time and "baked"
  into the points only when the animation settles, so subsequent cuts always
  operate on coherent geometry.
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
  region. No more "wasted" cuts.
- **Single-piece colorization**: across the whole cut path, exactly one (or a
  configurable number) of the new pieces inherits the cut line's color.
- **Color fade-back**: colored pieces gradually return to neutral grey over a
  random 10–20 s window. When a colored piece is itself recut, both halves
  inherit and continue the same fade; the newly chosen piece on the next cut
  starts a fresh one.
- **Cinematic camera mode** (toggle in settings): instead of showing the whole
  page, the camera pans + zooms onto a "zone" anchored to one of the original
  initial shapes, performs N successive cuts within that zone, then pans back
  out to the overview before picking a new zone. The framing is computed from
  the actual current bounds of the zone's descendants (which stay nearby
  thanks to SAT relaxation), with `ease-in-out` cubic transitions.
- **Translucent settings panel**: a slide-in panel (⚙ button, top right)
  exposes all animation parameters with live updates.

## Settings

| Group       | Parameter         | Notes                                       |
| ----------- | ----------------- | ------------------------------------------- |
| Setup       | Formes            | initial shape count (apply via *Recommencer*) |
| Setup       | Mode caméra       | toggles cinematic mode + camera section     |
| Caméra      | Padding           | extra margin around the framed cluster      |
| Caméra      | Zoom max          | zoom cap                                    |
| Caméra      | Vitesse pan       | pan-in / pan-out duration                   |
| Caméra      | Coupes/zone       | number of successive cuts per cluster       |
| Caméra      | Pause vue         | hold time at the overview between zones     |
| Coupe       | Vitesse ligne     | line draw + fade duration                   |
| Coupe       | Vitesse découpe   | piece animation total duration              |
| Coupe       | Élasticité        | elastic decay (lower = bouncier)            |
| Coupe       | Écartement        | max separation distance per cut             |
| Coupe       | Pièces colorées   | how many pieces inherit the cut color       |
| Couleur     | Fade min / max    | range for the fade-to-grey duration         |
| Apparence   | Fond              | background color                            |
| Apparence   | Trait             | base stroke color (= fade target)           |
| Apparence   | Épaisseur         | stroke width                                |
| Apparence   | Style             | rivets / solid / dashed / dotted            |

## Run it

Open `index.html` in any modern browser. No build step, no server.

- Click the canvas, or press `space` / `r`, to reset.
- Click ⚙ (top right) to open the settings panel.

## Notes

The implementation grew incrementally through a vibe-coding session, each
feature added on top of the previous one. The code is intentionally a single
file — the goal was a self-contained demo, not a framework.
