# Reservoir simulation grid

Three.js viewer for a synthetic reservoir simulation grid (50 x 50 x 16 cells, waterflood 2026-2038). No build step: Three.js 0.170 loads from CDN via an import map.

## Open
 https://stadniuk-roman.github.io/reservoir_simulation_grid/

## What each part shows

Synthetic model: 50 x 50 x 16 cells (80 m x 80 m x ~7 m), dome trap, sealing fault at I = 29 with 30 m throw, oil-water contact at 1640 m, 2 producers and 3 injectors in the west block, nothing in the east block. Story: by 2038 the waterflood has recovered ~43 % of the west block (37 % of field STOIIP, 255 MMstb); the sealed east block has no producer, so it still holds its oil at original pressure, which is why proposed well PX is there.

Model (synthetic, chosen to be plausible, not simulated):
- Saturation: Swc 0.15 (So max 0.85), Sor 0.20, 25 m transition zone above the OWC. The water front reaches a cell faster in good rock and deep layers (water under-runs oil), then So drops by a front jump plus a slow tail (Buckley-Leverett shape). Vertical sweep (permeability) and areal sweep (distance from injector-producer lines) limit how much movable oil leaves each cell.
- Pressure: oil gradient 0.07 bar/m above the OWC, brine 0.103 bar/m below, 8 bar initial overpressure. Injection replaces most of the produced volume: block average drops ~4 bar, with a sink around producers and a mound around injectors. The fault seals both ways.
- Volumes: STOIIP = pore volume x So x 6.29 / Bo (Bo = 1.2), so results are stock-tank barrels (MMstb). Depths are TVDSS.

| Feature | Three.js / GPU technique | Interview topic |
|---------|--------------------------|-----------------|
| Grid built off the main thread | Web Worker from a Blob URL; typed arrays returned as transferables (no copy) | keep the UI thread free, parsing GRDECL / binary results in a worker |
| Faulted corner-point cells | each cell = 6 quads with own vertices; corner depth uses the cell's own column, so the fault becomes a real step | corner-point grids, why cells are not plain boxes |
| Smaller vertex data | normals as `Int8` normalized, face UVs as `Uint8` normalized, `Uint32` index (960k vertices) | attribute quantization, GPU memory budget (29 MB here) |
| Properties over time | all 13 time steps of one property in one `R32F` `DataTexture`; shader reads with `texelFetch` and blends between steps | changing year or property = one uniform, 0 bytes uploaded |
| Value + I/J/K filter | vertex shader moves vertices of hidden cells outside the clip volume, GPU drops them | GPU filtering vs rebuilding geometry on CPU |
| Cell edges | per-face 0..1 coordinates + `fwidth` in the fragment shader, faded when cells get small | wireframe without line geometry |
| Picking | second `ShaderMaterial` with `PICKING` define writes cell id as RGB into a 1x1 render target (`setViewOffset`), read back with `readRenderTargetPixelsAsync`; shares uniforms, so it filters like the visible pass; stale results from overlapping clicks are dropped | GPU picking vs `Raycaster` on 480k triangles |
| STOIIP / remaining oil / recovery factor | pore volume x So x 6.29 / Bo, summed per step in the worker | basic reservoir engineering numbers, why 35-55 % is a normal waterflood recovery |

What a production version would add: draw only exposed faces (recomputed in a worker when the filter changes), load real simulator output (Eclipse / OPM `.EGRID` + `.UNRST`) and stream time steps on demand instead of keeping all of them in GPU memory, and add well completions and production charts linked to the 3D view.
