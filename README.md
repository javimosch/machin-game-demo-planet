# machin-demo-planet

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
- `alloc` raw buffers and `poke_f32`/`poke_u8` the vertex + color arrays, then `poke_i32`/`poke_ptr` a raylib `Mesh` struct (at its known field offsets);
- `UploadMesh(meshPtr, false)` — the `Mesh*` passed as a `ptr` (the call writes the VAO/VBO ids back into the buffer);
- `LoadModelFromMesh(meshPtr)` — the new **`*Mesh`** FFI convention (deref the pointer, pass the `Mesh` by value) → a `Model` (an opaque handle);
- `DrawModelEx` each frame, rotating the GPU-resident model to show the curvature.

So machin builds a C buffer + struct from scratch and hands it to OpenGL. The Tier-2 unlock in the [game-dev north star](https://github.com/javimosch/machin/blob/main/docs/NORTH-STAR-GAMEDEV.md) — the path to planet-scale static meshes.

## Build

Needs the `machin` compiler (**v0.47.0+**), a C compiler, **raylib**, and a display. A GUI binary links the system graphics stack, so it is **not** a no-dependency binary.

```bash
./build.sh            # → ./machin-demo-planet
./machin-demo-planet
```

`build.sh` uses a **system raylib** if installed; otherwise it **vendors raylib's prebuilt static release** into `vendor/` automatically — no root required.

## A note on struct offsets

The demo pokes raylib's `Mesh` fields at hard-coded byte offsets (`vertexCount@0`, `triangleCount@4`, `vertices@8`, `colors@48`, for raylib 5.0 / x86-64). Those are **layout-specific** — pin the raylib version (the vendored build does). A future machin feature (pointer-bearing `cstruct` fields) could let the C compiler compute the layout instead.

See [`planet.src`](planet.src). `build.sh` runs `machin encode` → canonical `planet.mfl`, then `machin build`.

## License

MIT
