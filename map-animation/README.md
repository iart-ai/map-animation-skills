# Map Animation

Build editorial, "Vox-style" animated maps for explainer and documentary video: a camera that zooms, orbits, and pans over a map while pins drop, routes draw on, regions highlight, and labels call out — all snapped to the signature 12fps stutter. Two production paths: photoreal base motion from Google Earth Studio composited in After Effects, and code-renderable vector maps (GeoJSON/SVG) driven from a coordinates array in Remotion or D3.

## When it activates

- "Make a Vox-style map animation" or "add map graphics to an explainer video."
- "Animate a map zoom" / "pan / orbit a camera over a map."
- "Draw a route on a map" or "drop pins on a map and animate them."
- "Highlight a country / region" and label it.
- "Use Google Earth Studio" (and bring the camera into After Effects).

## Example prompts

- "Make a Vox-style map that zooms from the globe to Kyiv, then draws a route to Lviv."
- "Highlight France on a map of Europe and label the capital."
- "I exported a camera from Google Earth Studio — bring it into After Effects and add animated pins."
- "Turn this list of coordinates into a data-driven map sequence and render an MP4."

## What's inside

- `SKILL.md` — picking the path (Earth Studio→AE vs. vector), the 12fps stutter cadence, the Earth Studio→After Effects camera import, GeoJSON/SVG vector map motion, pin/route/highlight/label overlays, the data-driven `beats` pattern, and a render-stills-then-encode verify loop.
- `references/earth-studio-to-ae.md` — Earth Studio keyframe + export settings, the After Effects `.jsx` camera-import workflow, parenting overlays to track points, lat/long ↔ screen-space mapping, and applying the 12fps stutter to the overlay precomp.
- `references/vector-maps.md` — GeoJSON projection setup, vector camera zoom/pan, pin-drop / route draw-on / region-highlight / label-callout recipes, the data-driven sequencing pattern, a D3/web variant, and the full 12fps stutter quantization recipe.

---
Part of **[Map Animation Skills](../)** · Built by **[iart.ai](https://iart.ai)** — the AI motion agent for editable, on-brand motion graphics.
