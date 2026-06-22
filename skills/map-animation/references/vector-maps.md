# Vector Maps — GeoJSON / SVG Cookbook

The code-renderable path: project GeoJSON to SVG, fly a transform-based camera to a lat/long, and drive pins, routes, region highlights, and labels from one coordinates array. Runnable patterns for Remotion (frame-driven) and D3/SVG (web), plus the full 12fps stutter recipe. This is the primary reference for the data-driven, deterministic look.

## 1. Projection: lat/long → screen pixels (the foundation)

Everything shares one projection so the basemap, pins, routes, and labels live in the same coordinate space.

```js
import { geoMercator, geoPath, geoInterpolate } from "d3-geo";

const W = 1920, H = 1080;
const projection = geoMercator().fitSize([W, H], geojson);  // fit a feature/collection to the frame
const toPath = geoPath(projection);                          // GeoJSON feature -> SVG "d"
const px = ([lng, lat]) => projection([lng, lat]);           // any coordinate -> [x, y] pixels
```

- **Projection choice:** `geoMercator` for familiar web-map look; `geoEqualEarth`/`geoNaturalEarth1` for honest area on world maps; `geoOrthographic` for a globe (and you can animate `.rotate()` to spin it).
- **`fitSize` / `fitExtent`** frame the map to your composition automatically — prefer it over hand-tuned `scale`/`translate`.
- Get basemap GeoJSON from Natural Earth (countries/states) or world-atlas TopoJSON (convert with `topojson.feature`).

Render the basemap:

```jsx
{geojson.features.map((feat) => (
  <path key={feat.id} d={toPath(feat)} fill="#1b2330" stroke="#0b0f14" strokeWidth={0.5} />
))}
```

## 2. The 12fps stutter (quantize the frame)

The signature cadence. Render the comp at 24fps but step graphic motion at 12fps by snapping the frame before you compute any animated value.

```js
import { useCurrentFrame, useVideoConfig, interpolate, spring } from "remotion";

const frame = useCurrentFrame();
const { fps } = useVideoConfig();
const STEP = Math.max(1, Math.round(fps / 12));   // 2 at 24fps, ~2-3 at 30fps
const f = Math.floor(frame / STEP) * STEP;         // quantized frame — use this for ALL graphic motion
```

Use `f` (not `frame`) everywhere you drive pins/routes/highlights/labels. Derive `STEP` from `fps` so the stutter survives an fps change. Keep any base/background camera on the raw `frame` if you want it smooth; stutter only the overlay layer.

In After Effects the equivalent is **Posterize Time** at 12 (or a 12fps precomp) on the overlay precomp — see `earth-studio-to-ae.md` §7.

## 3. Camera: zoom + pan to a lat/long

The vector "camera" is a transform on the map group. Compute the target's projected pixel, then translate so it centers and scale to zoom in.

```js
const [tx, ty] = px([targetLng, targetLat]);
const scale = interpolate(f, [0, 36], [1, 4], { extrapolateLeft: "clamp", extrapolateRight: "clamp" });
const x = W / 2 - tx * scale;
const y = H / 2 - ty * scale;
const transform = `translate(${x}px, ${y}px) scale(${scale})`;
```

Wrap the basemap **and** overlays in the same `<g style={{ transform }}>` so they zoom together. To keep label *text* readable while zooming, counter-scale labels by `1/scale` (zoom the map, not the type), or render labels outside the transformed group and place them with `px()` recomputed at the current scale.

For a smooth fly-between (A→B), interpolate center along the great circle and ease the zoom out-then-in:

```js
const mid = geoInterpolate([aLng, aLat], [bLng, bLat])(0.5);  // midpoint on the sphere
// zoom out toward mid in the first half, zoom into B in the second half (interpolate scale down then up)
```

## 4. Pins: drop, settle, pulse

Drive from a coordinates array; stagger entrances on the quantized step.

```jsx
const pins = [
  { name: "Kyiv", lng: 30.52, lat: 50.45 },
  { name: "Lviv", lng: 24.03, lat: 49.84 },
];

pins.map((p, i) => {
  const local = f - i * STEP * 4;                       // staggered start, quantized
  if (local < 0) return null;
  const drop = spring({ frame: local, fps, config: { damping: 12, stiffness: 120 } }); // overshoot -> settle
  const [cx, cy] = px([p.lng, p.lat]);
  const y = cy - 40 * (1 - drop);                        // fall into place from above
  const pulse = 1 + 0.15 * Math.max(0, Math.sin((local / STEP) * 0.6)); // optional ring
  return (
    <g key={p.name} transform={`translate(${cx} ${y})`} opacity={Math.min(1, drop * 1.5)}>
      <circle r={8 * pulse} fill="none" stroke="#0A84FF" opacity={0.4} />
      <circle r={5} fill="#0A84FF" />
    </g>
  );
});
```

## 5. Routes: draw-on between coordinates

Project each point of the route, join into one SVG path, normalize with `pathLength`, and stroke it on.

```jsx
const route = [[30.52, 50.45], [27.0, 49.0], [24.03, 49.84]];   // [lng, lat] waypoints
const d = route.map((c, i) => (i ? "L" : "M") + px(c).join(" ")).join(" ");
const draw = interpolate(f, [12, 60], [0, 1], { extrapolateLeft: "clamp", extrapolateRight: "clamp" });

<path d={d} fill="none" stroke="#FF9F0A" strokeWidth={3}
      pathLength={1} strokeDasharray={1} strokeDashoffset={1 - draw} strokeLinecap="round" />
```

- `pathLength={1}` normalizes any route length so one 0→1 value draws it.
- **Curved / flight-path arc:** densify the line first with `geoInterpolate` so a Mercator straight line becomes a proper great-circle curve:

```js
const arc = (a, b, n = 40) => Array.from({ length: n + 1 }, (_, i) => geoInterpolate(a, b)(i / n));
const d = arc([aLng, aLat], [bLng, bLat]).map((c, i) => (i ? "L" : "M") + px(c).join(" ")).join(" ");
```

- **Traveling dot leading the line:** place a circle at `offset-distance: ${draw * 100}%` along the same path (CSS `offset-path`) or sample the path at `draw` with `getPointAtLength`.

## 6. Region highlight: focus + context

Light the target feature, dim the rest. Color = meaning, never reassigned mid-piece.

```jsx
const TARGET = "France";
geojson.features.map((feat) => {
  const on = feat.properties.name === TARGET;
  const lit = on ? interpolate(f, [0, 12], [0, 1], { extrapolateRight: "clamp" }) : 0;
  return (
    <path key={feat.id} d={toPath(feat)}
      fill={on ? "#0A84FF" : "#1b2330"}
      fillOpacity={on ? 0.25 + 0.55 * lit : 0.35}
      stroke={on ? "#0A84FF" : "#0b0f14"} strokeWidth={on ? 1.5 : 0.5} />
  );
});
```

Alternatives: animate a `clip-path` wipe across the region, or pulse the stroke. Keep non-target regions visible but muted so the viewer keeps geographic context.

## 7. Labels & callouts

Anchor to `px([lng, lat])`, reveal after the pin lands, keep horizontal and inside the safe area.

```jsx
const labelIn = interpolate(f, [18, 30], [0, 1], { extrapolateLeft: "clamp", extrapolateRight: "clamp" });
const [lx, ly] = px([p.lng, p.lat]);
<g transform={`translate(${lx} ${ly - 14})`} opacity={labelIn}>
  <line x1={0} y1={0} x2={0} y2={-18} stroke="#F5F7FA" strokeWidth={1} />   {/* leader */}
  <text y={-24} textAnchor="middle" fontFamily="Inter" fontWeight={600} fontSize={28} fill="#F5F7FA">
    {p.name}
  </text>
</g>
```

- Never rotate labels with the map; keep type upright and legible.
- Counter-scale by `1/scale` if labels live inside the zoomed group, so text stays a fixed size while the map zooms.
- ≤ a couple of words per pin; use a callout box with a leader line for longer notes.

## 8. Data-driven sequencing (single source of truth)

One ordered `beats` array drives camera targets, pin order, route segments, labels, and total duration.

```jsx
import { Composition, Series } from "remotion";
import { z } from "zod";

export const schema = z.object({
  beats: z.array(z.object({
    lng: z.number(), lat: z.number(), label: z.string(),
    hold: z.number(), zoom: z.number().default(4), route: z.boolean().default(false),
  })),
});

const defaultProps = {
  beats: [
    { lng: 30.52, lat: 50.45, label: "Kyiv", hold: 48, zoom: 5 },
    { lng: 24.03, lat: 49.84, label: "Lviv", hold: 36, zoom: 6, route: true },
  ],
};

// Each beat = a Series.Sequence: camera flies to (lng,lat,zoom), pin drops, label pops,
// route draws if `route`. Compute duration from the data, not by hand:
export const calculateMetadata = ({ props }) => ({
  durationInFrames: props.beats.reduce((s, b) => s + b.hold, 0),
});

export const Root = () => (
  <Composition id="MapSeq" component={MapSeq} fps={24} width={1920} height={1080}
    schema={schema} defaultProps={defaultProps} calculateMetadata={calculateMetadata} />
);
```

Swap the `beats` array → a new map sequence with no re-keyframing. Verify ONE representative `beats` set with `remotion still` before batch-rendering many datasets.

## 9. D3 / web variant (no Remotion)

Same projection and recipes, driven by a paused/seekable timeline for headless screenshots:

```js
const projection = d3.geoMercator().fitSize([W, H], geojson);
const path = d3.geoPath(projection);
svg.selectAll("path").data(geojson.features).join("path").attr("d", path);
// Route draw-on with stroke-dashoffset; zoom via d3.zoom().scaleTo / a transform on the <g>.
// Freeze for a screenshot: build a master timeline, then ?t=N -> timeline.pause(); timeline.seek(N).
```

## Gotchas

- **Antimeridian / poles:** Mercator clips near the poles and can tear a feature that crosses ±180°. Use `geoEqualEarth`/`geoNaturalEarth1` for world views, or rotate the projection.
- **Coordinate order is `[lng, lat]`** in GeoJSON and `projection()` — not `[lat, lng]`. Swapping them sends every pin to the wrong place.
- **Determinism:** drive all motion from `f` (the quantized frame); no `Date.now()` / `Math.random()` / timers, or stills won't reproduce and CI renders drift.
- **Label readability while zooming:** counter-scale by `1/scale` or render labels outside the transformed group.
- The "Vox-style" descriptor is aesthetic shorthand for this editorial map look and implies no affiliation; the skill is named for the technique.

---
Built by **[iart.ai](https://iart.ai/?utm_source=github&utm_medium=readme&utm_campaign=map-animation-skills&utm_content=skill_footer&utm_term=map-animation)** — the AI motion agent for editable, on-brand motion graphics.
