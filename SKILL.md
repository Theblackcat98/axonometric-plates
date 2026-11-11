---
name: axonometric-plates
description: Write a new plate for the Axonometric Plates series — a single-file Three.js axonometric architectural drawing of a building or monument (orthographic orbit camera, drafting-paper overlay, flat colour wash + six-weight ink linework, seeded deterministic PRNG), matching the existing plates. Use when asked to create or add a plate, to draw a building or monument "in the same style" as the plates, or to extend the wonders/architecture series.
---

# Axonometric Plates

A plate is an architectural drawing that runs in the browser: one self-contained HTML file, a building built procedurally in Three.js and presented as an axonometric on drafting paper — flat ink-and-wash surfaces, one light direction, the structure opened up rather than hidden, free to orbit.

The existing plates in this repo are the specification by example: `plates/chichen-itza.html`, `plates/christ-the-redeemer.html`, `plates/colosseum.html`, `plates/great-wall.html`, `plates/hanging-gardens.html`, `plates/hanging-gardens-reconstruction.html`, `plates/hoogbouw.html`, `plates/machu-picchu.html`, `plates/petra.html`, `plates/pisa.html`, `plates/taj-mahal.html`. Before writing a new one, read `plates/template.html` (the canonical engine with a demo subject) and study whichever existing plate is closest to the new subject.

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

**Script structure** — numbered sections, in order: (1) PRNG + paper overlay, (2) scene + camera + controls + reset, (3) palette + wash/ink engine, (4) subject transform if any, (5+) builders, then subject assembly, site furniture, and finally geometry materialisation + render loop. `plates/template.html` is this skeleton with the engine already in place.

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
- `addArchBay(xL, zL, xR, zR, y0, y1, depth, opts)` — a framed arch opening along a wall segment (the Taj's `addMughalArchBay` is the reference implementation): solid piers at both ends, a recessed arch panel between them, an optional screen/jali inset in a contrasting palette, and a parapet band above. Arches are drawn as stepped or segmented profiles through `pushQuadWorld`/`addLineWorld` — never as a dark rectangle with an arch outline alone.
- `addBulbousDome(cx, cz, yBase, domeH, rBase, segs, opts)` — an onion/bulbous dome as a profile table of `[heightFraction, radiusFraction, palette]` rows lofted around the axis: swelling past the drum radius, an inverted lotus collar in terracotta at the neck, a finial on top. Latitudes get line rings only at profile breaks — a dome ruled like a cage reads as glass, not marble.
- `addChhatri(cx, cz, y0, radius, totalH, opts)` — the kiosk: a small drum (octagonal reads best at plate scale), columns or a cornice ring, and a miniature dome cap. Corner punctuation on roofs and minarets.
- **Site furniture** (every plate): a plaza plinth the subject stands on, a dashed survey grid beyond it (`addDashedLineWorld`, faint), sketched shrubs scattered by the PRNG outside the plinth, ghost context masses at the corners in the `ghost` palette (faint ink) — never centre-front, where they fight the subject — and where it earns its place, faint construction geometry: plumb lines, proportion circles, rays.

## Modeling the subject

The engine guarantees the *style*; it guarantees nothing about the *architecture*. A plate can obey every rule above and still be a lesser building — that is exactly what happened with the first Taj Mahal plate (Nov 2026): correct paper, correct ink, correct seed, but a mausoleum suggested by stacked boxes where the real one is arches, bays, and a bulbous dome. The replacement — built from the decomposition below — is the standard. Style is the floor, not the result.

- **Decompose before you draw.** Write down the subject's real architectural components and give *each one its own builder*: for the Taj that list is pishtaq (the great framed arch), two-storey niche bays wrapping the chamfered corners, cylindrical drum, bulbous dome with lotus collar and finial, guldasta pinnacles, chhatris, minaret tiers with balcony rings. If a signature element comes out as a plain box with lines on it, the model is wrong — stop and write the builder.
- **Proportions come from the real building.** Take two or three actual dimensions and derive the ratios (dome diameter vs. plinth width, minaret height vs. dome height, bay width vs. total facade). Eyeballed massing drifts; ratio-derived massing reads as the building even before the details land.
- **Repeated elements are one builder, called in a loop.** Bays, niches, merlons, columns: parameterise once, place by the count you researched. Hand-placed near-copies are how counts and spacings go quietly wrong.
- **Frame the subject, not the site.** Fit `frustumSize` so the building fills the sheet; let gardens, plazas, and context crop at the edges. Shrinking the subject to fit its surroundings is how a monument becomes a diagram of its grounds (the first Taj ran frustum 78 and read small; the rebuild's 54 reads as a monument).
- **Real neighbours are buildings, not blocks.** Flanking structures that exist (the Taj's mosque and jawab) get the same treatment at lower ink weight — massing, a face, a roof silhouette. A blank ghost box is for structures that are *absent or unknown*, not for ones you didn't model.
- **Context is part of the deliverable.** Whoever builds a plate needs this skill, the exemplar plates (`christ-the-redeemer`, `taj-mahal`) open beside them, *and* the decomposition list for the subject. A builder working from the style contract alone will produce style-correct, architecture-wrong work — plan the brief accordingly.

## Workflow

1. **Research the subject before drawing it.** The plates are credible because the counts are right: 80 bays and three arcades on the Colosseum, eight tiers on Pisa, six columns on the Treasury facade. Get the real proportions, the tier/bay/column counts, and the one or two structural facts the drawing should teach (Pisa's corrective curve, the Colosseum's exposed hypogeum). Decide what the cutaway or opening shows. The output of this step is the decomposition list from *Modeling the subject*, not a vibe.
2. **Choose the seed year and the palettes.** Paper, ink, and light never change; the subject's materials do (rose sandstones for Petra, cream marble + terracotta for Pisa). Add palettes, don't retune the engine.
3. **Start from `plates/template.html`.** Rename the title, set the seed, delete the demo rotunda, keep the engine and site furniture.
4. **Model in passes:** massing first (does the silhouette read?), then structure and courses, then the signature details, then the opening/cutaway, then furniture. Run it between passes.
5. **Fit the camera** (`frustumSize`, `targetY`) last, once the massing is final.
6. **Verify** (below). Fix what the screenshots show, not what the code suggests.
7. **Deliver** the standalone file as `plates/<slug>.html` in this repo. The file is the deliverable — it must open and run by itself.

## Verification

Not optional, and not "the code looks right":

- Render screenshots at desktop 1280×800 and mobile 390×844 and actually look at them.
- Adversarial pass on each shot: Is the subject centred and fully in frame at both aspects? Does the ink hierarchy read (silhouette bold, detail quiet)? Do ghost masses stay out of the subject's face? Any z-fighting shimmer, missing faces, or hatching on the lit side? Do the researched counts match what's drawn? Does every signature element read as itself (an arch as an arch, a dome as a dome) rather than as a box suggesting it? Is the caption on its paper patch, clear of the drawing, at both aspects? Are real neighbouring buildings modelled, not blank blocks?
- Orbit it: drag, zoom, double-click reset. The drawing must hold from other angles, not just the default diagonal.
- If the sandbox can't reach the CDNs, test with locally vendored copies of the same two scripts — but ship the CDN version.
- Confirm zero console errors and that the file's only external references are the two CDN scripts.

## Publishing (optional)

The plates are also exhibited in the Showcase on The BlackCat site. Packaging for the site — vendored Three.js swap, toolkit chrome, write-up, registry entry, thumbnails — follows the contract in that repo's `AGENTS.md`; treat it as a separate step from writing the plate, and don't let site chrome leak back into the standalone files here. The `banner-*.gif` files in this repo's `assets/` are the animated banners made for those exhibits.
