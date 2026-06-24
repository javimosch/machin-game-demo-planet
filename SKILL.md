---
name: machin-game-demo-planet
description: Build, run, and modify machin-game-demo-planet — a procedurally generated GPU mesh (planet-surface chunk) in machin (MFL) via raylib. Use when working on this repo, or as the worked example of machin's pointer/array FFI (alloc/poke raw buffers, the *T deref param, UploadMesh/LoadModelFromMesh) — building a C buffer/struct from MFL and handing it to the GPU.
---

# machin-game-demo-planet

A procedurally generated **GPU mesh** — a curved chunk of a planet's surface, written in [machin](https://github.com/javimosch/machin) (MFL) and drawn with raylib. It is the reference example for machin's **pointer/array FFI** (raw memory + the `*T` param convention).

> Shared game-dev setup, the FFI surface, and cross-cutting gotchas live in the canonical **[machin-gamedev skill](https://github.com/javimosch/machin/blob/main/skills/machin-gamedev/SKILL.md)**. This file is the specifics.

## Build & run

```bash
./build.sh                 # machin encode planet.src -> planet.mfl, then machin build -> ./machin-game-demo-planet
./machin-game-demo-planet
```

Needs `machin` **v0.48.0+** (pointer/array FFI), a C compiler, **raylib**, and a display. `build.sh` prefers a system raylib, else vendors the prebuilt static release into `vendor/` (no root).

## Pointer/array FFI: building a GPU mesh from MFL

machin v0.47.0/v0.48.0 added the pieces to hand C a raw buffer/struct:

- **Raw memory** (pointers are `int`s): `p := alloc(nbytes)` (zeroed), `poke_f32`/`poke_i32`/`poke_u8`/`poke_u16`/`poke_ptr(p, byteOffset, v)`, `peek_f32`/`peek_i32`, `free(p)`.
- **Pointer cstruct fields** (v0.48.0): a `cstruct` field may be `ptr` (a pointer, held as `int`), so you declare a struct like `Mesh` with its buffer fields and the C compiler lays it out.
- **Inout `T*` param** (v0.48.0): `fn UploadMesh(Mesh*, bool)` — pass a cstruct *variable* by pointer; the call writes the modified struct (the VAO/VBO ids) back into it. By value for the rest: `fn LoadModelFromMesh(Mesh) Model`.
- Other pointer forms: `ptr` passes a raw pointer (`void*` → any `T*`); `*T` derefs a raw pointer to a by-value struct.
- **Opaque `Model`** (`cstruct Model {}`): raylib's `Model` has pointers, so it's an opaque handle — received from `LoadModelFromMesh`, passed to `DrawModelEx`.

The flow (build once, draw forever):

```
vbuf := alloc(vcount*12)   // 3 f32 per vertex
cbuf := alloc(vcount*4)    // RGBA per vertex
// ... poke positions into vbuf, colors into cbuf ...
mesh := Mesh{vcount, vcount/3, vbuf, cbuf, 0, 0}   // a cstruct; ptr fields hold the buffers
UploadMesh(mesh, false)         // inout: GPU upload; writes vaoId/vboId back into mesh
model := LoadModelFromMesh(mesh)   // by value, now uploaded
// each frame:
DrawModelEx(model, pos, axis, angle, scale, tint)
```

## The Mesh cstruct (machin v0.48.0+)

Declare raylib's `Mesh` as a `cstruct` with the fields you touch — **pointer fields** (`ptr`) hold the raw buffers and the C compiler lays out the struct (no offsets):

```
cstruct Mesh { vertexCount i32  triangleCount i32  vertices ptr  colors ptr  vaoId u32  vboId ptr }
```

Field **names** must match raylib's `Mesh` (the boundary marshals by name); unset fields default to zero/NULL (`UploadMesh` handles missing texcoords/normals). `vaoId`/`vboId` must be declared so they round-trip — `UploadMesh` writes them and `LoadModelFromMesh` reads them. (Pre-v0.48.0 this was poked at hard-coded byte offsets into `alloc`'d memory; the cstruct form is version-robust.)

## Patterns worth copying

- **Build once.** A static mesh is uploaded a single time before the loop; the frame loop is just `DrawModelEx`. That's the whole point vs. immediate mode (machin-game-demo-terrain) which re-emits every frame.
- **Bake lighting into vertex colors.** The default material is unlit, so flat-shade in MFL (face normal · light) and store the result in the color buffer.
- **Non-indexed mesh.** 6 vertices per cell (two triangles), each triangle its own 3 vertices — makes per-triangle flat color trivial.
- **int/float discipline** as everywhere: world coords `float`, offsets/counts `int`, cross with `float()`/`int()`; `a < -b` → `a < 0.0 - b`.

## Modifying

- **Shape:** `height()`, `BASE_R()`, `LON()`/`LAT()` spans, `CELLS()` resolution (vertex count is `CELLS² · 6`).
- **Look:** the color thresholds and light dir in `shade_color`, the sky `ClearBackground`, the rotation in `DrawModelEx`.
- **Frontier:** a full planet would chunk this with LOD and add `free`/`UnloadModel` on chunk eviction; normals + a light shader for real lighting; instanced flora via `DrawMeshInstanced` (needs a `Matrix` array — more pointer FFI).
- After any edit to `planet.src`, re-run `./build.sh` (never hand-edit `planet.mfl` — it is generated).
