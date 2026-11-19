# Poncelet Engine — Design Document (Draft 0.1)

> Name: **Poncelet** (chosen 2026-11-19, after Jean-Victor Poncelet, projective geometry; see `docs/NAME_CHECK.md`).
> Author: Robert Boettcher, with Grok Bot. Status: first draft, 2026-11-19.
> Clean-room: Godot (MIT) was studied **for architecture only**. No Godot code, text, or assets are copied here.
> Everything below describes our own design.

---

## 1. Vision

A small, commercial-quality **desktop game engine with its core in Ada**. It ships games on **Steam** for
**Windows, macOS, and Linux** in **seven UI languages**. Its game logic is written so that the parts that
matter (simulation, rules, save files, maths) can later be **proven with SPARK without being rewritten**.

What makes it different:

1. **Correctness first.** The core is Ada 2012/2022 with contracts. Selected packages graduate to SPARK.
   Determinism (fixed-timestep simulation) makes replays, save files, and tests reliable.
2. **Thin, replaceable platform layer.** Windowing, GPU, audio, input, and Steam sit behind small Ada interfaces
   that bind to stable C APIs. The core never depends on them.
3. **Templates.** Robert's existing games (built with other bots) become starter templates a user can pick,
   with a defined import path.
4. **Localisation is built in, not bolted on.** All user-visible text goes through string IDs from day one.

## 2. Scope and non-goals

### In scope (v1)
- Desktop targets: Windows 10/11 x86-64, macOS 12+ (universal: arm64 + x86-64), Linux x86-64 (Steam Linux Runtime).
- **2D first** (sprites, tilemaps, UI, particles, 2D physics). The 3D path is designed for but not built in v1 (open question Q1).
- UI languages: **English (source), German, French, Russian, Simplified Chinese, Spanish, Italian**.
- Steamworks: achievements, stats, cloud saves, rich presence, overlay, language detection. Later: workshop and leaderboards.
- A development editor (tooling build) plus a runtime build for shipping games.
- A template and project-import system.

### Non-goals (for now)
- Mobile, consoles, VR/XR, and **Steam Deck-specific optimisation**. The Linux build will very likely run on Deck,
  because Deck runs ordinary Linux/Proton builds. We just won't tune controls, UI scale, or performance for it.
- A general-purpose scripting VM such as a GDScript equivalent. Game logic is Ada, compiled in (see §6.6).
- Competing with Godot or Unity on breadth. We aim for depth on a narrow set of game types.
- Open-source distribution. The repository is public but **source-available, all rights reserved** (§14).

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
| **Scene tree** of polymorphic nodes (deep inheritance, signals, virtual callbacks) | **Adapt** | Pointer-heavy, dynamic-dispatch trees are hard to prove. We keep the **authoring idea** (nested scenes, instancing, a hierarchy in the editor) but the runtime stores data in an **ECS-style World**: entities are handles, components live in bounded tables, and hierarchy is a `Parent` component. |
| Variant/dynamic object model with reflection | **Avoid** at runtime / **Adapt** for tools | Runtime is statically typed. Reflection for the editor and serialisation comes from **generated** Ada (a code generator emits component descriptors), not from runtime introspection. |
| **Resources**: reference-counted assets with a text format for scenes and resources, plus a binary format for export | **Adopt** | Text formats for source control (`.pscene`, `.pres`, TOML-like or JSON), and an exporter compiles them to binary packs with checksums. Assets are referenced by stable UUIDs, not paths. |
| **RenderingDevice** abstraction over Vulkan/D3D12/Metal, plus a GL compatibility renderer | **Adapt** | We don't write our own RHI. We use **SDL3 GPU**, which already abstracts Vulkan, Metal, and D3D12 (§6.2). |
| **TextServer** with complex shaping (HarfBuzz, ICU, FreeType) and a fallback | **Adopt** | FreeType + HarfBuzz via C bindings. Our seven languages need no RTL, but CJK line breaking and font fallback are required. |
| **Translation domains**, CSV/gettext catalogs, plural rules, locale fallback | **Adopt** | §8. |
| **Export templates**: pre-built runtime per platform plus a data pack | **Adopt** | Runtime binary per OS + `.ppak` data pack. Built per OS in CI (§10). |
| Editor built with the engine's own GUI | **Adapt (later)** | Start with Dear ImGui (via cimgui) as a dev tool and move to our own UI toolkit later (§11). |
| GDExtension (C ABI plug-ins) | **Defer** | Static linking only in v1. A C ABI plug-in API may come later. |
| Fixed physics tick decoupled from render (main timer sync) | **Adopt** | Fixed 60 Hz simulation with an accumulator. Rendering interpolates. Simulation is deterministic. |
| WorkerThreadPool and command queues to servers | **Adapt** | Ada tasks + protected objects for a job system. Simulation stays single-threaded in v1 for provability (§6.5). |
| Many optional modules (codecs, XR, networking) | **Avoid** in v1 | Keep the dependency set small (§7). |
| SCons build | **Avoid** | gprbuild + **Alire** (Ada's package manager and toolchain manager). |

## 4. Architecture overview

```
+---------------------------------------------------------------+
| Game (Ada)  — per-game packages; templates live here           |
+---------------------------------------------------------------+
| pc-game-api : Systems, Events, Input actions, UI, Save/Load    |   Plain Ada, contracts
+---------------------------------------------------------------+
| pc-core (SPARK-ready)                                          |
|  Math | World/ECS | Physics2D | Anim | Rules | Serial | I18n    |   no platform deps
|  RNG (deterministic) | Bounded containers | Time/Step           |
+---------------------------------------------------------------+
| pc-services (Ada interfaces, handles only)                     |
|  Render2D | Text | Audio | Input | Window | Files | Steam      |   SPARK_Mode Off
+---------------------------------------------------------------+
| pc-platform (thin Ada bindings to C)                           |
|  SDL3 (+SDL_GPU, audio, input) | FreeType | HarfBuzz           |
|  Steamworks flat C API | stb_image / libpng | Opus/Vorbis      |
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
| C. Ada + raw Vulkan (+ MoltenVK on macOS) + own windowing | Maximum control. | Huge effort (RHI, sync, macOS quirks). Not justified for a 2D-first engine. | Reject for v1 |
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
- Vec2/Vec3/Mat3/Mat4/Quat/Rect/Transform2D as records with `Pre`/`Post`. Example: normalising requires length > epsilon.
- No exceptions on the hot path. Contracts express the domain.

### 6.2 Rendering (service; SDL_GPU backend)
- v1: a **batched 2D renderer**: sprites and atlases, 9-slice, tilemaps, text glyph atlases, primitives,
  simple lights, and post-processing (bloom, colour grading). Render layers and cameras.
- Shaders are written once in HLSL or GLSL and compiled offline to SPIR-V/MSL/DXIL with SDL_shadercross.
  Compiled blobs go into the data pack.
- The 3D path (open question Q1) uses the same service boundary and adds a forward renderer later.

### 6.3 World / ECS (`Pc.World`)
- `Entity_Id` is a generational handle. Entities live in a bounded slot table (capacity set per game at
  compile time or load time).
- Components are stored in **dense arrays per component type** (struct-of-arrays) with sparse index maps.
  Generic packages: `Pc.World.Store (Component_Type, Capacity)`.
- Hierarchy: `Parent` and `Children_Range` components. World transforms are computed in a system pass, not on demand.
- Systems are plain procedures with explicit `in`/`in out` parameters. That makes data flow visible to SPARK
  flow analysis (`Global`/`Depends` can be added later without changing signatures).
- **Scenes** (authoring) are trees in the editor and serialise to `.pscene`. Loading a scene **instantiates
  entities** into the World. Scene instancing and nesting work as they do in Godot conceptually.

### 6.4 Physics 2D (`Pc.Physics2D`)
- Own implementation in Ada: AABB/circle/capsule/convex polygon shapes, a broadphase (uniform grid or
  sweep-and-prune), impulse solver, kinematic character controller, raycasts, triggers.
- This is a prime **SPARK candidate**: absence of runtime errors, bounded iterations, no division by zero.
- Alternative if we go 3D: bind Jolt Physics through a C shim (stays outside the SPARK boundary).

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
- Text: a deterministic, diff-friendly format for scenes and resources (stable key order, UUIDs).
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
| Dear ImGui + cimgui | dev editor | MIT | not shipped in games |

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
    candidate packages: `Pc.Math`, `Pc.Serial`, `Pc.I18n.Plural`, and `Pc.Physics2D.Broadphase`.
11. Mark packages with `pragma SPARK_Mode (On)` as soon as they pass flow analysis. The goal is that
    "SPARK later" means adding `Global`, `Depends`, and loop invariants, not refactoring.

## 10. Build, packaging, and Steam

### 10.1 Toolchain
- **Alire** manages GNAT toolchains per OS (`gnat_native` for Linux x86-64, Windows x86-64 (MinGW), and
  macOS arm64/x86-64) plus `gprbuild`. Robert's local GNAT 12.3 + gprbuild works for Linux development.
  Pin the toolchain version in `alire.toml`.
- Project files: `pc_core.gpr` (library, no deps), `pc_platform.gpr`, `pc_runtime.gpr`, `pc_editor.gpr`,
  and a per-game `game.gpr`. Scenario variables: `BUILD=debug|release|prove`, `TARGET_OS`.
- C dependencies are built with CMake or downloaded as prebuilt SDKs in CI and linked via gprbuild
  `Linker_Options`.

### 10.2 Per-OS packaging

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
- Bind the **flat C API** (`steam_api_flat.h`) with a thin Ada binding in `pc-platform`, then build
  `Pc.Services.Steam` on top. It degrades gracefully: with no Steam, all calls are no-ops, so the same build
  runs from itch.io or for development.
- Features by milestone: init/restart-check, overlay, language, achievements, stats (M4), Cloud saves,
  rich presence (M5), workshop for template and mod sharing (later).

## 11. Editor strategy

1. **M3–M5: dev editor = runtime + Dear ImGui panels** (scene tree, inspector generated from component
   descriptors, asset browser, localisation table view, play-in-editor). This is the cheapest way to a usable tool.
2. **Later (when selling the engine itself as a tool on Steam):** rebuild the editor UI on our own runtime UI
   toolkit, as Godot does. That proves the UI toolkit and gives a consistent, localised editor.
3. Editor UI language is selectable from the same seven languages, and the editor uses the same catalog system.

**Browser/WebAssembly preview for githack demos:** githack (raw.githack.com) serves static files from a public
repo, so HTML/JS/WASM pages work there. Compiling Ada to WASM needs **GNAT-LLVM**, which is still experimental
(projects like AdaWebPack exist), and SDL3 + emscripten + WebGPU/WebGL adds more risk.
Recommendation: **not in v1.** For githack, publish **static HTML pages** (design docs, screenshots, GIFs or
short videos, an interactive localisation report). Revisit a WASM build of `pc-core` with a JS renderer as an
experiment after M4. `pc-core` being platform-free makes this possible later.

## 12. Template system (folding in Robert's existing games)

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

## 13. Repository layout (planned; nothing created yet)

```
poncelet/                     (public, source-available — see LICENSE)
  alire.toml  poncelet.gpr
  core/        src/pc-*.ads|adb   tests/
  platform/    sdl3/ freetype/ harfbuzz/ steam/  (bindings; Steam SDK NOT committed)
  services/    render2d/ audio/ input/ text/ window/ steam/
  runtime/     main entry, loop
  editor/      imgui-based dev editor
  tools/       catalog compiler, asset packer, string-id generator, template tool
  templates/   <one dir per template>
  locale/      engine UI catalogs (en de fr ru zh_Hans es it)
  docs/        DESIGN.md, NAME_CHECK.md, html pages for githack
  ci/          GitHub Actions workflows per OS
```

## 14. Licensing posture (summary)
- The repository is **public** for githack links, with **all rights reserved / source-available, no reuse**.
  See `LICENSE`.
- Third-party notices (`THIRD_PARTY_NOTICES.md`) list SDL3, FreeType, HarfBuzz, Noto, and others.
- Keep the Steamworks SDK, signing keys, and Steam build credentials out of the repo (CI secrets only).

## 15. Milestones

| M | Name | Contents | Exit criterion |
|---|---|---|---|
| **M0** | Skeleton | Repo, Alire/gprbuild projects, `pc-core` with Math + contracts + tests, CI building on all three OSes, coding rules, LICENSE | `alr build` + tests green on Linux, Windows, and macOS |
| **M1** | Pixels | SDL3 binding, window, input, SDL_GPU sprite batcher, fixed-step loop, shader pipeline | A sprite moves at 60 Hz on all three OSes |
| **M2** | World and text | ECS World, scenes (text format), FreeType/HarfBuzz text, **i18n system with all seven languages** + pseudo-locale, GNATprove flow in CI | A demo menu switches live between the seven languages, including Chinese |
| **M3** | Gameplay | Physics2D, animation, audio mixer, save/load, action input with rebinding, runtime UI toolkit | A small playable game (first template candidate) |
| **M4** | Steam and packaging | Steamworks binding (achievements, stats, language), `.ppak` exporter, signed/notarised macOS app, Steam Runtime Linux build, SteamPipe CI upload to a private branch | Builds installable from Steam (private) on all three OSes |
| **M5** | Editor and templates | ImGui dev editor, template manifest/tooling, first 1–2 of Robert's games folded in, Cloud saves | Create a new project from a template and play it |
| **M6** | First sellable | Polish one game (or the template pack) to store quality: localisation QA by native speakers, store page in seven languages, achievements, crash reporting, settings menu (resolution, audio, language) | Steam review passed, release |
| M7+ | Growth | SPARK proofs of core packages, 3D path, workshop, engine-as-a-product editor, optional WASM experiment | — |

## 16. Risks
- **Small Ada game-dev ecosystem:** we own our bindings. Mitigation: keep the C surface tiny and generate bindings.
- **macOS toolchain** (Apple Silicon GNAT, notarisation): validate in M0, not M4.
- **CJK text quality** (line breaking, IME, font size): handle in M2 with a native-speaker review.
- **Determinism with floats across OSes:** use fixed point where it matters, and avoid `-ffast-math`.
- **Scope creep** (3D, editor): stick to 2D and a dev editor until M6.

## 17. Open questions for Robert
1. **2D only for v1, or is 3D needed soon?** This drives renderer, physics, and editor effort.
2. **What engines and languages were your other bot-built games written in** (Godot/GDScript, JS, Python, Ada)?
   This decides whether templates are imports or ports.
3. **What do we sell first:** a game built on the engine, a template pack, or the engine/editor itself as a Steam tool?
   This changes the editor priority and the EULA.
4. **Final name:** pick from the shortlist in `docs/NAME_CHECK.md` before the repo is created
   (repo, package prefix, and trademark search depend on it).
5. **macOS distribution:** are you willing to get an Apple Developer ID (signing and notarisation) and access
   to a Mac or CI macOS runners for testing?
6. Spanish: Spain (`spanish`) or Latin American (`latam`), or both?
7. GitHub account/org for the public repo, and should CI (GitHub Actions on all three OSes) run there?
8. Is professional translation planned, or bot-translated with native-speaker review?
9. Use Alire-managed toolchains (recommended for Windows and macOS), or stay on system GNAT 12.3?
