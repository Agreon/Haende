# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

"Hände weg" is a single self-contained web page (`public/index.html`) that watches the webcam and alerts the user when their fingers touch their face. Everything runs client-side in the browser; no images are stored or sent anywhere — keep it that way.

There is no build step, package manager, linter, or test suite. HTML, CSS, and an inline `<script type="module">` all live in the one file. The only external dependencies are loaded at runtime:
- `@mediapipe/tasks-vision@0.10.21` (JS bundle + WASM) from jsDelivr
- Hand/face landmarker `.task` models and the EfficientDet-Lite2 object detector from `storage.googleapis.com`
- Manrope / IBM Plex Mono from Google Fonts

## Running

Camera access requires a secure context, so serve the file from localhost rather than opening it directly:

```bash
python3 -m http.server 8000 -d public
```

Then open `http://localhost:8000/` and click "Kamera starten". Verification is manual (touch your face, hold a hand in front of it, cover the face, etc.).

## Deployment

Deployed as a Cloudflare Worker with static assets (custom domain `haende.agreon.de`), configured in `wrangler.jsonc`. Only `public/` is uploaded — anything placed outside it (repo docs, `.git`, config) stays private, so put every file the site needs inside `public/`.

## Conventions

- All UI text and code comments are in **German**; keep new strings and comments German.
- Colors are CSS custom properties on `:root` with a `prefers-color-scheme: dark` override. The canvas overlay and status pill use hardcoded colors that mirror the dark-theme palette (the video area is always dark).
- Code style is compact: short helpers, `$ = id => document.getElementById(id)`, sections marked with `// ---------- Name ----------` comments.

## Detection architecture

The core loop (`tick()`) runs at `FPS = 12`, driven by a **Web Worker timer** (created from a Blob URL in `startTicker()`) instead of `requestAnimationFrame`, because worker timers are throttled less in background tabs — the app is meant to run in a background tab. A `busy` flag drops ticks if one is still running.

Each tick:
1. **Face zone**: FaceLandmarker landmarks → convex hull (`convexHull`). The last face is remembered for `FACE_MEMORY_MS` because a hand covering the face often makes face detection fail exactly when a touch happens (`live: false` → dashed outline, "Gesicht verdeckt").
2. **Zones**: the hull is scaled around its centroid (`scalePoly`) — `1 + margin%` for contact, `1.35 + margin%` for the "near" warning.
3. **Depth filter**: there is no real depth, so a hand counts as "in front of the face" (and is ignored) when its size (wrist→middle-finger MCP, landmarks 0→9) divided by the eye distance (face landmarks 33→263) exceeds the `depth` slider value.
4. **Hit test**: the five fingertips (`FINGERTIPS`) are point-in-polygon tested (`inside`) against the zones.
5. **Drinking exemption**: at the mouth the cup is usually too occluded to detect, so the ObjectDetector (EfficientDet-Lite2, COCO `cup`/`bottle`/`wine glass`) runs on every 2nd tick whenever a hand is visible and looks for a drink overlapping a hand at roughly face height (`findDrink`). Once seen, the exemption is sticky: it is extended while a hand stays at the near zone (ignoring the depth filter) and expires `DRINK_HOLD_MS` after the hand leaves. The detector is optional — if it fails to load, `objDet` is `null` and everything else still works.
6. **Debounce**: contact must persist for `dwell` ms before `onTouch()` fires once; a new touch only counts after the hand has been away for `RELEASE_MS`. Manual pause and the 2-minute snooze suppress counting but detection/drawing continue.

Settings sliders are bound via `bindRange(id, fmt)` into the shared `S` object (`S.dwell`, `S.margin`, `S.depth`); each slider needs a matching `<output id="{id}Out">`. Video and canvas are mirrored with CSS `scaleX(-1)`, so drawing uses raw (unmirrored) landmark pixel coordinates.

`onTouch()` handles all alarm side effects: counter/log/longest-streak stats, Web Audio beep (the `AudioContext` is created on the start-button click to satisfy autoplay rules), red flash/outline, and a temporary tab-title change.
