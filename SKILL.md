---
name: axonometric-plates
description: Write a new plate for the Axonometric Plates series — a single-file Three.js axonometric architectural drawing of a building or monument (orthographic orbit camera, drafting-paper overlay, flat colour wash + six-weight ink linework, seeded deterministic PRNG), matching the seven existing plates. Use when asked to create or add a plate, to draw a building or monument "in the same style" as the plates, or to extend the wonders/architecture series.
---

# Axonometric Plates

A plate is an architectural drawing that runs in the browser: one self-contained HTML file, a building built procedurally in Three.js and presented as an axonometric on drafting paper — flat ink-and-wash surfaces, one light direction, the structure opened up rather than hidden, free to orbit.

The seven existing plates in this repo are the specification by example: `christ-the-redeemer.html`, `colosseum.html`, `hanging-gardens.html`, `hanging-gardens-reconstruction.html`, `hoogbouw.html`, `petra.html`, `pisa.html`. Before writing a new one, read `template.html` (the canonical engine with a demo subject) and study whichever existing plate is closest to the new subject.

## The contract

These are not defaults to tune per plate — they are what makes the plates a series.

**File shape**
- One HTML file, no build step, no assets. Two CDN scripts only: Three.js r128 (`cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js`) and OrbitControls (`cdn.jsdelivr.net/npm/three@0.128.0/examples/js/controls/OrbitControls.js`), then one inline `<script>`.
- Page background `#e3ded4`; renderer clear colour `0xe5e0d6`.
- Optional small hint line ("Drag to orbit · Scroll to zoom · Double-click to reset") bottom-left, offset from the right edge so it wraps clear on phones.

**Drafting paper** — a 2D canvas overlay above the WebGL canvas: fill `#ede9e0`, orthogonal grid every 28px at `rgba(130,120,108,0.09)`, per-pixel grain ±9 from its own `mulberry32(8841)`, `mix-blend-mode: multiply`, `opacity: 0.55`, `pointer-events: none`. Regenerated on resize.

**Camera** — `THREE.OrthographicCamera`, near −200 / far 300, positioned on the true axonometric diagonal `(d, targetY + 0.78·d, d)`, looking at the subject's vertical centre. Damped OrbitControls (`dampingFactor 0.08`); double-click restores the default position, target, and zoom 1. `frustumSize` and `targetY` are the two per-subject fit values: frame the subject at roughly 70% of frame height and check the portrait phone framing separately — a fit that works at 16:9 can crop at 9:19.5.

**Determinism** — all randomness flows through one `mulberry32` PRNG seeded with a year that matters to the subject (1931 the statue was dedicated, AD 80 the Colosseum, 1173 Pisa's first stone, 312 BC Petra as Nabataean capital). Never `Math.random()`. A plate renders identically on every load.

**Wash** — surfaces are raw vertex-coloured triangles, no lights, no materials with shading. Each material is a palette of five steps `{lit, mid, shade, top, bot}`. Face colour is chosen by `computeDirectionalHex` against a fixed light direction `(-0.35, 0.25, 0.90)`: `ny > 0.65` → top, `ny < −0.65` → bot, else dot product `> 0.28` → lit, `> −0.18` → mid, else shade. Per-face lightness jitter of ±0.009–0.018 keeps large walls from going flat. The wash mesh uses `MeshBasicMaterial({vertexColors: true, polygonOffset: true, polygonOffsetFactor: 1.4, polygonOffsetUnits: 1.4})` so ink sits cleanly on surfaces.

**Ink** — all linework is `LineSegments` accumulated into six layers, each with a fixed colour and opacity:

| layer | hex | opacity | carries |
| --- | --- | --- | --- |
| bold | `0x211b17` | 0.92 | primary silhouettes, arch rings, main edges (drawn as a triple stroke with small offsets) |
| medium | `0x332a24` | 0.80 | secondary edges, capitals, courses |
| fine | `0x4a3f37` | 0.44 | shadow hatching, stone texture |
| rust | `0x8c4531` | 0.64 | terracotta/brick course hatching, ornament |
| teal | `0x527d80` | 0.52 | intarsia, water, window reflections |
| faint | `0x6b6157` | 0.32 | site grid, ghost context, construction geometry |

`addLineWorld(p1, p2, layer, overshoot)` extends each segment slightly past its endpoints (default 0.022) and jitters the ends by ±0.007 from the PRNG (faint layer: no jitter). Bold is rationed — if everything is bold, nothing is.

**Script structure** — numbered sections, in order: (1) PRNG + paper overlay, (2) scene + camera + controls + reset, (3) palette + wash/ink engine, (4) subject transform if any, (5+) builders, then subject assembly, site furniture, and finally geometry materialisation + render loop. `template.html` is this skeleton with the engine already in place.

## Builders

Work in points-as-arrays `[x, y, z]`, in world units where the subject's real proportions set the scale.

- `pushQuadWorld / pushTriWorld(p…, hex, shadeShift)` — the only way wash enters the scene.
- `addLineWorld` — the only way ink enters the scene.
- `subjectPoint(x, y, z)` — every subject builder emits through this hook. Identity for an upright subject; Pisa overrides it with a curved leaning-spine table, which is how the whole tower bends without any builder knowing. Warped, leaning, or terraced subjects belong here, not in the builders.
- `addCylinder(y0, y1, rOuter, rInner, segs, pal, opts)` — drums and rings: optional `courses` (rust layer when the palette is brick/terracotta, fine otherwise), directional shade hatching on the unlit side only (`lightDot < −0.12`), silhouette bolding, hollow form with inner wall + annulus caps when `rInner > 0`.
- `addCircularCornice(y, h, rBase, segs, opts)` — three stacked rings (astragal / terracotta frieze / corona) plus radial dentil lines under the overhang.
- `addColumn(angle, radius, y0, y1, colRadius, opts)` — base plinth, fluted shaft, capital, placed radially; compose rings of them for colonnades.
- `addWorldBox(x, y, z, w, h, d, pal, layer)` — axis-aligned boxes with all six faces shaded from the palette: plinths, plaza decks, ghost masses, furniture.
- Subject-specific builders are expected and encouraged — lofted profiles for a statue's robe (Christ), arch galleries (Pisa, Colosseum), cliff strata and carved facade panels (Petra), stepped planted terraces (Hanging Gardens). Write them from the same primitives; don't import a new rendering path.
- **Site furniture** (every plate): a plaza plinth the subject stands on, a dashed survey grid beyond it (`addDashedLineWorld`, faint), sketched shrubs scattered by the PRNG outside the plinth, ghost context masses at the corners in the `ghost` palette (faint ink) — never centre-front, where they fight the subject — and where it earns its place, faint construction geometry: plumb lines, proportion circles, rays.

## Workflow

1. **Research the subject before drawing it.** The plates are credible because the counts are right: 80 bays and three arcades on the Colosseum, eight tiers on Pisa, six columns on the Treasury facade. Get the real proportions, the tier/bay/column counts, and the one or two structural facts the drawing should teach (Pisa's corrective curve, the Colosseum's exposed hypogeum). Decide what the cutaway or opening shows.
2. **Choose the seed year and the palettes.** Paper, ink, and light never change; the subject's materials do (rose sandstones for Petra, cream marble + terracotta for Pisa). Add palettes, don't retune the engine.
3. **Start from `template.html`.** Rename the title, set the seed, delete the demo rotunda, keep the engine and site furniture.
4. **Model in passes:** massing first (does the silhouette read?), then structure and courses, then the signature details, then the opening/cutaway, then furniture. Run it between passes.
5. **Fit the camera** (`frustumSize`, `targetY`) last, once the massing is final.
6. **Verify** (below). Fix what the screenshots show, not what the code suggests.
7. **Deliver** the standalone file as `<slug>.html` in this repo. The file is the deliverable — it must open and run by itself.

## Verification

Not optional, and not "the code looks right":

- Render screenshots at desktop 1280×800 and mobile 390×844 and actually look at them.
- Adversarial pass on each shot: Is the subject centred and fully in frame at both aspects? Does the ink hierarchy read (silhouette bold, detail quiet)? Do ghost masses stay out of the subject's face? Any z-fighting shimmer, missing faces, or hatching on the lit side? Do the researched counts match what's drawn?
- Orbit it: drag, zoom, double-click reset. The drawing must hold from other angles, not just the default diagonal.
- If the sandbox can't reach the CDNs, test with locally vendored copies of the same two scripts — but ship the CDN version.
- Confirm zero console errors and that the file's only external references are the two CDN scripts.

## Publishing (optional)

The plates are also exhibited in the Showcase on The BlackCat site. Packaging for the site — vendored Three.js swap, toolkit chrome, write-up, registry entry, thumbnails — follows the contract in that repo's `AGENTS.md`; treat it as a separate step from writing the plate, and don't let site chrome leak back into the standalone files here. The `banner-*.gif` files in this repo are the animated banners made for those exhibits.
