# machin-game-demo-planet

A procedurally generated **GPU mesh** — a curved chunk of a planet's surface, built **once** in raw memory by **[machin](https://github.com/javimosch/machin)** (MFL), uploaded to a GPU vertex buffer, and drawn as a real model. Heights and per-triangle shading are computed in MFL; the geometry lives in VRAM, not re-sent each frame.

Part of [**awesome-machin**](https://github.com/javimosch/awesome-machin) — the machin ecosystem.

> **Agents:** [`SKILL.md`](SKILL.md) covers the build, the raw-memory mesh upload, and the `Mesh` layout it pokes.

```
        .-~~~-.            machin — planet chunk
      .'  o    `.          oceans (blue) · land (green) · rock · snow
     /   ~~~~~   \         a curved surface tile, lit by face normals
    |  ~~~~~~~~~  |         — a GPU mesh built in MFL
     \   ~~~~~   /
      `.       .'
        `-...-'
```

## Why it exists

This is the dogfood that drove machin's **pointer/array FFI** (v0.47.0) — the first time machin hands C a **raw pointer/array**. The terrain demo before it streamed vertices every frame (immediate mode); this one builds a mesh **once** and uploads it to the GPU, which is how real engines work.

The pipeline, all in MFL:

- compute per-vertex positions on a displaced sphere (native math) and per-triangle flat-shaded colors;
- `alloc` raw buffers and `poke_f32`/`poke_u8` the vertex + color arrays;
- declare a raylib `Mesh` as a **`cstruct` with pointer fields** (`vertices ptr`, `colors ptr`, …) and build it by value — the C compiler lays out the struct, no hard-coded offsets;
- `UploadMesh(mesh, false)` — an **inout `Mesh*`** param: the mesh is passed by pointer and the call writes the VAO/VBO ids back into it;
- `LoadModelFromMesh(mesh)` by value → a `Model` (an opaque handle);
- `DrawModelEx` each frame, rotating the GPU-resident model to show the curvature.

So machin builds a C buffer + struct from scratch and hands it to OpenGL. The Tier-2 unlock in the [game-dev north star](https://github.com/javimosch/machin/blob/main/docs/NORTH-STAR-GAMEDEV.md) — the path to planet-scale static meshes.

## Build

Needs the `machin` compiler (**v0.48.0+**), a C compiler, **raylib**, and a display. A GUI binary links the system graphics stack, so it is **not** a no-dependency binary.

```bash
./build.sh            # → ./machin-game-demo-planet
./machin-game-demo-planet
```

`build.sh` uses a **system raylib** if installed; otherwise it **vendors raylib's prebuilt static release** into `vendor/` automatically — no root required.

## The Mesh cstruct

raylib's `Mesh` is declared as a `cstruct` with the fields the demo touches — pointer fields hold the raw buffers, and the C compiler computes the layout (no hard-coded offsets):

```
cstruct Mesh { vertexCount i32  triangleCount i32  vertices ptr  colors ptr  vaoId u32  vboId ptr }
fn UploadMesh(Mesh*, bool)        // inout — writes vaoId/vboId back into the mesh
fn LoadModelFromMesh(Mesh) Model  // by value, once uploaded
```

Field **names** must match raylib's `Mesh` (the boundary marshals by name); the other Mesh fields default to zero/NULL, which `UploadMesh` handles. Needs machin **v0.48.0+** (pointer-bearing `cstruct` fields + inout `T*`).

See [`planet.src`](planet.src). `build.sh` runs `machin encode` → canonical `planet.mfl`, then `machin build`.

## License

MIT
