# Poncelet Engine — Design Document (Draft 0.4)

> Name: **Poncelet** (chosen 2026-11-19, after Jean-Victor Poncelet, projective geometry; see `docs/NAME_CHECK.md`).
> Author: Robert Boettcher, with Grok Bot. Status: draft 0.4, 2026-11-19 (full 2D + 3D; editor is the first Steam product; Linux-first development, Steam/Apple deferred; AI-tool support is a core feature).
> Clean-room: Godot (MIT) was studied **for architecture only**. No Godot code, text, or assets are copied here.
> Everything below describes our own design.

---

## 1. Vision

A commercial **desktop game engine and editor with its core in Ada**, aiming to be **as capable as Godot in both 2D and 3D**.
The **first Steam product is the Poncelet editor itself** (sold as a Steam software/tool), bundled with starter
**template packs**. Games made with it ship on **Steam** for
**Windows, macOS, and Linux** in **seven UI languages**. Its game logic is written so that the parts that
matter (simulation, rules, save files, maths) can later be **proven with SPARK without being rewritten**.

What makes it different:

1. **Correctness first.** The core is Ada 2012/2022 with contracts. Selected packages graduate to SPARK.
   Determinism (fixed-timestep simulation) makes replays, save files, and tests reliable.
2. **Thin, replaceable platform layer.** Windowing, GPU, audio, input, and Steam sit behind small Ada interfaces
   that bind to stable C APIs. The core never depends on them.
3. **Editor as product.** The editor is first-class, with polished UX and a UI localised in all seven languages.
   It isn't a developer-only tool.
4. **Templates.** Robert's existing games (built with other bots) become template packs a user can pick,
   using a pack format defined early (§13).
5. **AI-tool friendly by design.** Users drive Poncelet with their *own* AI tools (Cursor, Claude, ChatGPT, Grok,
   Copilot, ...) through text formats, a JSON CLI, and a built-in MCP server. Poncelet bundles no model and stores no AI keys (§12).
6. **Localisation is built in, not bolted on.** All user-visible text goes through string IDs from day one.

## 2. Scope and non-goals

### In scope (v1 = first sellable editor release)
- Desktop targets: Windows 10/11 x86-64, macOS 12+ (universal: arm64 + x86-64), Linux x86-64 (Steam Linux Runtime).
- **Full 2D and full 3D, designed together from the start** (target: feature parity with Godot for typical indie
  games). 2D: sprites, tilemaps, 2D lights/shadows, particles, 2D physics. 3D: PBR, glTF import, lights and shadows,
  3D physics, skeletal animation, navigation. One architecture, one scene format, one editor.
  **Delivery order:** 2D ships first, then 3D lands on the same architecture (§16). Nothing in the 2D work may
  assume 2D-only.
- UI languages: **English (source), German, French, Russian, Simplified Chinese, Spanish, Italian**, for both
  the editor and games.
- **The editor is the product:** project manager, 2D and 3D viewports, inspector, asset pipeline, localisation
  editor, play-in-editor, exporter for all three OSes, and template browser.
- **Template packs** bundled with the editor. Their contents will be decided later with Robert's other game bots.
- Steamworks: for the editor product (achievements optional, Workshop for template packs later) and as a ready
  integration for users' games (achievements, stats, cloud saves, rich presence, overlay, language).

### Non-goals (for now)
- Mobile, consoles, VR/XR, and **Steam Deck-specific optimisation**. The Linux build will very likely run on Deck,
  because Deck runs ordinary Linux/Proton builds. We just won't tune controls, UI scale, or performance for it.
- A general-purpose scripting VM such as a GDScript equivalent. Game logic is Ada, compiled in (see §6.6).
- Matching every niche Godot feature (XR, mobile, web export, C#). We match Godot's **core 2D/3D capability
  for desktop games** and differ through correctness, Ada, and a curated template-first workflow.
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
|  Math | World/ECS | Physics 2D/3D API | Anim | Nav | Serial    |   no platform deps
|  Rules | I18n | Scene 2D/3D | Asset DB | Template packs          |
|  RNG (deterministic) | Bounded containers | Time/Step           |
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
| OS: Windows (D3D12/Vulkan) | macOS (Metal) | Linux (Vulkan)   |
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
- No exceptions on the hot path. Contracts express the domain.

### 6.2 Rendering (service; SDL_GPU backend)
One render service with a shared core (resources, render graph, cameras, materials) and two pipelines that can
be composited in the same frame: a 2D canvas over or under a 3D world, and 3D inside 2D via viewports.

**Shared core**
- GPU resources via SDL_GPU (Vulkan/Metal/D3D12) behind Ada handles: buffers, textures, samplers, pipelines.
- A small **render graph** (passes declared per frame, with automatic barriers and transient render targets)
  so 2D, 3D, shadows, and post-processing are just passes.
- **Cameras:** 2D (zoom, rotation, limits, smoothing) and 3D (perspective, orthographic, frustum offset).
  Multiple viewports and render-to-texture (minimaps, split-screen, editor previews).
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
- Hot reload: data assets reload live. Code changes need a rebuild (fast with gprbuild incremental builds).

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

### 6.9 Input
- Action mapping (`"move_left"`, `"confirm"`) with rebinding, keyboard/mouse and gamepads via SDL3.
  Steam Input works transparently through SDL. Rebinding UI is localised.

### 6.10 UI (runtime)
- Our own retained-mode UI in Ada: layouts (box, grid, anchors), widgets (label, button, slider, list,
  text field with IME), and themes. **Every label takes a string ID, not a literal.**
- IME support (needed for Chinese text entry) uses SDL3 text-input/IME events.
- The **same toolkit powers the editor** (docking, trees, property grids, code/text views, theming, DPI scaling).

### 6.11 Animation
- Keyframe tracks on any component property (2D and 3D), easing curves, events, and tweens.
- 3D skeletal animation from glTF, blend trees and state machines (data assets), root motion, IK (two-bone, look-at).
- 2D: sprite-frame animation, 2D skeletons (bones + mesh deform) post-v1.

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
9. **Deterministic RNG** (`Pc.Random`, e.g. PCG32/xoshiro) with explicit state passed in, not a global.
10. **Tooling:** compile with `-gnat2022 -gnata -gnatwa -gnatVa -gnatyy`, use AUnit or Alire-based tests, and
    run **GNATprove** in CI at `--mode=check` (flow) from M2 onward, then `--level=2` proof on the first
    candidate packages: `Pc.Math`, `Pc.Serial`, `Pc.I18n.Plural`, and `Pc.Physics.Broadphase_2D`.
11. Mark packages with `pragma SPARK_Mode (On)` as soon as they pass flow analysis. The goal is that
    "SPARK later" means adding `Global`, `Depends`, and loop invariants, not refactoring.

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
- Scene editor with **2D viewport** (gizmos, snapping, tilemap painter, 2D light/occluder editing) and
  **3D viewport** (move/rotate/scale gizmos, grid and snapping, camera fly/orbit, lighting preview, lightmap baking).
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

**Browser/WebAssembly preview for githack demos:** githack (raw.githack.com) serves static files from a public
repo, so HTML/JS/WASM pages work there. Compiling Ada to WASM needs **GNAT-LLVM**, which is still experimental
(projects like AdaWebPack exist), and SDL3 + emscripten + WebGPU/WebGL adds more risk.
Recommendation: **not in v1.** For githack, publish **static HTML pages** (design docs, screenshots, GIFs or
short videos, an interactive localisation report). Revisit a WASM build of `pc-core` with a JS renderer as an
experiment after M4. `pc-core` being platform-free makes this possible later.

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
| Resources (materials, physics shapes, animations, themes, tilesets) | `*.pres` | canonical JSON |
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
- Commands: `new` (from template), `build`, `run`, `test`, `export`, `screenshot` (render a scene/camera to PNG
  headlessly or offscreen), `validate`, `inspect` (scene tree, entity, component, resource, settings), `fmt`,
  `docs` (API lookup), `schema`, `mcp` (start the MCP server).
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
    `capture_frames` (short sequence), `read_logs`, `run_tests`, `build` (returns structured diagnostics).
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
  engine changes don't break templates.
- Templates can later be distributed as Steam Workshop items or DLC.

## 14. Repository layout (planned; nothing created yet)

```
poncelet/                     (public, source-available — see LICENSE)
  alire.toml  poncelet.gpr
  core/        src/pc-*.ads|adb   tests/
  platform/    sdl3/ freetype/ harfbuzz/ steam/  (bindings; Steam SDK NOT committed)
  services/    render2d/ audio/ input/ text/ window/ steam/
  runtime/     main entry, loop
  editor/      imgui-based dev editor
  tools/       catalog compiler, asset packer, string-id generator, template tool, API doc generator
  cli/         `poncelet` headless CLI (JSON output)
  mcp/         MCP server (stdio + local HTTP), shares Pc.Tools with the editor
  schemas/     JSON Schemas for all project file formats (versioned)
  AGENTS.md  llms.txt   (AI-tool guidance for the engine repo)
  templates/   <one dir per template>
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

| M | Name | Contents | Exit criterion |
|---|---|---|---|
| **M0** | Skeleton + CLI | Repo, Alire/gprbuild projects, `pc-core` (Math 2D+3D, contracts, tests), CI on all three OSes, coding rules, LICENSE, `AGENTS.md`. **`poncelet` CLI v0:** `new`, `build`, `test`, `validate` with the JSON envelope + structured GNAT diagnostics | `alr build` + tests green on Linux, Windows, and macOS; `poncelet build --json` reports file/line/column |
| **M1** | Pixels + formats | SDL3 binding, window/input, SDL_GPU render core + render graph, 2D batcher, fixed-step loop, shader pipeline. **All text formats + JSON Schemas specified** (`.pscene`, `.pres`, `.pimport`, `.ponpack`), canonical writer; CLI adds `run`, `screenshot`, `inspect`, `fmt`, `schema` | A sprite moves at 60 Hz; `poncelet screenshot` renders headlessly on Linux; format specs reviewed |
| **M2** | World, UI, i18n | ECS World with 2D/3D transforms, scenes, FreeType/HarfBuzz text, **seven-language i18n** + pseudo-locale, runtime/editor UI toolkit (docking, inspector widgets), GNATprove flow in CI. **MCP server v1** (stdio + local HTTP) in editor and headless mode: query/modify/play/screenshot/logs/tests tools, undo integration, confirmations; generated API docs (`api_docs`) | Editor shell switches live between seven languages; an external AI client creates an entity via MCP, and it is undoable in the editor |
| **M3** | 2D complete | Physics2D (own Ada), 2D lights/shadows, tilemaps, particles, animation, audio, save/load, input rebinding; editor 2D viewport, inspector, asset import, localisation editor, play-in-editor | A 2D template game built entirely in the editor |
| **M4** | Export (unsigned) | Exporter with `.ppak`; Linux export tested on Linux Mint; Windows/macOS exports **built in CI** (unsigned, smoke-tested only); `Pc.Services.Steam` stub; template-pack import | Exported template game runs on Linux Mint; Windows/macOS CI artifacts build and pass smoke tests |
| **M5** | 3D foundation | Clustered forward+ renderer, PBR, glTF import, cameras, directional/omni/spot lights, cascaded shadows, IBL/sky; Jolt 3D physics behind Ada API; 3D viewport and gizmos | A 3D template scene imported from glTF, lit, with physics, editable in the editor |
| **M6** | 3D complete | Skeletal animation + blend trees, lightmap baker, reflection probes, SSAO/SSR/post FX, LODs and culling, navmesh, material editor, particles 3D | 3D template game playable and exportable |
| **M7** | Maturity + product polish | Onboarding, tutorials, undo everywhere, crash recovery, stability/soak testing, editor localisation QA by native speakers, template packs (contents from the group chat) finalised and signed, EULA, documentation | Maturity gate: no known crashers, stable formats, 3D at least early-access quality |
| **M8** | Platforms + Steam (**first sellable**, gated on M7) | Only after the maturity gate: Steamworks partner sign-up, real Steamworks body, Apple Developer ID, macOS code signing + notarisation, Windows signing (optional), Steam Runtime Linux build, **real-hardware testing on macOS and Windows** (borrowed or rented machines / testers), SteamPipe depots, store page in seven languages | Steam review passed; **Poncelet Editor + template packs released** |
| M9+ | Growth | Own Ada 3D physics backend, SPARK proofs of core packages, dynamic GI, Workshop for packs, paid template DLC, visual material graph, optional WASM experiment | — |

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
- **Scope (Godot-level 2D + 3D + polished editor):** this is the largest risk. Mitigations: SDL_GPU + Jolt + cgltf
  instead of writing our own; ship 2D first; allow 3D as early access at M8; cut dynamic GI and node-graph
  editors to post-v1.
- **Bundling GNAT with a commercial editor:** use the FSF GCC toolchain under the GCC Runtime Library Exception
  and confirm redistribution terms (GPLv3 for compiler binaries means source offer obligations for the toolchain itself).
- **SDL_GPU feature ceiling for advanced 3D** (no ray tracing or mesh shaders): acceptable for v1. Revisit option C later.

## 18. Open questions for Robert
1. ~~2D only for v1, or is 3D needed soon?~~ **Answered (2026-11-19):** Poncelet must match Godot in 2D **and** 3D.
   Ship 2D first, then 3D on the same architecture.
2. **What engines and languages were your other bot-built games written in** (Godot/GDScript, JS, Python, Ada)?
   This decides whether templates are imports or ports. (Pack *contents* will be decided in the group chat.)
3. ~~What do we sell first?~~ **Answered (2026-11-19):** the engine + editor as a Steam software product, bundled with
   template packs. **New, to confirm:** the end-user licence for games made with Poncelet. Proposal: users own and may
   sell their games royalty-free, the runtime may be redistributed inside exported games, and engine/editor source
   may not be redistributed (§15).
4. ~~Final name~~ **Answered:** Poncelet.
5. ~~macOS distribution / Apple Developer ID~~ **Answered (2026-11-19): deferred.** No sign-ups (Apple or Steam)
   until Poncelet is stable and mature. Development is on Linux Mint, and macOS/Windows are compile and smoke-checked in CI.
   Signing, notarisation, and Steam come in M8.
6. Spanish: Spain (`spanish`) or Latin American (`latam`), or both?
7. Should CI (GitHub Actions on all three OSes) run in the public repo, or in a private mirror (to keep build
   details and secrets separate)?
8. Is professional translation planned, or bot-translated with native-speaker review?
9. Use Alire-managed toolchains (recommended for Windows and macOS), or stay on system GNAT 12.3?
