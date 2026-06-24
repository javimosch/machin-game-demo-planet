---
name: machin-demo-planet
description: Build, run, and modify machin-demo-planet — a procedurally generated GPU mesh (planet-surface chunk) in machin (MFL) via raylib. Use when working on this repo, or as the worked example of machin's pointer/array FFI (alloc/poke raw buffers, the *T deref param, UploadMesh/LoadModelFromMesh) — building a C buffer/struct from MFL and handing it to the GPU.
---

# machin-demo-planet

A procedurally generated **GPU mesh** — a curved chunk of a planet's surface, written in [machin](https://github.com/javimosch/machin) (MFL) and drawn with raylib. It is the reference example for machin's **pointer/array FFI** (raw memory + the `*T` param convention).

> Shared game-dev setup, the FFI surface, and cross-cutting gotchas live in the canonical **[machin-gamedev skill](https://github.com/javimosch/machin/blob/main/skills/machin-gamedev/SKILL.md)**. This file is the specifics.

## Build & run

```bash
./build.sh                 # machin encode planet.src -> planet.mfl, then machin build -> ./machin-demo-planet
./machin-demo-planet
```

Needs `machin` **v0.47.0+** (pointer/array FFI), a C compiler, **raylib**, and a display. `build.sh` prefers a system raylib, else vendors the prebuilt static release into `vendor/` (no root).

## Pointer/array FFI: building a GPU mesh from MFL

machin v0.47.0 added the pieces to hand C a raw buffer/struct:

- **Raw memory** (pointers are `int`s): `p := alloc(nbytes)` (zeroed), `poke_f32`/`poke_i32`/`poke_u8`/`poke_u16`/`poke_ptr(p, byteOffset, v)`, `peek_f32`/`peek_i32`, `free(p)`.
- **`*T` param**: `fn LoadModelFromMesh(*Mesh) Model` — the MFL arg is a pointer (`int`); the call emits `*(Mesh*)ptr`, passing the struct **by value**. To pass the pointer itself, use `ptr` (becomes `void*`, which C converts to any `T*`) — e.g. `fn UploadMesh(ptr, bool)` for `UploadMesh(Mesh*)`, which writes the VAO/VBO ids back into your buffer.
- **Opaque `Model`** (`cstruct Model {}`): raylib's `Model` has pointers, so it's an opaque handle — received from `LoadModelFromMesh`, passed to `DrawModelEx`.

The flow (build once, draw forever):

```
vbuf := alloc(vcount*12)   // 3 f32 per vertex
cbuf := alloc(vcount*4)    // RGBA per vertex
// ... poke positions into vbuf, colors into cbuf ...
m := alloc(112)            // sizeof(Mesh), zeroed
poke_i32(m, 0, vcount)  poke_i32(m, 4, vcount/3)
poke_ptr(m, 8, vbuf)    poke_ptr(m, 48, cbuf)
UploadMesh(m, false)       // GPU upload; writes vaoId@96 / vboId@104 into m
model := LoadModelFromMesh(m)   // *Mesh: deref m, pass Mesh by value
// each frame:
DrawModelEx(model, pos, axis, angle, scale, tint)
```

## The Mesh layout (raylib 5.0, x86-64)

`sizeof(Mesh) = 112`. Offsets you poke: `vertexCount@0` (i32), `triangleCount@4` (i32), `vertices@8` (ptr), `normals@32` (ptr), `colors@48` (ptr); raylib writes `vaoId@96` / `vboId@104`. **Layout-specific** — pin the raylib version (the vendored build does). Get offsets for another version with a tiny C program using `offsetof`.

## Patterns worth copying

- **Build once.** A static mesh is uploaded a single time before the loop; the frame loop is just `DrawModelEx`. That's the whole point vs. immediate mode (machin-demo-terrain) which re-emits every frame.
- **Bake lighting into vertex colors.** The default material is unlit, so flat-shade in MFL (face normal · light) and store the result in the color buffer.
- **Non-indexed mesh.** 6 vertices per cell (two triangles), each triangle its own 3 vertices — makes per-triangle flat color trivial.
- **int/float discipline** as everywhere: world coords `float`, offsets/counts `int`, cross with `float()`/`int()`; `a < -b` → `a < 0.0 - b`.

## Modifying

- **Shape:** `height()`, `BASE_R()`, `LON()`/`LAT()` spans, `CELLS()` resolution (vertex count is `CELLS² · 6`).
- **Look:** the color thresholds and light dir in `shade_color`, the sky `ClearBackground`, the rotation in `DrawModelEx`.
- **Frontier:** a full planet would chunk this with LOD and add `free`/`UnloadModel` on chunk eviction; normals + a light shader for real lighting; instanced flora via `DrawMeshInstanced` (needs a `Matrix` array — more pointer FFI).
- After any edit to `planet.src`, re-run `./build.sh` (never hand-edit `planet.mfl` — it is generated).
