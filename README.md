# MuMax3 Shape Editor (`shape-editor.html`)

A single static page (no dependencies, works offline) to preview, edit, and export MuMax3 geometries.
A script describing the geometry is mirrored to its SVG preview, and any edits in the WYSIWYG editor are mirrored in the code.
Open the file in any modern browser.

## 1. Shape API inventory (engine/shape.go)

`type Shape func(x, y, z float64) bool` — a point-membership predicate in metres, origin at the mesh centre.
That is all a previewer needs: sample the predicate per screen pixel (exact, WYSIWYG) and, for vector
export, translate the expression tree into SVG primitives.

**2D shapes** (z-independent, infinite along z) — bold = vector-exportable 1:1

| Constructor | Parameters | SVG form |
|---|---|---|
| **Circle** | diam | `<ellipse>` |
| **Ellipse** | diamx, diamy | `<ellipse>` |
| **Rect** | sidex, sidey | `<rect>` |
| **Square** | side | `<rect>` |
| **Triangle** | x0, y0, x1, y1, x2, y2 | `<polygon>` |
| **Line2D** | x1, y1, x2, y2, diam, cap (`round`/`flat`/`infinite`) | stroked `<line>`; diam = 0 (cell-intersection mode) → raster |
| **XRange / YRange** | a, b (half-open) | `<rect>` clipped to the view |
| **Universe** | – | `<rect>` covering the view |

**3D shapes** (the 2D view shows the slice at the chosen cell layer, 3D support planned for later)

| Constructor | Parameters | Slice at z |
|---|---|---|
| Cylinder | diam, height | circle if \|z\| ≤ h/2 |
| Cuboid | sidex, sidey, sidez | rect if \|z\| < sz/2 |
| Ellipsoid | diamx, diamy, diamz | ellipse scaled by √(1/4 − (z/dz)²) |
| Cone | diam, height (base at z = 0) | circle of diam·\|1 − z/h\| |
| Superball | diam, p | superellipse polygon |
| Line | x1, y1, z1, x2, y2, z2, diam, cap | Line2D with reduced width if z1 = z2, else raster |
| ZRange | z1, z2 | Universe or nothing |
| Layers / Layer / Cell | integer indices (need the mesh) | ZRange / cell box |

**Methods returning Shape**

| Method | 2D? | Notes |
|---|---|---|
| **Transl(dx, dy, dz)** | yes (dz shifts the slice) | `translate()` |
| **Scale(sx, sy, sz)** | yes | `scale()` |
| **RotZ(θ)** | yes | `rotate()` |
| **Repeat(px, py, pz)** | yes | clipped tile + `<use>` copies (raster if > 4000 tiles) |
| RotX(θ), RotY(θ) | 3D only | raster fallback in SVG |
| **Add / Sub / Intersect / Xor (b)** | boolean | union = group, others = SVG `<mask>` |
| **Inverse()** | boolean | masked rectangle |

Out of scope: `ImageShape`, `VoxelShape`, `GrainRoughness`, `ext_GeomEdge`, `ext_AllRegionShapes`,
`ext_InitGeomFromOVF`.

## 2–3. Previewer / editor

* Script pane accepts the Go-like mx3 subset: `a := expr`, `a = expr`, calls, method chains, `+ - * /`,
  comparisons, `pi`, `inf`, math functions, case-insensitive names. `SetMesh`, `SetGridSize`, `SetCellSize`,
  `SetPBC`, `SetGeom`, `DefRegion` and `EdgeSmooth = n` are understood; other statements (`Msat = …`,
  `run(…)`, `for {…}`) are listed as ignored, so a full simulation script can be pasted.
* Drawn items: `SetGeom(...)`, `DefRegion(id, ...)`, and every Shape variable not used later.
* Mesh given → simulation-area border + cell grid (decimated when dense). No mesh → 10 nm reference grid.
* Views: **Shapes** (exact predicate per pixel) and **Cells** (MuMax3 fill fraction per cell).
* Editor: shape bar adds `circle1 := Circle(100e-9)` etc.; drag to move (writes/updates a trailing
  `.Transl(...)`, snapping to cell edges and centres, Shift = free); Alt+drag moves the inner variable
  (e.g. `island` inside `geom`), mapped through Transl/Scale/RotZ; Shift+click multi-select, then
  Add / Sub / Intersect / Xor creates `group1 := a.Sub(b)`; *Set geometry* rewrites `SetGeom(...)`;
  double-click / Edit opens a parameter panel (nm, degrees) with append/remove transforms; Undo/Redo,
  Delete, arrow-key nudge. Edits only rewrite the touched statement; comments and other lines stay.
* 3D: predicates are already 3D and the layer selector slices them; the 3D toggle is disabled until implemented.
  A future View3D only needs its own render/pick over the same `item.shape.f`.

## 4. SVG export / import

1 SVG user unit = 1 nm and `width/height` are in px, so 1 nm = 1 px in Edge/Chrome/Inkscape.
Content sits in `<g transform="scale(1,-1)">` (MuMax3 y-up). The export area is the mesh (or the shapes'
bounding box). Each item is a `<g id="mx3-item-i" mx3:name=…>`; mesh grid in `<g id="mx3-mesh">`.
The full script is stored in `<metadata><mx3:script>` (namespace `urn:mumax3:shape-editor`); Import reads
it back (round-trip is lossless). Arbitrary third-party SVGs are not imported.

## 5. Grayscale PNG for ImageShape

Exports an 8-bit grayscale PNG, 1 pixel per cell (Nx × Ny, top row = largest y), for the chosen layer,
using the exact `geom.setGeom` / `cellVolume` algorithm with EdgeSmooth 0–8.

Two facts from the MuMax3 source that matter here:

* `ImageShape` treats **dark** pixels as inside (`r+g+b < 1.5·max`). The default polarity is therefore
  *black = inside*; *white = inside* is available but must be inverted before use with ImageShape.
* `ImageShape` is binary: gray cells are thresholded at 50 % fill (value ≤ 127 → inside). The gray levels
  document the partial filling but MuMax3 does not use them through ImageShape. Keeping fractional fill
  in a simulation would need an OVF geometry (`ext_InitGeomFromOVF`), which could be a later export.

Not modelled: `TotalShift` (moving frame) and PBC repetition in the preview.

