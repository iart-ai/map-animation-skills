# Google Earth Studio → After Effects — Camera Import Cookbook

The photoreal base-motion path: keyframe a cinematic camera over Google's 3D globe in the browser, export an image sequence plus a 3D camera-track script, and rebuild that exact camera in After Effects so your pins, labels, and routes stick to the right place on the map.

## 1. What Earth Studio exports

Earth Studio (browser tool, earth.google.com/studio) renders two coupled artifacts:

- **Image sequence** — the rendered photoreal frames (JPG or PNG), one per frame, at your chosen resolution/fps. This is your base footage layer.
- **3D camera track (`.jsx`)** — an After Effects script that, when run, rebuilds the camera's position, rotation, and field of view keyframe-for-keyframe, plus **track-point nulls** for any geographic coordinates you marked. This is what lets 2D overlays lock to 3D map locations.

The two are frame-locked: frame N of the image sequence corresponds to frame N of the imported camera. Never resample/retime one without the other or overlays drift.

## 2. Keyframe the camera (in Earth Studio)

Animate slowly and deliberately — this style establishes place, it does not whip.

- Set keyframes on the timeline for **camera position** (latitude, longitude, altitude), **tilt** (pitch), **heading** (rotation), and **field of view**.
- Common moves:
  - **Globe → city zoom:** start high altitude over the globe, end low over the target. Pull FOV slightly to flatten at the end.
  - **Orbit:** hold position/altitude, sweep heading 30–90° around a landmark.
  - **Pan along a border/route:** keep altitude/tilt fixed, move position along the line.
- **Easing:** per keyframe, set ease in/out (Earth Studio's auto-ease is a good start). Avoid linear — it reads robotic. Ease into and out of every hold.
- **Mark coordinate points:** add the lat/long of each place you'll pin/label so they ship as track points in the `.jsx`.
- Keep camera moves smooth at the full project fps (e.g. 24) — the 12fps stutter is applied later to the *overlay*, never to the base camera.

## 3. Export / render settings

- **Render** → set **Resolution** (render larger than final — e.g. 4K for a 1080p edit — so you have room to reframe/scale overlays).
- **Frame rate**: match your edit (24 or 30fps). The base stays smooth at this rate.
- **Format**: image sequence (PNG for alpha/quality, JPG for size).
- **CRITICAL — enable the After Effects / 3D Camera Export option**: check "3D Camera Export" (a.k.a. the After Effects checkbox) so the export bundles the `.jsx` script and the track-point data alongside the frames. Without this you only get footage and cannot rebuild the camera.
- Export. You'll receive the image-sequence folder + the `.jsx` (and a tracking-data file it reads).

## 4. Import into After Effects

1. **Import the image sequence** as footage: File → Import → File, select the first frame, check **"Image Sequence"**, and set the footage's frame rate to match the Earth Studio export (right-click → Interpret Footage → Main → Frame Rate).
2. **Run the camera script**: File → Scripts → **Run Script File…** → select the exported `.jsx`. It builds a new comp (or layers into the current one) containing:
   - a **3D camera** with keyframes matching the Earth Studio move,
   - the **image-sequence layer** referenced as the background,
   - **track-point null layers** for each marked coordinate.
3. Confirm the comp's fps, resolution, and duration match the export. Scrub — the camera should fly exactly as keyframed in the browser.

> If the script can't find the footage, point it at the image-sequence folder when prompted, or pre-import the sequence so it resolves by name.

## 5. Lock overlays to map locations (lat/long ↔ screen space)

The track-point nulls are the bridge from 3D map coordinates to 2D screen space:

- Each track null sits, in 3D space, at the geographic point you marked in Earth Studio. As the camera moves, the null's **screen position** follows the correct spot on the map automatically.
- **Parent** your 2D pin/label/route layers to the relevant track null (or use an expression to read the null's `toComp([0,0,0])` position) so they ride the camera move and stay glued to their location.
- For a coordinate you did *not* mark in Earth Studio, add it back in the browser and re-export, or approximate by parenting to the nearest track null and offsetting.

```jsx
// AE expression on a 2D pin layer's Position: follow a 3D track null's screen position
L = thisComp.layer("Track: Kyiv");
L.toComp([0,0,0]);   // 3D null -> 2D comp pixels, every frame, as the camera moves
```

## 6. Build the overlay precomp

Keep all graphics (pins, routes, highlights, labels) in **one precomp** layered over the base footage. This isolates them so the stutter and grade apply only to graphics, not the photoreal plate.

- Pin drop: position keyframes (down-into-place) + scale overshoot; ease out.
- Route: a shape/mask path stroked on via **Trim Paths** (Add → Trim Paths, animate End 0→100%).
- Region highlight: a masked solid or shape over the area, animate opacity/clip in.
- Labels: text layers parented to track nulls; keep them upright (do not inherit map rotation) and inside the title-safe area.

## 7. Apply the 12fps stutter — overlay only

Make the graphics step at 12fps inside the 24fps master. Two equivalent methods:

- **Posterize time:** select the overlay precomp layer → Effect → Time → **Posterize Time**, set Frame Rate to **12**. (Or as an expression on a property: `posterizeTime(12); value`.)
- **Nested 12fps precomp:** set the overlay precomp's own frame rate to 12fps and place it inside the 24fps master comp.

Leave the base footage layer and the 3D camera at the full rate (24fps) — they stay smooth. Stutter the graphics layer only; stuttering the plate looks like dropped frames, not a deliberate style.

## 8. Finish

- Color-grade the base footage (the Earth Studio plate often needs contrast/saturation to feel editorial).
- Add a subtle vignette/atmosphere if the move is cinematic.
- Render the master comp (overlay stuttered, base smooth) to your delivery codec.

## Gotchas

- Forgetting the **3D Camera Export** checkbox → no `.jsx`, no camera, overlays can't track. Re-export.
- Footage frame rate not matching the camera keyframes → progressive overlay drift. Interpret footage to the exact export fps.
- Retiming the footage without retiming the camera (or vice versa) breaks the lock — retime the whole comp together.
- Earth Studio imagery has usage/attribution terms (Google) — keep the on-screen attribution Earth Studio bakes in, or follow its current export terms.
- This skill is named for the technique, not any outlet; the "Vox-style" descriptor is aesthetic shorthand and implies no affiliation.
