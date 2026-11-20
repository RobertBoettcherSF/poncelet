# Poncelet Engine — Design Document (Draft 0.10)

> Name: **Poncelet** (chosen 2026-11-19, after Jean-Victor Poncelet, projective geometry; see `docs/NAME_CHECK.md`).
> Author: Robert Boettcher, with Grok Bot. Status: draft 0.10, 2026-11-20 (all signature features in the *first* sellable release, none deferred to M9: anamorphic projections, portals/mirrors, AR calibration, Mandelbulb raymarching, infinite zoom, SPH fluids, destructible voxels, dual contouring — §19.5. Prior 0.9: name kept: Poncelet, after Jean-Victor Poncelet and projective geometry; signature features *beyond* Godot: projective geometry, fractals, and density, §19 and Appendix A.21; plan updated. Prior 0.8: reference implementations from four bot-built games folded in: concrete host ABI v1, event ring, buffers, F9 report format, web host standard, seed parity, Reckoning as first template pack and M1–M4 acceptance test, Appendix B; web-safe Ada subset + numeric host ABI from proven Ada-to-WASM bots; games target all platforms: desktop, web, Android, iOS, consoles via partners, XR; renderer tiers; RTL later; community asset library; multiplayer, dynamic GI, video, visual shaders and CSG moved into the first sellable release; step plan in docs/IMPLEMENTATION_PLAN.md; Godot parity audit + matrix in Appendix A; full 2D + 3D; editor is the first Steam product; Linux-first development, Steam/Apple deferred; AI-tool support is a core feature).
> Clean-room: Godot (MIT) was studied **for architecture only**. No Godot code, text, or assets are copied here.
> Everything below describes our own design.

---

## 1. Vision

A commercial **game engine and editor with its core in Ada**, aiming to be **as capable as Godot in both 2D and 3D**.
The **editor is a desktop application** (Windows, macOS, Linux), and the **first Steam product is the editor itself**
(sold as Steam software), bundled with starter **template packs**. **Games made with Poncelet target all platforms:**
desktop (Steam and elsewhere), **web (Ada → WebAssembly)**, **Android and iOS**, **consoles via third-party porting
partners**, and **VR/AR via OpenXR**, in **seven UI languages**. Its game logic is written so that the parts that
matter (simulation, rules, save files, maths) can later be **proven with SPARK without being rewritten**.

What makes it different:

1. **Correctness first.** The core is Ada 2012/2022 with contracts. Selected packages graduate to SPARK.
   Determinism (fixed-timestep simulation) makes replays, save files, and tests reliable.
2. **Thin, replaceable platform layer.** Windowing, GPU, audio, input, storage, and store services (Steam,
   mobile stores, consoles) sit behind small Ada interfaces. The core never depends on them, which is what makes
   web, mobile, console, and XR targets possible (§10.4).
3. **Editor as product.** The editor is first-class, with polished UX and a UI localised in all seven languages.
   It isn't a developer-only tool.
4. **Templates.** Robert's existing games (built with other bots) become template packs a user can pick,
   using a pack format defined early (§13).
5. **AI-tool friendly by design.** Users drive Poncelet with their *own* AI tools (Cursor, Claude, ChatGPT, Grok,
   Copilot, ...) through text formats, a JSON CLI, and a built-in MCP server. Poncelet bundles no model and stores no AI keys (§12).
6. **Localisation is built in, not bolted on.** All user-visible text goes through string IDs from day one.
7. **Signature features beyond Godot.** Projective geometry, fractals, and density are first-class in the math core,
   the renderer, the editor, and the tools API — not add-ons — and **all of them ship in the first sellable release**
   (decided 2026-11-20). This is the naming tie-in (Jean-Victor Poncelet founded modern projective geometry) and the
   product's technical identity on top of Godot-level 2D/3D parity, besides everything being Plain Ada (§19).

## 2. Scope and non-goals

### Editor hosts vs. game targets (corrected 2026-11-19)
- **Editor hosts:** desktop only — Windows 10/11 x86-64, macOS 12+ (universal), Linux x86-64. The editor is
  primarily used on desktop, and a mobile or browser editor isn't planned.
- **Game export targets:** **all platforms.**

| Target | Toolchain / API | Renderer tier (§6.2a) | Placement |
|---|---|---|---|
| Windows, macOS, Linux | GNAT native, SDL3, SDL_GPU (D3D12/Metal/Vulkan) | High-end (low-end PCs: compatibility) | M1–M4; signed store builds M8 |
| Web (browser) | Ada → WebAssembly (AdaWebPack GNAT-LLVM wasm32 + wasm-ld, proven by Robert's CDDA In ADA and Roguelike Jungle bots; §10.5), WebGL2 (WebGPU later) | Compatibility (WebGPU: mobile tier later) | Toolchain spike M1, 2D M4, 3D M6, WebGPU M9 |
| Android (arm64, x86-64) | GNAT cross / GNAT-LLVM + Android NDK, SDL3, Vulkan or GLES3 | Mobile (old devices: compatibility) | Dev builds M6; Play Store signing M8 (gated) |
| iOS / iPadOS (arm64) | GNAT-LLVM arm64-apple-ios + Xcode toolchain, SDL3, Metal | Mobile | CI compile M6; device builds and App Store M8 (gated on Mac + Apple account) |
| Consoles | Third-party porting partners under platform NDAs; console-friendly platform layer from M0 | Partner-provided backend | Post-launch (M9), partner-driven |
| VR/AR (OpenXR) | OpenXR loader, Vulkan path (PC VR) and Android (standalone headsets) | High-end / mobile with stereo views | Render hooks M7, OpenXR M9 |

**Why this placement:**
- **Web is pre-launch.** Robert's Roguelike Jungle and CDDA In ADA bots already build Ada games to WebAssembly
  (Reckoning even runs on iPhone; Hood Simulator and Colony4X are JavaScript games), githack can host web
  builds for demos, and web export is a strong selling point for the editor.
- **Android and iOS are pre-launch as dev builds** (M6), so mobile constraints shape the renderer and input early.
  Store signing waits for the sign-up gate in M8, like Steam and Apple.
- **Consoles and XR are post-launch.** Console SDKs are NDA-only and partner-driven. XR needs the finished 3D
  renderer and the mobile tier. The architecture is prepared for both before launch.

### In scope (v1 = first sellable editor release)
- **Full 2D and full 3D, designed together from the start** (target: feature parity with Godot for typical indie
  games). 2D: sprites, tilemaps, 2D lights/shadows, particles, 2D physics. 3D: PBR, glTF import, lights and shadows,
  3D physics, skeletal animation, navigation. One architecture, one scene format, one editor.
  **Delivery order:** 2D ships first, then 3D lands on the same architecture (§16). Nothing in the 2D work may
  assume 2D-only.
- UI languages: **English (source), German, French, Russian, Simplified Chinese, Spanish, Italian**, for both
  the editor and games.
- **The editor is the product:** project manager, 2D and 3D viewports, inspector, asset pipeline, localisation
  editor, play-in-editor, exporter for desktop, web, Android, and iOS (consoles via partners), and an asset library
  browser (curated paid template packs at launch, community library post-launch, §13.3).
- **Template packs** bundled with the editor. Their contents will be decided later with Robert's other game bots.
- Steamworks: for the editor product (achievements optional, Workshop for template packs later) and as a ready
  integration for users' games (achievements, stats, cloud saves, rich presence, overlay, language).
- **Signature features beyond Godot (§19), all in the first sellable release:** a projective math core and editor
  perspective tools, including anamorphic / non-standard projections, portals and mirrors, and AR camera-calibration
  homographies; fractal generators (noise, L-systems, IFS, escape-time shader nodes, seeded world generation),
  Mandelbulb-style SDF raymarching with LOD, and deterministic infinite zoom; first-class density fields (fog/smoke,
  scattering, heat maps, editor painting, isosurfaces), SPH fluids, destructible voxel terrain, and dual contouring.
  Nothing in the three pillars is deferred to M9.

### Non-goals (for now)
- **A mobile or browser version of the editor** (the editor is a desktop application).
- **Steam Deck-specific tuning** in v1. Linux and Proton builds run on Deck, and controller-first UI from the console
  rules (§10.4) helps.
- A general-purpose scripting VM such as a GDScript equivalent. Game logic is Ada, compiled in (see §6.6).
- Godot's scripting languages (GDScript, C#): game code is Ada.
  Goal: **functional equivalence with Godot 4.x for games on all platforms.** Every Godot capability is either planned with a
  milestone or listed as an intentional difference in the **parity matrix (Appendix A)**. The first sellable
  release (M8) includes the core **plus high-level multiplayer, dynamic GI, video playback, the visual shader
  editor, and CSG** (decided 2026-11-19), **and the whole of the three signature pillars** (decided 2026-11-20, §19).
  Only the non-signature long tail is post-launch (M9: OpenXR runtime, RTL, community library, consoles, WebGPU).
- Open-source distribution. The repository is public but **source-available, all rights reserved** (§15).

## 3. What we learned from Godot (adopt / adapt / avoid)

Godot's tree: `core/` (math, object model, variant, io, string/translation, templates/containers, os),
`servers/` (rendering, physics 2D/3D, audio, display, text, navigation), `scene/` (2d, 3d, gui, main, resources,
animation), `drivers/` (vulkan, d3d12, metal, gles3, wasapi, coreaudio, pulseaudio, alsa, sdl), `platform/`
(linuxbsd, macos, windows, web, android, ios), `modules/` (optional features such as physics backends, text
shaping, codecs), `editor/` (built on the engine's own GUI), and `main/` (boot, main loop, fixed-step timer sync).

| Godot idea | Verdict | Our version |
|---|---|---|
| **Servers**: subsystems (render, physics, audio, text) behind a narrow API, addressed by opaque resource IDs, and runnable on their own thread | **Adopt** | "Services" with **generational handles** (`Handle := (Index, Generation)`). Opaque handles are ideal for SPARK: no pointers cross the boundary, and invariants are easy to state. |
| `core/` with no dependency on scene or rendering | **Adopt** | `pc-core` has zero platform dependencies and can be checked in SPARK mode. |
| **Scene tree** of polymorphic 2D and 3D nodes (deep inheritance, signals, virtual callbacks) | **Adapt** | Pointer-heavy, dynamic-dispatch trees are hard to prove. We keep the **authoring idea** (nested scenes, instancing, a hierarchy in the editor) but the runtime stores data in an **ECS-style World**: entities are handles, components live in bounded tables, and hierarchy is a `Parent` component. |
| Variant/dynamic object model with reflection | **Avoid** at runtime / **Adapt** for tools | Runtime is statically typed. Reflection for the editor and serialisation comes from **generated** Ada (a code generator emits component descriptors), not from runtime introspection. |
| **Resources**: reference-counted assets with a text format for scenes and resources, plus a binary format for export | **Adopt** | Documented, deterministic text formats with published JSON Schemas (`.pscene`, `.pres`, `.pimport`; §12.1), and an exporter compiles them to binary packs with checksums. Assets are referenced by stable UUIDs, not paths. |
| **RenderingDevice** abstraction over Vulkan/D3D12/Metal, plus a GL compatibility renderer | **Adapt** | We don't write our own RHI. We use **SDL3 GPU**, which already abstracts Vulkan, Metal, and D3D12 (§6.2). |
| **TextServer** with complex shaping (HarfBuzz, ICU, FreeType) and a fallback | **Adopt** | FreeType + HarfBuzz via C bindings. Our seven languages need no RTL, but CJK line breaking and font fallback are required. |
| **Translation domains**, CSV/gettext catalogs, plural rules, locale fallback | **Adopt** | §8. |
| **Export templates**: pre-built runtime per platform plus a data pack | **Adopt** | Runtime binary per OS + `.ppak` data pack. Built per OS in CI (§10). |
| Editor built with the engine's own GUI | **Adopt** | The editor is the product, built on our own Ada UI toolkit (themes, docking, localisation). Dear ImGui is used only for internal debug overlays (§11). |
| GDExtension (C ABI plug-ins) | **Defer** | Static linking only in v1. A C ABI plug-in API may come later. |
| Fixed physics tick decoupled from render (main timer sync) | **Adopt** | Fixed 60 Hz simulation with an accumulator. Rendering interpolates. Simulation is deterministic. |
| WorkerThreadPool and command queues to servers | **Adapt** | Ada tasks + protected objects for a job system. Simulation stays single-threaded in v1 for provability (§6.5). |
| Many optional modules (codecs, XR, networking) | **Avoid** in v1 | Keep the dependency set small (§7). |
| SCons build | **Avoid** | gprbuild + **Alire** (Ada's package manager and toolchain manager). |

## 4. Architecture overview

```
+---------------------------------------------------------------+
| Tools: Editor (GUI) | `poncelet` CLI (JSON) | MCP server       |   all share one Ada tools API
+---------------------------------------------------------------+
| Game (Ada)  — per-game packages; templates live here           |
+---------------------------------------------------------------+
| pc-game-api : Systems, Events, Input actions, UI, Save/Load    |   Plain Ada, contracts
+---------------------------------------------------------------+
| pc-core (SPARK-ready)                                          |
|  Math (incl. projective) | World/ECS | Physics 2D/3D API | Anim | Nav    |   no platform deps
|  Serial | Rules | I18n | Scene 2D/3D | Asset DB | Template packs  |
|  RNG | Noise / Fractal | Density fields | Bounded containers | Time/Step |
+---------------------------------------------------------------+
| pc-services (Ada interfaces, handles only)                     |
|  Render (2D batch + 3D clustered) | Physics3D backend | Text   |   SPARK_Mode Off
|  Audio | Input | Window | Files | Steam                         |
+---------------------------------------------------------------+
| pc-platform (thin Ada bindings to C)                           |
|  SDL3 (+SDL_GPU, audio, input) | FreeType | HarfBuzz           |
|  Jolt (C shim) | cgltf | KTX/Basis | Steamworks flat C API      |
|  stb_image | Opus/Vorbis | meshoptimizer                        |
+---------------------------------------------------------------+
| Desktop (D3D12/Vulkan/Metal/GL) | Web (WASM + WebGL2/WebGPU)     |
| Android (Vulkan/GLES3) | iOS (Metal) | Consoles (partners) | XR |
+---------------------------------------------------------------+
```

The dependency rule is strict: arrows point **down only**. `pc-core` must build with
`gprbuild -XPLATFORM=none` and pass its unit tests headless.

## 5. Key architecture decision: Ada main program over C libraries via SDL3

### 5.1 Options evaluated

| Option | Pros | Cons | Verdict |
|---|---|---|---|
| **A. Ada main + SDL3 (window, input, audio, SDL_GPU) via thin bindings** | One C API covers Windows, macOS, and Linux. SDL_GPU (SDL 3.2+) wraps Vulkan, Metal, and D3D12, so macOS works without MoltenVK. Steam itself ships SDL. zlib licence. Small surface to bind. | SDL3 Ada bindings are young, so we generate or own a thin binding (`gcc -fdump-ada-spec` + curation). Shaders need cross-compilation (SDL_shadercross / SPIR-V → MSL/DXIL). | **Chosen** |
| B. Ada + SDL2 (SDLAda) + OpenGL 3.3 | Mature Ada binding (SDLAda). GL is simple. | OpenGL is deprecated on macOS (frozen at 4.1, could be removed). SDL2 is in maintenance mode. | Fallback / prototype only |
| C. Ada + raw Vulkan (+ MoltenVK on macOS) + own windowing | Maximum control. | Huge effort (RHI, sync, macOS quirks). Not justified while SDL_GPU covers the same backends. | Reject; revisit only if SDL_GPU limits 3D features (e.g. ray tracing, mesh shaders) |
| D. C/C++ host (e.g. raylib, sokol, or a Godot fork) with Ada core as a library | Fastest to pixels. | Two languages own the main loop, which makes the build and the debugger harder. A Godot fork conflicts with clean-room and licence goals. | Keep as an escape hatch: the core is a library anyway |
| E. raylib via Ada bindings | Very easy. | OpenGL-only (same macOS concern). Less control over the GPU pipeline. | Prototype tool only |

**Decision:** Option A. The Ada program owns `main` and the game loop, and calls SDL3 through a thin binding
confined to `pc-platform`. Because `pc-core` is platform-free, Option D stays possible later if we ever need a
C++ host.

### 5.2 Main loop
1. Poll SDL events → translate them into engine `Input_Event` records → push to the input service.
2. Accumulate wall time. Run `Simulation.Step` (fixed dt = 1/60 s) N times (clamped). The core is pure and deterministic.
3. Extract a **render snapshot** (plain records) from the World, then interpolate.
4. The Render2D service builds GPU command buffers from the snapshot and presents.
5. Pump Steam callbacks (`SteamAPI_RunCallbacks`) once per frame.

## 6. Core subsystems (Ada)

### 6.1 Math (`Pc.Math`)
- `Real` is `Float` (IEEE 32-bit) for rendering-facing data. Simulation can opt into **fixed point**
  (`type Fixed is delta 2.0**(-16) range ...`) where cross-platform bit-exact determinism matters
  (lockstep or replays), decided per game.
- Vec2/Vec3/Vec4/Mat3/Mat4/Quat/Rect/AABB/Plane/Frustum/Transform2D/Transform3D as records with `Pre`/`Post`.
  Example: normalising requires length > epsilon.
- Large 3D worlds: an optional `Long_Float` (double) world-position mode with camera-relative rendering, similar
  in spirit to large-world builds elsewhere. Decided per project.
- **Projective geometry is first-class** in `Pc.Math.Projective`: homogeneous points, lines and planes (including
  points at infinity), incidence, cross-ratio, duality, and projective transforms. The 3D camera model and the
  editor's perspective tools are built on these types, not on ad-hoc 4×4 tricks (§19.1). Euclidean Vec/Mat/Quat
  remain the everyday API; projective types are used where the geometry needs them.
- No exceptions on the hot path. Contracts express the domain.

### 6.2 Rendering (service; SDL_GPU backend)
One render service with a shared core (resources, render graph, cameras, materials) and two pipelines that can
be composited in the same frame: a 2D canvas over or under a 3D world, and 3D inside 2D via viewports.

**Shared core**
- GPU resources via SDL_GPU (Vulkan/Metal/D3D12) behind Ada handles: buffers, textures, samplers, pipelines.
- A small **render graph** (passes declared per frame, with automatic barriers and transient render targets)
  so 2D, 3D, shadows, and post-processing are just passes.
- **Cameras:** 2D (zoom, rotation, limits, smoothing) and 3D (perspective, orthographic, frustum offset), with a
  **projective camera model** underneath (§19.1): homogeneous clip space, a robust infinite far plane, and
  first-class 2.5D / fake-3D projection modes. Multiple viewports and render-to-texture (minimaps, split-screen,
  editor previews).
- **Shaders:** written once in HLSL and compiled offline to SPIR-V/MSL/DXIL with SDL_shadercross. A **material
  system** with uber-shader permutations plus user shaders in a small Poncelet shader dialect, compiled by the
  editor. Pipeline cache per GPU.

**2D pipeline**
- Batched sprite and canvas renderer: atlases, 9-slice, tilemaps (chunked static buffers), text (SDF glyph atlases),
  primitives and polylines, particles (GPU compute where available, CPU fallback).
- 2D lights (point, directional) with normal maps and 2D shadow casting (occluder polygons → 1D shadow maps).
- Canvas layers, y-sort, parallax, and pixel-perfect mode.

**3D pipeline: clustered forward+ renderer**
- **Depth pre-pass**, then **light clustering** (froxel grid, built by a compute shader) and **forward shading**
  with all clustered lights. MSAA-friendly, handles transparency well, and copes with many lights. That fits
  indie-scale scenes and integrated GPUs (MacBooks) better than a deferred G-buffer.
- **PBR materials:** metallic-roughness (glTF 2.0 model) with albedo, normal, ORM, emission, alpha
  mask and blend, clearcoat, double-sided, and vertex colours. Image-based lighting from environment maps
  (prefiltered specular + irradiance), sky (procedural and HDRI), fog, and volumetric fog later.
- **Lights:** directional, point (omni), spot, area (later), with a per-light shadow toggle.
- **Shadows:** cascaded shadow maps (4 cascades, PCF/PCSS soft filtering) for directional lights, cube or
  dual-paraboloid shadows for point lights, perspective maps for spots, and a shadow atlas with per-light resolution.
- **Global illumination:** v1 has SSAO + baked lightmaps (xatlas unwrap + CPU/GPU baker) + reflection probes.
  Later: dynamic GI (SDFGI/DDGI-style probes) as a stretch goal.
- **Post-processing:** HDR, tonemapping (ACES, AgX), bloom, SSR, DOF, auto-exposure, colour grading LUT, FXAA/TAA.
- **Geometry:** meshes with LODs (meshoptimizer), frustum and occlusion culling (Hi-Z), GPU instancing, MultiMesh
  for foliage, skinned meshes (GPU skinning), blend shapes, terrain (heightmap chunks) post-v1.

**glTF 2.0 import**
- cgltf (MIT) parses `.gltf`/`.glb`. The importer converts to our binary mesh, material, skeleton, and animation
  resources (meshoptimizer for vertex cache and LODs, MikkTSpace tangents, KTX2/Basis texture compression).
  Imported scenes become `.pscene` sub-scenes that can be re-imported without losing user overrides.
- Also supported: OBJ (static) and image formats via stb_image. FBX is out of scope (users convert via Blender).

### 6.2a Renderer tiers (one scene, three renderers)
Like Godot, Poncelet has **three renderers behind one render service**. Scenes, materials, and the render graph are
shared, and each renderer implements the same `Render_Method` interface. Projects choose a default tier per export
target, and users can switch at runtime where it makes sense.

| Tier | Backend | Targets | Features |
|---|---|---|---|
| **High-end ("Clustered")** | SDL_GPU: D3D12, Vulkan, Metal | Desktop, PC VR | Everything in §6.2: clustered forward+, compute particles, dynamic GI, volumetric fog, SSR/SSIL, TAA/FSR |
| **Mobile** | SDL_GPU: Vulkan (Android), Metal (iOS/Apple Silicon); WebGPU later | Phones, tablets, standalone XR, mid-range PCs | Single-pass forward with per-object light limits (e.g. 8 omni + 8 spot), fewer post effects, baked GI and light probes, half-res effects, ASTC/ETC2 textures, tile-friendly passes (no expensive full-screen copies) |
| **Compatibility** | OpenGL 3.3 core / OpenGL ES 3.0 / **WebGL2** through SDL3 GL contexts (Emscripten on web) | Web, old or low-end PCs, older Android devices | Forward rendering, limited lights, lightmaps, basic shadows, MSAA/FXAA, 2D fully supported, no compute-dependent effects |

- The shader dialect compiles to SPIR-V/MSL/DXIL (high-end/mobile) and to **GLSL 330 / GLSL ES 300** (compatibility)
  via SPIRV-Cross. Materials declare which features they need, and the editor warns when a material uses a feature
  the target tier lacks, then falls back where it can.
- **2D is identical on all tiers.** 3D degrades gracefully: unsupported effects are disabled with an editor warning.
- **Order:** high-end first (M1/M5); compatibility 2D in M4 (needed for web); compatibility 3D and mobile tier in M6.

### 6.3 World / ECS (`Pc.World`)
- `Entity_Id` is a generational handle. Entities live in a bounded slot table (capacity set per game at
  compile time or load time).
- Components are stored in **dense arrays per component type** (struct-of-arrays) with sparse index maps.
  Generic packages: `Pc.World.Store (Component_Type, Capacity)`.
- Hierarchy: `Parent` and `Children_Range` components. World transforms are computed in a system pass, not on demand.
- **2D and 3D from day one:** `Transform_2D` and `Transform_3D` components (an entity has one or the other).
  Typed component sets cover the Godot-equivalent node kinds: Sprite, TileMap, Camera2D/3D, MeshInstance, Light
  (directional/omni/spot), Body2D/3D, Area2D/3D, CharacterBody2D/3D, AnimationPlayer, AudioSource2D/3D, NavAgent.
- Systems are plain procedures with explicit `in`/`in out` parameters. That makes data flow visible to SPARK
  flow analysis (`Global`/`Depends` can be added later without changing signatures).
- **Scenes** (authoring) are trees in the editor and serialise to `.pscene`. Loading a scene **instantiates
  entities** into the World. Scene instancing and nesting work as they do in Godot conceptually.

### 6.4 Physics 2D and 3D (`Pc.Physics`)
One engine-side API (`Pc.Physics.Api_2D`, `Pc.Physics.Api_3D`): bodies (static, kinematic, dynamic), shapes, joints,
areas/triggers, ray and shape queries, character controllers, collision layers and masks. Backends sit behind it.

**2D: our own Ada implementation (decided)**
- Shapes: circle, capsule, box, convex polygon, segment, and tilemap collision. Broadphase: dynamic AABB tree.
  Sequential-impulse solver with warm starting, CCD for fast bodies, and joints (pin, spring, groove).
- A prime **SPARK candidate**: absence of runtime errors, bounded iterations, no division by zero, deterministic.

**3D: evaluation**

| Criterion | **Jolt Physics via C shim** | **Own Ada 3D physics** |
|---|---|---|
| Capability | Production-proven (Horizon Forbidden West; Godot's optional 3D backend). Convex, mesh, height field, compound shapes, constraints, vehicles, ragdolls, soft bodies, multithreaded, deterministic option | Must be built: GJK/EPA, convex hull, mesh BVH, constraint solver, CCD, character controller, vehicles. Years to reach Jolt quality |
| Effort to Godot-level 3D | Weeks (C shim over its C++ API + Ada binding) | 1–2 engineer-years+ |
| SPARK/correctness | Outside the SPARK boundary; contracts on the Ada wrapper only | Fully provable over time |
| Licence | MIT | ours |
| Determinism | Cross-platform determinism available as a compile option | under our control |
| Risk | C++ shim maintenance, ABI per OS (manageable) | schedule, solver robustness |

**Decision:** ship 3D on **Jolt behind our Ada API** (we write a small `extern "C"` shim, about 1–2 k lines, since
Jolt is C++). In parallel, grow an **own Ada 3D backend** for the simple, provable subset (queries, triggers, kinematic
character controller, boxes and spheres and capsules). It can become the default later if it reaches parity. The
backend choice is a project setting, as in Godot. Game logic only sees the Ada API, so switching backends doesn't
change game code.

**Navigation:** 2D/3D navmesh baking (Recast via C shim, or our own for 2D) and path queries in Ada (A* over
navmesh polygons, a SPARK candidate). Avoidance (RVO) later.

### 6.5 Concurrency
- v1: simulation is single-threaded. Asset loading and audio mixing run in Ada tasks, and communication uses
  protected objects with bounded queues (a Ravenscar/Jorvik-compatible style, which SPARK supports).
- Later: a job system for render-snapshot building and physics islands.

### 6.6 Game logic and "scripting"
- Game code is **Ada packages compiled with the engine**. There's no VM. This gives contracts, speed, and
  provability, and matches Robert's "logic in plain Ada" requirement.
- Data-driven pieces cover what designers would otherwise script: state machines, dialogue graphs, loot and
  spawn tables, and tweens are **data assets** interpreted by verified Ada interpreters.
- Hot reload: data assets reload live. Code changes reload too, see §6.6a.

### 6.6a Iteration speed without a scripting language (parity with GDScript workflows)
Godot users expect "edit a script, see it immediately". We match that without an interpreter:
- **Game code as a hot-reloadable shared library.** In the editor and in play mode, the game's Ada packages are built
  as a shared library (`libgame.so` / `.dll` / `.dylib`) that the runtime loads. All game *state* lives in the
  engine-owned World (components are plain records, §6.3), not in library globals. So on reload the engine:
  1. pauses the simulation at a frame boundary;
  2. unloads the old library and loads the new one;
  3. checks component layouts by descriptor hash. If a layout changed, it round-trips the affected components
     through the canonical serialiser with defaults for new fields;
  4. resumes. Typical edit → running time target: **under 3 s** for a small change (incremental gprbuild + relink).
- Exported games link statically, so there's no library loading in shipped builds.
- **Data assets** (state machines, dialogue, tweens, loot and spawn tables, animation graphs) cover the work
  designers would script, edited live without any compile.
- **Expression fields:** a small, sandboxed, total expression language (arithmetic, comparisons, component reads,
  no loops or side effects) for data-asset conditions and formulas. It's interpreted by verified Ada code, a SPARK
  candidate, with no general scripting VM.
- **Decision:** no general-purpose scripting language in v1. We'll revisit after M8 if users ask for one. A
  sandboxed language would plug into the same tools API.


### 6.7 Serialisation and saves (`Pc.Serial`)
- Text: **canonical JSON** for scenes, resources, and import settings (UTF-8, LF, two-space indent, fixed key order,
  one entity per block, stable UUIDs, no floats in exponent form, trailing newline). Byte-identical output for
  identical data, so diffs are minimal and reviewable. See §12.1.
- Binary: versioned little-endian records with a header (magic, format version, CRC32/xxHash).
  Explicit `Read`/`Write` procedures, with no `'Read`/`'Write` streams on access types.
- Save games: versioned, with migration functions `Migrate_Vn_To_Vn1` that have contracts. Saves go to Steam Cloud.

### 6.8 Audio (service)
- Uses SDL3 audio streams for device output. The **mixer is written in Ada** (voices, buses, volume and pan,
  fades, simple DSP). Decoding is Ogg Vorbis/Opus via C libraries (stb_vorbis or libopus).
- Alternative: miniaudio (public domain / MIT-0) if we need more features quickly.
- **Parity features** (Appendix A.10): bus graph with an effect set (reverb, EQ, compressor, limiter, delay, chorus,
  filters, distortion, pitch shift). 2D/3D positional sources with attenuation curves, Doppler, listeners, reverb
  zones. Sound containers (random, sequence, no-repeat), polyphony and voice limits. Adaptive music (clips,
  transitions, stems, beat-synced switching). Procedural generator streams. Surround 5.1/7.1. Formats WAV, QOA,
  Ogg Vorbis/Opus, MP3. Microphone capture, TTS, and MIDI input come later.

### 6.9 Input
- Action mapping (`"move_left"`, `"confirm"`) with rebinding, keyboard/mouse and gamepads via SDL3.
  Steam Input works transparently through SDL. Rebinding UI is localised.
- **Touch and mobile input:** multi-touch, gestures (tap, swipe, pinch, rotate), virtual joystick and buttons as UI
  widgets, accelerometer and gyro, safe-area insets, on-screen keyboard. The input model is designed for this in M3
  and shipped on devices in M6. On the web, DOM events are mapped by the platform layer.
- **XR input:** OpenXR action sets map onto the same action system (M9).

### 6.10 UI (runtime)
- Our own retained-mode UI in Ada: layouts (box, grid, anchors), widgets (label, button, slider, list,
  text field with IME), and themes. **Every label takes a string ID, not a literal.**
- IME support (needed for Chinese text entry) uses SDL3 text-input/IME events.
- The **same toolkit powers the editor** (docking, trees, property grids, code/text views, theming, DPI scaling).
- **Parity features** (Appendix A.11): containers (box, grid, flow, margin, centre, aspect, split, foldable, scroll,
  tabs), stretch modes and content scaling, rich text with a small localisable markup (styles, colours, links, inline
  images, effects), tree/table, searchable dropdowns, colour picker, transformable controls, drag-and-drop, a theme
  editor with vector and 9-patch styleboxes, and multi-line text and code editing.
- **Accessibility:** UI exposes an accessibility tree to screen readers via the AccessKit C API (Windows, macOS,
  Linux), for both the editor and games. Paired with platform TTS later.

### 6.11 Animation
- Keyframe tracks on any component property (2D and 3D), easing curves, events, and tweens.
- 3D skeletal animation from glTF, blend trees and state machines (data assets), root motion, IK (two-bone, look-at).
- 2D: sprite-frame animation, 2D skeletons (bones + mesh deform) post-v1.
- **Parity features** (Appendix A.9): an animation graph (state machine, 1D/2D blend spaces, blend/add/one-shot
  nodes), event and audio tracks, FABRIK/CCD IK chains, spring bones, and humanoid retargeting on import.

### 6.12 Plugins and extensions (Ada)
- **Editor plugins** in Ada, built as shared libraries against a versioned `Pc.Editor.Plugin` API. They can add
  docks, inspectors for custom components, viewport gizmos, menu and command-palette entries, importers and
  exporters, and custom node types for the material graph.
- **Runtime extensions:** Ada library projects (an Alire crate or a gpr project) linked into games. A **stable C ABI**
  (`pc_ext.h`) lets native libraries (C, C++, Rust, Zig) register services and components (the GDExtension
  equivalent). The ABI version is checked at load time.
- **Safety:** plugins run in-process with the user's trust, are explicitly enabled per project, and their
  operations go through the undo system. The MCP server can list plugins but not install them.
- **Distribution:** inside template packs, and via Steam Workshop later.

### 6.13 Networking and multiplayer (`Pc.Net`)
- **Low level:** TCP/UDP sockets on GNAT.Sockets (non-blocking, IPv4/IPv6), WebSocket client and server, an
  HTTP/1.1 client with TLS via an mbedTLS binding and a bundled CA store (M6).
- **High level (M7, required for the first sellable release):** deterministic-simulation-friendly multiplayer with entity replication (snapshot + delta,
  interest management), typed remote calls generated from Ada specs with contracts on the payload, an
  authority/ownership model, reliable/unreliable/ordered channels over ENet (binding) or our own reliable-UDP
  layer (a SPARK candidate), and optional rollback for fixed-step games.
- **Extras:** UPnP (miniupnpc), Steam Networking Sockets backend once Steam is enabled (M8+), WebRTC (libdatachannel,
  M9). Editor support: launch multiple instances for client/server testing, plus a network profiler.

### 6.14 Additional parity subsystems (summary; details in Appendix A)
- **3D rendering additions:** clustered decals, SSIL, froxel volumetric fog with fog volumes, area lights, SSS,
  transmittance, refraction, resolution scaling with FSR1/FSR2, visibility ranges, texture streaming, and VoxelGI-
  and probe-based dynamic GI (M7, pre-launch).
- **3D tools:** primitive meshes, a mesh builder API, GridMap-equivalent 3D tiles, Curve3D paths, CSG (Manifold),
  and glTF export.
- **2D tools:** Curve2D paths, a 2D geometry library (polygon booleans and triangulation), terrain auto-tiling, and a 2D SDF.
- **Windowing/OS:** multiple windows, native file dialogs, borderless/exclusive fullscreen, tray, process
  spawning, and opening URLs.
- **Production:** video playback (Theora or AV1), deterministic movie capture, runtime loading of user content,
  mod/DLC packs, an object/memory profiler, Tracy integration, and shell completion.
- **Signature features (§19):** projective math and editor guides (including anamorphic cameras, portals/mirrors,
  and AR calibration), fractal generators and shader nodes (including Mandelbulb raymarching and infinite zoom),
  and density-field resources (fog, scattering, heat maps, isosurfaces, SPH fluids, destructible voxels, dual
  contouring). These go *beyond* the Godot-parity list and all ship in the first release.

### 6.15 Seeded, replayable simulation, automated players, and batch reports
First requested by Robert's Hood Simulator and Colony4X bots. Draft 0.8 makes it concrete using the four reference
implementations (Appendix B): Reckoning (CDDA In ADA), Roguelike Jungle, Colony4X and Hood Simulator.

**6.15.1 Seeds and randomness (Poncelet standard)**
- Every run has an explicit **world seed** (map, content, world events) and a separate **AI seed** (decisions of
  AI-controlled actors and of the autoplayer). Hood Simulator showed why both are needed: wins on the default map hid
  a stall that only a new map exposed, so a batch varies **both**.
- `Pc.Random` keeps **explicit state** (never a package global; Reckoning's global `S.Seed` is the counter-example)
  and offers **named streams** derived from the two seeds (Roguelike Jungle pattern: `Map`, `Loot`, `Combat`,
  `Weather`, `Align`, …). A game declares its streams as an enumeration. Streams are seeded with
  `Derive (Seed, Stream_Tag)` (SplitMix32 finaliser), so adding a draw in one system never shifts another.
- **Default generator for new games: PCG32** (64-bit state, 32-bit output). **Compatibility generators**, bit-exact
  with the reference games, exist only for seed-parity ports: `Mulberry32` (Colony4X, Hood Simulator),
  `Xorshift32_Star` (Roguelike Jungle) and `Lcg_1664525` (Reckoning: `V := V * 1664525 + 1013904223`, state `V / 2`,
  result `(V / 65536) mod N`).
- Integer helpers only in core: `Below (N)` = `(U64 (Next) * N) >> 32` (no modulo bias, and bit-identical to the JS
  idiom `Math.floor (next() * N)` for N < 2**21), `Chance (P_Raw)` compares against a 32-bit threshold, and
  `Weighted`, `Shuffle` (Fisher–Yates) and `Bell` (mean of three) follow the Hood Simulator `rng.ts` semantics in
  integer form. Contracts: `Pre => N > 0, Post => V < N`.
- Simulation code has no floats where results must match across targets: binary **fixed-point** types
  (Roguelike Jungle `RJ.Fixed`: `delta 2.0**(-4)` money, `2.0**(-16)` probability, 32-bit storage) are the pattern
  for `Pc.Fixed` game modes (§6.1). No wall-clock time, no unordered iteration, no `Math.random` equivalent.

**6.15.2 Clock, commands and time cost**
- The simulation advances only through `pc_tick (n)` and `pc_act (...)` (§10.5). A project chooses its **tick unit**:
  a fixed frame (default 1/60 s), a game minute (Roguelike Jungle: 1 tick = 1 game minute, 1× speed = 10 ticks/s
  of real time), or a turn (Reckoning, Colony4X). Hood Simulator's weekly turn with minute ticks inside maps to
  "turn = N ticks".
- **Chunking must not matter:** ticking 1+7+80+3+640+33 times must give the same hash as one call with the sum.
  This is a CI test (Roguelike Jungle `determinism.sh` feeds WASM uneven chunks against a native run).
- **One command entry point** for the human, NPCs, rival AIs and the autoplayer. Every command has a **time cost**.
  While an actor is busy, its commands are rejected with `Busy`. A **failed or blocked command still costs time**
  where the player's attempt would (Reckoning spends a turn on a failed try; a bump costs 2–3 extra turns).
  The result is a numeric status (Roguelike Jungle `Result` enum; Hood Simulator `ok/fail` with a reason id), never
  text.

**6.15.3 State hash, saves and replays**
- `Pc.Hash`: 64-bit FNV-1a over a **canonical serialisation** of the deterministic state (fixed field order,
  little-endian), exported as `pc_hash_lo/hi`. Games add fields through `Hash_Field` registrations. Roguelike
  Jungle's per-subsystem hashes (`Map_Hash`, `Region_Hash`) become **sub-hashes** so desync reports name the system.
- **Two save forms:** (a) the **replay save** = seeds + engine/game version + action log (Colony4X: "saves are
  logs", `replay (log, upto_turn)`); (b) the **snapshot save** = a versioned `Pc.Serial` blob with magic, format
  version and CRC32, written into a dedicated host buffer (`pc_buffer_ptr/len (Save)`), never a raw record address.
  A loader that sees an unknown version refuses cleanly (Reckoning `w_valid`; Colony4X ignores older autosaves).
- `.preplay` = canonical JSON header (`schema`, stamp, seeds, test key, tick unit) + one line per command
  `tick actor verb a1 a2 a3` + periodic `#hash tick lo hi` lines. Playback verifies every hash and reports the
  **first divergent tick and sub-hash**. In-memory replay logs are capped (Reckoning: 50 000 inputs) and the cap is
  reported, not silent.

**6.15.4 Automated players (rules distilled from 80+ Hood Simulator versions and Reckoning)**
1. The autoplayer **is a player**: it uses only the public command API and only information a human could know.
   Report-only data is never read by the AI. Rival AIs reuse the same module.
2. **Coverage:** every player action must be reachable by the autoplayer (launch, craft, confirm a dangerous move,
   fly home, …), or batches never find the bug. A coverage counter per verb is part of every batch summary.
3. **One currency:** decisions compare a single fitness value (toward the chosen goal minus costs, per actor-hour).
4. **Estimates use the same path code as the simulation** (straight-line estimates were 2× off on roads).
5. Every "set aside" rule has a way back **the AI can observe** (a target parked "until seen elsewhere" stalled one
   game for 77 weeks).
6. **Stuck detection:** if the clock does not advance for 50 autoplayer calls, count `stuck`, force a wait and go on
   (Reckoning `auto_run`). `auto_step` returning `< 0` means "stop".
7. Optional **decision matrix** per period and order type (proposed, given, held, done, ran out, value, cost terms,
   net, approved vs. spent, top hold reason) and an optional per-order trace (travel/execute/wait minutes,
   re-routes, end state). A fix names the matrix cell it should move and is proven only when that cell moves.

**6.15.5 Batch simulation (`poncelet sim`)**
- `poncelet sim <project> --games 200 --ticks 14400 --world-seeds 1..200 --ai-seeds 1..200 --autoplayer default
  [--scenario stocked] --jobs 8 --out reports/` runs headless in parallel processes.
- **Fair lineups:** batches rotate factions, start slots, professions and start locations from the seed
  (Colony4X `fairLineup`; Reckoning `profession = seed mod 5`, `location = (seed / 5) mod 3`), or one faction
  skews the result.
- Output: one **game report** per run (the F9 format, §6.15.6), a CSV row per run with the declared metrics, and a
  **batch summary** in Markdown + JSON: runs, errors, stuck, survival/length mean/median/min/max, end causes with
  counts, win types and max win share, per-group breakdowns (faction, profession, personality), ms per tick and
  worst AI step, verb coverage, and desynced runs with their `.preplay` attached.
- **`poncelet sim --compare old/ new/`** prints the lean regression table used by every reference bot after a
  change: errors, stuck, mean/median length, causes of death or win types, and metric deltas.
- **Long runs write periodic report files** (Hood Simulator: every 52 game weeks into a user-picked folder) instead
  of being stopped for debugging. The compact text form starts with `BUILD <stamp>` and `TEST KEY <stamp> | world
  seed W | ai seed A | <mode> | <options>`, then `SUMMARY …` lines.

**6.15.6 The Poncelet report (F9) format, `poncelet-report` v1**
Reconciled from Reckoning's `debugReport()`, Roguelike Jungle's `rj-feedback` v1, Colony4X's `colony4x-log` and
Hood Simulator's report files. One schema for the in-game F9 key (rebindable; also a visible Debug button on touch),
the MCP tool `debug_report`, crash reports and batch runs.
- **Header:** `schema {name: "poncelet-report", version: 1}`, `stamp` (§10.6), engine version, game id/version,
  `reason` (`f9`, `crash`, `batch`, `contract`), `created_at` (UTC), seeds, `test_key`, tick/turn/day.
- **Player note:** `expected` (required for a manual F9 report: "what did you expect?"), so a report always states
  the bug.
- **Reproduction:** the action log (`replay`), the state hash and sub-hashes, scenario/start options.
- **State:** registered stats (`Report_Metric`), per-entity tables (units, creatures) as numeric fields, view or
  camera snapshot, `turn_summaries` / timeline, end cause.
- **Events:** the last N events with `dropped` count; host log lines with the same dropped count.
- **Errors:** contract failures and last-chance reports (exception, file, line), console errors.
- **Environment:** platform, user agent, screen and DPR, standalone/PWA flag, language, settings, audio state
  (unlocked, bus volumes, context), performance (ms/tick, worst frame).
- **Screenshot:** optional PNG with the stamp drawn in (Roguelike Jungle), so even a picture names its build.
- **Opening F9 pauses the game.** Outputs: JSON (canonical) and Markdown (note, view, screenshot, JSON block). File
  name `<game>-report-<version>-s<world seed>-t<tick>.json`. Delivery: share sheet where supported, else download
  (WebKit on `file://` cannot keep writing a file, so no live file streaming there). Collection starts at launch,
  so early errors are included.

**6.15.7 Replay tests and milestones**
- Golden replays in `tests/replays/` run in CI and through `poncelet test --replays`; each template ships at least
  one. **Native-vs-WASM parity:** the same seeds give the same hash natively and in WASM (Roguelike Jungle: 20 seeds ×
  43 200 ticks).
- **Seed parity for ports** (§13.4): a port of a reference game must reproduce the reference's observable trace for
  a fixed seed set.
- **Milestones:** seeds and the headless hash in M0; `Pc.Random` (+ compatibility generators), `Pc.Fixed` and
  `Pc.Hash` in M1; replay record/playback, autoplayer interface, `poncelet sim` + `--compare`, and the F9 report in
  M3; CI cross-platform and native-vs-WASM determinism in M4; multiplayer desync detection reuses the hashes in M7.

## 7. Third-party components (all compatible with a closed, commercial product)

| Component | Use | Licence | Notes |
|---|---|---|---|
| GNAT (FSF GCC) 12.3+ / Alire toolchains | compiler | GPLv3 **with GCC Runtime Library Exception** | Fine for closed-source games. **Do not** use the old GNAT Community 2021 runtime (pure GPL). |
| SDL3 | window, input, audio, GPU | zlib | |
| SDL_shadercross | offline shader conversion | zlib | build-time only |
| FreeType | font rasterising | FTL (BSD-like, credit required) or GPLv2 | choose FTL; add the credit |
| HarfBuzz | text shaping | Old MIT | |
| stb_image / stb_vorbis | decoding | public domain / MIT | |
| Noto Sans, Noto Sans SC | fonts | SIL OFL 1.1 | embedding OK |
| Steamworks SDK | Steam | Valve SDK licence | **Must not be committed to the public repo.** Fetched locally/CI secret. |
| Jolt Physics | 3D physics backend | MIT | via our C shim |
| cgltf | glTF 2.0 import | MIT | editor/importer |
| meshoptimizer | LODs, vertex cache | MIT | importer |
| MikkTSpace | tangents | zlib | importer |
| KTX-Software / Basis Universal | texture compression | Apache-2.0 | importer + runtime transcode |
| xatlas | lightmap UV unwrap | MIT | editor baker |
| Recast/Detour | navmesh baking | zlib | optional |
| msdfgen | MSDF fonts | MIT | |
| ufbx | FBX import | MIT / public domain | importer |
| mbedTLS | TLS for HTTP/WebSocket | Apache-2.0 | M6 |
| ENet / miniupnpc | multiplayer transport / UPnP | MIT / BSD | M7 |
| AccessKit (C API) | screen-reader accessibility | MIT / Apache-2.0 | M7 |
| Manifold | CSG | Apache-2.0 | M5 |
| Tracy (client) | profiling | BSD-3 | dev builds |
| dr_mp3, tinyexr, libwebp | decoding | public domain / BSD | |
| V-HACD / CoACD | convex decomposition | BSD / MIT | editor |
| libtheora or dav1d | video | BSD | M6 |
| Dear ImGui + cimgui | internal debug overlays | MIT | not shipped in the editor UI or games |

## 8. Localisation design

### 8.1 Languages and IDs

| Language | Our locale tag | Steam API language | Notes |
|---|---|---|---|
| English | `en` (source) | `english` | fallback for everything |
| German | `de` | `german` | ~30% text expansion; decimal comma |
| French | `fr` | `french` | narrow no-break space before `: ; ! ?`; plural: 0 and 1 are singular |
| Russian | `ru` | `russian` | 3 plural forms (one/few/many) + other; Cyrillic glyphs |
| Simplified Chinese | `zh-Hans` | `schinese` | CJK font, no word spaces, UAX #14 line breaking, IME |
| Spanish | `es` | `spanish` (Spain); `latam` exists separately | open question Q6: Spain vs Latin American Spanish |
| Italian | `it` | `italian` | decimal comma |

Steam reports the user's language via `SteamApps()->GetCurrentGameLanguage()`. We map it to our tag and fall
back to the OS locale (SDL3 `SDL_GetPreferredLocales`) when Steam isn't running, and then to `en`.

### 8.2 String IDs
- Hierarchical English identifiers: `menu.main.start_game`, `hud.ammo_count`, `item.sword.name`.
  Identifiers are always ASCII English (Robert's requirement). Values are UTF-8.
- At build time, a generator emits an Ada enumeration `String_Id` from the master catalog. **A typo in an ID
  becomes a compile error**, and lookups are array indexing with no hash maps. This is also SPARK-friendly
  (`Lookup` is total, with `Post => Lookup'Result'Length > 0`).
- Placeholders are named, not positional: `"{count} coins"`. The formatter checks at build time that every
  translation uses the same placeholder set as English.

### 8.3 Catalog format
- **Master:** one UTF-8 CSV/TSV per domain (`ui`, `items`, `dialogue`) with columns
  `id, context, max_len, en, de, fr, ru, zh_Hans, es, it`. It's easy to hand to translators and diff in git.
- **Interchange:** export and import gettext `.po` per language for professional translators and CAT tools.
- **Runtime:** compiled to a binary `.plang` per locale (offset table + UTF-8 blob) inside the data pack.
  The game loads only the active locale.
- Plurals: CLDR plural rules implemented in Ada per language (en/de/es/it: one/other; fr: one for 0–1/other;
  ru: one/few/many/other; zh: other only). Keys like `inventory.items#one`, `#few`, ...
- Build checks: missing translations (warn, fall back to `en`), placeholder mismatch (error), and over-length
  strings against `max_len` and real measured text width per font (warn).
- A pseudo-locale `xx` (accented, +40% length) is used for testing UI layout.

### 8.4 Fonts and text
- Latin + Cyrillic: Noto Sans (or a licensed brand font). It covers all of `de fr es it ru`.
- Chinese: Noto Sans SC is very large (tens of MB across weights). We **subset at build time** to the glyphs
  used in the `zh-Hans` catalog plus GB2312 level 1 (~3,700 common hanzi) for player names and IME input.
- Glyph atlases are rendered at runtime with FreeType into GPU textures, with SDF for scalable UI text.
  Shaping uses HarfBuzz. Font fallback chain per locale: `zh-Hans → [Noto SC, Noto Sans]`.
- Number and date formatting per locale lives in `Pc.I18n.Format`, a small CLDR subset (decimal and grouping
  separators, ordering).
- **Right-to-left (BiDi) text: documented later milestone (M9).** None of our seven languages needs it, but games may
  add Arabic, Hebrew, or Persian. Plan: the Unicode BiDi algorithm via SheenBidi or FriBidi (or our own Ada
  implementation, a SPARK candidate), HarfBuzz RTL shaping, cursor and selection in mixed-direction text, and automatic
  UI mirroring (containers flip, icons with a "mirror" flag). **Preparation from M2:** text is stored in logical order,
  layout uses start/end instead of left/right, and no code assumes left-to-right.

## 9. SPARK-readiness coding rules (Plain Ada now, SPARK later)

These rules apply to everything in `pc-core` and are recommended in game code.

1. **Contracts everywhere:** `Pre`, `Post`, `Type_Invariant` (on private types), `Static_Predicate` and
   `Dynamic_Predicate`, and **ranged subtypes** for every quantity (`subtype Health is Integer range 0 .. 9_999`).
   Use `pragma Assert` and `Loop_Invariant`-style asserts in loops (later these become `pragma Loop_Invariant`).
2. **Assertion policy:** `pragma Assertion_Policy (Check)` in debug and test builds. In release, runtime checks
   stay on except in measured hot spots (decided per package after profiling).
3. **No access types in `pc-core`.** Use handles and bounded arrays. If unavoidable, use only the SPARK
   ownership-compatible patterns (single owner, no aliasing).
4. **No exception handlers in core logic.** Errors are status results (`type Result (Ok : Boolean) is record ...`).
   `raise` is allowed only for "impossible" states, which are later proven unreachable.
5. **Functions have no side effects.** State changes go through procedures with `out`/`in out` parameters.
   Avoid hidden global state. Package-level state is encapsulated, ready for `Abstract_State`/`Global`.
6. **No controlled types, no `'Access`/`'Unchecked_Access`, no `Unchecked_Conversion` in core.** Fixed-capacity
   containers come from our own generics or SPARKlib formal containers.
7. **Tagged types only when needed**, and then they obey Liskov rules (SPARK checks `Pre'Class`/`Post'Class`).
   Prefer variant records and generics.
8. **Every C binding sits in a package with `SPARK_Mode => Off`**, wrapped by an Ada spec whose contracts
   restate the C function's preconditions.
9. **Deterministic RNG** (`Pc.Random`: PCG32 default, named streams, bit-exact compatibility generators for ports;
   §6.15.1) with explicit state passed in, not a global. Array aggregates use `[...]` under `-gnat2022`, so code is
   warning-free on GNAT 12.3 and 14.
10. **Tooling:** compile with `-gnat2022 -gnata -gnatwa -gnatVa -gnatyy`, use AUnit or Alire-based tests, and
    run **GNATprove** in CI at `--mode=check` (flow) from M2 onward, then `--level=2` proof on the first
    candidate packages: `Pc.Math`, `Pc.Serial`, `Pc.I18n.Plural`, and `Pc.Physics.Broadphase_2D`.
11. Mark packages with `pragma SPARK_Mode (On)` as soon as they pass flow analysis. The goal is that
    "SPARK later" means adding `Global`, `Depends`, and loop invariants, not refactoring.
12. **Portability restrictions for `pc-core` = the web-safe Ada subset** (§10.5, enforced by a configuration-pragmas
    file `core.adc`): no tasking, no allocators or heap containers, no exception handlers, no Text_IO or direct OS/file
    access, and fixed-size data only. This keeps the core
    usable on WebAssembly, iOS, consoles, and restricted Ada runtimes. Threads, files, and time come from services.

## 10. Build, packaging, and Steam

### 10.0 Development policy (decided 2026-11-19)
- **Primary development and testing happen on Robert's Linux Mint laptop (x86-64)** with GNAT 12.3 + gprbuild
  (Alire optional). Every feature must run and pass tests there first.
- **No sign-ups for now:** no Steamworks partner account and no Apple Developer ID until Poncelet is stable and
  mature. macOS and Windows remain **essential targets**, delivered later.
- **Keep the platform layer portable from day one:** no Linux-only APIs outside `pc-platform`, and only
  cross-platform C dependencies (SDL3, FreeType, HarfBuzz, Jolt). OS-specific code is isolated in
  `pc-platform/<os>/` bodies selected by gprbuild scenario.
- **CI on free GitHub Actions runners:** Linux builds and full tests. **Windows (`windows-latest`) and macOS
  (`macos-latest`, arm64) compile and smoke-test only** (build, run headless core tests, start the runtime with a
  null/offscreen window if possible). There is no signing, notarisation, or packaging for distribution. A red
  Windows/macOS build blocks merging, just like Linux.
- **Steamworks stays behind a compile-time stub** (`STEAM=stub`, the default) until the late Steam milestone.
  Nothing from the Steamworks SDK is needed to build.

### 10.1 Toolchain
- **Alire** manages GNAT toolchains per OS (`gnat_native` for Linux x86-64, Windows x86-64 (MinGW), and
  macOS arm64/x86-64) plus `gprbuild`. Robert's local GNAT 12.3 + gprbuild on Linux Mint is the reference development toolchain. CI uses Alire for Windows/macOS.
  Pin the toolchain version in `alire.toml`.
- Project files: `pc_core.gpr` (library, no deps), `pc_platform.gpr`, `pc_runtime.gpr`, `pc_editor.gpr`,
  and a per-game `game.gpr`. Scenario variables: `BUILD=debug|release|prove`, `TARGET_OS`.
- C dependencies are built with CMake or downloaded as prebuilt SDKs in CI and linked via gprbuild
  `Linker_Options`.

### 10.2 Per-OS packaging (target state; signing and Steam parts deferred to milestone M8, see §16)

| OS | Build host | Output | Steam specifics |
|---|---|---|---|
| Windows | GitHub Actions `windows-latest` (MinGW GNAT from Alire) | `Game.exe` + `SDL3.dll` + `steam_api64.dll` + `game.ppak` | optional Authenticode signing |
| macOS | `macos-latest` (arm64) + x86-64 runner | `Game.app` **universal binary** (`lipo`), libs bundled under `Contents/Frameworks` with rpath | **Code signing + notarisation** (Apple Developer ID, about US$99/yr) to avoid Gatekeeper warnings; `libsteam_api.dylib` |
| Linux | `ubuntu` inside the **Steam Runtime 3 "sniper" SDK** container | `game.x86_64` + `libSDL3.so` + `libsteam_api.so` (rpath `$ORIGIN`) | Target the Steam Linux Runtime so it runs on any distro (and on Deck) |

- Depots: one depot per OS plus a shared data depot. Upload with SteamPipe (`steamcmd` + app build VDF) from CI
  using a **dedicated build account**.
- Store-required assets (capsules and so on) are localised for the seven languages. Steam store language
  checkboxes are interface/subtitles only unless voice is added.

### 10.3 Steamworks integration
- **Now:** `Pc.Services.Steam` is defined as an Ada spec with contracts. The only body built by default is a
  **compile-time stub** (gprbuild scenario `STEAM=stub`): every call is a no-op, language comes from the OS locale,
  and achievements are logged locally. Game and editor code call the service as if Steam existed.
- **Later (M7, after maturity and a partner account):** add a `STEAM=real` body that binds the **flat C API**
  (`steam_api_flat.h`) with a thin Ada binding in `pc-platform`. No game code changes. It degrades gracefully: with no Steam, all calls are no-ops, so the same build
  runs from itch.io or for development.
- Features by milestone: init/restart-check, overlay, language, achievements, stats, Cloud saves,
  rich presence (all in M7), workshop for template and mod sharing (later).

### 10.4 Web, mobile, console, and XR export

**Web (Ada → WebAssembly).** Two of Robert's bots have proven Ada-to-WASM experience: **Roguelike Jungle** and
**CDDA In ADA** ("Reckoning", which also runs on iPhone in Safari). See §10.5 for the resulting decisions.
Hood Simulator and Colony4X are JavaScript games. **Toolchain details are refined in the group chat, based on
these two setups.** Baseline plan:
- GNAT-LLVM targeting `wasm32` (Emscripten or a WASI-based setup, whichever those bots have proven), with a
  restricted Ada runtime. `pc-core` already follows the portability rules (§9 rule 12).
- Rendering: compatibility renderer on WebGL2 (M4 2D, M6 3D). WebGPU via the mobile tier later (M9).
- Audio: WebAudio with an AudioWorklet feeding the Ada mixer. Input: DOM keyboard, mouse, touch, and Gamepad API
  events. Saves: IndexedDB. Data: `.ppak` streamed via fetch, with progress UI.
- Threads: single-threaded by default. Optional SharedArrayBuffer threads when the host sends COOP/COEP headers.
- Hosting: static files, so exports can be demoed through githack links or any static host.

**Android and iOS.**
- Toolchains: GNAT cross (aarch64-linux-android) or GNAT-LLVM with the NDK for Android, and GNAT-LLVM for
  arm64-apple-ios. Both are validated by CI spikes early (M1/M6). SDL3 provides windowing, touch, and lifecycle on both.
- App lifecycle: pause/resume/background as engine events, save on suspend, GPU context-loss handling, and
  orientation and safe areas.
- Packaging: Android APK/AAB via Gradle templates. iOS Xcode project export, built on macOS.
- **Gated:** Play Store signing and Apple App Store submission come in M8 after the sign-up gate, like Steam.
  Unsigned debug APKs can run on an Android phone or emulator earlier (M6).

**Consoles (third-party partners).** As with Godot, console ports are done by licensed partners under
platform NDAs, in the style of W4 Games. Platform SDKs never enter this public repo. To stay console-friendly from M0:
- static linking, and no runtime code loading in shipped builds (hot reload is editor-only);
- all OS access through services, with fixed memory budgets and fixed-capacity containers in the core;
- controller-first UI navigation (focus graph) and suspend/resume lifecycle events;
- save data, achievements/trophies, and user accounts through a generic `Pc.Services.Platform` (Steam is one backend);
- deterministic, assertion-checked code that helps certification.
- Risk: partners need a GNAT/GNAT-LLVM toolchain for each console target. That's discussed with partners post-launch (M9).

**XR (OpenXR).**
- Pre-launch preparation (M7): render-graph support for multiple views (stereo/multiview), XR camera components,
  and an action-system mapping.
- OpenXR integration post-launch (M9): PC VR via the Vulkan path, standalone Android headsets via the mobile tier,
  hand tracking, passthrough AR where the runtime supports it. WebXR follows WebGPU.

### 10.5 Web-safe Ada subset and the numeric host ABI (decided 2026-11-19, made concrete in draft 0.8)
Reconciled from the two proven Ada-to-WASM games (details and sources in Appendix B):
- **Reckoning (CDDA In ADA) v0.54:** AdaWebPack GNAT-LLVM `wasm32`, `-gnatp -O2`,
  `wasm-ld --no-entry --export-dynamic --allow-undefined --strip-all`, base64-inlined into one HTML file. About 45
  integer-only exports (`w_new`, `act`, `view`, `ev_count/ev_code/ev_arg`, `w_stat`, `w_state_ptr/len`, `auto_act`,
  …), a 256-entry numeric event queue, a raw state blob for saves, imports via a JS `Proxy` that returns 0.
- **Roguelike Jungle v0.04:** AdaWebPack 24, `-O2 -gnat2022 -gnatwa`, **checks on**, a last-chance handler that
  calls the import `rj_abort (line)`, an **explicit export list**, `-z stack-size=1048576
  --initial-memory=16777216`, `memcpy/memmove/memset` linked from `libgnat.a`, a tick clock, an `Order` command with
  a result enum, a 32-bit state hash, and a native-vs-WASM determinism test.

**Poncelet decisions:**
1. **Web-safe Ada subset for `pc-core`** (unchanged): no tasking, no heap, no exception handlers, no Text_IO or other
   OS units, fixed-size arrays and records only, enforced by `core.adc` (§9 rule 12). Native runners and tools (the
   `poncelet sim` driver, Reckoning's `auto_run`) may use Text_IO and handlers because they are not the core.
2. **Runtime checks policy** (unchanged from 0.7.1, now with the evidence): dev builds on all targets keep checks
   (`-gnata`, no `-gnatp`) plus the last-chance handler. Reckoning's `-gnatp` let an out-of-range index write into
   the wrong cell and surface turns later. Release builds may suppress checks **only** for SPARK Silver units.
3. **`Pc.Host_ABI` v1 (exported C convention, scalars only):**

   | Export | Meaning | Source pattern |
   |---|---|---|
   | `pc_abi_version` | ABI version (host refuses a mismatch) | RJ `rj_ver_major/minor` |
   | `pc_init (world_seed, ai_seed)` | new world | `w_new`, `rj_init` |
   | `pc_load (kind) → status` | validate + load the bytes the host wrote into buffer `kind` | `w_valid` |
   | `pc_tick (n)` | advance n ticks (chunk-independent) | `rj_tick` |
   | `pc_act (actor, verb, a1, a2, a3) → status` | one command; status ≥ 0 = time cost, < 0 = result code | `act`, `rj_order` + `rj_last` |
   | `pc_auto_step (actor) → status` | one autoplayer step; < 0 = stop | `auto_act` |
   | `pc_busy (actor)`, `pc_clock (field)` | busy flag; tick/day/minute/turn | `rj_busy`, `rj_day` |
   | `pc_read_stat (id)`, `pc_read (table, index, field)` | game stats and per-entity fields | `w_stat`, `unit_stat`, `mon_stat`, `rj_actor_*` |
   | `pc_query (kind, a1, a2, a3)` | small queries: look/examine layers, danger of a move, routes | `look_l`, `danger`, `rj_find_route` |
   | `pc_event_count`, `pc_event_read (i, field)`, `pc_event_dropped`, `pc_event_clear` | event queue (below) | `ev_*` |
   | `pc_hash_lo/hi`, `pc_subhash (system)` | state hash | `rj_hash` |
   | `pc_buffer_ptr (kind)`, `pc_buffer_len (kind)`, `pc_buffer_cap (kind)` | linear-memory buffers | `w_state_ptr/len` |
   | `pc_view_set (w, h)`, `pc_view_sync` | viewport size and snapshot refresh | `set_view`, `view_sync` |
   | `pc_audio_ctx (bus)` | context id the host maps to music/ambience | `music_ctx` |

   Buffer kinds: `Save` (Pc.Serial snapshot), `Draw` (render snapshot / draw list), `Text` (UTF-8 blob, optional),
   `Report` (numeric report block). Many tiny getters are fine for a 48×48 tile view (Roguelike Jungle); bigger
   scenes use the `Draw` buffer. The packed cell format of the 2D tile view is `glyph (bits 0..15) | colour << 16
   | visible << 24 | seen << 25` and a "lit" layer is `class << 16 | id` (Reckoning); 3D and sprite scenes use the
   draw-list layout of §6.2.
4. **Event queue:** fixed-capacity ring (default 1024) of integer records `{sequence, tick, code, actor, a1, a2, a3}`
   with a `dropped` counter (Reckoning silently stopped at 256). It is filled during `pc_act`/`pc_tick` and drained by
   the host after each call (`pc_event_clear`). Codes are a game enumeration registered with names, so the F9 report
   and the editor can print them; Hood Simulator's `PresentationEvent {sequence, week, minute, kind, actorId, pos}`
   and Colony4X's `logEv (kind, data)` fit the same record. **Strings never cross the ABI.** The host maps codes to
   text (via `Pc.Text` catalogs, §8), SFX and voice in one place.
5. **Imports:** the real imports are `pc_lch (file_ptr, line)` (last-chance handler; the host reads the NUL-terminated
   file name from memory and shows exception, file and line), `pc_log (code, a1, a2)` and `pc_now_ms` (UI only, never
   simulation). `memcpy/memmove/memset` come from the Ada runtime objects, not from the host. The `Proxy` default of 0
   stays as a **startup guard**, and dev builds **log every unknown import name** so a typo is not hidden.
6. **Elaboration:** if the module exports `*___elab[bs]` functions, the host calls them in order after instantiate
   (Reckoning). Poncelet's own core avoids elaboration code where possible (`pragma Preelaborate`).
7. **No `Unchecked_Conversion` on a record address.** Save and render data are produced into the buffers by
   `Pc.Serial` / the draw-list writer (Reckoning's `W_State_Ptr` breaks §9 rule 6 and has no schema).
8. **Each target supplies renderer, input, audio and saves** behind the services (SDL3 + SDL_GPU on desktop/mobile;
   WebGL2 first and WebGPU later on the web; WebAudio; DOM/Gamepad input; IndexedDB with a localStorage fallback).
   The web host standard is §10.6.
9. **Build recipe (web):**
   - Compiler: AdaWebPack 24 (GNAT-LLVM, LLVM 16) for `--target=wasm32`, flags `-O2 -gnat2022 -gnatwa -gnata` and
     `--RTS=<adawebpack>/rts-native`. Release adds per-unit suppression for proven units only.
   - Linker: `wasm-ld --no-entry --strip-all --allow-undefined --export=<list>` with the export list **generated from
     the `Pc.Host_ABI` spec** (smaller and reviewable; `--export-dynamic` only in a debug variant),
     `-z stack-size=1048576 --initial-memory=16777216`, plus `s-memcop.o s-memmov.o s-memset.o` from `libgnat.a`.
   - The AdaWebPack compiler itself needs `libgnat-14.so`, `libLLVM-16.so` and `libclang-cpp.so.16`.
     `tools/get-wasm-toolchain.sh` installs them **into the project tree without sudo** (the Roguelike Jungle recipe,
     with the GNAT 14 runtime taken from the GNAT-FSF-builds tarball instead of Alire), and `wasm-ld` comes from
     `lld` or the pip package `ziglang`.
   - **Native binaries link the GNAT runtime statically** (`-static` binder switch). Roguelike Jungle's runner failed
     on a second machine with "libgnat-14.so not found"; the same sources build and run with GNAT 12.2 when linked
     against that compiler's runtime.
   - Code must compile **warning-free on GNAT 12.3 (Robert's laptop) and GNAT 14**: under `-gnat2022`, array
     aggregates use `[...]` (GNAT 12 warns on `(...)`, seen in Roguelike Jungle).
10. **Packaging and deployment:** the default export is one self-contained HTML file (base64 wasm + JS host + small
    data pack). Music and large assets stay beside it and load lazily with a progress bar (Reckoning keeps MP3s next
    to the HTML); large games use a streamed `.ppak`. Stamp, `version.txt` and publishing follow §10.6.

### 10.6 Web host standard (from the reference implementations)
The reference web host (`hosts/web/`, M4) and every exported web game follow these rules. Native hosts follow the
same rules where they apply (stamp, F9, confirms, audio buses).
1. **Boot:** decode the inlined wasm, instantiate with the real imports wrapped in the dev `Proxy`, run elaboration,
   check `pc_abi_version`, call `pc_init`. On a last-chance report, stop the loop and show an overlay with exception,
   file, line and stamp, and keep F9 working.
2. **Build stamp on every screen:** `vMAJOR.MINOR bYYYYMMDD-HHMM <short-sha>`, with a trailing `+` when the tree was
   dirty. It sits above every overlay (splash, menu, in-game, F9, screenshots). Every push bumps the version; a stamp is
   never reused. The same string is in `version.txt` beside the HTML, and in every report (`BUILD` line / `stamp`).
   Dev builds show a bright badge; public builds may dim it but keep the string.
3. **Update check:** the page polls `version.txt` from a **raw, uncached origin** (`gist.githubusercontent.com/
   <user>/<gist>/raw/version.txt` or `raw.githubusercontent.com/<user>/<repo>/<branch>/…/version.txt`), **never from
   githack**, compares the `b` timestamp, and offers a reload banner.
4. **Publish targets** (both supported by `poncelet publish web`):
   - **Gist:** `git clone` the gist (binaries cannot be pushed with `gh gist edit`), copy the HTML + `version.txt` +
     assets, commit, push. Play URL `https://gist.githack.com/<user>/<gist>/raw/<file>.html?v=<N>`.
   - **Repository build branch** (Hood Simulator): commit the build to `build/<version>`, play URL
     `https://raw.githack.com/<user>/<repo>/build/<version>/play/index.html`. Commit source, then build, then push.
   - **Always curl-verify before announcing:** fetch the raw URL and the githack URL and grep for the new stamp.
     githack can serve a stale file for 30 seconds to several minutes even with `?v=N`. If it is stale, print the
     **commit-pinned** URL (`…/raw/<commit>/<file>.html` or `rawcdn.githack.com/…/<commit>/…`).
5. **Audio:** four buses with separate gain (**SFX, Music, Ambient, Voice**), plus a master. Unlock on the **first
   `pointerdown`/`touchstart` anywhere** (create/resume the `AudioContext` and play a one-sample buffer; start every
   `<audio>` element once muted for iOS), and keep a silent keep-alive for the iOS silent switch. On `visibilitychange`
   mute Ambient (and Voice); Music may continue. The core picks a **context id** (`pc_audio_ctx`; Reckoning: 0 none,
   1 surface, 2 bunker calm, 3 bunker active, 4 tension, 5 reflection) and the host crossfades. Voice plays one line
   at a time, and the **newest waiting line wins** (no queue dump). A 0–100 link-quality stat may drive a radio effect
   (≥70 clear, ≥35 light static, ≥15 dropouts, below that mostly noise). Generated WebAudio ambience driven by game
   state (time of day, rain, height) is allowed; browser SpeechSynthesis is a placeholder only.
6. **Input:** arrows, vi keys and numpad, matching both `key` and `code` (NumLock differs by browser); touch,
   wheel/pinch zoom; virtual controls on phones. Every gameplay key has a visible on-screen equivalent.
7. **Confirms and warnings:** play actions use an **in-game confirm** (Enter/Y confirm, Esc/N cancel), never
   `window.confirm`. Warn **only when the warning is real** (a deterministic lethal step, a link about to fail);
   warning spam teaches players to click through. The core supplies the reason (`pc_query (Danger, dx, dy)`).
8. **Saves:** IndexedDB (localStorage fallback) holding the `Save` buffer as base64 plus the replay log; autosave;
   an incompatible version is ignored with a notice. Long-run report files go to a user-picked folder where the
   browser allows it (`showDirectoryPicker`), else to downloads.
9. **F9:** the report of §6.15.6. Collection starts at page open; the overlay pauses the game; JSON and Markdown;
   share sheet or download.
10. **Testing:** a headless smoke test at a phone viewport (390×844): no console errors, the stamp equals
    `version.txt`, the core ticks, input and overlays work, screenshot saved. UI tests run in **WebKit, Firefox and
    Chromium** (Colony4X: GNOME Web/WebKit behaves differently, and `display:flex` can override `hidden`, so modal
    visibility is asserted).
11. **One-way dependency:** the simulation never imports host, UI, audio or renderer code (Hood Simulator checks this
    with grep; Poncelet enforces it with project dependencies plus a CI check, §14).

## 11. Editor strategy: the editor is the first product

The first thing sold on Steam is **Poncelet Editor** (Steam category: Software → Game Development), bundled
with template packs. So the editor gets product-level UX, not internal-tool UX.

**Principles**
1. Built on our **own Ada UI toolkit** (§6.10), the same one games use. This dogfoods the toolkit, gives
   one theme and DPI system, and makes everything localisable.
2. **Fully localised in all seven languages**, including menus, inspector property names (via string IDs
   generated from component descriptors), error messages, docs tooltips, and the template browser. The language
   follows Steam/OS and can be switched live.
3. **Discoverable and fast:** command palette, undo/redo for everything (command pattern over the World),
   autosave and crash recovery, and keyboard shortcuts with rebinding.
4. **Onboarding:** first-run wizard → pick a template pack → playable project in under a minute, with interactive
   tutorials authored as data.

**Feature set for the first sellable release (2D complete, 3D complete or early access per milestone)**
- Project manager (create from template, open recent, import template pack, version check).
- Scene editor with **2D viewport** (gizmos, snapping, tilemap painter, 2D light/occluder editing, **perspective
  grids and horizon / vanishing-point guides**) and **3D viewport** (move/rotate/scale gizmos, grid and snapping,
  camera fly/orbit, lighting preview, lightmap baking, **density-field painting and isosurface preview**).
- Scene tree dock, inspector (generated from component descriptors), asset browser with thumbnails, and an import
  dock (glTF, textures, audio, fonts with re-import settings).
- Animation editor (timeline, curves), material editor (parameters first, node graph later), particle editor.
- **Localisation editor:** a spreadsheet view of string IDs × seven languages, missing/overlong highlighting,
  and PO export/import.
- Play-in-editor (separate process for crash isolation) with a live inspector of the running World. Debugger for
  Ada game code via GDB integration (breakpoints) + engine profiler.
- **Code workflow:** game logic is Ada. The editor generates package stubs for new components and systems,
  runs `alr build`/`gprbuild` (bundled GNAT toolchain per OS, licence-checked under the GCC Runtime Library
  Exception) and shows errors clickably. Integrates with external editors (VS Code + Ada Language Server, GNAT Studio).
- **Exporter:** one-click builds for Windows, macOS, and Linux, plus Steamworks toggles (app ID, achievements) for
  the user's game.

**Distribution of the editor on Steam:** Windows, macOS, and Linux depots. Updates go through Steam branches
(stable/beta). The GNAT toolchain ships as a separate depot or optional DLC-like download to keep the base install small.

**Web builds for githack demos:** web export (§10.4) produces static files, so exported games and template demos
can be linked through raw.githack.com from M4 onward. The editor itself stays a desktop application.

## 12. AI-tool integration (core feature)

**Principle:** people will mostly drive Poncelet with **their own AI subscriptions** (Cursor, Claude, ChatGPT, Grok,
GitHub Copilot, and others). Poncelet makes itself maximally legible and controllable for those tools.
**Poncelet pays for no AI, bundles no AI model, and never stores AI API keys.** The editor, CLI, and MCP server are
one Ada **tools API** (`Pc.Tools`) exposed three ways, so the GUI, scripts, and agents can all do the same things.

### 12.1 Text-first project data with published schemas
- Every piece of project data is a documented, deterministic text file:

| Data | File | Format |
|---|---|---|
| Scenes (2D/3D) | `*.pscene` | canonical JSON |
| Resources (materials, physics shapes, animations, themes, tilesets, **density fields**) | `*.pres` | canonical JSON |
| Import settings per asset | `<asset>.pimport` | canonical JSON |
| Project settings, input map | `project.toml` | TOML (stable key order) |
| Templates / template packs | `template.toml`, `pack.toml` | TOML |
| Localisation catalogs | `locale/*.csv` (+ `.po` export) | UTF-8 CSV |
| Data-driven logic (state machines, dialogue, loot tables) | `*.pdata` | canonical JSON |

- **JSON Schema (2020-12)** for every format, versioned and published in `schemas/` (also served by the CLI and MCP).
  Each file carries `"$schema"` and `"format_version"`. Migrations are explicit and covered by tests.
- **Canonical writer:** the editor, CLI, and MCP always re-serialise canonically, so an AI's hand edit and an editor
  edit produce the same bytes. `poncelet fmt` normalises files, and `poncelet validate` checks schemas and references.
- Binary data (textures, meshes, audio) stays as source files (PNG, glTF, OGG). Derived caches live in `.poncelet/cache/`
  (git-ignored).

### 12.2 Headless CLI `poncelet` (from M0/M1)
- Commands: `new` (from template), `build`, `run` (with `--world-seed`/`--ai-seed`), `sim` (batch automated-player
  runs writing report files, §6.15), `replay` (record/verify `.preplay`), `test` (incl. `--replays`), `export`, `screenshot` (render a scene/camera to PNG
  headlessly or offscreen), `validate`, `inspect` (scene tree, entity, component, resource, settings), `fmt`,
  `docs` (API lookup), `schema`, `mcp` (start the MCP server), **`gen`** (seeded fractal noise, L-systems, IFS,
  density fields, and world chunks, §19), and **`projective`** (incidence, cross-ratio, homography, and camera
  queries, JSON in / JSON out).
- **Machine-readable output:** `--json` gives one JSON object (or NDJSON for streams) with a stable, versioned
  envelope `{ "ok", "command", "version", "result", "diagnostics": [...] }`. Exit codes are documented.
- **Precise diagnostics:** every error has `file`, `line`, `column`, `end_line`/`end_column`, `code` (e.g.
  `PC-SCENE-0042`), `severity`, `message` (English, plus a localised `message_l10n`), and a `hint`. GNAT compiler and
  contract-failure messages are parsed into the same structure.
- Deterministic: the same inputs give the same outputs (fixed seeds for `test`/`screenshot`) so agents can compare runs.

### 12.3 Built-in MCP server (with the editor shell, M2)
- Implements the **Model Context Protocol** in Ada, in **both the editor and headless mode** (`poncelet mcp`).
  Transports: **stdio** (for desktop AI clients and IDEs) and **local HTTP** (Streamable HTTP, bound to `127.0.0.1`).
- **Tools** (initial set):
  - Query: `scene_tree`, `get_entity`, `list_components`, `get_component`, `find_entities` (by name, type, or tag),
    `get_resource`, `project_settings`, `list_assets`.
  - Modify: `create_entity`, `delete_entity`, `set_component`, `add_component`, `remove_component`,
    `instantiate_scene`, `save_scene`, `set_translation` (catalog entry).
  - Run: `play` / `stop` play-in-editor, `step_frames`, `send_input` (scripted input), `capture_screenshot`,
    `capture_frames` (short sequence), `read_logs`, `run_tests`, `build` (returns structured diagnostics),
    `run_sim` (batch automated-player runs, §6.15), `debug_report` (F9-style report), `verify_replay`.
  - Signature features (§19): `projective_query` (incidence, cross-ratio, homography, camera),
    `set_perspective_guides`, `generate_fractal` (noise, L-system, IFS, escape-time preview),
    `sample_density`, `paint_density`, `scatter_from_density`, `generate_world` (seeded chunk / terrain).
  - Knowledge: `api_docs` (search the generated Ada API reference including contracts), `schema`, `examples`.
- **Resources:** project files, schemas, logs, and docs exposed as MCP resources. **Prompts:** curated task prompts
  (e.g. "add a component", "localise a scene").
- **Safe by default:**
  - Local only (stdio or loopback HTTP with a per-session random token). No remote binding unless the user
    explicitly enables it in settings.
  - **Project-scoped:** file access is limited to the open project directory. Nothing outside it, and no arbitrary
    shell execution. `build`/`run_tests` invoke only the project's own build.
  - **User confirmation in the editor for destructive operations** (delete entities or files, overwrite scenes,
    export, mass edits above a threshold). Headless mode requires an explicit `--allow-destructive` flag.
  - **Undo integration:** every MCP change is a normal editor command, grouped per tool call, labelled
    "AI: <tool>" in the undo history and fully reversible. There's an activity log panel of all agent actions.
  - Read-only mode switch, and rate limits.

### 12.4 AI-oriented documentation
- Every template and new project ships **`AGENTS.md`** (how to build, test, run, the conventions, where things are,
  "do and don't") and **`llms.txt`** (an index of the docs most relevant to the project) at the project root.
- **API reference generated from the Ada specs** (`.ads`) by our doc tool: signatures, `Pre`/`Post`/`Type_Invariant`
  contracts, ranged subtypes, and doc comments. Output as Markdown + JSON (for `api_docs` / `poncelet docs`), versioned
  per engine release.
- Examples: small, compiling, tested snippets per component and system, extracted from the test suite so they never rot.
- The engine repository itself also has an `AGENTS.md` for contributors' AI tools.

### 12.5 Ada contracts as the feedback loop for AI-written code
- Game code is Ada, so the compiler and contracts give AI tools fast, precise feedback. Type errors, range
  violations, and contract failures arrive as structured diagnostics with locations (§12.2).
- Debug builds run with all assertions on. Contract failures during `run`/`test`/`play` are reported with the
  failing contract, the actual values, and the source location.
- Optional GNATprove flow analysis via `poncelet build --prove` for packages marked `SPARK_Mode`.
- Templates include a test harness (scripted input + golden screenshots) so agents can verify behaviour, not just compilation.

### 12.6 What Poncelet does not do
- No bundled AI model, no AI API calls, no AI accounts, **no storage of AI keys**, and no telemetry of AI usage.
  The user's AI client connects to Poncelet, never the other way round.

## 13. Template system and template pack format (defined early, from M1)

The editor ships with **template packs**. Their contents (which of Robert's games) will be decided later in a group
chat with his other game bots. The **format and import mechanism are defined now** so that work can start in parallel.

### 13.1 Template pack (`.ponpack`)
- A zip container with `pack.toml` at its root (id, version, `engine_min`, localised name and description IDs,
  thumbnail, list of templates, author, licence terms for the pack contents, content hash).
- Contains one or more **templates** (below), shared assets, and per-language catalogs for its own strings.
- Signed: a SHA-256 manifest + Ed25519 signature by Robert's publisher key, so bundled and DLC packs are
  verifiable. Unsigned community packs are allowed with a warning.
- Distribution: bundled in the editor depot, as Steam DLC (paid packs), or via Steam Workshop (later).
- Versioned with migration: the editor can upgrade templates from older `engine_min` versions with scripted migrations.

### 13.2 Templates

- A **template** is a project directory plus a manifest `template.toml`:
  ```toml
  [template]
  id = "arcade_shooter"            # English identifier
  name_id = "template.arcade_shooter.name"   # localised display name
  description_id = "template.arcade_shooter.desc"
  engine_min = "0.4.0"
  genre = ["action", "2d"]
  thumbnail = "thumb.png"
  [contents]
  scenes = ["scenes/main.pscene"]
  code = ["src/"]                  # Ada packages
  assets = ["assets/"]
  locales = ["locale/ui.csv"]
  [provenance]
  original_project = "<repo or bot name>"
  licence = "All rights reserved, Robert Boettcher"
  ```
- **"New project from template"** copies the files, renames the root package and project file
  (`Arcade_Shooter` → user's name), assigns new UUIDs, and registers the project.
- **Importers** for Robert's existing games depend on what they were built with (open question Q2):
  - **Ada games:** move them into the template layout and adapt them to the World API.
  - **Godot games:** a converter for `.tscn`/`.tres` scene structure, sprites, and tilemaps is feasible.
    GDScript logic must be **re-written in Ada** (manually or with bot assistance). It's documented as a port, not an import.
  - **Web (JS/TS) or Python/pygame games:** import assets, levels, and text. Port logic manually.
- Each folded-in game also gets a **parity test** (golden screenshots and a scripted input replay) so that
  engine changes don't break templates. Ports of Robert's reference games additionally pass **seed parity** (§13.4).
- Templates can later be distributed as Steam Workshop items or DLC.

### 13.3 Asset library (curated paid packs + community)
- **Curated paid template packs** (Robert's, signed) ship at launch: bundled with the editor and as Steam DLC (M8).
- **Asset/plugin package format and editor library client** (M7): browse, search, install, and update packages
  (templates, assets, plugins, shaders) from configured sources: the curated index, local folders, or Steam Workshop.
  Each package declares its licence (for example CC0, CC-BY, MIT, or proprietary/paid). The client shows licences
  before install and records attributions in the project's `THIRD_PARTY_NOTICES`.
- **Community asset library service** (M9, post-launch): a hosted index where users publish free or paid assets.
  It needs moderation, malware scanning for plugins, ratings, and a licence policy. Hosting and revenue share are open
  question 13. Steam Workshop is an alternative or complementary channel.
- **Why post-launch:** running a moderated public service is an operations commitment that only makes sense once the
  editor has users. The package format and client come earlier, so templates and plugins already use it.

### 13.4 Reference template packs and seed parity (draft 0.8)
Robert's four bot-built games are the first template content and, more importantly, the acceptance tests for the
engine (sources and contributions in Appendix B).
1. **Reckoning (CDDA In ADA) is the first template pack and the acceptance test for M1–M4.** It is already plain
   Ada on the web-safe subset, so each milestone must run it:
   - **M1:** the Reckoning core builds with the Poncelet web toolchain (checks on, last-chance handler) and runs
     behind a `Pc.Host_ABI` adapter in the WASM spike host; its native runner produces the same numbers.
   - **M3:** Reckoning's autoplayer runs under `poncelet sim` (200 seeds) and the summary matches the shape of its
     own batch (runs, survival days, causes, stuck = 0, errors = 0). Its F9 data fits `poncelet-report` v1.
   - **M4:** Reckoning exports from the editor as one stamped HTML file through the reference web host, publishes
     with curl-verify, and plays on a phone browser (input, audio unlock, saves, F9).
   - **Pack contents** (game, not engine): mapgen, units and AI policies, missions, Venus scenario, splash rain,
     glyph/colour tables, and the event/Look/radio/Help string catalogs moved into `Pc.Text` catalogs.
2. **Later template ports with seed-parity checks.** A port is accepted only when, for a fixed seed set, it
   reproduces the reference game's **observable trace**: the per-turn/day summary metrics and event sequence, and
   for Ada references also the state hash.
   - **Roguelike Jungle** (Ada, M5): the native `rj_sim` and the ported template give identical JSON lines (hash,
     funds, events, day) for 20 seeds × 43 200 ticks, natively and in WASM.
   - **Colony4X** (JavaScript, M7): the Ada port uses `Mulberry32` and integer `Below`. Its trace for seed 7919 after
     40 all-AI turns must match the reference `col-feedback` sample (whose JS replay hash `5fd168f5` was re-verified
     on 2026-11-19), plus a 24-seed `sim.js` batch comparison.
   - **Hood Simulator** (TypeScript + Three.js, M7; needs 3D city, navigation and vehicles from M6): the sim port uses
     `Mulberry32`; parity on the weekly `SUMMARY`/timeline lines for city seed 1 / Auto seed 1 and a seed batch. The
     Three.js renderer and React UI are not ported; they are rebuilt with Poncelet scenes and UI.
   - Where floating-point JS logic cannot be matched bit-for-bit, parity falls back to **statistical parity** over
     a batch (same distributions within stated tolerances), and the port documents each such system.
3. Parity tests live in `templates/<id>/tests/parity/` and run with `poncelet test --parity`.

## 14. Repository layout (planned; nothing created yet)

```
poncelet/                     (public, source-available — see LICENSE)
  alire.toml  poncelet.gpr
  core/        src/pc-*.ads|adb   tests/
  platform/    sdl3/ freetype/ harfbuzz/ steam/  (bindings; Steam SDK NOT committed)
  services/    render2d/ audio/ input/ text/ window/ steam/
  runtime/     main entry, loop
  hosts/       web/ (reference JS host, §10.6)  native/ (SDL3 host)  headless/ (sim + replay driver)
  editor/      imgui-based dev editor
  tools/       catalog compiler, asset packer, string-id generator, template tool, API doc generator
  cli/         `poncelet` headless CLI (JSON output)
  mcp/         MCP server (stdio + local HTTP), shares Pc.Tools with the editor
  schemas/     JSON Schemas for all project file formats (versioned)
  AGENTS.md  llms.txt   (AI-tool guidance for the engine repo)
  templates/   <one dir per template>, e.g. reckoning/ (first pack, §13.4), each with tests/parity/
  locale/      engine UI catalogs (en de fr ru zh_Hans es it)
  docs/        DESIGN.md, NAME_CHECK.md, html pages for githack
  ci/          GitHub Actions workflows per OS
```

## 15. Licensing posture (summary)
- The repository is **public** for githack links, with **all rights reserved / source-available, no reuse**.
  See `LICENSE`.
- Third-party notices (`THIRD_PARTY_NOTICES.md`) list SDL3, FreeType, HarfBuzz, Noto, Jolt, and others.
- **The editor product needs a separate EULA** (Steam allows a custom EULA). Proposed terms, *to be confirmed by Robert*
  (open question Q3): users **own the games they make and may sell them** with no royalty, and the **Poncelet runtime
  and the required third-party libraries may be redistributed inside exported games** (binary form only). The editor
  and engine source code may not be redistributed. Template-pack content may be used in the user's games.
  The repository's all-rights-reserved LICENSE covers the source. The EULA covers the product.
- Keep the Steamworks SDK, signing keys, and Steam build credentials out of the repo (CI secrets only).

## 16. Milestones (target: sellable editor + template packs on Steam)

2D ships first, then 3D on the same architecture. The editor grows from M2 onward, not at the end.
**Decided 2026-11-19:** high-level multiplayer, dynamic GI, video playback, the visual shader editor, and CSG are
**required for the first sellable release (M8)**. **Decided 2026-11-20:** every feature in the three signature
pillars (§19) is also required for M8; none of it is deferred to M9. M9 is post-launch long tail only (OpenXR
runtime, RTL, community library, consoles, WebGPU, and similar).
The step-by-step plan with verification commands is in **`docs/IMPLEMENTATION_PLAN.md`**.

| M | Name | Contents | Exit criterion |
|---|---|---|---|
| **M0** | Skeleton + CLI | gprbuild projects (Alire optional), `pc-core` (Math 2D+3D, **`Pc.Math.Projective`** (§19.1), World tables, i18n catalogs with plurals, text scene round-trip), coding rules, LICENSE, `AGENTS.md`. **`poncelet` CLI v0:** `version`, `new`, `validate`, `run --headless`, `test`, `--json`, `--lang`. CI: Linux build+tests, Windows/macOS compile; **portability rules** (`core.adc`: no tasking/allocators/OS access in core; console-friendly service boundaries); `run --headless --world-seed --ai-seed` (seeded, exactly replayable); **numeric host ABI v0** (`Pc.Host_ABI`) + last-chance handler (§10.5) | `./build.sh` + `./run-tests.sh` green on Linux Mint; `poncelet run --headless` prints a deterministic hash |
| **M1** | Pixels + formats | Local SDL3 build script, SDL3 binding, window/input, SDL_GPU render core + render graph, 2D batcher, fixed-step loop, offline shader pipeline. **All text formats + JSON Schemas** (`.pscene`, `.pres`, `.pimport`, `.ponpack`), canonical writer; CLI `screenshot`, `inspect`, `fmt`, `schema`; **WebAssembly toolchain spike** (headless core in the browser/Node, same hash as native); render service with a `Render_Method` interface for three tiers | A sprite moves at 60 Hz on Mint; `poncelet screenshot` writes a PNG; schemas validate samples |
| **M2** | World, UI, i18n, MCP | Generic component stores, hierarchy, events/tags, FreeType/HarfBuzz text + CJK fonts, pseudo-locale, UI toolkit (containers, widgets, themes, docking), editor shell + project manager, undo system, **MCP server v1**, generated API docs, GNATprove flow in CI | Editor shell switches live between seven languages; an AI client creates an entity via MCP and it can be undone |
| **M3** | 2D complete + hot reload | Own Ada 2D physics, 2D lights/shadows, tilemaps + terrains, particles, animation + tweens, audio mixer v1, save/load, input rebinding, rich text, **game-library hot reload**, editor 2D viewport, inspector, asset import, localisation editor, play-in-editor; input model incl. touch, gestures, and virtual controls (testable with mouse emulation); logical-order text layout (RTL-ready); **`Pc.Random`, replay record/playback (`.preplay`), golden replay tests, autoplayer interface, `poncelet sim` batch runs with per-game report files (F9-style), in-game F9 debug report**; **fractal generators** (`Pc.Noise` fBm/ridged/domain-warp, Diamond-square, L-systems, IFS) tied to world seeds; **2D projective editor tools** (perspective grids, horizon/vanishing-point guides, 2.5D projection mode) (§19) | A 2D template game built and live-edited entirely in the editor |
| **M4** | Export + tooling | Exporter with `.ppak` (Linux tested on Mint; Windows/macOS built in CI, unsigned), Steam stub, template-pack import, profiler + Tracy, movie capture, module-stripped exports; **compatibility renderer 2D** (GL 3.3 / GLES3 / WebGL2), **web export of 2D games** (WASM + WebAudio + DOM input + IndexedDB), githack-hostable demos; cross-platform determinism checks of replays in CI; single-file HTML web export with version stamp + githack check/commit-pinned fallback; release check policy (checks off only for SPARK Silver units) | Exported 2D template runs on Mint; Windows/macOS CI artifacts pass smoke tests |
| **M5** | 3D foundation + CSG | Clustered forward+ renderer, PBR, glTF import, cameras, lights, cascaded shadows, sky/IBL, MSAA/FXAA/TAA, Jolt 3D physics behind Ada API, 3D viewport + gizmos, primitive meshes, mesh builder, **CSG (Manifold)**, collision generation; **projective camera model** and **infinite far plane** built on `Pc.Math.Projective`; **anamorphic and non-standard projections**; **SDF fractal distance estimators** (Mandelbulb and similar) as a core math package (§19) | A glTF scene with CSG blockout, lit and shadowed, with physics, editable in the editor |
| **M6** | 3D complete + video + networking base | Skeletal animation + animation graph + IK/retargeting, lightmap baker, reflection probes, SSAO/SSIL/SSR, decals, volumetric fog, GPU particles, LOD/culling, navmesh + avoidance, GridMap, FBX import, FSR1, sockets/HTTP(S)/WebSocket, **video playback**, audio parity (buses, effects, 3D audio, adaptive music); **mobile renderer tier**, **compatibility renderer 3D**, **web 3D export**, **Android dev builds** (debug APK, lifecycle, touch, sensors), **iOS CI compile**; **density-field resources** (sparse/dense, text serialisation, painting, scattering, heat maps, physics queries, fog/smoke driven by fields) and **homography-based decals/projectors**; **portal/mirror math** and **planar mirror rendering**; **2D SPH** fluid solver; **sparse voxel volume** resources (§19) | 3D template game playable and exportable, with an in-game video cutscene |
| **M7** | Advanced features | **Dynamic GI** (VoxelGI-style volumes + probe-based large-world GI), **visual shader editor** (node graph → shader dialect, **fractal noise and escape-time nodes**), **high-level multiplayer** (replication, typed remote calls, ENet, UPnP, multi-instance testing, network profiler), editor plugins + C ABI extensions, accessibility (AccessKit); **asset/plugin package format + library client**; XR preparation (multi-view render graph, XR camera, action mapping); controller-first UI focus navigation; **seeded fractal/density world generation**, fractal density volumes, marching-cubes and **dual-contouring** isosurfaces; **portal rendering**; **AR camera-calibration homographies**; **3D SPH** and SPH rendering; **destructible voxel terrain** and voxel physics; **SDF raymarched fractals (Mandelbulb) with LOD**; **deterministic infinite zoom** (double + perturbation); mobile/web fractal budgets (§19) | A multiplayer template runs on two instances on one machine; a dynamically lit 3D scene uses node-graph materials; signature demos (portals, Mandelbulb or deep zoom, SPH, destructible voxels) run on the high-end tier |
| **M8** | Beta, maturity gate, platforms + Steam (**first sellable**) | **8a Beta:** onboarding, tutorials, crash recovery, soak tests, native-speaker localisation QA, template packs finalised + signed, EULA, docs. **8b Maturity gate:** no known crashers, formats frozen. **8c Release:** Steamworks + Apple sign-ups, real Steamworks body, macOS signing/notarisation, Windows signing (optional), Steam Runtime build, real-hardware tests on macOS/Windows, SteamPipe, store page in seven languages; **8c also:** Play Store signing/AAB, iOS device builds + App Store/TestFlight (needs Mac access + Apple account), curated paid template packs as DLC | Steam review passed; **Poncelet Editor + template packs released** |
| **M9** | Post-launch | **OpenXR VR/AR** (the AR calibration *toolkit* already shipped in M7), **BiDi/RTL text + UI mirroring**, **community asset library service**, **console ports via partners**, WebGPU for web/mobile, mobile store services (IAP, ads, notifications), FSR2, texture streaming, glTF export, Workshop, mic/TTS, MIDI, webcam, WebRTC, VRS, ARM64 desktop, dedicated-server exports, own Ada 3D physics, SPARK proofs. Signature pillars are not in M9; they shipped in M0–M7 (§19.5). | — |

**Platform placement (draft 0.7):** web 2D in M4 and 3D in M6, Android/iOS dev builds in M6 with store signing in
M8 (gated), and consoles, OpenXR, RTL, and the community library post-launch in M9. Reasons are in §2 and §13.3.

**Reference games as acceptance tests (draft 0.8):** Reckoning must build and run on the engine at M1 (WASM spike),
M3 (`poncelet sim` batch, F9 report) and M4 (stamped single-HTML web export); Roguelike Jungle is ported with hash
parity in M5, Colony4X and Hood Simulator with seed parity in M7, and all four packs are finalised and signed in M8
(§13.4, Appendix B).

**Schedule realism:** this is a multi-year roadmap. Adding the five pre-launch parity features moved M8 out by roughly
two to three milestone-lengths compared with draft 0.5. Draft 0.9 added the signature *core* (projective math, fractal
generators, density fields) across M0–M7 and left the heavy items in M9. **Draft 0.10 pulls every signature feature
into M5–M7**, which grows M6 and especially M7 (portals, SPH, voxels, dual contouring, Mandelbulb raymarching,
infinite zoom, AR calibration). That is a further **one and a half to two milestone-lengths** on top of draft 0.9.
**Cumulative delay versus draft 0.5 is about four to five milestone-lengths.** The mitigations are reusing proven
C libraries (Jolt, Manifold, ENet, dav1d/Theora, cgltf), writing SPH, voxels, and isosurfaces in Plain Ada as small
verifiable steps, running independent signature tracks in parallel inside M6–M7, and AI-assisted development
through the CLI and MCP. This is an honest slip: the unique-value pillars are now all on the critical path to M8.

## 17. Risks
- **AI tools making destructive or wrong edits:** mitigated by project scoping, confirmations, undo grouping,
  schema validation on every write, and an activity log.
- **MCP spec evolution:** isolate the protocol layer in `mcp/`, track spec versions, and test against several clients.
- **Small Ada game-dev ecosystem:** we own our bindings. Mitigation: keep the C surface tiny and generate bindings.
- **macOS/Windows without local hardware:** compile and smoke-test in CI from M0 so portability never rots.
  Real-hardware testing and notarisation come in M8. Risk: GPU or driver issues found late, so keep shaders and
  SDL_GPU usage conservative and test early with community testers if possible.
- **CJK text quality** (line breaking, IME, font size): handle in M2 with a native-speaker review.
- **Determinism with floats across OSes:** use fixed point where it matters, and avoid `-ffast-math`.
- **Scope (Godot-level 2D + 3D + polished editor + the whole of the signature pillars):** this is the largest risk,
  and draft 0.10 makes it larger by moving Mandelbulb raymarching, infinite zoom, SPH, destructible voxels, dual
  contouring, portals, anamorphic projections, and AR calibration into the first release. Mitigations: SDL_GPU +
  Jolt + cgltf instead of writing our own; ship 2D first; allow 3D as early access at M8 if needed; break the
  heavy signature work into small verifiable steps on parallel tracks in M6–M7 (§19.5); per-tier fallbacks so a
  low-end target still ships (baked meshes, cheaper fluids, single-bounce mirrors). Dynamic GI and the visual
  shader editor stay pre-launch (decided 2026-11-19). See the delay estimate in §16.
- **Signature-feature cost on mobile and web:** fractal shaders, volumetric density, isosurfaces, raymarching,
  SPH, voxels, and portals can blow frame-time and memory budgets. Mitigations: per-tier quality presets and hard
  caps (§19), editor warnings when a material or field exceeds the target tier, baked or lower-resolution
  fallbacks on the compatibility renderer, and golden-frame plus budget tests on the mobile and WebGL2 paths.
- **SPH, voxels, and deep zoom determinism:** particle neighbourhoods, sparse voxel edits, and perturbation zooms
  are easy to make platform-unstable. Mitigations: fixed-size neighbourhoods, canonical serialisation, world-seed
  sub-streams, and native-vs-WASM hash tests on every generator and solver step.
- **Projective numerics:** homogeneous coordinates and an infinite far plane are easy to get subtly wrong
  (w=0, cancellation, reversed depth). Mitigations: contracts on w and on incidence tests, known-answer tests
  (cross-ratio, duality, vanishing points), and screenshot tests of the infinite far plane on all three renderer
  tiers.
- **Bundling GNAT with a commercial editor:** use the FSF GCC toolchain under the GCC Runtime Library Exception
  and confirm redistribution terms (GPLv3 for compiler binaries means source offer obligations for the toolchain itself).
- **Ada toolchains for web, iOS, and consoles:** GNAT-LLVM for wasm32 and arm64-apple-ios is less mature than
  native GNAT. Mitigations: early CI spikes (M1, M6), the proven Ada-to-WASM setups of Roguelike Jungle and CDDA In ADA (§10.5), a restricted
  runtime, and core portability rules. Console toolchains are a partner topic.
- **Three renderers to maintain:** shared render graph, materials, and shader dialect, golden screenshots per tier
  in CI, and a clear feature matrix with editor warnings.
- **Store policies and sign-ups** (Google Play, Apple, console certification): all gated to M8/M9, with no accounts before the maturity gate.
- **SDL_GPU feature ceiling for advanced 3D** (no ray tracing or mesh shaders): acceptable for v1. Revisit option C later.

## 18. Open questions for Robert
1. ~~2D only for v1, or is 3D needed soon?~~ **Answered (2026-11-19):** Poncelet must match Godot in 2D **and** 3D.
   Ship 2D first, then 3D on the same architecture.
2. **What engines and languages were your other bot-built games written in?** *Partly answered:* Hood Simulator and
   Colony4X are JavaScript games (so their templates are ports to Ada), and Roguelike Jungle and CDDA In ADA (Reckoning) are Ada games built to WASM.
   This decides whether templates are imports or ports. **Draft 0.8:** all four packages studied (Appendix B);
   Reckoning is the first pack, the others are seed-parity ports (§13.4).
3. ~~What do we sell first?~~ **Answered (2026-11-19):** the engine + editor as a Steam software product, bundled with
   template packs. **New, to confirm:** the end-user licence for games made with Poncelet. Proposal: users own and may
   sell their games royalty-free, the runtime may be redistributed inside exported games, and engine/editor source
   may not be redistributed (§15).
4. ~~Final name~~ **Answered (2026-11-19) and confirmed kept (2026-11-20):** Poncelet, after Jean-Victor Poncelet
    and projective geometry. Memorability was raised and accepted. Signature focus: projective geometry, fractals,
    and density (§19).
5. ~~macOS distribution / Apple Developer ID~~ **Answered (2026-11-19): deferred.** No sign-ups (Apple or Steam)
   until Poncelet is stable and mature. Development is on Linux Mint, and macOS/Windows are compile and smoke-checked in CI.
   Signing, notarisation, and Steam come in M8.
6. Spanish: Spain (`spanish`) or Latin American (`latam`), or both?
7. Should CI (GitHub Actions on all three OSes) run in the public repo, or in a private mirror (to keep build
   details and secrets separate)?
8. Is professional translation planned, or bot-translated with native-speaker review?
9. Use Alire-managed toolchains (recommended for Windows and macOS), or stay on system GNAT 12.3?
10. ~~Parity timing~~ **Answered (2026-11-19):** high-level multiplayer, dynamic GI, video playback, the visual shader
    editor, and CSG must all be in the first sellable release (M8). They're scheduled in M5–M7 and M9 is long tail only.
11. **Android testing:** do you have an Android phone or tablet for M6 dev builds, or should we rely on the Android
    emulator (which needs KVM on the Mint laptop)?
12. **iOS:** iOS builds and App Store submission need a Mac (or a cloud Mac) and an Apple account, both gated to M8.
    Is that acceptable, with CI compile-only before then?
13. **Community asset library:** self-hosted service vs. Steam Workshop vs. both; free only or paid assets with a
    revenue share? (Post-launch, M9.)
14. ~~Web toolchain~~ **Answered in draft 0.8 (§10.5.9):** AdaWebPack 24 (GNAT-LLVM) with checks on and a
    last-chance handler, explicit export list, WebGL2 first and WebGPU later, four WebAudio buses (§10.6).
15. **Default RNG for new games:** PCG32 (proposed) or mulberry32 (already used by Colony4X and Hood Simulator)?
    Compatibility generators exist either way.
16. **Web toolchain on your laptop:** OK for `tools/get-wasm-toolchain.sh` to download AdaWebPack 24 and the GNAT 14
    runtime library into the project tree (no sudo, no Alire)? Native builds stay on your GNAT 12.3.
17. **Default publish target for Poncelet demos and exported games:** a gist (as Reckoning and Roguelike Jungle do)
    or a `build/<version>` branch in the repository (as Hood Simulator does)? Both will be supported.
18. **Template code ownership:** may Reckoning, Roguelike Jungle, Colony4X and Hood Simulator content ship inside
    the commercial editor as template packs under the Poncelet EULA (all are yours), and which of them should be paid
    DLC rather than bundled?
19. ~~Signature-feature cut for the first release~~ **Answered (2026-11-20):** **all** of the three signature
    pillars ship in the first sellable release. Nothing in §19 is deferred to M9. Anamorphic / non-standard
    projections, portals and mirrors, AR camera-calibration homographies, Mandelbulb-style SDF raymarching with
    LOD, deterministic infinite zoom (double + perturbation), SPH fluids, destructible voxel terrain, and dual
    contouring are scheduled in M5–M7 (§19.5). The OpenXR *runtime* stays in M9; the calibration toolkit does not.
20. **Signature showcase pack:** should the first Steam release bundle a small official template that exercises
    the three pillars (perspective guides, portals or an anamorphic camera, a seeded fractal world or
    Mandelbulb / deep zoom, a painted density field, SPH or destructible voxels), or is a tutorial plus docs
    enough?

## 19. Signature features beyond Godot

Poncelet's baseline is Godot-level 2D and 3D for typical indie games (Appendix A). On top of that, three pillars
are first-class in the engine, the editor, and the tools API: **projective geometry**, **fractals**, and
**density**. They are why the product is called Poncelet, they are designed into `pc-core` rather than
shipped as plugins, and **all of them ship in the first sellable release** (decided 2026-11-20; §19.5).

Jean-Victor Poncelet (1788–1867) founded modern projective geometry. Naming the engine after him is not just
ornament: homogeneous coordinates, points at infinity, incidence, duality and the cross-ratio are types and
operations in `Pc.Math`, and the camera and perspective tools are built on them.

**Rules that apply to all three pillars**
- **Plain Ada, contracts, SPARK-ready, platform-free.** Types and algorithms live in `pc-core`. They build with
  `gprbuild -XPLATFORM=none`, use no pointers, heap, tasking, or `Text_IO`, and state `Pre`/`Post` and ranged
  subtypes (§9, §10.5). Shader and GPU work sits in services, behind handles.
- **Deterministic and seeded.** Anything that generates world content or affects simulation takes the project's
  **world seed** (and a documented sub-stream), is bit-stable across editor, CLI, MCP, and exported games, and
  contributes to the 64-bit state hash (§6.15). Visual-only shader previews may use ordinary floats; anything
  that feeds gameplay prefers integer or fixed-point paths, or a documented float exception.
- **Renderer tiers (§6.2a).** High-end (clustered), mobile, and compatibility (GL 3.3 / GLES3 / WebGL2) all
  accept the same scenes and resources. Each feature names what every tier actually does, and the editor warns
  when a resource exceeds the target tier.
- **AI-tool visible.** Resources are text (`.pres` / `.pscene`) with JSON Schemas. The CLI (`poncelet gen`,
  `poncelet projective`) and MCP tools in §12 expose the same operations as the editor.
- **Tests.** Known-answer vectors, golden hashes, and (where it is a picture) golden screenshots per renderer
  tier, including a WebGL2 path once that export exists.

### 19.1 Projective geometry

**What it gives game makers.** A consistent language for perspective: vanishing points, horizons, points at
infinity, projections from 3D onto a picture plane, and mappings between planes (homographies). That helps 2D
artists and level designers lay out believable perspective, makes 2.5D and fake-3D modes first-class instead of
a shader trick, and gives 3D cameras a numerically honest far plane and projector/decal math.

**Engine and editor features**
- Homogeneous points, lines and planes as first-class math types, including **points at infinity**.
- Incidence (meet / join), **cross-ratio**, and **duality** utilities with contracts.
- Projective transforms and a **camera model** built on them (perspective, orthographic, 2.5D / fake-3D
  projection modes).
- Editor **vanishing-point and perspective tools:** perspective grids, horizon and vanishing-point guides for
  2D art and level design.
- **Homography-based decals, projectors, and texture mapping** (map an image onto a quad or from one plane
  onto another).
- A **robust infinite far plane** (reversed-Z / infinite projection) so distant objects and skies do not fight
  the depth buffer.
- **Anamorphic and other non-standard projections** as first-class camera modes (editor controls, all renderer
  tiers that draw 3D).
- **Portal and mirror reflection math**, planar mirror rendering, and portal rendering (recursion cap, teleport,
  editor placement).
- **Camera-calibration and AR-friendly homographies:** checkerboard / still-image calibration, CLI and editor
  workflow, apply a computed camera to a scene. The OpenXR runtime itself stays post-launch (M9); the toolkit
  ships in the first release so AR projects can use it on desktop and hook to OpenXR later.

**In the Ada core.** `Pc.Math.Projective` defines `Point_H`, `Line_H`, `Plane_H`, and `Projective_Transform` as
records of homogeneous coordinates. Points at infinity are ordinary values with `W = 0`, guarded by contracts
(no silent division by zero; callers use `Is_Finite` / `Is_At_Infinity`). Meet, join, dual, and `Cross_Ratio`
have `Pre`/`Post` that state incidence and invariance. v1 uses IEEE `Float` (and `Long_Float` where calibration
or large worlds need it) with documented epsilon contracts; a rational/integer homogeneous path for exact
tool-side incidence is a later SPARK candidate, not a first-release requirement. The package is platform-free
and is a planned SPARK graduate.

**Renderer tiers.** The projective camera, infinite far plane, and anamorphic / non-standard projections are
used on every tier that draws 3D (high-end, mobile, compatibility, WebGL2). 2.5D projection modes and 2D
guides are CPU/editor work plus a small vertex transform, so they work on all tiers. Homography projectors use
the clustered decal path on high-end, a cheaper projector on mobile, and a single-projector fallback on
compatibility / WebGL2. Planar mirrors and portals use render-to-texture on high-end and mobile (with a
recursion and bounce cap); compatibility / WebGL2 get a single-bounce mirror and a simpler portal (one
offscreen, no recursive portals). Calibration is core math plus editor/CLI, so it runs everywhere, including
headless.

**CLI / MCP.** `poncelet projective` and MCP `projective_query` / `set_perspective_guides` answer incidence,
cross-ratio, vanishing-point, and homography queries as JSON, and write guides into a scene. `poncelet
calibrate` / MCP `calibrate_camera` consume a documented checkerboard (or marked stills) and write a camera.
Portal and mirror placement goes through the ordinary scene tools plus `set_portal` / `set_mirror`.

**Acceptance (first release).** Known-answer tests: a documented complete quadrangle's cross-ratio; duality
is involutive; a point at infinity lies on a given line; a homography maps four documented points to four
others. A 2D scene with vanishing-point guides round-trips in `.pscene`. A 3D demo with a distant object and
an infinite far plane produces a stable golden screenshot on high-end and on the compatibility / WebGL2 path
(no far-plane clipping, no depth fighting with the sky). An anamorphic camera matches a documented projection
matrix. A planar mirror and a pair of portals produce golden screenshots (and, for portals, a teleport that
preserves a seeded replay). A checkerboard fixture yields a camera whose reprojection error is under a
documented threshold.

### 19.2 Fractals

**What it gives game makers.** Procedural plants, terrain, patterns, and spaces that stay interesting at many
scales, all driven by the world seed. Artists get visual-shader nodes; designers get seeded generators that
the CLI and an autoplayer can batch-test.

**Engine and editor features**
- **L-systems** for plants and trees (turtle interpretation to meshes or 2D strokes, iteration cap, seed).
- **Iterated function systems (IFS)** for self-similar shapes.
- **Fractal noise:** fBm, ridged, and domain-warped noise for terrain and materials, plus **Diamond-square**
  heightmaps, all in `Pc.Noise` / `Pc.Fractal` (pure Ada).
- **Escape-time** Mandelbrot / Julia materials as **visual shader editor nodes** (and equivalent dialect code).
- **Fractal level and world generation** tied to the world seed: chunks, biomes, and tile or height layers that
  hash-match across platforms.
- **Budget and LOD guarantees** on mobile and web: quality presets, iteration caps, and editor warnings.
- **SDF raymarched fractals** (Mandelbulb and similar) as a renderer feature, with LOD, a mobile iteration
  budget, and a pre-baked mesh or impostor fallback on the compatibility tier.
- **Infinite-zoom** escape-time precision: a deterministic double-precision path and a perturbation method for
  deep zooms, with native-vs-WASM hash or image-diff tests.

**In the Ada core.** Generators are deterministic functions of `(world seed, sub-stream, parameters)` and
write bounded outputs (degree caps on L-systems, iteration caps on IFS, grid sizes on noise). Contracts enforce
those bounds. Shader nodes compile to the Poncelet shader dialect; they are data in `.pres`, not generated Ada
on the hot path. World generation that affects gameplay is hashed into the simulation state (§6.15).

**Renderer tiers.** Noise, L-systems, and IFS produce meshes, instances, heightmaps, or textures — those run
on every tier. Escape-time looks are fragment shaders and run on WebGL2 as well as high-end and mobile, with
iteration caps per quality preset. Deep zoom uses the CPU/double or perturbation path for the region of
interest and a shader preview at the current cap. SDF raymarching is high-end first, with a tight iteration
budget on mobile and a pre-baked mesh or impostor fallback on compatibility / WebGL2.

**CLI / MCP.** `poncelet gen noise|lsystem|ifs|world --world-seed …` and MCP `generate_fractal` /
`generate_world` write resources or scene chunks and print the content hash. `poncelet gen sdf` / `raymarch`
and `poncelet zoom` (double and perturbation) expose the Mandelbulb and deep-zoom paths.

**Acceptance (first release).** The same seed and parameters produce the same hash on Linux and in the WASM
spike / web export for noise, Diamond-square, a documented L-system tree, and an IFS. A visual-shader graph
with fBm and a Julia node compiles and renders a golden screenshot on high-end and on WebGL2 at the "low"
budget preset. A budget test fails the build if a mobile/web preset exceeds its iteration or sample cap. A
documented Mandelbulb distance estimator matches known answers; a high-end raymarch screenshot is golden; the
compatibility path shows the baked-mesh fallback without crashing. A deep-zoom (double and perturbation) of a
documented Julia/Mandelbrot location matches a reference within tolerance and hashes stably on native and WASM.

### 19.3 Density

**What it gives game makers.** A single **density-field** resource that many systems can read: fog and smoke,
vegetation and crowd placement, AI influence / heat maps, physics queries ("how thick is it here?"), and, at a
basic level, terrain isosurfaces. Artists paint it; tools generate it from noise or fractals; the file is text
and deterministic.

**Engine and editor features**
- Density fields as a first-class resource: scalar and vector volumes, dense or sparse grids, stored as a
  typed `.pres` (canonical JSON, optional binary cache under `.poncelet/cache/`).
- **Volumetric fog and smoke** (and simple cloud volumes using the same field) can be driven by a density field (extending the froxel fog volumes of M6).
- **Scattering:** vegetation, props, and crowds placed from a density map, seeded and repeatable.
- **AI influence / heat maps** and **density-aware physics queries** (point sample, filtered query, ray march
  through a field) in `pc-core`.
- **Editor painting** of density (2D maps and 3D volumes) with brush, erase, and fill-from-generator.
- **Isosurface terrain** from a scalar field: marching cubes and **dual contouring** (sharp features, LOD,
  editor bake) in the first release.
- **SPH-style particle and fluid density** in 2D and 3D, coupled to density fields, with editor emitters.
- **Destructible voxel terrain:** a sparse voxel volume resource, editing and destruction (undoable,
  serialised), mesh extraction, and physics / collision.

**In the Ada core.** `Pc.Density` is a bounded, platform-free resource: resolution, bounds, sparsity, and
sample values are all under contracts. Sampling is a pure function. Serialisation is canonical and
order-stable so two saves of the same field match byte-for-byte after `poncelet fmt`. Fields that affect
simulation are included in the state hash. Painting and generation are editor/CLI commands (undoable).

**Renderer tiers.** Sampling, scattering, heat maps, and physics queries are core work and run everywhere,
including headless and web. Volumetric lighting of a field uses froxel fog on high-end, a lower-resolution
or half-res volume on mobile, and baked slices or a cheaper 2D/height fog fallback on compatibility / WebGL2.
Marching-cubes and dual-contouring meshes are ordinary meshes once built, so they draw on all tiers;
generation cost is budgeted and can be baked at edit time. SPH runs the solver in the core (headless-capable);
high-end draws particles or a surface, mobile uses a tighter particle cap, and compatibility may draw a cheaper
billboard or baked surface. Voxel destruction is core state; mesh rebuild is budgeted and can be asynchronous
in the editor, with a lower-resolution mesh on mobile and a baked mesh on compatibility where live edits are
off.

**CLI / MCP.** `poncelet gen density` and MCP `sample_density`, `paint_density`, `scatter_from_density` read
and write `.pres` fields and placement results, all schema-validated. `poncelet sph`, `poncelet voxels` (and
matching MCP tools) run a seeded step, carve, and extract meshes.

**Acceptance (first release).** A painted or generated field round-trips through `.pres` with a stable hash.
The same seed places the same scatter set on native and WASM. A point-sample and a ray-march query match known
answers. A fog-volume demo driven by a field renders on high-end and degrades without crashing on
compatibility / WebGL2. A small marching-cubes mesh and a dual-contouring mesh from a documented field match
golden hashes (dual contouring keeps a documented sharp edge). A 2D and a 3D SPH fixture are seed-stable
(native vs WASM) and render without exploding the particle bound. A voxel volume carves under a seeded tool,
rebuilds a mesh, answers a collision query, and round-trips through `.pres`.

### 19.4 Interactions

The pillars are meant to combine:
- **Fractal density volumes:** a density field can be filled from fractal noise (fBm, ridged, domain-warp) or
  an IFS, then used for fog, scattering, or an isosurface (marching cubes or dual contouring). This path is in
  the first release.
- **Projective rendering of density and fractals:** cameras, projectors, and 2.5D modes use the projective
  model, so a density volume, a decal, or a fractal material is imaged through the same camera and
  vanishing-point tools as everything else, including the infinite far plane.
- **Seeded pipelines:** `generate_world` can chain noise → density field → scatter / isosurface in one
  deterministic command, which `poncelet sim` and golden tests can replay.

### 19.5 First-release cut (all in)

**Decided 2026-11-20:** every feature in the three pillars ships in the first sellable release (M8). Nothing
here is deferred to M9. OpenXR as a runtime, RTL, the community library, consoles, and WebGPU remain post-launch;
they are not signature-pillar work.

| Feature | Built in |
|---|---|
| `Pc.Math.Projective` (homogeneous types, points at infinity, incidence, cross-ratio, duality, transforms) | M0 |
| Editor perspective grids, horizon and vanishing-point guides; 2.5D / fake-3D projection modes | M3 |
| Fractal noise (fBm, ridged, domain-warp), Diamond-square, L-systems, IFS | M3 |
| Projective camera model, infinite far plane | M5 |
| Anamorphic and other non-standard projections | M5 |
| SDF fractal distance estimators (Mandelbulb and similar), known-answer tests | M5 |
| Density-field resource, fog/smoke, scatter, heat maps, physics queries, editor painting; homography projectors | M6 |
| Portal and mirror math; planar mirror rendering | M6 |
| 2D SPH fluid solver (seeded, bounded, hashed) | M6 |
| Sparse voxel volume resource and text serialisation | M6 |
| Escape-time visual-shader nodes, mobile/web budget caps; seeded fractal/density world generation; marching cubes | M7 |
| Portal rendering (recursion cap, teleport, editor) | M7 |
| AR camera-calibration homographies (still-image / checkerboard toolkit; OpenXR runtime remains M9) | M7 |
| 3D SPH, SPH rendering, coupling to density fields, editor emitters | M7 |
| Destructible voxel terrain (editing, persistence) and voxel physics / collision | M7 |
| Dual contouring (sharp features, LOD, editor bake) from density and voxels | M7 |
| SDF raymarched fractals (Mandelbulb) with LOD and a compatibility mesh/impostor fallback | M7 |
| Deterministic infinite zoom: double-precision path and perturbation method | M7 |

A rational / integer homogeneous path for exact tool-side incidence is a SPARK graduation of `Pc.Math.Projective`,
not a separate deferred product feature.

## Appendix A. Godot 4.x parity matrix (audit of 2026-11-19, draft 0.5)

Sources: the public Godot 4.x "List of features" documentation page and the directory layout of the Godot
repository (`modules/`, `servers/`). Capabilities are **paraphrased in our own words**. No Godot text or code is copied.
Draft 0.10 update: every Appendix A.21 signature row is scheduled in M0–M7 (first release); none remain in M9.
Draft 0.9 update: Appendix A.21 lists **signature features beyond Godot parity** (projective geometry, fractals,
density; §19). Draft 0.7 update: games now target all platforms (web, Android, iOS, consoles via partners, XR); three renderer
tiers; RTL and the community asset library are scheduled post-launch. Draft 0.6 update: multiplayer, dynamic GI, video, the visual shader editor, and CSG were moved to pre-launch
milestones (M5–M7). Everything still marked M9 is post-launch long tail.
"0.4 status" is how DESIGN.md draft 0.4 covered each capability before this audit. The "Poncelet equivalent" column
is the plan added in draft 0.5. Milestones refer to §16.

**Before (draft 0.4), 174 capabilities:** Covered 44 · Partial 48 · Missing 68 · Intentionally different 14.  
**After (draft 0.5):** every Covered/Partial/Missing item has a planned equivalent and milestone (161); 13 intentional differences remain (FBX moved from "out of scope" to planned).

Legend: C = covered, P = partial, M = missing, I = intentionally different.

### A.1 Platforms

| Godot capability | 0.4 | Poncelet equivalent (current) | Milestone |
|---|:-:|---|---|
| Windows/macOS/Linux x86-64 editor + games | C | Native desktop targets; Linux-first development, Win/macOS via CI (§10) | M0–M8 |
| ARM64 Windows/Linux, 32-bit builds | M | ARM64 Linux/Windows added when GNAT + SDL3 CI runners are available; no 32-bit | M9 |
| macOS universal (arm64 + x86-64) | C | lipo universal app (§10.2) | M8 |
| Android / iOS export | I | Android (arm64/x86-64) and iOS (arm64) game export: mobile renderer tier, touch, sensors, app lifecycle; unsigned dev builds first, store signing gated (§10.4) | M6 (dev builds) / M8 (store signing) |
| Web (HTML5) export | I | Ada → WebAssembly (GNAT-LLVM toolchain, AdaWebPack GNAT-LLVM, web-safe subset and numeric host ABI per §10.5), WebGL2 via the compatibility renderer, WebAudio, DOM input, IndexedDB saves; WebGPU later (§10.4) | M1 (spike) / M4 (2D) / M6 (3D) / M9 (WebGPU) |
| Consoles (via third parties) | I | Console ports by third-party partners under platform NDAs (W4-Games-style); console-friendly platform layer from M0 (§10.4) | M0 (rules) / M9 (partner ports) |
| Headless/server mode (no GPU) | P | `poncelet run --headless` with null render/audio services; dedicated-server exports | M1 (CLI), M9 (server export) |

### A.2 Editor

| Godot capability | 0.4 | Poncelet equivalent (current) | Milestone |
|---|:-:|---|---|
| Scene tree editor, inspector | C | Scene dock + generated inspector (§11) | M2–M3 |
| Built-in script/text editor with highlighting | P | Built-in Ada code editor (syntax highlight, ALS-driven completion, diagnostics) + external editors | M5 |
| External editor integration (VS Code, Vim…) | C | ALS / GNAT Studio / VS Code (§11) | M3 |
| Script debugger, thread debugging | P | GDB/MI integration: breakpoints, stepping, per-task views, contract-failure stop | M5 |
| Visual CPU/GPU profiler, custom monitors, Tracy/Perfetto | P | Frame profiler (CPU scopes + GPU timestamp queries), custom monitors, Tracy client via C binding | M4 |
| Run code in editor (tool scripts, inspector buttons) | M | Editor-side Ada 'tool systems' loaded with the game library; inspector actions declared in component descriptors | M5 |
| Live code reload | M | Hot-reload of the game as a shared library, World state kept engine-side (§6.6a) | M3 |
| Live scene editing in running game / remote inspector | P | Play-in-editor process with live World inspector and two-way property edits over the tools protocol | M3 |
| Camera override / select objects in running game | M | Debug camera override + picking in play mode via tools protocol | M5 |
| Embedded game panel, multiple simultaneous instances | M | Embed play view in editor; launch N instances (client/server testing) | M5 / M7 |
| Time scale, pause, frame advance, mute | P | Fixed-step loop makes this trivial: time scale, pause, `step_frames`, mute bus | M3 |
| Ruler, vertex snapping, focus follow | M | Viewport measurement tool, vertex/surface snapping, follow selection | M3 (2D) / M5 (3D) |
| Offline built-in class reference | C | Generated API reference from Ada specs (§12.4) | M2 |
| Editor localised in many languages | C | Seven languages (§8, §11) | M2 |
| Editor plugins | M | Ada editor-plugin API (docks, inspectors, importers, gizmos, menu commands) loaded as shared libraries (§6.12) | M7 |
| Asset library / store, download projects | P | Package format + editor library client (M7), curated paid template packs (M8), community asset library service (M9) (§13.3) | M7 / M8 / M9 |
| Project manager | C | Project manager with templates (§11) | M2 |

### A.3 2D rendering

| Godot capability | 0.4 | Poncelet equivalent (current) | Milestone |
|---|:-:|---|---|
| Sprites, polygons, lines (Polygon2D/Line2D) | P | Sprite batcher + textured polygons and polylines with joints/caps | M1–M3 |
| Animated sprites | C | Sprite-frame animation (§6.11) | M3 |
| Parallax layers | C | Parallax (§6.2) | M3 |
| 2D lights with normal + specular maps, hard/soft shadows | C | 2D lights + occluder shadows (§6.2) | M3 |
| 2D SDF of occluders (2D GI, particle collision) | M | Real-time 2D SDF from occluders exposed to shaders and particles | M6 |
| Fonts: FreeType, MSDF, bitmap fonts, colour emoji, outlines, variable fonts, OpenType features, oversampling, subpixel/LCD | P | FreeType + HarfBuzz + MSDF (msdfgen, MIT); bitmap fonts; COLR/CBDT emoji; outlines; variable axes; oversampling | M2 / M6 |
| SVG import with runtime re-rasterisation | M | SVG via a small rasteriser (e.g. lunasvg/nanosvg, MIT) for icons and DPI-aware textures | M5 |
| GPU and CPU particles (2D) | C | Particles (§6.2) | M3 |
| 2D HDR, debanding, HDR output | M | Optional HDR canvas, dithering; HDR display output when SDL3 exposes it | M6 |
| Pseudo-3D in 2D | M | Y-sort + z-index + first-class 2.5D / fake-3D projection modes and vanishing-point guides (§19.1) | M3 (guides, 2.5D mode) / M5 (projective camera) |

### A.4 2D tools

| Godot capability | 0.4 | Poncelet equivalent (current) | Milestone |
|---|:-:|---|---|
| TileMaps (layers, terrains/autotile, physics, navigation) | P | Tilemap layers, terrain auto-tiling rules, per-tile collision/nav/occluders, tile animation | M3 |
| 2D camera smoothing and drag margins | C | Camera2D (§6.2) | M3 |
| Path2D / PathFollow2D | M | Curve2D resource + path-follow component, editable in viewport | M3 |
| 2D geometry helpers (clipping, triangulation, convex) | M | Pc.Geometry_2D (polygon boolean via Clipper2 binding or own, triangulation) — SPARK candidate | M3 |

### A.5 3D rendering

| Godot capability | 0.4 | Poncelet equivalent (current) | Milestone |
|---|:-:|---|---|
| Forward+ clustered renderer | C | Clustered forward+ (§6.2) | M5 |
| Lower-end / compatibility renderer | I | Three renderer tiers (§6.2a): compatibility renderer on OpenGL 3.3 / GLES 3.0 / WebGL2, plus a mobile tier | M4 (compat 2D) / M6 (compat 3D + mobile) |
| Perspective / ortho / frustum-offset cameras, depth prepass | C | §6.2 | M5 |
| Variable rate shading | M | Only if SDL_GPU exposes it; otherwise deferred | M9 |
| PBR: diffuse/specular modes incl. toon, ORM, normal, parallax, detail maps | P | Uber-material gains toon/lambert modes, parallax occlusion, detail layers | M6 |
| Subsurface scattering, transmittance, screen-space refraction, proximity/distance fade | M | Screen-space SSS, transmittance, rough refraction, soft particles and dither fade | M6 / M9 |
| Directional/omni/spot lights, area lights, light size, distance fade | P | Area (rect) lights, light size/penumbra, per-light energy and distance fade | M6 |
| Shadows: PSSM cascades, cube/dual-paraboloid, projector textures, PCSS, bias controls | C | §6.2 (+ projector textures added) | M5–M6 |
| Baked lightmaps (GPU bake, SH directional, shadowmask, denoise, probes) | P | GPU compute baker, SH directional, shadowmask, OIDN optional, light probes for dynamics | M6 |
| VoxelGI (dynamic voxel GI) | M | Voxel cone-traced GI volume (baked voxelisation, dynamic lights) | M7 |
| SDFGI (large-world dynamic GI) | P | Probe-based dynamic GI (DDGI-style) as our large-world equivalent | M7 |
| SSIL (screen-space indirect) | M | SSIL pass alongside SSAO | M6 |
| Reflection probes, SSR, mixed reflections | C | §6.2 | M6 |
| Decals (albedo/normal/ORM/emission, clustered) | M | Clustered decals with fade and channel masks; homography-based projectors (§19.1) go beyond basic decals | M6 |
| Sky: HDRI, procedural, physical sky, custom sky shaders | P | Physically based sky + custom sky shaders | M5–M6 |
| Depth and height fog, aerial perspective, sun scatter | P | Height fog, sky-tinted fog, sun scatter | M5 |
| Volumetric fog + fog volumes with shaders | P | Froxel volumetric fog reusing light clusters; fog volume component (box/ellipsoid/cone/cylinder/3D texture); can be driven by density-field resources (§19.3) | M6 |
| GPU particles: sub-emitters, trails, attractors, collision (SDF/heightmap), custom shaders | P | Compute particles with sub-emitters, trails, attractors, SDF/heightfield collision | M6 |
| CPU particles | C | §6.2 | M3 |
| Post: tonemappers, auto exposure, DOF bokeh, SSAO, glow modes + lens dirt, LUT, BCS | C | §6.2 (+ lens dirt, BCS adjustments) | M5–M6 |
| Texture filtering per use, VRAM compression BC/ASTC/ETC2/Basis | P | BC1–7 on desktop, ASTC/ETC2 on mobile, Basis Universal/KTX2 transcoding for web and mixed targets | M5 / M6 |
| MSAA, FXAA, TAA, SSAA, alpha-to-coverage | P | MSAA (2D+3D), FXAA, TAA, SSAA, alpha-to-coverage/hash | M5–M6 |
| Resolution scaling + FSR1 / FSR2 upscaling | M | 3D resolution scale with bilinear/FSR1 (MIT shader); FSR2/3 via FidelityFX SDK (MIT) later | M6 / M9 |
| LOD (auto mesh LOD), occlusion culling, visibility ranges/HLOD | C | meshoptimizer LODs, Hi-Z culling (§6.2); visibility ranges added | M6 |
| MultiMesh / instancing | C | §6.2 | M5 |
| Texture streaming | M | Mip streaming with residency budget | M9 |

### A.6 3D tools

| Godot capability | 0.4 | Poncelet equivalent (current) | Milestone |
|---|:-:|---|---|
| Built-in primitive meshes | M | Primitive mesh generators (box, sphere, capsule, cylinder, torus, plane…) | M5 |
| GridMap (3D tiles) | M | 3D tile grid component + palette painter in editor | M6 |
| CSG for prototyping | M | CSG via Manifold (Apache-2.0) C binding | M5 |
| Procedural geometry API | M | Pc.Mesh_Builder (surface tool equivalent) | M5 |
| Path3D / PathFollow3D | M | Curve3D + path-follow | M6 |
| glTF export (editor and runtime) | M | glTF writer (cgltf write) for scenes/meshes | M9 |
| Procedural noise | M | Pc.Noise (Perlin/Simplex/cellular, fBm, ridged, domain-warp) — pure Ada, SPARK candidate; fractal generators and shader nodes go further (§19.2) | M3 / M7 (shader nodes) |

### A.7 Physics

| Godot capability | 0.4 | Poncelet equivalent (current) | Milestone |
|---|:-:|---|---|
| 2D: static/animatable/rigid/character bodies, areas, joints | C | Own Ada 2D physics (§6.4) | M3 |
| 2D shapes incl. world boundary, collision polygons from sprites | P | Add world-boundary plane and sprite-outline polygon generation | M3 |
| 3D: static/animatable/rigid/character bodies, areas, joints | C | Jolt behind Ada API (§6.4) | M5 |
| Vehicles, soft bodies, ragdolls | P | Jolt vehicle/soft-body/ragdoll exposed via Ada API + editor ragdoll generator | M6 |
| Raycasts / shape casts / queries | C | §6.4 | M3 / M5 |
| Physics interpolation | C | Render interpolation of fixed-step state (§5.2) | M1 |
| Trimesh / convex collision generation, convex decomposition | M | Editor generators: trimesh, convex hull, V-HACD/CoACD decomposition | M5 |
| Interchangeable physics backends | C | Backend setting (§6.4) | M5 |

### A.8 Navigation

| Godot capability | 0.4 | Poncelet equivalent (current) | Milestone |
|---|:-:|---|---|
| A* (grid and graph) 2D/3D | P | Pc.Nav.AStar for grids and graphs (SPARK candidate) | M3 |
| Navmesh baking editor + runtime, 2D/3D | C | §6.4 navigation | M3 / M6 |
| Dynamic obstacle avoidance (RVO) | P | RVO2-style avoidance in Ada | M6 |
| Navigation links, layers, regions | M | Off-mesh links, nav layers/costs, multiple regions | M6 |

### A.9 Animation

| Godot capability | 0.4 | Poncelet equivalent (current) | Milestone |
|---|:-:|---|---|
| Animate any property, custom interpolation, Bézier curves | C | §6.11 | M3 |
| Method-call and audio tracks | P | Event tracks call registered Ada handlers; audio tracks | M3 |
| AnimationTree: state machines, blend spaces 1D/2D, blend trees | P | Data-driven animation graph: state machine, 1D/2D blend spaces, blend/add/one-shot nodes | M6 |
| Skeletal animation, skins, blend shapes | C | §6.2/§6.11 | M6 |
| IK (two-bone, look-at, FABRIK/CCD), skeleton modifiers | P | Two-bone, look-at, FABRIK/CCD chains, spring bones | M6 |
| Retargeting (bone maps, humanoid profile) | M | Retargeting via humanoid bone map on glTF import | M6 |
| Root motion | C | §6.11 | M6 |
| 2D skeletons / mesh deform | C | §6.11 (post-v1 noted) | M6 |
| Tweens | C | §6.11 | M3 |

### A.10 Audio

| Godot capability | 0.4 | Poncelet equivalent (current) | Milestone |
|---|:-:|---|---|
| Buses, routing, effects (reverb, EQ, compressor, delay, chorus, limiter…) | P | Ada mixer with bus graph and effect set (reverb, EQ, compressor, limiter, delay, chorus, filters, distortion) | M3 / M6 |
| 2D/3D positional audio, Doppler, listeners | P | Panning, attenuation curves, Doppler, listener components, reverb zones | M3 / M5 |
| Surround 5.1/7.1 output | M | SDL3 multichannel output + speaker mapping | M6 |
| Polyphony, random pitch/volume, sequential/random containers | M | Sound containers (random/sequence/no-repeat), voice limits | M3 |
| Real-time pitch scaling | M | Pitch-shift effect | M6 |
| Interactive/adaptive music | M | Music clips with transitions, stems, beat-synced switching | M6 |
| Procedural audio generation | M | Generator streams (callback into Ada) | M6 |
| Microphone input | M | SDL3 capture devices | M9 |
| Text-to-speech | M | Platform TTS (Speech Dispatcher / SAPI / AVSpeech) via thin bindings, tied to accessibility | M9 |
| MIDI input | M | MIDI input via platform APIs (ALSA seq / WinMM / CoreMIDI) | M9 |
| Formats WAV (+QOA/ADPCM), Ogg Vorbis, MP3 | P | WAV, QOA, Ogg Vorbis/Opus, MP3 (dr_mp3, public domain) | M3 |

### A.11 GUI / Control system

| Godot capability | 0.4 | Poncelet equivalent (current) | Milestone |
|---|:-:|---|---|
| Buttons, check/radio, line/text edit, code editor widget | P | Full widget set incl. multi-line text and code edit | M2–M5 |
| Popup/option menus with search, scrollbars, tree/table, colour picker | P | Menus, searchable dropdowns, tree/table, colour picker (RGB/HSV/OKHSL) | M2–M3 |
| Rich text (markup, inline images, animated effects) | M | Rich text label with a small markup (bold/italic/colour/links/images/effects) — localisable | M3 |
| Anchors, containers (box, grid, flow, margin, aspect, split, foldable) | P | Container set incl. flow, aspect, split, foldable | M2 |
| Stretch modes / multi-resolution / any aspect | M | Canvas/viewport stretch modes, content scale, aspect handling | M2 |
| Theme editor, vector styleboxes, texture styleboxes | P | Theme resource + editor, flat/rounded/shadow styleboxes, 9-patch styleboxes | M3 |
| Rotation/scale of controls, drag-and-drop | M | Transformable controls, DnD API | M3 |
| BiDi / RTL typesetting and UI mirroring | I | Documented later milestone: BiDi algorithm (SheenBidi/FriBidi or own Ada), HarfBuzz RTL shaping, UI mirroring; logical-order text model from M2 (§8.4) | M9 |
| Accessibility / screen readers | M | Accessibility tree for UI via AccessKit C API (MIT/Apache) — editor and games | M7 |

### A.12 Input

| Godot capability | 0.4 | Poncelet equivalent (current) | Milestone |
|---|:-:|---|---|
| Action map, remapping, deadzones, axis from two actions | C | §6.9 (+ deadzones and axis pairs) | M2–M3 |
| Physical-key mapping (layout independent) | M | Scancode-based bindings via SDL3 | M2 |
| Mouse capture/confine, custom cursors, raw input | M | SDL3 relative mode, cursor images, system cursors | M2 |
| Gamepads (multiple), LED, motion sensors | P | Up to 8 pads, rumble, LED, gyro via SDL3 | M3 |
| Pen/tablet pressure and tilt | M | SDL3 pen API | M6 |
| Touch/virtual joystick (mobile) | I | Multi-touch, gestures, virtual joystick/buttons, sensors, safe areas (§6.9) | M3 (model) / M6 (devices) |

### A.13 Scripting & extensibility

| Godot capability | 0.4 | Poncelet equivalent (current) | Milestone |
|---|:-:|---|---|
| GDScript (interpreted, optionally typed) | I | Game logic in compiled Ada with contracts; iteration via hot reload + data assets (§6.6a) | M3 |
| C# / .NET | I | Not supported; Ada only | — |
| Signals and groups | P | Typed event channels + tags/groups queries in the World | M2 |
| GDExtension (C ABI native extensions) | P | Ada plugin API + a stable C ABI for native libraries (§6.12) | M7 / M9 |
| Statically linked engine modules, feature-stripped builds | M | Module flags in gprbuild + export build profile that strips unused services | M4 |
| Low-level server access bypassing scene layer | C | Services addressed by handles (§3) | M1 |

### A.14 Shaders

| Godot capability | 0.4 | Poncelet equivalent (current) | Milestone |
|---|:-:|---|---|
| Custom 2D vertex/fragment/light shaders | P | Poncelet shader dialect with 2D stages | M3 |
| Custom 3D vertex/fragment/light/sky/fog shaders | P | 3D stages incl. sky and fog volumes | M5–M6 |
| GLSL-like shader language | C | Shader dialect (§6.2) | M3 |
| Visual shader editor + plugins | P | Node-graph material editor generating dialect code; custom nodes via plugins | M7 |
| Procedural textures via shaders (drawable textures) | M | Render-to-texture + compute writes to textures | M6 |

### A.15 Networking

| Godot capability | 0.4 | Poncelet equivalent (current) | Milestone |
|---|:-:|---|---|
| TCP/UDP sockets | M | Pc.Net.Sockets on GNAT.Sockets (non-blocking, IPv4/IPv6) | M6 |
| HTTP/HTTPS client (bundled certs) | M | HTTP/1.1 client in Ada + TLS via mbedTLS binding (Apache-2.0), CA bundle | M6 |
| High-level multiplayer: RPC, replication, authority, spawners | M | Entity replication + typed remote calls with contracts, authority model, interest management, rollback-friendly deterministic sim | M7 |
| ENet reliable UDP | M | ENet binding (MIT) or own reliable-UDP layer (SPARK candidate) | M7 |
| WebSocket client/server | M | WebSocket over Pc.Net (also used by MCP HTTP transport) | M6 |
| WebRTC | M | libdatachannel (MPL-2.0, dynamic link) binding | M9 |
| UPnP port mapping | M | miniupnpc binding (BSD) | M7 |
| Steam networking (relay) | P | Steam Networking Sockets backend behind Pc.Net when Steam is enabled | M8+ |

### A.16 Import

| Godot capability | 0.4 | Poncelet equivalent (current) | Milestone |
|---|:-:|---|---|
| glTF 2.0 (incl. runtime loading) | P | cgltf import (§6.2) + runtime glTF loading API | M5 / M6 |
| FBX | I | Previously out of scope; now planned via ufbx (MIT) importer for parity | M6 |
| .blend via Blender CLI | M | Optional importer calling user-installed Blender for glTF export | M6 |
| Collada (.dae) | I | Not planned (legacy format); convert via Blender | — |
| OBJ | C | §6.2 | M5 |
| Images (PNG, JPG, WebP, TGA, BMP, HDR, EXR, DDS, KTX, SVG) | P | stb_image + tinyexr + libwebp + KTX2/DDS + SVG | M3 / M5 |
| Custom import plugins | M | Importer plugin interface (§6.12) | M7 |
| MikkTSpace tangents | C | §7 | M5 |

### A.17 Internationalisation

| Godot capability | 0.4 | Poncelet equivalent (current) | Milestone |
|---|:-:|---|---|
| Full Unicode incl. emoji | P | UTF-8 throughout; colour emoji fonts | M2 / M6 |
| System font fallback | M | Fontconfig / DirectWrite / CoreText fallback lookup | M6 |
| CSV / gettext catalogs, POT/PO generation | C | §8.3 | M2 |
| Automatic translation of UI, tr() | C | String IDs (§8.2) | M2 |
| Plurals and contexts | C | §8.3 | M2 |
| Pseudolocalisation | C | §8.3 | M2 |
| BiDi/RTL | I | See GUI row (§8.4) | M9 |

### A.18 Windowing & OS

| Godot capability | 0.4 | Poncelet equivalent (current) | Milestone |
|---|:-:|---|---|
| Multiple windows, popups, always-on-top, transparent/borderless, passthrough | M | SDL3 multi-window service (editor docks can float), window flags | M5 |
| Fullscreen (borderless/exclusive) | P | Settings menu covers resolution; add exclusive/borderless modes | M3 |
| Native file dialogs | M | SDL3 file dialog API | M3 |
| Tray icon, global menu (macOS), CSD | M | SDL3 tray/menu where supported | M9 |
| Execute processes, open URLs, CLI args | P | Pc.OS (spawn, open URL, args) — CLI already exists | M2 |
| Clipboard, IME | C | §6.10 | M2 |

### A.19 Files & resources

| Godot capability | 0.4 | Poncelet equivalent (current) | Milestone |
|---|:-:|---|---|
| Text and binary scene/resource formats | C | §6.7, §12.1 | M1 |
| File access, compression, encryption | P | Pc.Files with zstd/deflate; optional pack encryption (AES via mbedTLS) | M4 |
| JSON / INI-style config read-write | C | Canonical JSON + TOML (§12.1) | M1 |
| XML parsing | M | Minimal XML reader (Ada) for tooling/import | M9 |
| Runtime loading of images/audio/fonts/zip without import | M | Runtime loaders API (mods, user content) | M6 |
| Pack files, single-file executables, extra packs for mods/DLC | P | `.ppak` + embedded-in-executable mode + additional packs for mods/DLC | M4 / M7 |

### A.20 Misc / production

| Godot capability | 0.4 | Poncelet equivalent (current) | Milestone |
|---|:-:|---|---|
| Video playback (Theora) | M | Video player: Theora (BSD) or AV1 via dav1d (BSD) + Opus | M6 |
| Movie Maker mode (offline capture, perfect pacing) | M | Deterministic capture to PNG sequence/WAV (+ ffmpeg if installed); also used by MCP `capture_frames` | M4 |
| CLI for automation, CI export | C | `poncelet` CLI (§12.2) | M0–M4 |
| Shell completion | M | bash/zsh/fish completion generated from CLI definition | M2 |
| Coloured rich console output | M | Structured log levels with colour; JSON logs for tools | M1 |
| Webcam/camera feed | M | SDL3 camera API | M9 |
| Object/memory profiler (object DB snapshots) | M | World snapshot diff + allocation stats panel | M5 |
| XR (OpenXR, visionOS, WebXR) | I | OpenXR VR/AR: PC VR (Vulkan) and standalone Android headsets, hand tracking, passthrough; WebXR after WebGPU (§10.4) | M7 (render hooks) / M9 (OpenXR) |
| Mobile features (IAP, ads, PiP) | I | Optional platform-service modules: in-app purchases, ads, notifications, PiP behind `Pc.Services.Store`/`Platform` | M9 |
| Licence: open source MIT | I | Commercial, all rights reserved (§15) | — |

### A.21 Signature features beyond Godot (draft 0.9, updated 0.10: all first-release)

These rows are **not Godot parity items**. Godot already has useful pieces in the same neighbourhood
(procedural noise, fog volumes, decals, perspective cameras, some 2.5D helpers); the point of §19 is that
Poncelet treats projective geometry, fractals, and density as first-class pillars with a shared Ada core,
seeded determinism, and tool/CLI/MCP exposure. The first column is the Poncelet feature.

| Poncelet feature | vs typical Godot-level offer | Plan | Milestone |
|---|---|---|---|
| Homogeneous points/lines/planes, points at infinity, incidence, cross-ratio, duality (`Pc.Math.Projective`) | Beyond (math libraries exist; not a first-class engine/editor pillar) | SPARK-ready core types and contracts, §19.1 | M0 |
| Projective camera model, infinite far plane, 2.5D / fake-3D projection modes | Beyond (cameras and some 2.5D helpers exist; not a projective model) | Built on `Pc.Math.Projective`; all renderer tiers | M3 (2.5D) / M5 (camera, far plane) |
| Perspective grids, horizon and vanishing-point guides | Beyond | 2D/3D editor guides, CLI/MCP `set_perspective_guides` | M3 |
| Homography-based decals, projectors, texture mapping | Beyond clustered decals (A.5) | Projectors from plane-to-plane homographies; cheap fallback on compatibility | M6 |
| Anamorphic / non-standard projections; portals and mirrors; AR camera calibration | Beyond | Anamorphic cameras in M5; portal/mirror math and planar mirrors in M6; portal rendering and still-image calibration toolkit in M7 (OpenXR runtime remains M9) | M5 / M6 / M7 |
| Fractal noise (fBm, ridged, domain-warp), Diamond-square | Extends procedural noise (A.6) | Pure Ada `Pc.Noise`, seeded, hashed | M3 |
| L-systems and IFS generators | Beyond | Deterministic mesh/stroke generation, iteration caps | M3 |
| Escape-time Mandelbrot/Julia visual-shader nodes + mobile/web budgets | Beyond generic shader graphs (A.14) | Nodes in the M7 visual shader editor; caps per tier | M7 |
| Seeded fractal world / level generation | Beyond | `poncelet gen world`, MCP `generate_world`, seed parity | M7 |
| SDF raymarched fractals (Mandelbulb) with LOD; infinite-zoom perturbation | Beyond | Distance estimators in M5; raymarch + LOD/fallback and deterministic double/perturbation zoom in M7; high-end first, mesh/impostor on compatibility | M5 / M7 |
| Density-field resource (scalar/vector, dense/sparse, text `.pres`) | Beyond fog volumes as a one-off | `Pc.Density`, canonical serialisation, editor painting | M6 |
| Fog/smoke driven by density; scatter; AI heat maps; density physics queries | Extends volumetric fog (A.5) and queries (A.7) | Shared field resource; works headless and on web | M6 |
| Basic marching-cubes isosurface from a density field | Beyond | Baked mesh from a documented field, golden hash | M7 |
| Fractal density volumes (noise/IFS → field → fog/scatter/mesh) | Beyond | Full first-release pipeline (noise/IFS → field → fog/scatter/mesh, including dual contouring) | M7 |
| SPH fluid density; destructible voxels; dual contouring | Beyond | 2D SPH and voxel resources in M6; 3D SPH, voxel destruction/physics, and dual contouring in M7 | M6 / M7 |

### A.99 Intentional differences (honest summary, draft 0.7)
The earlier "desktop only" differences are **removed**: web, Android, iOS, consoles (via partners), XR, the
compatibility/mobile renderers, RTL text, and a community asset library are all planned (see rows above). Remaining differences:
1. **Game code language:** compiled **Ada with contracts** instead of GDScript or C#. Iteration speed comes from hot
   reload and data assets (§6.6a).
2. **Editor hosts:** the editor runs on desktop (Windows, macOS, Linux). Godot's experimental Android/web editors
   have no Poncelet equivalent. Games still target every platform.
3. **No Collada import** (convert via Blender). FBX is planned (ufbx).
4. **Licence and business model:** commercial, all rights reserved, sold on Steam, instead of MIT/open development.
   Curated paid template packs sit alongside the community library.

## Appendix B. Reference implementations (draft 0.8)
Four packages from Robert's game bots (received 2026-11-19, studied in full; not committed to this repository).
They are read for architecture, interfaces and test method. Engine code is re-written for `Pc.*` with contracts; it
is not copied.

### B.1 The four packages

| Package | Language / build | Version | Contributes to Poncelet |
|---|---|---|---|
| **Reckoning** (CDDA In ADA) | Plain Ada → wasm32 (AdaWebPack), one HTML file, gist + githack | v0.54 | Host ABI shape (≈45 integer exports), command codes with time costs, numeric event queue + one host switch as the localisation seam, packed view cells, look layers, byte-blob save idea, music context, radio SNR, in-game confirms, stamp + `version.txt` + curl-verify, 200-seed native batch (`auto_run`) and summary, rich F9 JSON; **first template pack** (§13.4) |
| **Roguelike Jungle** | Plain Ada, native + wasm32 from the same sources, gprbuild | v0.04 (`771f5c3`) | Named RNG streams (xorshift32*), binary fixed point, tick clock with real-time-with-pause, one `Order` entry point with busy/time cost and result enum, state hash + native-vs-WASM chunked determinism test, **last-chance handler**, explicit export list and link flags, toolchain fetch script, F9 report with schema, required "expected" note, Markdown variant and stamped screenshot, phone-viewport smoke test |
| **Colony4X** | Plain JavaScript + canvas, no build | v0.9 (`90b4ee2`) | Pure seeded sim with one `act`, mulberry32, FNV-1a state hash, **saves = action logs** with exact replay, data-only events (~60 kinds), `col-feedback` report, fair lineups for batches, `sim.js` batch stats, WebKit/Firefox/Chromium UI tests |
| **Hood Simulator** | TypeScript + React + Three.js (Vite) | 1.2.82 (`099e8c9`) | Two seeds (city + AI), one explicit `Rng` (mulberry32), command API returning ok/fail, presentation events with sequence/time/pos, one-way sim→view dependency, save schema checked at build, autoplayer rules from 80+ versions, decision matrix and per-order traces, periodic report files with `BUILD`/`TEST KEY`/`SUMMARY` lines, build-branch publishing |

### B.2 Conflicts and how Poncelet resolves them

| Topic | Reference variants | Poncelet standard |
|---|---|---|
| RNG algorithm | LCG global (Reckoning), xorshift32* streams (RJ), mulberry32 (Colony4X, Hood) | `Pc.Random`: explicit state, named streams, **PCG32 default**, the three others as bit-exact compatibility generators for ports (§6.15.1) |
| Seeds | one seed (Reckoning, RJ, Colony4X), two seeds (Hood) | world seed + AI seed, streams derived from them |
| Runtime checks | `-gnatp` (Reckoning) vs. checks on + LCH (RJ) | checks on in dev on all targets; release suppression only for SPARK Silver units (§10.5.2) |
| Export list | `--export-dynamic` (Reckoning) vs. explicit `--export=` list (RJ) | explicit list generated from `Pc.Host_ABI`; `--export-dynamic` only in a debug variant |
| `memcpy` & co. | JS imports (Reckoning) vs. objects from `libgnat.a` (RJ) | runtime objects; the host does not implement them |
| LCH payload | line only (RJ) | file pointer + line, host reads the file name from memory |
| Command shape | `act (code, dx, dy)` (Reckoning), `order (code)` + `last` (RJ), `act (S, {type, ints})` (Colony4X), API functions → ok/fail (Hood) | `pc_act (actor, verb, a1, a2, a3) → status` (≥ 0 time cost, < 0 result code), same entry for player, NPCs and autoplayer |
| Events | 256 (code, arg) cleared per act, silent overflow (Reckoning); counter only (RJ); kind + data (Colony4X); sequence/week/minute/kind/actor/pos (Hood) | ring of `{sequence, tick, code, actor, a1, a2, a3}` with `dropped` counter and registered names (§10.5.4) |
| Time | turns (Reckoning, Colony4X), minute ticks + JS speed (RJ), weekly turns with minutes (Hood) | per-project tick unit, `pc_tick (n)` chunk-independent (§6.15.2) |
| State hash | none (Reckoning), custom 32-bit (RJ), FNV-1a 32 over JSON (Colony4X) | 64-bit FNV-1a over canonical serialisation + named sub-hashes |
| Saves | raw record bytes via `Unchecked_Conversion`, version check only (Reckoning); action log (Colony4X); JSON schema (Hood) | replay save (log) **and** versioned `Pc.Serial` snapshot in a host buffer; schema documented |
| Save compatibility | "doesn't matter" (Reckoning, house rule) | engine formats are versioned from M0 and frozen at the M8 maturity gate; templates may stay unversioned before that |
| F9 report | four different shapes | `poncelet-report` v1 (§6.15.6), plus the compact `BUILD`/`TEST KEY`/`SUMMARY` text form for long runs |
| Version check | gist raw (Reckoning, RJ) vs. build branch on githack (Hood) | both publish targets; `version.txt` always polled from a raw origin, never githack (§10.6) |
| Rendering | canvas text grid (Reckoning, RJ, Colony4X), Three.js (Hood) | same Ada snapshot/draw-list ABI; reference web host uses WebGL2 (canvas only for the M1 spike) |
| Native runtime | dynamic `libgnat-14.so` (RJ runner failed on another machine) | static GNAT runtime in all native binaries; compile clean on GNAT 12.3 and 14 |

### B.3 Consolidated lessons (deduplicated)
1. Keep checks on in development; a last-chance handler turns a silent WASM failure into file + line on screen.
2. The core is a hard subset: no tasking, heap, handlers, Text_IO or floats where results must match; fixed point instead.
3. No hidden globals: pass world and RNG state explicitly (needed for SPARK and for exact replays).
4. One RNG source, explicit and saved with the world; never `Math.random` or wall-clock time in rules.
5. Two seeds (world and AI) and vary both in batches; rotate factions/start slots for fairness.
6. One command entry point for players, NPCs and autoplayer; every command has a time cost; failures still cost time.
7. Determinism must not depend on frame chunking; test native vs. WASM with uneven chunks.
8. Log actions, not just state: a bug report is seeds + action log; saves can be logs.
9. Events carry numbers, not text; one host table turns them into words, sound and voice (the localisation seam).
10. Never expose a record address through `Unchecked_Conversion`; use buffers with a schema and a version.
11. The autoplayer is a real player: public API only, player-knowable data only, coverage of every action.
12. Estimates use the simulation's own pathing; every set-aside rule needs an observable way back; one fitness currency.
13. Prefer report files and numeric decision matrices to live debugging; a fix names the metric it should move.
14. A visible stamp on every screen, in `version.txt` and in every report; bump on every push.
15. githack caches: poll raw origins, curl-verify before announcing, give a commit-pinned URL when stale.
16. Mobile audio unlocks on the first tap anywhere; separate SFX/Music/Ambient/Voice buses; mute ambience when hidden.
17. In-game confirms only, and warnings only when they are real.
18. Single-file HTML for code and small data; big media beside it, loaded lazily.
19. Test in WebKit, Firefox and Chromium at a phone viewport; map both `key` and `code`.
20. The `Proxy`-returns-0 import guard prevents startup traps but hides typos; log unknown imports in dev.
21. Link native tools statically and keep code warning-free on both GNAT 12 and GNAT 14.

### B.4 Engine vs. template content

| Goes into the engine (`Pc.*`, hosts, tools) | Stays in template packs |
|---|---|
| `Pc.Host_ABI`, event ring, buffers, LCH import, export-list generator | Command/verb sets, event code lists, stat ids |
| `Pc.Random` (+ compatibility generators), `Pc.Fixed`, `Pc.Hash` | Fixed-point unit types of a game (money, heat, …) |
| Command dispatcher with busy/time cost, autoplayer interface, stuck detection, coverage counters | Autoplayer valuations, personalities, utility brains |
| 2D grid/tilemap with layers, Bresenham LOS, FOV, fixed-bound BFS pathing, route graph + fog of war | Tile enums, glyph/colour tables, opacity and cone constants, map generators |
| Link-quality 0–100 with hard cut, audio context ids, radio SNR effect in the host | Antennas, jammers, music tracks, voice lines |
| Dialogue/look/examine id protocol, in-game confirm, danger-query protocol | Dialogue text, Look/Help text, danger rules |
| `poncelet sim`, `--compare`, report format, periodic report files, decision-matrix support | Metric choices and summary groupings of a game |
| Web host: boot, stamp, update check, publish + curl-verify, audio buses, input, saves, F9, smoke tests | Splash effects (Reckoning rain), menus' content, achievements' rules |
| Mission/achievement **bitfield hook** | Missions and achievements themselves |

## Changelog

- **0.10 (2026-11-20).** All three signature pillars are first-release core features (decided 2026-11-20).
  Everything previously marked "later / M9" in §19 — anamorphic projections, portals/mirrors, AR calibration,
  Mandelbulb raymarching, infinite zoom, SPH fluids, destructible voxels, dual contouring — moves into M5–M7.
  Q19 resolved; Q20 (showcase pack) left open. Schedule estimate revised in §16. Implementation plan updated
  (see that file for the new totals).
- **0.9 (2026-11-20).** Name kept: Poncelet, after Jean-Victor Poncelet and projective geometry. New §19
  **Signature features beyond Godot** (projective geometry, fractals, density) with a first-release vs later cut
  (superseded by 0.10). Mentioned in the vision, scope, architecture, math, cameras, editor, CLI/MCP, and
  milestones. Appendix A.21 adds beyond-parity rows.
- **0.8 (2026-11-19).** Reference implementations (Reckoning, Roguelike Jungle, Colony4X, Hood Simulator),
  host ABI v1, F9 report format, web host standard, seed parity; plan at 119 steps. See the draft 0.8 status
  line and Appendix B for the full record.
