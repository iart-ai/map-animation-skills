---
name: map-animation
description: This skill should be used when the user asks to "make a Vox-style map animation", "animate a map zoom", "draw a route on a map", "highlight a country/region", "use Google Earth Studio", "drop pins on a map and animate them", "pan/orbit a camera over a map", or "add map graphics to an explainer video". Covers Earth Studio→After Effects camera import, GeoJSON/SVG vector map motion, pins/routes/highlights/labels, the 12fps stutter cadence, and data-driven Remotion renders.
version: 0.1.0
---

# Map Animation

Build editorial, "Vox-style" animated maps: a camera that zooms, orbits, and pans over a base map while location pins drop, routes draw on, regions highlight, and labels call out — all snapped to a deliberate 12fps stutter so motion reads as graphic, not photographic. Two production paths: photoreal base motion from **Google Earth Studio** composited in After Effects, and code-renderable **vector maps** (GeoJSON/SVG) driven from a coordinates array in Remotion or D3.

## When to use

- A documentary/explainer beat that establishes place: zoom from globe to city, orbit a landmark, pan along a border.
- Show movement across geography: a route drawing between two points, a supply chain, a migration, a flight path.
- Highlight a country, state, or region and label it; build a step-by-step geographic walkthrough.
- Turn a list of coordinates/events into a repeatable, data-driven map sequence for video.

## Pick the path

| You need | Path | Output |
|---|---|---|
| Photoreal terrain, real satellite/3D buildings, cinematic camera | **Google Earth Studio → After Effects** | Image sequence + camera `.jsx`, composited in AE |
| Clean editorial vector look, full control, data-driven | **Vector (GeoJSON/SVG)** in Remotion or D3 | Code-rendered MP4/GIF or web SVG |
| Pins / routes / highlights / labels on top of either | **Overlay layer** | AE shape/null layers OR React/SVG components |

Most "Vox-style" pieces combine them: Earth Studio for the base camera move, vector overlays for the pins, routes, and labels on top.

## The signature look: 12fps stutter

The single most recognizable trait of this style is **animating the map graphics at 12fps inside a 24fps edit**. The base footage and audio run at 24 (smooth); the graphic layer — camera-driven pins, route draw-on, label pops — steps at 12. The result reads as intentional, hand-built, "infographic" motion rather than soft video.

- **Math:** at 24fps, hold each graphic-layer value for 2 frames (update on even frames only). At 30fps, hold ~2–3 frames (≈12.5/10fps).
- **After Effects:** apply `posterizeTime(12)` to the layer/comp, OR nest the graphics in a 12fps precomp placed inside the 24fps master.
- **Remotion / code:** snap the frame before computing any animated value:

```js
// Quantize the timeline to 12fps "steps" while the comp renders at 24fps.
const STEP = Math.round(fps / 12);              // 2 at 24fps
const f = Math.floor(frame / STEP) * STEP;       // use `f`, not `frame`, for graphic motion
const progress = interpolate(f, [0, 48], [0, 1], { extrapolateRight: "clamp" });
```

Keep the *base* footage/camera smooth (24fps) and stutter only the *graphic overlay* — stuttering everything looks like dropped frames, not style.

## Path A — Google Earth Studio → After Effects

Earth Studio is a browser tool that keyframes a camera over Google's 3D globe and exports two things you import into AE: an **image sequence** (the rendered frames) and a **camera-track `.jsx`** script that rebuilds the exact 3D camera in After Effects.

1. **Keyframe in Earth Studio** (browser): set keyframes for camera position (lat/long/altitude), tilt, heading, and field of view. Use the auto-ease or set custom ease per keyframe. Keep moves slow and deliberate — establish, don't whip.
2. **Export → render**: choose the image sequence (JPG/PNG), set resolution and fps, and **enable the "3D Camera Export" / After Effects checkbox** so it ships the `.jsx` and track-point files alongside the frames.
3. **In After Effects**: import the image sequence as footage, then **File → Scripts → Run Script File…** and run the exported `.jsx`. It creates a comp with a 3D camera and null layers whose keyframes match the Earth Studio move frame-for-frame.
4. **Parent overlays to the track**: parent your pin/label null layers to the imported 3D camera/track nulls so they stick to the right map location as the camera moves (the `.jsx` includes track points for any coordinates you marked in Earth Studio).
5. **Apply the stutter** to the overlay precomp only (`posterizeTime(12)`), color-grade the base footage, then render.

See `references/earth-studio-to-ae.md` for the exact keyframe/export settings, the `.jsx` import step, and how track points map screen-space to lat/long.

## Path B — Vector maps (GeoJSON / SVG)

For the clean editorial look, project GeoJSON to SVG paths and animate them. Use a real projection so coordinates land correctly.

```js
import { geoMercator, geoPath } from "d3-geo";
const projection = geoMercator().fitSize([1920, 1080], geojson);  // fit map to frame
const toPath = geoPath(projection);
const [x, y] = projection([lng, lat]);          // any lat/long → screen pixel
```

Render `geojson.features` as `<path d={toPath(f)} />`. The same `projection([lng,lat])` converts every pin coordinate to a screen position, so pins, routes, and labels all share one coordinate space as the camera (a `scale`/`translate` transform) moves.

**Camera move** in vector land is a CSS/transform zoom-and-pan toward a target lat/long:

```js
const [tx, ty] = projection([targetLng, targetLat]);
const scale = interpolate(f, [0, 36], [1, 4], { extrapolateRight: "clamp" });
// transform-origin at the target so the zoom homes in on it
const transform = `translate(${960 - tx*scale}px, ${540 - ty*scale}px) scale(${scale})`;
```

## Overlays: pins, routes, highlights, labels

**Pin drop** (drops in, settles, optional pulse) — stagger across a coordinates array:

```jsx
const pins = [{ name:"Kyiv", lng:30.52, lat:50.45 }, { name:"Lviv", lng:24.03, lat:49.84 }];
pins.map((p, i) => {
  const local = f - i * STEP * 4;                 // stagger, quantized to the 12fps step
  const drop = spring({ frame: local, fps, config:{ damping: 12 } });   // overshoot = settle
  const [x, y] = projection([p.lng, p.lat]);
  return <Pin key={p.name} x={x} y={y - 40*(1-drop)} opacity={local > 0 ? 1 : 0} />;
});
```

**Route draw-on** — build the path with the projection, then stroke it on with `stroke-dashoffset`:

```js
// route = [[lng,lat], ...]; project each point, join into an SVG path
const d = route.map((c,i) => (i ? "L" : "M") + projection(c).join(" ")).join(" ");
// in the component: pathLength=1, strokeDashoffset = 1 - drawProgress
const draw = interpolate(f, [12, 60], [0, 1], { extrapolateLeft:"clamp", extrapolateRight:"clamp" });
```

`pathLength="1"` normalizes any route length so one progress value (0→1) draws it. Add a traveling dot at `offset-distance` to lead the line. For a great-circle/curved flight path, sample intermediate points with `d3.geoInterpolate(a, b)`.

**Region highlight** — fill/stroke the matched GeoJSON feature, dim the rest (focus + context):

```js
const isTarget = (f) => f.properties.name === "France";
// target: fill accent, opacity 1, stroke up; others: opacity .35
```

Animate the highlight in by interpolating fill opacity, or wipe it with a clip-path mask. Keep color = meaning and never reassign it mid-piece.

**Label callout** — pin label or leader-line box; fade/slide in after its pin lands, anchored to `projection([lng,lat])` so it tracks the camera. Keep labels horizontal and inside the safe area; never let them rotate with the map.

## Data-driven sequencing

Drive the whole sequence from one ordered array so the map narrates itself and re-renders from data:

```jsx
const beats = [
  { lng: 30.52, lat: 50.45, label: "Kyiv",  hold: 48 },
  { lng: 24.03, lat: 49.84, label: "Lviv",  hold: 36, route: true },
];
// Each beat = a Sequence: camera flies to (lng,lat), pin drops, label pops, route draws if set.
// Total duration = sum(holds); compute it in calculateMetadata, not by hand.
```

One array → camera targets, pin order, route segments, and labels stay in sync. Swap the data, get a new map sequence — no re-keyframing.

## Output checklist

- Camera moves are slow and deliberate (establish, don't whip); eases on every keyframe.
- Graphic overlay stutters at 12fps; base footage/camera stays smooth (24fps).
- Pins/routes/labels stay locked to their lat/long as the camera moves (parented to track / shared projection).
- Routes draw on in reading order; highlights use focus + context (target lit, rest dimmed).
- Labels horizontal, legible, inside the safe area; color grammar consistent.
- Data-driven path: one coordinates/beats array is the single source of truth.

## Deliver & verify (rendered stills → MP4)

> **Packaged helper** (`scripts/`): tile your stills with `scripts/contact-sheet.sh sheet.png f-hook.png f-mid.png f-end.png`, then assert the encode with `scripts/probe-mp4.sh out.mp4 [WxH] [fps]`. See `scripts/README.md`.

The code-renderable path is a Remotion composition — frame-deterministic, so any exact frame renders headlessly with **no seek harness**. Use this tier when the deliverable is an MP4/GIF (vector map, data-driven, exact labels) or batches per dataset; for an Earth Studio + AE comp, the deliverable is the AE project + rendered footage instead, verified by scrubbing the master comp.

**Output contract:**
- A Remotion project with the composition registered (`<Composition>` + zod `schema` + `defaultProps`), all motion frame-driven (no timers / `Date.now()` / `Math.random()`).
- Deliverable = the rendered `out/*.mp4` (plus the project + the `beats`/coordinates array, so the map can be re-rendered from new data).
- The 12fps stutter is computed from `fps` (quantize the frame), not hardcoded — so it survives an fps change.
- Duration data-dependent (sum of beat holds)? compute it in `calculateMetadata`, not by hand.

**Verify loop — render stills → inspect → encode.** Render single frames first (cheap, no encode), inspect them, encode only once the frames are right.

```bash
# Frame-exact stills at start / each beat / end — render with the SHIPPED props (the real beats array)
npx remotion still MapSeq out/f-start.png --frame=0   --props='{...}'
npx remotion still MapSeq out/f-beat2.png --frame=N   --props='{...}'   # mid a beat's hold
npx remotion still MapSeq out/f-end.png   --frame=L   --props='{...}'   # L = durationInFrames - 1

# Inspect each PNG — FIDELITY: every pin sits on the correct lat/long, the route connects the right
# cities, the highlighted region is the intended one, labels are spelled right and readable. And land
# on TWO adjacent frames inside a move to confirm the overlay holds for 2 frames (the 12fps step),
# while the base camera is still advancing every frame.
# ARTIFACTS: labels off-canvas/clipped, pins drifting off their coordinate, route overshooting,
# missing map tiles/fonts, projection cutoff at the antimeridian.

# Only after the stills check out, encode:
npx remotion render MapSeq out/map.mp4 --props='{...}'
```

- `npx remotion compositions` reads `durationInFrames`/`fps` to pick the end frame and per-beat sample frames.
- **Data-driven / batch:** verify ONE representative `beats` array via stills *before* batch-rendering all datasets — catch a projection/label bug once, not N times.
- **README demo GIF for free:** `npx remotion render MapSeq out/demo.gif --codec=gif`.

**Before you finish:**
1. `npx remotion still` renders cleanly at frame 0, mid each beat, and last — no errors, no missing tiles/fonts.
2. Every pin/route/label lands on the correct projected coordinate at every checked frame (spot-check a known city).
3. Two adjacent overlay frames are identical (12fps step holds) while the base camera advanced — stutter is real and only on the graphic layer.
4. Frame-driven only — no `Date.now()` / `Math.random()` / timers; the **shipped** beats render correctly (not just `defaultProps`).
5. Full MP4 encoded and plays; (optional) GIF rendered for the README.

## Reference files

- `references/earth-studio-to-ae.md` — Google Earth Studio keyframe + export settings, the After Effects `.jsx` camera-import workflow, parenting overlays to track points, lat/long ↔ screen-space mapping, and applying the 12fps stutter to the overlay precomp.
- `references/vector-maps.md` — GeoJSON projection setup, vector camera zoom/pan to a lat/long, pin-drop / route draw-on / region-highlight / label-callout recipes, the data-driven `beats` pattern, and the full 12fps stutter quantization recipe for code and AE.
