# Poncelet — Implementation Plan (M0 → M8)

> Companion to `docs/DESIGN.md` (draft 0.6). Copyright (c) 2026 Robert Boettcher. All rights reserved.
> Status: plan only. No code has been committed yet. Dated 2026-11-19.

## How to use this plan

- Steps are small and **verifiable on Robert's Linux Mint laptop** (x86-64, GNAT 12.3 + gprbuild). Each step lists:
  goal, packages/files, contracts and tests, the **command to run**, done criteria, and dependencies.
- Commands assume the repository root. `./build.sh` builds everything that's available, and `./run-tests.sh [name]` runs
  smoke tests and prints `PASS`/`FAIL` per test plus a final summary.
- **Tests:** few smoke tests per step until Alpha (Robert's preference), plus golden hashes and screenshots where
  determinism makes them cheap.
- **Rules for every step:** Plain Ada with SPARK-ready contracts (DESIGN §9), `-gnata` in dev builds, 0 warnings with
  `-gnatwa`, English identifiers, user-visible text only via `String_Id`, and new file formats with schemas (DESIGN §12.1).
- **Platforms:** develop and test on Linux Mint. CI compiles and smoke-tests Windows/macOS on free runners. No sign-ups
  until step M8.6.
- **One-time system packages (from M1 onward):** building SDL3 locally needs build tools and display headers, roughly
  `sudo apt install cmake ninja-build pkg-config libx11-dev libxext-dev libxrandr-dev libxcursor-dev libxi-dev`
  `libxss-dev libxkbcommon-dev libwayland-dev libegl1-mesa-dev libvulkan-dev libpulse-dev libasound2-dev`.
  `tools/deps.sh` prints the exact missing list. M0 needs none of this.

## Overview

| Milestone | Theme | Steps |
|---|---|---|
| M0 | Skeleton + CLI | 8 |
| M1 | Pixels + formats | 9 |
| M2 | World, UI, i18n, MCP | 10 |
| M3 | 2D complete + hot reload | 11 |
| M4 | Export + tooling | 8 |
| M5 | 3D foundation + CSG | 10 |
| M6 | 3D complete + video + networking base | 11 |
| M7 | Advanced features | 9 |
| M8 | Beta, maturity gate, platforms + Steam | 9 |
| **Total** | | **85** |

Critical path: M0 → M1 (SDL3, render core) → M2 (World, UI, tools API) → M3 (2D) → M5 (3D core) → M6 → M7 (GI,
multiplayer, visual shaders) → M8. M4 (export) can run in parallel with early M5 once M3.8 is done. Networking base
M6.10 depends only on M0, so it can start early.

## M0 — Skeleton + CLI

Only GNAT 12.3 + gprbuild needed. No SDL3, no Alire, no sudo.

### M0.1 Repository skeleton and build scripts
- **Goal:** Projects build with one command and report warnings clearly.
- **Packages/files:** `poncelet.gpr` (scenarios `BUILD=dev|release`), `build.sh`, `run-tests.sh`, `.gitignore`, `AGENTS.md`, `README_QUICKSTART.md`
- **Contracts/tests:** Dev switches `-gnat2022 -gnata -gnatwa -gnatVa -gnato -g`. `build.sh` counts warnings and fails on errors
- **Verify:** `./build.sh`
- **Done when:** Prints toolchain versions, `BUILD OK`, `0 warnings`; `bin/poncelet` exists
- **Depends on:** —

### M0.2 Pc.Math (2D + 3D)
- **Goal:** Vectors, matrices, quaternions, Transform2D/3D with contracts.
- **Packages/files:** `src/core/pc.ads`, `src/core/pc-math.ads/.adb`
- **Contracts/tests:** `Pre`/`Post` on Normalize, From_Axis_Angle, Rotate. Smoke test: rotate (1,0,0) by 90° around Z gives (0,1,0); identity composition
- **Verify:** `./run-tests.sh math`
- **Done when:** `PASS math`
- **Depends on:** M0.1

### M0.3 Pc.World fixed-capacity tables
- **Goal:** Generational entity handles and component slots without heap or access types.
- **Packages/files:** `src/core/pc-world.ads/.adb`
- **Contracts/tests:** `Type_Invariant` (alive count = Count); Create/Destroy/Is_Alive postconditions; generation bump on destroy. Test: create 3, destroy 1, stale handle is dead
- **Verify:** `./run-tests.sh world`
- **Done when:** `PASS world`
- **Depends on:** M0.2

### M0.4 Pc.Text localisation core
- **Goal:** String IDs, seven catalogs, plural rules, placeholder formatting.
- **Packages/files:** `locale/ui.tsv`, `locale/plurals.tsv`, `tools/gen_catalog.py` (maintainer only), generated `src/core/pc-l10n_data.ads`, `src/core/pc-text.ads/.adb`
- **Contracts/tests:** Post on `Plural` encodes the CLDR rules. Generator rejects missing translations and placeholder mismatches. Test: ru 1/2/5/11/21/22/25, fr 0/1/2, zh other
- **Verify:** `./run-tests.sh text`
- **Done when:** `PASS text`; `./bin/poncelet version --lang ru` prints Russian
- **Depends on:** M0.1

### M0.5 Pc.Scene text format v0
- **Goal:** Load and save the canonical scene JSON (`.pscene`) with precise diagnostics.
- **Packages/files:** `src/core/pc-scene.ads/.adb`, `tests/data/valid.pscene`, `tests/data/invalid_*.pscene`
- **Contracts/tests:** Parser has no access types and works on an in-memory buffer. Diagnostics carry code, line, and column. Test: load → save → load → save gives identical bytes; invalid file reports the exact line/column
- **Verify:** `./run-tests.sh scene`
- **Done when:** `PASS scene`
- **Depends on:** M0.3

### M0.6 Deterministic headless simulation
- **Goal:** Fixed-step update and a world hash for reproducibility.
- **Packages/files:** `Pc.World.Step`, `Pc.World.Hash`
- **Contracts/tests:** Hash over quantised values. Test: 600 frames run twice give the same hash, matching `tests/golden_hash.txt`
- **Verify:** `./bin/poncelet run tests/data/valid.pscene --headless --frames 600`
- **Done when:** Prints `World hash: <16 hex digits>` identical to the golden file
- **Depends on:** M0.5

### M0.7 `poncelet` CLI v0
- **Goal:** Subcommands `version`, `new <dir>`, `validate <scene>`, `run <scene> --headless --frames N`, `test`; options `--json`, `--lang`.
- **Packages/files:** `src/cli/poncelet.adb`, `src/cli/pc-cli*.ad?`, `src/cli/pc-io.ad?` (SPARK_Mode Off)
- **Contracts/tests:** JSON envelope `{ok, command, version, result, diagnostics}`. Exit codes 0/1/2. Messages localised in seven languages
- **Verify:** `./bin/poncelet new /tmp/demo && ./bin/poncelet validate /tmp/demo/scenes/main.pscene --json`
- **Done when:** `"ok":true`; `--lang zh` output is Chinese; invalid scene exits 1 with file:line:column
- **Depends on:** M0.4, M0.5, M0.6

### M0.8 Smoke test runner and CI
- **Goal:** One command proves M0 works, and CI keeps all three OSes compiling.
- **Packages/files:** `tests/pc_tests.adb`, `run-tests.sh`, `.github/workflows/ci.yml` (Linux build+tests, Windows/macOS compile)
- **Contracts/tests:** About 8 smoke tests (Robert prefers few until Alpha). CLI golden checks
- **Verify:** `./run-tests.sh`
- **Done when:** Ends with `ALL TESTS PASSED`; CI green
- **Depends on:** M0.7

## M1 — Pixels + formats

First step that needs SDL3. It's built locally into `third_party/` by a script, without sudo, once the build dependencies are installed.

### M1.1 Local dependency builder
- **Goal:** Build SDL3 (and later FreeType, HarfBuzz) from source into `third_party/prefix`.
- **Packages/files:** `tools/deps.sh`, `deps.lock` (pinned versions + SHA-256)
- **Contracts/tests:** Script checks for cmake/gcc and X11/Wayland headers, and prints the exact `apt install` line if any are missing
- **Verify:** `./tools/deps.sh sdl3`
- **Done when:** `third_party/prefix/lib/libSDL3.so` exists; `DEPS OK`
- **Depends on:** M0.8

### M1.2 SDL3 thin binding
- **Goal:** Minimal Ada binding: init, window, events, timing, GPU device.
- **Packages/files:** `src/platform/sdl3/sdl3*.ads` (SPARK_Mode Off), `poncelet_platform.gpr`
- **Contracts/tests:** Each C call is wrapped in an Ada procedure with a `Pre`; errors become status results
- **Verify:** `./build.sh && ./bin/poncelet demo window`
- **Done when:** A window opens and closes cleanly with Esc; exit code 0
- **Depends on:** M1.1

### M1.3 Main loop with fixed step + interpolation
- **Goal:** 60 Hz simulation decoupled from rendering.
- **Packages/files:** `src/runtime/pc-runtime-loop.ad?`
- **Contracts/tests:** Accumulator is clamped (no spiral of death). Test: headless loop with a fake clock gives the same step count
- **Verify:** `./run-tests.sh loop`
- **Done when:** `PASS loop`
- **Depends on:** M1.2

### M1.4 Render core on SDL_GPU
- **Goal:** Device, swapchain, buffers, textures, pipelines behind handles.
- **Packages/files:** `src/services/render/pc-render*.ad?`
- **Contracts/tests:** Generational handles for GPU resources. Invalid-handle use is caught by contracts
- **Verify:** `./bin/poncelet demo clear --frames 120`
- **Done when:** Window shows the clear colour; no GPU validation errors in the log
- **Depends on:** M1.3

### M1.5 Offline shader pipeline
- **Goal:** Shaders compiled to SPIR-V/MSL/DXIL at build time.
- **Packages/files:** `shaders/*.hlsl`, `tools/shaders.sh` (SDL_shadercross)
- **Contracts/tests:** Build fails if a shader doesn't compile, with file:line diagnostics
- **Verify:** `./tools/shaders.sh && ./build.sh`
- **Done when:** All shader blobs generated; `SHADERS OK`
- **Depends on:** M1.4

### M1.6 2D sprite batcher
- **Goal:** Textured quads with batching and a 2D camera.
- **Packages/files:** `src/services/render/pc-render-batch2d.ad?`, PNG loading (stb_image binding)
- **Contracts/tests:** Batch capacity is a contract. Test: 10,000 sprites in ≤ 10 draw calls
- **Verify:** `./bin/poncelet demo sprites`
- **Done when:** Sprite moves smoothly at 60 Hz; the FPS overlay shows the draw-call count
- **Depends on:** M1.5

### M1.7 Render graph v0
- **Goal:** Declarative passes with automatic transitions.
- **Packages/files:** `src/services/render/pc-render-graph.ad?`
- **Contracts/tests:** Pass DAG validated (acyclic, inputs exist) with contracts
- **Verify:** `./run-tests.sh rendergraph`
- **Done when:** `PASS rendergraph`
- **Depends on:** M1.6

### M1.8 Text formats + JSON Schemas
- **Goal:** Specify `.pscene`, `.pres`, `.pimport`, `.ponpack`, and `project.toml`, plus a canonical writer.
- **Packages/files:** `schemas/*.schema.json`, `docs/FORMATS.md`, `Pc.Serial.Canonical`
- **Contracts/tests:** Round-trip byte-identical for all sample files. The schema validator runs in tests
- **Verify:** `./bin/poncelet validate samples/ --json`
- **Done when:** All samples valid; one broken sample reports a precise location
- **Depends on:** M0.5

### M1.9 CLI: screenshot, inspect, fmt, schema
- **Goal:** Agent-friendly inspection and rendering.
- **Packages/files:** `src/cli/pc-cli-screenshot.adb` and others
- **Contracts/tests:** `screenshot` is deterministic (fixed seed, fixed frame)
- **Verify:** `./bin/poncelet screenshot samples/sprites.pscene --frame 60 --out /tmp/s.png`
- **Done when:** PNG written and its hash matches the golden
- **Depends on:** M1.6, M1.8

## M2 — World, UI, i18n, MCP

### M2.1 Generic component stores
- **Goal:** `Pc.World.Store (Component, Capacity)` struct-of-arrays plus a descriptor registry.
- **Packages/files:** `src/core/pc-world-store.ad?`, `pc-world-descriptors.ad?`
- **Contracts/tests:** Dense/sparse invariants as `Type_Invariant`
- **Verify:** `./run-tests.sh world`
- **Done when:** `PASS world` (stores)
- **Depends on:** M0.3

### M2.2 Hierarchy, transforms 2D/3D, events, tags
- **Goal:** Parent/children, world-transform pass, typed event channels, tag queries.
- **Packages/files:** `pc-world-hierarchy`, `pc-world-events`
- **Contracts/tests:** No cycles (contract on Set_Parent). Test: nested transforms
- **Verify:** `./run-tests.sh hierarchy`
- **Done when:** `PASS hierarchy`
- **Depends on:** M2.1

### M2.3 FreeType + HarfBuzz text rendering
- **Goal:** Shaped text with glyph atlases, Noto Sans + Noto Sans SC subset.
- **Packages/files:** `src/services/text/*`, `tools/deps.sh freetype harfbuzz`, `tools/subset_fonts.sh`
- **Contracts/tests:** Atlas capacity contracts. Golden screenshots for the seven languages
- **Verify:** `./bin/poncelet demo text --lang zh`
- **Done when:** Chinese, Russian, and German text render correctly
- **Depends on:** M1.6

### M2.4 UI toolkit v1
- **Goal:** Containers, widgets, themes, stretch modes, IME.
- **Packages/files:** `src/ui/pc-ui*.ad?`
- **Contracts/tests:** Layout invariants (no negative sizes). Pseudo-locale layout test
- **Verify:** `./bin/poncelet demo ui --lang xx`
- **Done when:** No clipped text in the pseudo-locale
- **Depends on:** M2.3

### M2.5 Undo/command system
- **Goal:** Every edit is a command and can be reversed.
- **Packages/files:** `src/tools/pc-tools-commands.ad?`
- **Contracts/tests:** Undo(Do(x)) = x as a postcondition. Property test over random edits
- **Verify:** `./run-tests.sh undo`
- **Done when:** `PASS undo`
- **Depends on:** M2.1

### M2.6 Editor shell + project manager
- **Goal:** Window with docks, project list, new project from template.
- **Packages/files:** `src/editor/*`
- **Contracts/tests:** Editor strings only via String_Id
- **Verify:** `./bin/poncelet-editor`
- **Done when:** Project manager opens; language switch works live
- **Depends on:** M2.4, M2.5

### M2.7 Pc.Tools API
- **Goal:** One Ada API shared by the editor, CLI, and MCP.
- **Packages/files:** `src/tools/pc-tools.ad?`
- **Contracts/tests:** Destructive operations need an explicit confirmation token
- **Verify:** `./run-tests.sh tools`
- **Done when:** `PASS tools`
- **Depends on:** M2.5

### M2.8 MCP server v1
- **Goal:** stdio + loopback HTTP with query/modify/play/screenshot/log/test tools.
- **Packages/files:** `src/mcp/*`
- **Contracts/tests:** Project-scoped paths; per-session token; confirmation for destructive tools
- **Verify:** `./bin/poncelet mcp --stdio < tests/mcp/list_tools.json`
- **Done when:** Valid JSON-RPC responses; `create_entity` appears in the undo history
- **Depends on:** M2.7

### M2.9 API docs generator
- **Goal:** Markdown + JSON reference from `.ads` incl. contracts.
- **Packages/files:** `tools/apidoc/*`
- **Contracts/tests:** Every public subprogram is documented (CI check)
- **Verify:** `./bin/poncelet docs Pc.Math.Normalize --json`
- **Done when:** Returns the signature and Pre/Post
- **Depends on:** M0.2

### M2.10 GNATprove flow in CI
- **Goal:** Flow analysis on SPARK_Mode packages.
- **Packages/files:** CI job (gnatprove via Alire in CI only)
- **Contracts/tests:** `--mode=flow` clean for Pc.Math, Pc.Text, Pc.Scene
- **Verify:** (CI)
- **Done when:** CI job green
- **Depends on:** M0.8

## M3 — 2D complete + hot reload

### M3.1 2D physics v1
- **Goal:** Bodies, shapes, broadphase, solver, queries.
- **Packages/files:** `src/core/pc-physics-2d*`
- **Contracts/tests:** Bounded iterations; no division by zero (contracts). Stacking test
- **Verify:** `./run-tests.sh physics2d`
- **Done when:** `PASS physics2d`; boxes stack stably
- **Depends on:** M2.2

### M3.2 Character controller + areas + joints
- **Goal:** Gameplay-level 2D physics.
- **Packages/files:** `pc-physics-2d-character`, `-joints`
- **Contracts/tests:** Slope and step limits as ranged subtypes
- **Verify:** `./bin/poncelet demo platformer`
- **Done when:** Character runs and jumps; trigger events fire
- **Depends on:** M3.1

### M3.3 Tilemaps + terrains
- **Goal:** Layers, autotile rules, per-tile collision/nav.
- **Packages/files:** `pc-tilemap*`
- **Contracts/tests:** Rule matching test
- **Verify:** `./bin/poncelet demo tilemap`
- **Done when:** Tile painting autotiles correctly
- **Depends on:** M1.6

### M3.4 2D lights + shadows + particles
- **Goal:** Lights, occluders, CPU/GPU particles.
- **Packages/files:** `pc-render-light2d`, `pc-particles`
- **Contracts/tests:** Particle pool capacity contracts
- **Verify:** `./bin/poncelet demo lights2d`
- **Done when:** Shadows follow the light; particles spawn
- **Depends on:** M1.7

### M3.5 Animation + tweens
- **Goal:** Property tracks, sprite frames, events, tweens.
- **Packages/files:** `pc-anim*`
- **Contracts/tests:** Track time monotonic (Pre)
- **Verify:** `./run-tests.sh anim`
- **Done when:** `PASS anim`
- **Depends on:** M2.2

### M3.6 Audio mixer v1
- **Goal:** Buses, positional 2D, containers, Ogg/WAV/MP3/QOA.
- **Packages/files:** `src/services/audio/*`
- **Contracts/tests:** Mixer buffers bounded; no allocation in the audio callback
- **Verify:** `./bin/poncelet demo audio`
- **Done when:** Sound pans with position; no glitches
- **Depends on:** M1.2

### M3.7 Input actions + rebinding
- **Goal:** Action map, deadzones, gamepads, physical keys.
- **Packages/files:** `pc-input*`
- **Contracts/tests:** Rebinding persists to `project.toml`
- **Verify:** `./bin/poncelet demo input`
- **Done when:** Rebinding works for keyboard and gamepad
- **Depends on:** M1.2

### M3.8 Save/load
- **Goal:** Versioned saves with migrations.
- **Packages/files:** `pc-serial-save*`
- **Contracts/tests:** Migrate_Vn_To_Vn1 contracts
- **Verify:** `./run-tests.sh save`
- **Done when:** `PASS save` (v1→v2 migration)
- **Depends on:** M1.8

### M3.9 Game-library hot reload
- **Goal:** Rebuild the game `.so` and reload with World state kept.
- **Packages/files:** `src/runtime/pc-runtime-reload*`, `pc-cli-run` (`--watch`)
- **Contracts/tests:** Descriptor hash check; layout migration via the canonical serialiser
- **Verify:** `./bin/poncelet run samples/platformer --watch`
- **Done when:** Edit a `.adb` → change visible in < 3 s without losing state
- **Depends on:** M2.1, M3.2

### M3.10 Editor 2D viewport + inspector + import
- **Goal:** Gizmos, snapping, tilemap painter, asset import dock.
- **Packages/files:** `src/editor/viewport2d*`, `inspector*`, `import*`
- **Contracts/tests:** All edits go through undo
- **Verify:** `./bin/poncelet-editor samples/platformer`
- **Done when:** Build a level with mouse only, undo/redo works
- **Depends on:** M2.6, M3.3

### M3.11 Localisation editor + play-in-editor
- **Goal:** Spreadsheet view of catalogs; play in a separate process with live inspector.
- **Packages/files:** `src/editor/l10n*`, `play*`
- **Contracts/tests:** Missing/overlong highlighting
- **Verify:** `./bin/poncelet-editor samples/platformer`
- **Done when:** Edit a German string, see it in-game live
- **Depends on:** M3.10

## M4 — Export + tooling

### M4.1 Data pack `.ppak`
- **Goal:** Binary pack with checksums and an embedded-in-executable mode.
- **Packages/files:** `pc-pack*`
- **Contracts/tests:** CRC/xxHash verified on load
- **Verify:** `./bin/poncelet export --target linux samples/platformer /tmp/out`
- **Done when:** `/tmp/out/platformer.x86_64` runs
- **Depends on:** M3.8

### M4.2 Linux export runtime
- **Goal:** Stripped runtime, rpath `$ORIGIN`, bundled libs.
- **Packages/files:** `src/runtime/*`, `tools/export_linux.sh`
- **Contracts/tests:** `ldd` shows only bundled + system libs
- **Verify:** `/tmp/out/platformer.x86_64`
- **Done when:** Game runs on Mint outside the repo
- **Depends on:** M4.1

### M4.3 Windows/macOS cross builds in CI
- **Goal:** Unsigned exports built on free runners; smoke-tested headless.
- **Packages/files:** `.github/workflows/export.yml`
- **Contracts/tests:** Headless run hash equals the Linux hash
- **Verify:** (CI)
- **Done when:** Artifacts built; hashes match
- **Depends on:** M4.2

### M4.4 Steam stub service
- **Goal:** `Pc.Services.Steam` spec + stub body.
- **Packages/files:** `src/services/steam/*`
- **Contracts/tests:** Stub logs achievements; language from OS
- **Verify:** `./run-tests.sh steamstub`
- **Done when:** `PASS steamstub`
- **Depends on:** M2.7

### M4.5 Template-pack import
- **Goal:** `.ponpack` install and new-from-template.
- **Packages/files:** `pc-templates*`
- **Contracts/tests:** Signature verified; unsigned packs give a warning
- **Verify:** `./bin/poncelet new /tmp/g --template samples/demo.ponpack`
- **Done when:** Project created and runs
- **Depends on:** M1.8

### M4.6 Profiler + Tracy
- **Goal:** Frame profiler, GPU timestamps, Tracy client.
- **Packages/files:** `pc-profile*`
- **Contracts/tests:** Zero cost when disabled (compile-time switch)
- **Verify:** `./bin/poncelet run samples/platformer --profile`
- **Done when:** Profiler panel and Tracy capture work
- **Depends on:** M3.10

### M4.7 Movie capture
- **Goal:** Deterministic PNG/WAV capture.
- **Packages/files:** `pc-capture*`
- **Contracts/tests:** Frame pacing locked to the fixed step
- **Verify:** `./bin/poncelet capture samples/platformer --frames 120 --out /tmp/cap`
- **Done when:** 120 PNG frames + WAV
- **Depends on:** M4.2

### M4.8 Module-stripped exports
- **Goal:** Export profile removes unused services.
- **Packages/files:** `poncelet.gpr` module flags
- **Contracts/tests:** Size report
- **Verify:** `./bin/poncelet export --profile minimal ...`
- **Done when:** Smaller binary; game still runs
- **Depends on:** M4.2

## M5 — 3D foundation + CSG

### M5.1 3D math/scene types + cameras
- **Goal:** Transform3D components, perspective/ortho cameras, frustum.
- **Packages/files:** `pc-scene-3d*`
- **Contracts/tests:** Frustum plane normalisation contracts
- **Verify:** `./run-tests.sh scene3d`
- **Done when:** `PASS scene3d`
- **Depends on:** M2.2

### M5.2 Mesh resources + primitives + mesh builder
- **Goal:** GPU meshes, primitive generators, procedural builder.
- **Packages/files:** `pc-mesh*`
- **Contracts/tests:** Index bounds contracts
- **Verify:** `./bin/poncelet demo primitives`
- **Done when:** Primitives render
- **Depends on:** M1.7

### M5.3 Clustered forward+ core
- **Goal:** Depth prepass, light clustering compute, forward shading.
- **Packages/files:** `pc-render-3d-clustered*`
- **Contracts/tests:** Cluster grid sizes as ranged subtypes
- **Verify:** `./bin/poncelet demo lights3d`
- **Done when:** 256 point lights at 60 FPS on Robert's laptop GPU
- **Depends on:** M5.2

### M5.4 PBR materials + sky/IBL
- **Goal:** Metallic-roughness, environment lighting, procedural/HDRI sky.
- **Packages/files:** `pc-material*`, `shaders/pbr*`
- **Contracts/tests:** Golden screenshots
- **Verify:** `./bin/poncelet screenshot samples/pbr.pscene --out /tmp/p.png`
- **Done when:** Matches golden within tolerance
- **Depends on:** M5.3

### M5.5 Shadows
- **Goal:** Cascaded directional, omni, spot, shadow atlas, PCF/PCSS.
- **Packages/files:** `pc-render-shadows*`
- **Contracts/tests:** Cascade split monotonic (Post)
- **Verify:** `./bin/poncelet demo shadows`
- **Done when:** Stable, acne-free shadows
- **Depends on:** M5.3

### M5.6 glTF import + OBJ
- **Goal:** cgltf, MikkTSpace, meshoptimizer, KTX2.
- **Packages/files:** `src/import/gltf*`
- **Contracts/tests:** Re-import keeps overrides
- **Verify:** `./bin/poncelet import samples/models/helmet.glb`
- **Done when:** Imported scene renders correctly
- **Depends on:** M5.4

### M5.7 Antialiasing
- **Goal:** MSAA, FXAA, TAA.
- **Packages/files:** `pc-render-aa*`
- **Contracts/tests:** —
- **Verify:** `./bin/poncelet demo aa`
- **Done when:** Modes switch at runtime
- **Depends on:** M5.3

### M5.8 Jolt 3D physics
- **Goal:** C shim + Ada API, bodies, shapes, queries, character controller.
- **Packages/files:** `third_party/jolt_shim/*`, `pc-physics-3d*`
- **Contracts/tests:** Ada wrapper contracts (valid handles, finite values)
- **Verify:** `./bin/poncelet demo physics3d`
- **Done when:** Stack of boxes, character walks on slopes
- **Depends on:** M5.1

### M5.9 CSG (Manifold)
- **Goal:** Union/subtract/intersect nodes, editor preview, bake to mesh.
- **Packages/files:** `pc-csg*`, Manifold binding
- **Contracts/tests:** Manifold result checked (watertight)
- **Verify:** `./bin/poncelet demo csg`
- **Done when:** CSG blockout updates live
- **Depends on:** M5.2

### M5.10 3D viewport + gizmos + collision generation
- **Goal:** Editor 3D editing, trimesh/convex/decomposition.
- **Packages/files:** `src/editor/viewport3d*`
- **Contracts/tests:** Undo for all gizmo edits
- **Verify:** `./bin/poncelet-editor samples/3d`
- **Done when:** Place, transform, and CSG-edit objects
- **Depends on:** M5.6, M5.8, M5.9

## M6 — 3D complete + video + networking base

### M6.1 Skeletal animation + skins + blend shapes
- **Goal:** GPU skinning from glTF.
- **Packages/files:** `pc-anim-skeleton*`
- **Contracts/tests:** Bone index bounds
- **Verify:** `./bin/poncelet demo skinning`
- **Done when:** Animated character plays
- **Depends on:** M5.6

### M6.2 Animation graph + IK + retargeting
- **Goal:** State machine, blend spaces, FABRIK/CCD, humanoid map.
- **Packages/files:** `pc-anim-graph*`
- **Contracts/tests:** Graph validation (no dangling transitions)
- **Verify:** `./bin/poncelet demo locomotion`
- **Done when:** Walk/run blend and foot IK
- **Depends on:** M6.1

### M6.3 Lightmaps + probes + reflection probes
- **Goal:** GPU baker, SH probes, reflection probes.
- **Packages/files:** `pc-bake*`
- **Contracts/tests:** Bake deterministic for a fixed seed
- **Verify:** `./bin/poncelet bake samples/room`
- **Done when:** Baked scene matches golden
- **Depends on:** M5.5

### M6.4 SSAO, SSIL, SSR, decals, volumetric fog, post FX
- **Goal:** Screen-space effects and fog volumes.
- **Packages/files:** `pc-render-post*`, `-decals`, `-fog`
- **Contracts/tests:** Effect toggles are per-quality preset
- **Verify:** `./bin/poncelet demo fx`
- **Done when:** All effects toggle without artefacts
- **Depends on:** M5.4

### M6.5 GPU particles 3D + LOD/culling + FSR1
- **Goal:** Compute particles, LODs, Hi-Z culling, resolution scaling.
- **Packages/files:** `pc-particles-gpu*`, `pc-render-cull*`
- **Contracts/tests:** Particle buffer bounds
- **Verify:** `./bin/poncelet demo forest`
- **Done when:** Large scene at 60 FPS with FSR1
- **Depends on:** M5.3

### M6.6 Navigation 2D/3D
- **Goal:** Navmesh bake, A*, avoidance, links.
- **Packages/files:** `pc-nav*`
- **Contracts/tests:** Path validity Post (connected polygons)
- **Verify:** `./run-tests.sh nav`
- **Done when:** `PASS nav`
- **Depends on:** M3.1, M5.8

### M6.7 GridMap + Curve3D + FBX
- **Goal:** 3D tiles, paths, ufbx importer.
- **Packages/files:** `pc-gridmap*`, `pc-curve*`, `src/import/fbx*`
- **Contracts/tests:** —
- **Verify:** `./bin/poncelet import samples/models/test.fbx`
- **Done when:** FBX renders like the glTF version
- **Depends on:** M5.6

### M6.8 Audio parity
- **Goal:** Effects, 3D audio + Doppler, adaptive music, surround.
- **Packages/files:** `src/services/audio/*`
- **Contracts/tests:** No allocation in the callback (checked)
- **Verify:** `./bin/poncelet demo audio3d`
- **Done when:** Positional 3D audio and music transitions
- **Depends on:** M3.6

### M6.9 Video playback
- **Goal:** Theora or AV1 (dav1d) + Opus, video texture, cutscenes.
- **Packages/files:** `pc-video*`
- **Contracts/tests:** Audio/video sync within 40 ms
- **Verify:** `./bin/poncelet demo video`
- **Done when:** Cutscene plays in sync
- **Depends on:** M6.8

### M6.10 Networking base
- **Goal:** Sockets, HTTP(S) via mbedTLS, WebSocket.
- **Packages/files:** `pc-net-sockets*`, `pc-net-http*`, `pc-net-ws*`
- **Contracts/tests:** Timeouts mandatory (contracts)
- **Verify:** `./run-tests.sh net`
- **Done when:** `PASS net` (loopback)
- **Depends on:** M0.8

### M6.11 3D template game
- **Goal:** A complete small 3D game as a template.
- **Packages/files:** `templates/3d_*`
- **Contracts/tests:** Golden replay test
- **Verify:** `./bin/poncelet run templates/3d_demo --headless --frames 3600`
- **Done when:** Replay hash matches
- **Depends on:** M6.1–M6.9

## M7 — Advanced features

### M7.1 Dynamic GI: voxel volumes
- **Goal:** Voxelise static geometry, cone-trace with dynamic lights.
- **Packages/files:** `pc-gi-voxel*`
- **Contracts/tests:** Volume bounds contracts
- **Verify:** `./bin/poncelet demo gi_voxel`
- **Done when:** Moving light changes indirect lighting
- **Depends on:** M6.3

### M7.2 Dynamic GI: probe-based large world
- **Goal:** DDGI-style probe grid with ray-traced/raster updates.
- **Packages/files:** `pc-gi-probes*`
- **Contracts/tests:** Probe budget per frame (Pre)
- **Verify:** `./bin/poncelet demo gi_world`
- **Done when:** Open scene with dynamic sun and indirect light
- **Depends on:** M7.1

### M7.3 Visual shader editor
- **Goal:** Node graph → shader dialect, preview, custom nodes.
- **Packages/files:** `src/editor/shadergraph*`, `pc-shader-graph*`
- **Contracts/tests:** Graph type-checking (port types as Ada enums)
- **Verify:** `./bin/poncelet-editor samples/materials`
- **Done when:** Build a material by nodes; it compiles and renders
- **Depends on:** M5.4, M2.6

### M7.4 Multiplayer transport
- **Goal:** ENet binding (or own reliable UDP), channels.
- **Packages/files:** `pc-net-transport*`
- **Contracts/tests:** Sequence-number invariants
- **Verify:** `./run-tests.sh transport`
- **Done when:** `PASS transport` (loss simulation)
- **Depends on:** M6.10

### M7.5 Replication + typed remote calls
- **Goal:** Snapshots/deltas, interest management, generated RPC stubs with contracts.
- **Packages/files:** `pc-net-replication*`, `tools/rpcgen`
- **Contracts/tests:** Payload contracts checked on receive
- **Verify:** `./bin/poncelet run templates/mp_demo --instances 2`
- **Done when:** Two instances see each other move
- **Depends on:** M7.4

### M7.6 UPnP + multi-instance testing + network profiler
- **Goal:** Port mapping, editor N-instance launch, bandwidth graphs.
- **Packages/files:** `pc-net-upnp*`, `src/editor/netprof*`
- **Contracts/tests:** —
- **Verify:** `./bin/poncelet-editor templates/mp_demo`
- **Done when:** Launch 3 instances from the editor; profiler shows traffic
- **Depends on:** M7.5

### M7.7 Plugins + C ABI
- **Goal:** Editor plugin API, runtime extensions, `pc_ext.h`.
- **Packages/files:** `src/editor/plugins*`, `include/pc_ext.h`
- **Contracts/tests:** ABI version check on load
- **Verify:** `./bin/poncelet-editor --plugin samples/plugins/hello`
- **Done when:** Plugin dock appears; can be disabled
- **Depends on:** M2.6

### M7.8 Accessibility
- **Goal:** AccessKit tree for UI.
- **Packages/files:** `src/ui/pc-ui-access*`
- **Contracts/tests:** Every focusable widget has a role and a localised name
- **Verify:** (manual) Orca screen reader on Mint
- **Done when:** Orca reads editor menus
- **Depends on:** M2.4

### M7.9 Multiplayer template
- **Goal:** Template pack entry with an online-ready sample.
- **Packages/files:** `templates/mp_*`
- **Contracts/tests:** Replay + desync detection test
- **Verify:** `./run-tests.sh mp`
- **Done when:** `PASS mp`
- **Depends on:** M7.5

## M8 — Beta, maturity gate, platforms + Steam

No sign-ups before step M8.6 (Robert's decision).

### M8.1 Onboarding + tutorials
- **Goal:** First-run wizard, interactive tutorials as data.
- **Packages/files:** `src/editor/onboarding*`, `tutorials/*`
- **Contracts/tests:** Tutorial steps validated by schema
- **Verify:** `./bin/poncelet-editor --first-run`
- **Done when:** New user reaches a playable template in < 1 minute
- **Depends on:** M7

### M8.2 Crash recovery + autosave + soak tests
- **Goal:** Recover unsaved work; 24 h soak runs.
- **Packages/files:** `src/editor/recovery*`, `tests/soak/*`
- **Contracts/tests:** —
- **Verify:** `./run-tests.sh soak --hours 1`
- **Done when:** No crash, no memory growth
- **Depends on:** M8.1

### M8.3 Template packs finalised
- **Goal:** Contents from the group chat with Robert's game bots; signed packs.
- **Packages/files:** `templates/*`, `packs/*.ponpack`
- **Contracts/tests:** Parity tests per template
- **Verify:** `./bin/poncelet validate packs/ --json`
- **Done when:** All packs valid and signed
- **Depends on:** M7.9

### M8.4 Localisation QA
- **Goal:** Native-speaker review of all seven languages.
- **Packages/files:** `locale/*`
- **Contracts/tests:** No missing or overlong strings (CI)
- **Verify:** `./bin/poncelet l10n report --json`
- **Done when:** 0 missing, 0 overlong
- **Depends on:** M8.1

### M8.5 Maturity gate
- **Goal:** Freeze formats, close all crashers, docs complete.
- **Packages/files:** `docs/*`, `schemas/*` (frozen v1)
- **Contracts/tests:** Gate checklist
- **Verify:** `./run-tests.sh all`
- **Done when:** All green; Robert signs off the gate
- **Depends on:** M8.2–M8.4

### M8.6 Accounts (after the gate)
- **Goal:** Steamworks partner + Apple Developer ID.
- **Packages/files:** —
- **Contracts/tests:** —
- **Verify:** —
- **Done when:** Accounts active
- **Depends on:** M8.5

### M8.7 Real Steamworks + signing
- **Goal:** Steam body, macOS signing/notarisation, Windows signing (optional), Steam Runtime build.
- **Packages/files:** `src/services/steam/real*`, `tools/sign_*`
- **Contracts/tests:** —
- **Verify:** (CI + test machines)
- **Done when:** Signed builds install on real macOS/Windows hardware
- **Depends on:** M8.6

### M8.8 Store + depots
- **Goal:** SteamPipe, store page in seven languages, EULA.
- **Packages/files:** `steam/*.vdf`
- **Contracts/tests:** —
- **Verify:** (CI)
- **Done when:** Private branch installs from Steam
- **Depends on:** M8.7

### M8.9 Release
- **Goal:** Steam review and launch.
- **Packages/files:** —
- **Contracts/tests:** —
- **Verify:** —
- **Done when:** **Poncelet Editor + template packs on sale**
- **Depends on:** M8.8

## Definition of done for a milestone

1. All steps' verify commands pass on Linux Mint, and `./run-tests.sh` ends with `ALL TESTS PASSED`.
2. CI is green (Linux tests, Windows/macOS compile + smoke).
3. 0 compiler warnings in the dev build; GNATprove flow is clean for SPARK_Mode packages (from M2).
4. DESIGN.md and the Appendix A parity rows for the milestone are updated; new strings are translated in all seven languages.
5. Robert has run the verify commands himself and confirmed.
