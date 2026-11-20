# Poncelet — Implementation Plan (M0 → M8, plus M9 outlook)

> Companion to `docs/DESIGN.md` (draft 0.10). Copyright (c) 2026 Robert Boettcher. All rights reserved.
> Status: plan only. No code has been committed yet. Dated 2026-11-20. Draft 0.10 puts **all** signature
> features in the first release (M0–M7); nothing in DESIGN §19 is deferred to M9.

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
  until step M8.6. Game export targets: desktop, web (M4/M6), Android and iOS (M6 dev builds, M8 store), consoles
  via partners and OpenXR post-launch (M9). Web follows the proven Ada-to-WASM setups of Robert's Roguelike Jungle
  and CDDA In ADA bots: web-safe Ada subset, numeric host ABI, AdaWebPack GNAT-LLVM (DESIGN §10.5). Hood Simulator
  and Colony4X are JavaScript games.
- **Reference games (DESIGN Appendix B):** unzip Robert's four packages into `refs/` (gitignored, never committed). Reckoning is
  the acceptance test for M1 (M1.11), M3 (M3.15) and M4 (M4.15); Roguelike Jungle (M5.11), Colony4X (M7.13) and Hood Simulator
  (M7.14) are template ports that must pass seed parity against the reference.
- **One-time system packages (from M1 onward):** building SDL3 locally needs build tools and display headers, roughly
  `sudo apt install cmake ninja-build pkg-config libx11-dev libxext-dev libxrandr-dev libxcursor-dev libxi-dev`
  `libxss-dev libxkbcommon-dev libwayland-dev libegl1-mesa-dev libvulkan-dev libpulse-dev libasound2-dev`.
  `tools/deps.sh` prints the exact missing list. M0 needs none of this.

## Overview

| Milestone | Theme | Steps |
|---|---|---|
| M0 | Skeleton + CLI | 13 |
| M1 | Pixels + formats | 11 |
| M2 | World, UI, i18n, MCP | 10 |
| M3 | 2D complete + hot reload | 17 |
| M4 | Export + tooling | 15 |
| M5 | 3D foundation + CSG | 14 |
| M6 | 3D complete + video + networking base | 21 |
| M7 | Advanced features | 29 |
| M8 | Beta, maturity gate, platforms + Steam | 10 |
| M9 | Post-launch (outlook) | 6 |
| **Total** | | **146** |

Critical path: M0 → M1 (SDL3, render core, WASM spike) → M2 (World, UI, tools API) → M3 (2D) → M5 (3D core) → M6 → M7 (GI,
multiplayer, visual shaders, remaining signature work) → M8. M4 (export) can run in parallel with early M5 once M3.8 is done. Networking base
M6.10 depends only on M0, so it can start early. Signature-feature steps (DESIGN §19) are now on the path to M8:
M0.13 after M0.2, M3.16–M3.17 with 2D, M5.12–M5.14 with cameras and SDF math, M6.16–M6.21 with density, mirrors,
2D SPH and voxel resources. Inside M7, four tracks can run in parallel after their deps: portals and calibration
(M7.17–M7.19), SPH (M7.20–M7.21), voxels and dual contouring (M7.22–M7.25), raymarch and infinite zoom (M7.26–M7.29).

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
- **Goal:** Vectors, matrices, quaternions, Transform2D/3D with contracts. Projective types follow in M0.13.
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

### M0.6 Seeded deterministic headless simulation
- **Goal:** Fixed-step update with explicit world seed and AI seed, plus a world hash for exact replay.
- **Packages/files:** `Pc.World.Step`, `Pc.World.Hash`
- **Contracts/tests:** Hash over quantised values; seeds passed explicitly (no global RNG, no wall clock). Test: same seeds give the same hash twice; a different AI seed gives a different hash
- **Verify:** `./bin/poncelet run tests/data/valid.pscene --headless --frames 600 --world-seed 1 --ai-seed 2`
- **Done when:** Prints `World hash: <16 hex digits>` identical to `tests/golden_hash.txt`
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

### M0.9 Web-safe Ada subset for the core
- **Goal:** Keep `pc-core` in the web-safe subset (DESIGN §10.5) so it runs on WebAssembly, iOS, consoles, and restricted runtimes.
- **Packages/files:** `core.adc` (configuration pragmas: `No_Tasking`, `No_Allocators`, `No_Exception_Handlers`, no Text_IO/OS units), `poncelet.gpr`
- **Contracts/tests:** Build fails if the core uses tasking, allocators, exception handlers, or Text_IO/Directories; fixed-size data only
- **Verify:** `./build.sh`
- **Done when:** `BUILD OK` with `core.adc` active; a deliberate violation in a scratch file is rejected
- **Depends on:** M0.8

### M0.10 Numeric host ABI v1 core + last-chance handler (from Reckoning `world.ads` and RJ `rj-lch`)
- **Goal:** Freeze `Pc.Host_ABI` v1 as in DESIGN §10.5 decision 3: `pc_abi_version`, `pc_init (world_seed, ai_seed)`, `pc_tick (n)`, `pc_act (actor, verb, a1, a2, a3) → status`, `pc_busy`, `pc_clock`, `pc_read_stat`, `pc_read`, the event ring (`pc_event_count/read/dropped/clear`, records `{sequence, tick, code, actor, a1, a2, a3}`), `pc_hash_lo/hi`, `pc_buffer_ptr/len/cap (kind)`. Shape taken from Reckoning's `world.ads` (integer-only C exports) with state passed explicitly instead of a package global. Last-chance handler as in Roguelike Jungle's `rj-lch`, but passing file pointer + line to the import `pc_lch`.
- **Packages/files:** `src/host/pc-host_abi.ads/.adb`, `src/core/pc-events.ads/.adb` (fixed ring, default 1024), `src/host/pc-last_chance.ads/.adb`, `tests/host_abi_test.c` (C harness), `tools/gen_exports.py` (export list from the spec, used by M1.10)
- **Contracts/tests:** Contracts: every export total over its integer domain (out-of-range index returns 0, never raises); event ring `Post => Count <= Capacity`, overflow increments `Dropped`; a deliberate failing assertion reaches `pc_lch` with file and line; no `Unchecked_Conversion` of a record address (grep check)
- **Verify:** `./run-tests.sh hostabi`
- **Done when:** `PASS hostabi`: the C harness drives init/act/tick/events/hash through the ABI, gets the same hash as `poncelet run`, sees `dropped = 5` after pushing capacity+5 events, and prints `LCH pc-host_abi.adb:<line>` for the forced failure
- **Depends on:** M0.9

### M0.11 Reference packages and known-answer vectors
- **Goal:** Make the four reference packages usable as test oracles without committing them: they are unzipped into `refs/` (gitignored). A script builds Roguelike Jungle's native `rj_sim` with your GNAT 12.3 (statically linked, see DESIGN §10.5 decision 9) and writes known-answer vectors for the later RNG/hash/parity steps.
- **Packages/files:** `refs/README.md` (where to unzip CDDA-In-ADA.zip, roguelike-jungle.zip, Colony4X.zip, Hood-Simulator.zip), `.gitignore` entry `refs/*/`, `tools/refvectors.sh`, `tests/vectors/rj_sim_s1-5_t1440.jsonl`, `tests/vectors/mulberry32.txt`, `tests/vectors/lcg_reckoning.txt`, `tests/vectors/xorshift32s.txt`
- **Contracts/tests:** Vectors are plain text with a header naming the source package, version and commit; the script fails loudly if a package is missing
- **Verify:** `./tools/refvectors.sh`
- **Done when:** Prints `rj_sim: 5 seeds OK (GNAT 12.3, static)`, `mulberry32/lcg/xorshift32*: 1000 values each`, and, if `node` is installed, `colony4x seed 7919 t40 replay hash 5fd168f5 OK`
- **Depends on:** M0.8

### M0.12 Pc.Random, Pc.Fixed and Pc.Hash (extracted from Roguelike Jungle, Reckoning and the JS references)
- **Goal:** Deterministic maths for every target (DESIGN §6.15.1, §6.15.3): `Pc.Random` with explicit state and **named streams** derived from world/AI seed (`Derive` = SplitMix32), PCG32 default, compatibility generators `Mulberry32`, `Xorshift32_Star`, `Lcg_1664525`; integer `Below`, `Chance`, `Weighted`, `Shuffle`, `Bell`. `Pc.Fixed`: generic binary fixed-point pattern from `RJ.Fixed` (32-bit storage). `Pc.Hash`: 64-bit FNV-1a over a canonical field stream with named sub-hashes.
- **Packages/files:** `src/core/pc-random.ads/.adb`, `pc-random-compat.ads/.adb`, `pc-fixed.ads`, `pc-hash.ads/.adb`, `tests/random_test.adb`
- **Contracts/tests:** `pragma SPARK_Mode (On)` on all three; `Below: Pre => N > 0, Post => V < N`; no floats; streams independent (drawing from `Loot` never changes `Map`); known-answer vectors from M0.11 match bit for bit; `Below` equals JS `Math.floor (next() * N)` for N < 2**21
- **Verify:** `./run-tests.sh random`
- **Done when:** `PASS random`: 4 generators × 1000 vectors identical, stream-independence check passes, hash of a fixed field list equals the committed value; GNAT 12.3 build has 0 warnings
- **Depends on:** M0.11, M0.2


### M0.13 Pc.Math.Projective (signature: projective geometry)
- **Goal:** Homogeneous points, lines and planes (including points at infinity), incidence, cross-ratio, duality, and projective transforms as a SPARK-ready, platform-free core package (DESIGN §19.1).
- **Packages/files:** `src/core/pc-math-projective.ads/.adb`, `tests/math_projective*`
- **Contracts/tests:** `Pre`/`Post` on Meet, Join, Dual, Cross_Ratio, and on `Is_Finite` / `Is_At_Infinity` (no silent division by w = 0). Known-answer tests: a documented complete quadrangle's cross-ratio; duality is involutive; a point at infinity lies on a given line; a documented homography maps four points to four others
- **Verify:** `./run-tests.sh math_projective`
- **Done when:** `PASS math_projective`
- **Depends on:** M0.2

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
- **Goal:** Device, swapchain, buffers, textures, pipelines behind handles; render service exposes a `Render_Method` interface so the mobile and compatibility tiers can plug in later.
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

### M1.10 WebAssembly toolchain spike (AdaWebPack GNAT-LLVM)
- **Goal:** Build the headless core + host ABI to wasm32 with AdaWebPack 24 (GNAT-LLVM): `-O2 -gnat2022 -gnatwa -gnata`, `wasm-ld --no-entry --strip-all --allow-undefined --export=<generated list> -z stack-size=1048576 --initial-memory=16777216` plus `s-memcop.o s-memmov.o s-memset.o` from `libgnat.a`. `tools/get-wasm-toolchain.sh` installs AdaWebPack, LLVM 16 libs and the GNAT 14 runtime library into `toolchain/` without sudo (Roguelike Jungle recipe, GNAT-FSF-builds tarball instead of Alire). JS host: real imports (`pc_lch`, `pc_log`, `pc_now_ms`) inside the dev Proxy that logs unknown names; runs `___elab*`; checks `pc_abi_version`.
- **Packages/files:** `tools/get-wasm-toolchain.sh`, `tools/wasm/build.sh`, `hosts/headless/run.mjs`, `hosts/web/boot.js`, `web/headless.html` (single file, base64-inlined wasm)
- **Contracts/tests:** Same world hash as native; **chunking test**: ticks fed as 1,7,80,3,640,33,… give the native hash; a forced assertion shows `file:line` in the console and an overlay; unknown import names are logged
- **Verify:** `./tools/get-wasm-toolchain.sh && ./tools/wasm/build.sh && node hosts/headless/run.mjs tests/data/valid.pscene --ticks 43200 --seeds 1..5 --chunks uneven`
- **Done when:** `PASS 5/5 native == wasm (uneven chunks)`; `web/headless.html` works when opened directly from disk
- **Depends on:** M0.10, M0.12

### M1.11 Reckoning acceptance A1: Reckoning core on the Poncelet web toolchain
- **Goal:** First acceptance test (DESIGN §13.4). Move Reckoning v0.54's Ada sources into `templates/reckoning/` as template content and add a thin adapter that exposes them through `Pc.Host_ABI` (`w_new`→`pc_init`, `act`→`pc_act`, `ev_*`→event ring, `w_stat`→`pc_read_stat`, `auto_act`→`pc_auto_step`, save blob→`pc_buffer (Save)`). Build with checks **on** (Reckoning shipped with `-gnatp`, so latent range errors are expected and fixed in the template).
- **Packages/files:** `templates/reckoning/src/` (world.ads/.adb, reckoning.ads/.adb), `templates/reckoning/abi/reckoning-pc_adapter.ads/.adb`, `templates/reckoning/template.toml`, `templates/reckoning/NOTES.md` (list of check failures found and fixed)
- **Contracts/tests:** No engine package depends on the template; the adapter has no game logic; every check failure found is listed with seed, turn and fix
- **Verify:** `./tools/wasm/build.sh templates/reckoning && node hosts/headless/run.mjs templates/reckoning --seeds 1..20 --auto 3000 --compare-native`
- **Done when:** `PASS 20/20 native == wasm` for hash and stats 2, 3, 46, 50; no last-chance report in 20 seeds × 3000 auto steps
- **Depends on:** M1.10

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

### M3.12 Touch and gesture input model
- **Goal:** Multi-touch, gestures, virtual joystick/buttons, safe areas (device testing comes in M6).
- **Packages/files:** `pc-input-touch*`, `pc-ui-virtual_controls*`
- **Contracts/tests:** Gesture recogniser state machine with contracts; mouse-emulated touch tests
- **Verify:** `./bin/poncelet demo touch --emulate-touch`
- **Done when:** Pinch/swipe/virtual stick work with mouse emulation
- **Depends on:** M3.7

### M3.13 Replays, saves and golden replay tests
- **Goal:** `.preplay` per DESIGN §6.15.3 (canonical JSON header + `tick actor verb a1 a2 a3` lines + `#hash` lines), playback that names the first divergent tick and sub-hash, the two save forms (replay save = seeds + action log, as Colony4X; versioned `Pc.Serial` snapshot with magic/version/CRC in `pc_buffer (Save)`), golden replays in CI. Uses `Pc.Random`/`Pc.Hash` from M0.12.
- **Packages/files:** `src/core/pc-replay*`, `src/core/pc-serial*` (snapshot header), `tests/replays/*`
- **Contracts/tests:** Record → play back → identical hash; a deliberately changed rule reports the first divergent tick and sub-hash; an unknown save version is refused cleanly; log cap reported
- **Verify:** `./bin/poncelet replay verify tests/replays/platformer_001.preplay`
- **Done when:** `REPLAY OK` (all checkpoint hashes match); a tampered replay reports the divergent frame
- **Depends on:** M3.8

### M3.14 Autoplayer, `poncelet sim`, `--compare` and the `poncelet-report` v1 (F9)
- **Goal:** Autoplayer interface with the rules of DESIGN §6.15.4 (public command API only, verb coverage counter, stuck detection after 50 calls without clock progress, `< 0` = stop, optional decision matrix and per-order trace). `poncelet sim` with world + AI seed ranges, fair lineups, parallel jobs, per-run report, CSV, batch summary, periodic report files (`BUILD` / `TEST KEY` / `SUMMARY` lines) and `--compare old new`. In-game F9 (and Debug button) writing `poncelet-report` v1 (§6.15.6): required `expected` note, replay, hashes, stats, events with `dropped`, errors, environment, stamped screenshot; JSON + Markdown.
- **Packages/files:** `src/core/pc-autoplay.ads/.adb`, `src/cli/poncelet-sim.adb`, `src/cli/poncelet-report.ads/.adb`, `schemas/poncelet-report-1.json`, `src/runtime/pc-debug_report.adb`
- **Contracts/tests:** Report validates against `schemas/poncelet-report-1.json`; batch summary lists errors, stuck, mean/median length, end causes and verb coverage; `--compare` of a run with itself shows all deltas 0
- **Verify:** `./bin/poncelet sim samples/platformer --games 10 --ticks 36000 --world-seeds 1..10 --ai-seeds 1..10 --out /tmp/rep && ./bin/poncelet sim --compare /tmp/rep /tmp/rep && ./bin/poncelet validate /tmp/rep --json`
- **Done when:** 10 reports + `summary.md` + `runs.csv` written, `--compare` prints `no change`, all reports valid; F9 in `poncelet run samples/platformer` writes JSON + Markdown with the stamp in the screenshot
- **Depends on:** M3.13

### M3.15 Reckoning acceptance A3: Reckoning batch and F9 under the engine
- **Goal:** Run Reckoning's autoplayer through `poncelet sim` (200 seeds, profession = seed mod 5, location = (seed / 5) mod 3, scenarios default/stocked/fvs) and map its F9 data into `poncelet-report` v1. Reckoning's own `auto_run` (from `refs/`) is the oracle.
- **Packages/files:** `templates/reckoning/sim.toml` (metrics, lineup, end-cause names), `templates/reckoning/report_map.ads/.adb`, `templates/reckoning/tests/replays/rk_seed1.preplay`
- **Contracts/tests:** Per-seed CSV columns turns/days/cause/waves/alerts equal `refs/.../auto_run` output for seeds 1..20; batch summary shape matches `samples/batch-v020-summary.md` (runs, survival mean/median/min/max, causes, stuck, errors)
- **Verify:** `./bin/poncelet sim templates/reckoning --games 200 --ticks 14400 --world-seeds 1..200 --out /tmp/rk && ./tools/compare_refs.sh reckoning /tmp/rk`
- **Done when:** `stuck 0, errors 0`, `PASS 20/20 identical to auto_run`, golden replay passes in `./run-tests.sh replays`
- **Depends on:** M3.14, M1.11


### M3.16 Fractal generators (noise, L-systems, IFS)
- **Goal:** Seeded, deterministic fractal generators in `pc-core`: fBm, ridged and domain-warped noise, Diamond-square, L-systems, and IFS, all tied to the world seed (DESIGN §19.2).
- **Packages/files:** `src/core/pc-noise*`, `src/core/pc-fractal*`, `cli` `gen` subcommand, schemas for generator parameters
- **Contracts/tests:** Iteration and degree caps as ranged subtypes; the same `(world seed, parameters)` yields the same content hash. Smoke: a documented L-system tree, an IFS, a Diamond-square heightmap, and fBm sample grid
- **Verify:** `./run-tests.sh fractal && ./bin/poncelet gen noise --world-seed 1 --kind fbm --out /tmp/n.pres && ./bin/poncelet gen lsystem --world-seed 1 --out /tmp/t.pres`
- **Done when:** `PASS fractal`; native hash equals the WASM/headless hash for the documented cases
- **Depends on:** M0.12, M0.13, M1.8

### M3.17 2D projective editor tools (guides, 2.5D)
- **Goal:** Perspective grids, horizon and vanishing-point guides in the 2D viewport, plus a first-class 2.5D / fake-3D projection mode, all stored in `.pscene` (DESIGN §19.1).
- **Packages/files:** `src/editor/guides-projective*`, `pc-scene` projection-mode component, MCP `set_perspective_guides`, CLI `poncelet projective`
- **Contracts/tests:** Guide parameters are ranged; scene round-trip is byte-stable after `fmt`. A documented vanishing-point construction matches `Pc.Math.Projective` known answers
- **Verify:** `./run-tests.sh guides_projective && ./bin/poncelet projective --query vanishing --in tests/projective/quad.json`
- **Done when:** `PASS guides_projective`; guides appear in the 2D viewport and survive save/reload
- **Depends on:** M3.10, M0.13, M2.8

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

### M4.9 Compatibility renderer (2D)
- **Goal:** OpenGL 3.3 / GLES 3.0 / WebGL2 renderer for 2D with the shared render graph.
- **Packages/files:** `src/services/render/compat/*`, SPIRV-Cross shader output to GLSL 330/ES 300
- **Contracts/tests:** Golden screenshots equal to the high-end tier within tolerance
- **Verify:** `./bin/poncelet run samples/platformer --renderer compat`
- **Done when:** Same picture as the high-end renderer on Mint
- **Depends on:** M3.4

### M4.10 Web export (2D)
- **Goal:** Export 2D games to WebAssembly: numeric host ABI, WebGL2 (compatibility tier), WebAudio, DOM/Gamepad input, IndexedDB saves; single-file HTML by default, multi-file with streamed `.ppak` for large games.
- **Packages/files:** `src/platform/web/*`, `tools/export_web.sh`, `hosts/web/*` (reference host, M4.13), `tools/inline.py` (base64 wasm + stamp into one HTML; large media stays beside it, lazy-loaded)
- **Contracts/tests:** Replay hash identical to native; numeric event codes mapped to localised text/sound host-side or via Pc.Text
- **Verify:** `./bin/poncelet export --target web samples/platformer /tmp/web && python3 -m http.server -d /tmp/web 8000`
- **Done when:** Game plays at http://localhost:8000 in Firefox/Chrome; same replay hash as native
- **Depends on:** M1.10, M4.9

### M4.11 Cross-platform determinism in CI
- **Goal:** Golden replays verified on Linux, Windows, macOS, and web.
- **Packages/files:** `.github/workflows/replays.yml`
- **Contracts/tests:** Linux/Windows/macOS native hashes identical for the golden replays; native vs. WASM identical for 20 seeds × 43 200 ticks with uneven chunks (Roguelike Jungle `determinism.sh` method)
- **Verify:** (CI)
- **Done when:** All platforms report identical checkpoint hashes for fixed-point replays
- **Depends on:** M4.3, M4.10, M3.13

### M4.12 Release check policy
- **Goal:** Release builds suppress checks only for SPARK Silver-proven units; all others keep checks.
- **Packages/files:** `tools/gen_suppress.sh` (generates per-unit `pragma Suppress` config from gnatprove results)
- **Contracts/tests:** CI fails if a non-proven unit has checks suppressed
- **Verify:** `./build.sh release`
- **Done when:** Report lists suppressed units, all of them proven
- **Depends on:** M2.10

### M4.13 Reference web host (DESIGN §10.6)
- **Goal:** The JS host every web export uses: boot + Proxy + LCH overlay; stamp `vX.YY bYYYYMMDD-HHMM <sha>[+]` above every overlay; `version.txt` update banner polling a raw origin; four audio buses (SFX, Music, Ambient, Voice) with first-tap unlock, iOS keep-alive, hidden-tab muting, `pc_audio_ctx` crossfade, newest-voice-line-wins; keys + `code` mapping, touch, pinch; in-game confirm driven by `pc_query (Danger)`; IndexedDB saves (Save buffer + replay log); F9 report.
- **Packages/files:** `hosts/web/boot.js`, `stamp.js`, `audio.js`, `input.js`, `confirm.js`, `saves.js`, `report.js`, `hosts/web/tests/smoke.mjs`
- **Contracts/tests:** Smoke test at 390×844 in WebKit, Firefox and Chromium: no console errors, stamp equals `version.txt`, core ticks, input moves the player, confirm overlay opens/cancels, F9 writes a valid report, modal visibility asserted
- **Verify:** `./tools/web/smoke.sh /tmp/web/platformer.html --browsers webkit,firefox,chromium`
- **Done when:** `PASS 3/3 browsers`; screenshot shows the stamp
- **Depends on:** M4.10

### M4.14 `poncelet publish web` with curl-verify
- **Goal:** Publish to a gist (git clone/commit/push, binaries included) or to a `build/<version>` repository branch; then curl the raw and githack URLs for the new stamp and print the commit-pinned fallback if githack is stale. Real publishing only after your explicit go per release.
- **Packages/files:** `src/cli/poncelet-publish.adb`, `tools/web/publish.sh`
- **Contracts/tests:** `--dry-run` prints the exact git commands and URLs without pushing; the verify step greps for the exact stamp string
- **Verify:** `./bin/poncelet publish web /tmp/web --target gist --dry-run`
- **Done when:** Dry run lists clone/commit/push commands, play URL with `?v=N`, raw `version.txt` URL and the commit-pinned fallback form
- **Depends on:** M4.13

### M4.15 Reckoning acceptance A4: stamped single-HTML web export
- **Goal:** Export Reckoning through the engine: one HTML file + `version.txt` + lazy MP3s, reference web host, Reckoning event/Look/radio/Help text moved into `Pc.Text` catalogs (English first, other six languages by M8.4).
- **Packages/files:** `templates/reckoning/locale/*.tsv`, `templates/reckoning/assets/audio/`, `templates/reckoning/export_presets.toml`
- **Contracts/tests:** Smoke test passes for Reckoning; save → reload → identical hash; F9 report from the browser validates
- **Verify:** `./bin/poncelet export --target web templates/reckoning /tmp/rk-web && ./tools/web/smoke.sh /tmp/rk-web/reckoning.html`
- **Done when:** `PASS 3/3 browsers`; you play it on your phone from a local or published URL and the stamp matches
- **Depends on:** M4.13, M3.15

## M5 — 3D foundation + CSG

### M5.1 3D math/scene types + cameras
- **Goal:** Transform3D components, perspective/ortho cameras, frustum. The projective camera model and infinite far plane land in M5.12.
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

### M5.11 Roguelike Jungle template port (hash parity)
- **Goal:** Port Roguelike Jungle v0.04 onto the engine: `RJ.Rng` → `Pc.Random` (`Xorshift32_Star` streams), `RJ.Fixed` → `Pc.Fixed`, `rj_*` exports → `Pc.Host_ABI` adapter, shell features → reference web host.
- **Packages/files:** `templates/roguelike_jungle/` (src, abi adapter, tests/parity/)
- **Contracts/tests:** Parity oracle = `refs/roguelike-jungle` `rj_sim`: hash, funds_raw, events, day identical for 20 seeds × 43 200 ticks, natively and in WASM
- **Verify:** `./bin/poncelet test --parity templates/roguelike_jungle`
- **Done when:** `PASS 20/20 identical (native) / 20/20 (wasm)`
- **Depends on:** M4.15, M0.12


### M5.12 Projective camera model + infinite far plane
- **Goal:** 3D (and 2.5D) cameras built on `Pc.Math.Projective`, including a robust infinite far plane / reversed-Z path that works on every renderer tier (DESIGN §19.1). Anamorphic and other non-standard modes follow in M5.13.
- **Packages/files:** `pc-camera-projective*`, hook into `pc-scene-3d` and the render graph
- **Contracts/tests:** No silent w = 0; frustum and projection contracts. Golden screenshot: a distant object against the sky with no far-plane clipping or depth fighting, on high-end and on the compatibility path when that exists
- **Verify:** `./run-tests.sh camera_projective && ./bin/poncelet demo far_plane && ./bin/poncelet screenshot samples/far_plane --out /tmp/far.png`
- **Done when:** `PASS camera_projective`; screenshot matches golden within tolerance on the available tiers
- **Depends on:** M5.1, M0.13, M1.7


### M5.13 Anamorphic and non-standard projections
- **Goal:** First-class anamorphic and other non-standard camera projections built on `Pc.Math.Projective`, with editor controls and support on every 3D renderer tier (DESIGN §19.1, §19.5).
- **Packages/files:** `pc-camera-nonstandard*`, editor camera panel, a sample scene
- **Contracts/tests:** Projection parameters as ranged subtypes; a documented anamorphic matrix matches a known-answer test. Golden screenshot on high-end and on the compatibility path when it exists
- **Verify:** `./run-tests.sh camera_nonstandard && ./bin/poncelet demo anamorphic && ./bin/poncelet screenshot samples/anamorphic --out /tmp/ana.png`
- **Done when:** `PASS camera_nonstandard`; screenshots match golden within tolerance
- **Depends on:** M5.12

### M5.14 SDF fractal distance estimators
- **Goal:** Platform-free distance estimators for Mandelbulb (and similar) fractals in `Pc.Fractal.SDF`, with contracts and known-answer tests, ready for the M7 raymarch path (DESIGN §19.2).
- **Packages/files:** `src/core/pc-fractal-sdf.ads/.adb`, `tests/sdf_fractal*`
- **Contracts/tests:** Parameter bounds (iterations, power, escape radius) as ranged subtypes. Known-answer distances at documented points (inside, on, and outside a reference Mandelbulb); no silent overflow
- **Verify:** `./run-tests.sh sdf_fractal`
- **Done when:** `PASS sdf_fractal`; native hash of the sample grid equals the WASM/headless hash
- **Depends on:** M3.16, M0.2

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
- **Goal:** Screen-space effects and fog volumes. Density-field-driven fog and homography projectors land in M6.16–M6.17.
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

### M6.12 Mobile renderer tier
- **Goal:** Single-pass forward renderer with per-object light limits, tile-friendly passes, ASTC/ETC2.
- **Packages/files:** `src/services/render/mobile/*`
- **Contracts/tests:** Light limits as ranged subtypes; golden screenshots
- **Verify:** `./bin/poncelet run templates/3d_demo --renderer mobile`
- **Done when:** Runs on Mint with the mobile tier; looks as specified
- **Depends on:** M5.3

### M6.13 Compatibility renderer 3D + web 3D
- **Goal:** 3D subset on GL/WebGL2 and web export of 3D games.
- **Packages/files:** `src/services/render/compat/3d*`
- **Contracts/tests:** Unsupported features produce editor warnings, not crashes
- **Verify:** `./bin/poncelet export --target web templates/3d_demo /tmp/web3d`
- **Done when:** 3D template plays in the browser
- **Depends on:** M4.10, M6.12

### M6.14 Android dev builds
- **Goal:** Debug APK with lifecycle, touch, sensors, Vulkan/GLES fallback (unsigned, no store).
- **Packages/files:** `src/platform/android/*`, `tools/export_android.sh`, Gradle template
- **Contracts/tests:** Suspend/resume saves state; context loss handled
- **Verify:** `./bin/poncelet export --target android templates/3d_demo /tmp/apk && adb install /tmp/apk/game-debug.apk`
- **Done when:** Game runs on an Android phone or the emulator
- **Depends on:** M6.12, M3.12

### M6.15 iOS CI compile
- **Goal:** GNAT-LLVM arm64-apple-ios build of runtime + template in CI (no device, no signing).
- **Packages/files:** `.github/workflows/ios.yml`, `src/platform/ios/*`
- **Contracts/tests:** Core portability rules hold for iOS
- **Verify:** (CI)
- **Done when:** iOS build artifact compiles and links
- **Depends on:** M6.12


### M6.16 Density-field resource + uses
- **Goal:** First-class density fields (scalar/vector, dense/sparse) as `.pres` resources, with canonical text serialisation, editor painting v0, scattering, AI heat maps, density-aware physics queries, and volumetric fog/smoke driven by a field (DESIGN §19.3).
- **Packages/files:** `src/core/pc-density*`, schemas, editor paint panel, CLI `gen density`, MCP `sample_density` / `paint_density` / `scatter_from_density`, fog service hook
- **Contracts/tests:** Bounds, resolution and sparsity under contracts; `.pres` round-trip is byte-stable; the same seed places the same scatter set; point-sample and ray-march queries match known answers. Fog demo degrades without crashing on compatibility / WebGL2
- **Verify:** `./run-tests.sh density && ./bin/poncelet gen density --world-seed 1 --out /tmp/d.pres && ./bin/poncelet demo density_fog`
- **Done when:** `PASS density`; native and WASM hashes match for the documented field and scatter set
- **Depends on:** M1.8, M0.12, M6.4, M2.8

### M6.17 Homography-based decals and projectors
- **Goal:** Decals and projectors defined by a plane-to-plane homography from `Pc.Math.Projective`, with a cheap single-projector fallback on the compatibility / WebGL2 tier (DESIGN §19.1, §19.4).
- **Packages/files:** `pc-render-projectors*`, material/decal resource fields, a sample scene
- **Contracts/tests:** Homography maps four documented corners correctly (reuse M0.13 vectors). Golden screenshot of a projected texture on a plane, high-end and compatibility
- **Verify:** `./run-tests.sh projectors && ./bin/poncelet demo projectors`
- **Done when:** `PASS projectors`; screenshots match golden within tolerance
- **Depends on:** M6.4, M0.13, M5.12


### M6.18 Portal and mirror math
- **Goal:** Plane-pair portal and planar-mirror geometry in `pc-core`: homographies, clip planes, and teleport transforms, with contracts (DESIGN §19.1).
- **Packages/files:** `src/core/pc-portal.ads/.adb`, `tests/portal_math*`
- **Contracts/tests:** A documented portal pair maps a point and a ray onto the exit portal; a mirror plane reflects a ray; w = 0 stays guarded. Known-answer tests only (no GPU)
- **Verify:** `./run-tests.sh portal_math`
- **Done when:** `PASS portal_math`
- **Depends on:** M0.13, M5.12

### M6.19 Planar mirror rendering
- **Goal:** Planar mirrors via render-to-texture or stencil, one bounce, using M6.18 math, with a single-bounce fallback on compatibility / WebGL2 (DESIGN §19.1).
- **Packages/files:** `pc-render-mirrors*`, a sample scene, MCP/editor placement hook
- **Contracts/tests:** Bounce cap is a ranged subtype (1 on compatibility, small N on high-end). Golden screenshot of a documented mirror scene on high-end and compatibility
- **Verify:** `./run-tests.sh mirrors && ./bin/poncelet demo mirrors && ./bin/poncelet screenshot samples/mirrors --out /tmp/mirrors.png`
- **Done when:** `PASS mirrors`; screenshots match golden within tolerance; no crash on the compatibility path
- **Depends on:** M6.18, M1.7, M5.3

### M6.20 SPH fluid solver (2D)
- **Goal:** A bounded, seeded 2D SPH solver in `pc-core`: density, pressure, viscosity, integrate, write to a density field (DESIGN §19.3).
- **Packages/files:** `src/core/pc-sph.ads/.adb`, `pc-sph-2d*`, `tests/sph2d*`, CLI `poncelet sph`
- **Contracts/tests:** Particle count and neighbourhood size as ranged subtypes; no out-of-bounds writes. A documented fixture is seed-stable and hashes identically on native and WASM after N steps. Energy/volume stay within documented bounds (no explosion)
- **Verify:** `./run-tests.sh sph2d && ./bin/poncelet sph --dim 2 --world-seed 1 --steps 120 --out /tmp/sph2d.pres`
- **Done when:** `PASS sph2d`; native and WASM hashes match
- **Depends on:** M6.16, M0.12

### M6.21 Sparse voxel volume resource
- **Goal:** A sparse voxel volume as a first-class `.pres` resource: sample, set, canonical text serialisation, and a bounded memory cap (DESIGN §19.3).
- **Packages/files:** `src/core/pc-voxels.ads/.adb`, schemas, CLI `poncelet voxels inspect`, `tests/voxels_res*`
- **Contracts/tests:** Resolution, bounds and occupancy under contracts; `.pres` round-trip is byte-stable; a documented volume hashes identically on native and WASM
- **Verify:** `./run-tests.sh voxels_res && ./bin/poncelet voxels inspect tests/voxels/unit.pres`
- **Done when:** `PASS voxels_res`; hashes match
- **Depends on:** M1.8, M6.16

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
- **Goal:** Node graph → shader dialect, preview, custom nodes. Fractal noise and escape-time nodes land in M7.15.
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
- **Contracts/tests:** Payload contracts checked on receive; desync detection reuses replay checkpoint hashes (M3.13)
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

### M7.10 Asset/plugin package format + library client
- **Goal:** Install, update, and remove packages (templates, assets, plugins) from curated index, local folders, or Workshop; licences shown.
- **Packages/files:** `pc-packages*`, `src/editor/library*`
- **Contracts/tests:** Licence field mandatory; attributions written to THIRD_PARTY_NOTICES
- **Verify:** `./bin/poncelet package install samples/packages/demo_assets.ponpack --project /tmp/g`
- **Done when:** Package installed; licence and attribution recorded
- **Depends on:** M4.5

### M7.11 XR preparation
- **Goal:** Multi-view render graph, XR camera components, action mapping hooks (no OpenXR yet).
- **Packages/files:** `pc-render-graph` multiview, `pc-xr-types*`
- **Contracts/tests:** Stereo view count as a ranged subtype
- **Verify:** `./bin/poncelet demo stereo`
- **Done when:** Side-by-side stereo preview renders correctly
- **Depends on:** M6.12

### M7.12 Controller-first UI focus navigation
- **Goal:** Focus graph for gamepad/remote navigation (console- and TV-friendly).
- **Packages/files:** `pc-ui-focus*`
- **Contracts/tests:** Every focusable widget is reachable (test)
- **Verify:** `./bin/poncelet demo ui --gamepad`
- **Done when:** Whole demo UI usable with a gamepad only
- **Depends on:** M2.4

### M7.13 Colony4X template port (seed parity)
- **Goal:** Port Colony4X v0.9's JavaScript rules to Ada (`Mulberry32`, integer `Below`, action log saves, data-only events → event codes, `fairLineup`, `sim.js` metrics); canvas UI rebuilt with Poncelet 2D + UI.
- **Packages/files:** `templates/colony4x/` (src, locale, tests/parity/)
- **Contracts/tests:** Seed 7919, 40 all-AI turns: per-turn summaries and event sequence equal the reference `col-feedback` sample; 24-seed batch win types and turn statistics equal `node refs/.../sim.js` (statistical parity documented for any system where float JS cannot be matched)
- **Verify:** `./bin/poncelet test --parity templates/colony4x`
- **Done when:** `PASS seed 7919 trace identical`, `PASS batch 24 seeds`
- **Depends on:** M5.11, M3.14

### M7.14 Hood Simulator template port (seed parity)
- **Goal:** Port Hood Simulator 1.2.82's `src/sim` rules to Ada (`Mulberry32`, city + Auto seeds, command API → `pc_act`, presentation events → event ring, autoplayer rules, weekly report lines); city, characters and vehicles rebuilt with Poncelet 3D, navigation (M6.6) and UI.
- **Packages/files:** `templates/hood_simulator/` (src, scenes, tests/parity/)
- **Contracts/tests:** City seed 1 / Auto seed 1: weekly `SUMMARY` timeline lines equal the reference run (`refs/.../summary-city1-autoseed1-won-w145.txt`) or are within the documented statistical tolerance; sim never imports renderer/UI (CI check)
- **Verify:** `./bin/poncelet test --parity templates/hood_simulator`
- **Done when:** `PASS timeline w1–w145` (or tolerance report), seed batch summary written
- **Depends on:** M7.13, M6.6, M6.11


### M7.15 Visual shader fractal nodes + budgets
- **Goal:** Visual shader nodes for fractal noise (fBm, ridged, domain-warp) and escape-time Mandelbrot/Julia, with per-tier iteration/sample caps and editor warnings (DESIGN §19.2).
- **Packages/files:** shader-graph node definitions, dialect builtins, quality-preset tables, a sample material
- **Contracts/tests:** Port types stay in the graph type-checker (M7.3). The "low" mobile/web preset is rejected if it exceeds its cap. Golden screenshots on high-end and WebGL2 at the low preset
- **Verify:** `./run-tests.sh shader_fractal && ./bin/poncelet-editor samples/materials/julia.pshader --screenshot /tmp/julia.png --tier compatibility`
- **Done when:** `PASS shader_fractal`; budget test fails a deliberately over-budget graph; screenshots match golden
- **Depends on:** M7.3, M3.16, M6.13

### M7.16 Fractal/density world generation + marching cubes
- **Goal:** A seeded pipeline `noise/IFS → density field → scatter or marching-cubes mesh`, exposed as `poncelet gen world` and MCP `generate_world`, plus fractal density volumes at v1 quality (DESIGN §19.3–§19.4).
- **Packages/files:** `pc-gen-world*`, `pc-iso-cubes*`, editor "generate world" command, schemas
- **Contracts/tests:** The same world seed produces the same field hash, scatter set, and mesh hash on native and WASM. Marching cubes on a documented field matches a golden hash. Output is a normal mesh that draws on all renderer tiers
- **Verify:** `./run-tests.sh worldgen && ./bin/poncelet gen world --world-seed 1 --out /tmp/wg/`
- **Done when:** `PASS worldgen`; hashes match across native and WASM
- **Depends on:** M3.16, M6.16, M5.2, M2.8


### M7.17 Portal rendering, teleport, and editor
- **Goal:** Portal rendering (offscreen or stencil, recursion cap), teleport of bodies and cameras, and editor placement, built on M6.18 (DESIGN §19.1).
- **Packages/files:** `pc-render-portals*`, `pc-portal-teleport*`, editor portal tool, a sample scene, MCP `set_portal`
- **Contracts/tests:** Recursion cap as a ranged subtype (0 extra recursions on compatibility: one offscreen). Teleport preserves a seeded replay. Golden screenshots of a documented portal pair on high-end and a simpler one-offscreen path on compatibility
- **Verify:** `./run-tests.sh portals && ./bin/poncelet demo portals && ./bin/poncelet screenshot samples/portals --out /tmp/portals.png`
- **Done when:** `PASS portals`; screenshots match golden; a body walking through a portal is replay-stable
- **Depends on:** M6.18, M6.19, M5.8, M3.13

### M7.18 Camera calibration math
- **Goal:** Checkerboard / still-image camera calibration and AR-friendly homographies in `Pc.Calib`, with contracts and known-answer fixtures (DESIGN §19.1). The OpenXR runtime stays in M9.
- **Packages/files:** `src/core/pc-calib.ads/.adb`, `tests/calib_math*`, documented checkerboard fixtures
- **Contracts/tests:** Reprojection error of a documented fixture is under a stated threshold; degenerate point sets are rejected by `Pre`. Homography and pose round-trip within tolerance
- **Verify:** `./run-tests.sh calib_math`
- **Done when:** `PASS calib_math`
- **Depends on:** M0.13, M5.12

### M7.19 Calibration workflow (CLI, editor, MCP)
- **Goal:** `poncelet calibrate` and MCP `calibrate_camera` consume still images or a marked checkerboard, write a camera into a scene, and show residual error in the editor (DESIGN §19.1).
- **Packages/files:** `cli/calibrate*`, editor calibration panel, sample images under `tests/calib/`
- **Contracts/tests:** Schema-validated camera output; a documented image set reproduces the M7.18 pose within tolerance. Headless path needs no webcam (webcam capture stays M9)
- **Verify:** `./bin/poncelet calibrate tests/calib/board/ --out /tmp/cam.json && ./run-tests.sh calib_workflow`
- **Done when:** `PASS calib_workflow`; residual under the documented threshold
- **Depends on:** M7.18, M2.8, M5.12

### M7.20 SPH fluid solver (3D)
- **Goal:** Extend the 2D SPH solver to 3D with the same contracts, seeds, and hash discipline (DESIGN §19.3).
- **Packages/files:** `pc-sph-3d*`, `tests/sph3d*`
- **Contracts/tests:** Same bounds as M6.20 in 3D; a documented dam-break fixture is seed-stable on native and WASM; particle cap is enforced
- **Verify:** `./run-tests.sh sph3d && ./bin/poncelet sph --dim 3 --world-seed 1 --steps 120 --out /tmp/sph3d.pres`
- **Done when:** `PASS sph3d`; hashes match
- **Depends on:** M6.20

### M7.21 SPH rendering, density coupling, and emitters
- **Goal:** Draw SPH particles (and an optional surface), couple the solver to a density field, and add editor emitters (DESIGN §19.3).
- **Packages/files:** `pc-sph-render*`, density-field hook, editor emitter, a sample scene
- **Contracts/tests:** Particle and neighbour caps per renderer tier. A seeded emitter plus 60 steps writes a density field whose hash is stable. Golden screenshot on high-end; compatibility draws a cheaper path without crashing
- **Verify:** `./run-tests.sh sph_render && ./bin/poncelet demo sph && ./bin/poncelet screenshot samples/sph --out /tmp/sph.png`
- **Done when:** `PASS sph_render`; hashes and screenshots match
- **Depends on:** M7.20, M6.16, M6.5

### M7.22 Voxel editing and destruction
- **Goal:** Undoable, serialised voxel editing and destruction (carve, stamp, CSG subtract) on the M6.21 volume, with a seeded tool path for tests (DESIGN §19.3).
- **Packages/files:** `pc-voxels-edit*`, editor voxel brush, CLI `poncelet voxels carve`, MCP `carve_voxels`
- **Contracts/tests:** Occupancy stays inside the memory cap; edits are ordinary undo commands. A documented carve sequence round-trips through `.pres` and hashes identically on native and WASM
- **Verify:** `./run-tests.sh voxels_edit && ./bin/poncelet voxels carve tests/voxels/unit.pres --tool sphere --out /tmp/carved.pres`
- **Done when:** `PASS voxels_edit`; hashes match; undo restores the original hash
- **Depends on:** M6.21, M2.5, M5.9

### M7.23 Voxel collision and physics
- **Goal:** Collision queries and Jolt/2D-physics shapes from a voxel volume, so a carved hole is walkable and ray-castable (DESIGN §19.3).
- **Packages/files:** `pc-voxels-physics*`, `tests/voxels_phys*`
- **Contracts/tests:** Point, ray, and shape queries match known answers on a documented volume. A seeded body falling into a carved well is replay-stable
- **Verify:** `./run-tests.sh voxels_phys`
- **Done when:** `PASS voxels_phys`
- **Depends on:** M7.22, M5.8, M3.13

### M7.24 Dual contouring isosurface
- **Goal:** Dual contouring from a density field or voxel volume, preserving a documented sharp edge better than marching cubes (DESIGN §19.3).
- **Packages/files:** `pc-iso-dc*`, `tests/dual_contour*`
- **Contracts/tests:** Index and vertex bounds; a documented sharp-edge field produces a mesh whose hash is stable and whose Hausdorff distance to the feature is under a stated tolerance (tighter than the M7.16 marching-cubes mesh of the same field)
- **Verify:** `./run-tests.sh dual_contour && ./bin/poncelet gen world --iso dual-contour --world-seed 1 --out /tmp/dc/`
- **Done when:** `PASS dual_contour`; native and WASM hashes match
- **Depends on:** M7.16, M6.21

### M7.25 Dual contouring LOD and editor bake
- **Goal:** LOD levels, quality presets, and an editor bake of dual-contouring meshes, usable on every renderer tier (DESIGN §19.3).
- **Packages/files:** editor bake command, LOD tables, a sample
- **Contracts/tests:** LOD count and cell size as ranged subtypes. A "low" bake stays under a documented vertex budget. Output is an ordinary mesh that draws on high-end and compatibility
- **Verify:** `./run-tests.sh dual_contour_lod && ./bin/poncelet bake iso samples/dc_field --quality low --out /tmp/dc_low.mesh`
- **Done when:** `PASS dual_contour_lod`; budget test fails a deliberately oversized request
- **Depends on:** M7.24, M6.5

### M7.26 SDF raymarch (Mandelbulb) high-end path
- **Goal:** A high-end raymarch pass for Mandelbulb-style SDF fractals, using M5.14 estimators, with quality presets and a preview in the editor (DESIGN §19.2).
- **Packages/files:** `pc-render-sdf-raymarch*`, a sample material/scene, CLI `poncelet gen raymarch`
- **Contracts/tests:** Iteration and step caps as ranged subtypes. Golden screenshot of a documented Mandelbulb on the high-end tier
- **Verify:** `./run-tests.sh raymarch && ./bin/poncelet demo mandelbulb && ./bin/poncelet screenshot samples/mandelbulb --out /tmp/mb.png`
- **Done when:** `PASS raymarch`; screenshot matches golden within tolerance
- **Depends on:** M5.14, M5.3, M7.3

### M7.27 Raymarch LOD and compatibility fallback
- **Goal:** LOD / mobile iteration budgets for SDF raymarching, plus a pre-baked mesh or impostor fallback on compatibility / WebGL2 (DESIGN §19.2).
- **Packages/files:** quality-preset tables, mesh-bake from the SDF, a compatibility sample
- **Contracts/tests:** The "low" mobile preset is rejected if it exceeds its cap. Compatibility shows the baked fallback and does not attempt unbounded raymarch. Golden screenshot of the fallback
- **Verify:** `./run-tests.sh raymarch_lod && ./bin/poncelet screenshot samples/mandelbulb --tier compatibility --out /tmp/mb_compat.png`
- **Done when:** `PASS raymarch_lod`; budget test fails an over-budget graph; fallback screenshot matches golden; no crash on WebGL2 when that export exists
- **Depends on:** M7.26, M7.16, M6.13

### M7.28 Infinite zoom: double-precision escape-time
- **Goal:** A deterministic double-precision escape-time path for deep Mandelbrot/Julia zooms, hashed and comparable on native and WASM (DESIGN §19.2).
- **Packages/files:** `pc-zoom-double*`, `tests/zoom_double*`, CLI `poncelet zoom --method double`
- **Contracts/tests:** A documented location and zoom level produce the same iteration grid hash on native and WASM. No silent overflow; zoom and iteration caps are ranged
- **Verify:** `./run-tests.sh zoom_double && ./bin/poncelet zoom --method double --in tests/zoom/locus.json --out /tmp/z_d.pres`
- **Done when:** `PASS zoom_double`; hashes match
- **Depends on:** M7.15, M0.12

### M7.29 Infinite zoom: perturbation method
- **Goal:** A deterministic perturbation deep-zoom that matches the M7.28 double-precision reference within a documented tolerance, at a cost suitable for interactive previews (DESIGN §19.2).
- **Packages/files:** `pc-zoom-perturb*`, `tests/zoom_perturb*`, CLI `poncelet zoom --method perturbation`
- **Contracts/tests:** For a documented locus, perturbation vs double reference stays inside tolerance; the perturbation result hashes stably on native and WASM. A golden preview image is optional but preferred
- **Verify:** `./run-tests.sh zoom_perturb && ./bin/poncelet zoom --method perturbation --in tests/zoom/locus.json --out /tmp/z_p.pres`
- **Done when:** `PASS zoom_perturb`; hashes match; vs-reference check is inside tolerance
- **Depends on:** M7.28

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
- **Goal:** Finalise and sign the bundled template packs: Reckoning (first pack), Roguelike Jungle, Colony4X, Hood Simulator, plus the engine demo templates; all seven languages; each with golden replays and parity tests. If Robert agrees (DESIGN Q20), include a small official template that exercises the three signature pillars (perspective guides, portals or anamorphic camera, seeded fractal world or Mandelbulb/deep zoom, painted density, SPH or destructible voxels).
- **Packages/files:** `templates/*`, `packs/*.ponpack`
- **Contracts/tests:** Every pack passes `poncelet test --replays --parity`; signatures verify
- **Verify:** `./bin/poncelet validate packs/ --json`
- **Done when:** All packs valid and signed
- **Depends on:** M7.9, M7.17, M7.21, M7.23, M7.27, M7.29

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

### M8.8 Mobile store packaging + signing
- **Goal:** Signed Android AAB for Google Play; iOS device builds, signing, and TestFlight/App Store (needs Mac access + Apple account).
- **Packages/files:** `tools/export_android.sh --release`, `tools/export_ios.sh`
- **Contracts/tests:** —
- **Verify:** (CI + Mac)
- **Done when:** Signed builds install on real devices
- **Depends on:** M8.6, M6.14, M6.15

### M8.9 Steam store + depots + curated packs
- **Goal:** SteamPipe, store page in seven languages, EULA, curated paid template packs as DLC.
- **Packages/files:** `steam/*.vdf`, `packs/*`
- **Contracts/tests:** —
- **Verify:** (CI)
- **Done when:** Private branch installs from Steam; DLC packs install
- **Depends on:** M8.7

### M8.10 Release
- **Goal:** Steam review and launch of the editor; mobile/web export available to users.
- **Packages/files:** —
- **Contracts/tests:** —
- **Verify:** —
- **Done when:** **Poncelet Editor + template packs on sale**
- **Depends on:** M8.8, M8.9

## M9 — Post-launch (outlook)

Planned after launch. Steps are coarser and will be refined then. Signature-pillar work is not in this
milestone; it was scheduled in M0–M7 in draft 0.10 (DESIGN §19.5). M9.1 OpenXR can consume the M7 calibration
toolkit when the runtime lands.

### M9.1 OpenXR VR/AR
- **Goal:** PC VR via Vulkan, standalone Android headsets via the mobile tier, hand tracking, passthrough.
- **Packages/files:** `src/platform/xr/*`
- **Contracts/tests:** XR input mapped to actions
- **Verify:** `./bin/poncelet run samples/xr_demo --xr`
- **Done when:** Demo runs on an OpenXR runtime (e.g. Monado/SteamVR)
- **Depends on:** M7.11

### M9.2 BiDi/RTL text + UI mirroring
- **Goal:** Unicode BiDi, RTL shaping, mixed-direction editing, mirrored layouts; Arabic/Hebrew test catalogs.
- **Packages/files:** `pc-text-bidi*`, `pc-ui` mirroring
- **Contracts/tests:** BiDi algorithm conformance tests (Unicode test files)
- **Verify:** `./run-tests.sh bidi`
- **Done when:** `PASS bidi`; RTL demo UI mirrors correctly
- **Depends on:** M2.4

### M9.3 Community asset library service
- **Goal:** Hosted index with publishing, moderation, malware scanning for plugins, ratings, licences; optional paid assets.
- **Packages/files:** separate service repo (TBD)
- **Contracts/tests:** Licence policy enforced on upload
- **Verify:** (service)
- **Done when:** Users can publish and install community packages from the editor
- **Depends on:** M7.10

### M9.4 Console partner kit
- **Goal:** Documentation and test suite for porting partners; platform-service conformance tests.
- **Packages/files:** `docs/CONSOLE_PORTING.md`, `tests/platform_conformance/*`
- **Contracts/tests:** Conformance suite passes on all public platforms
- **Verify:** `./run-tests.sh conformance`
- **Done when:** Kit ready to hand to a partner under NDA
- **Depends on:** M7.12

### M9.5 WebGPU path
- **Goal:** Mobile tier on WebGPU for web (and WebXR later).
- **Packages/files:** `src/services/render/webgpu/*`
- **Contracts/tests:** Golden screenshots
- **Verify:** (browser)
- **Done when:** 3D template runs with WebGPU in Chrome/Firefox
- **Depends on:** M6.13

### M9.6 Mobile store services
- **Goal:** Optional IAP, ads, notifications modules behind platform-service interfaces.
- **Packages/files:** `src/services/store/*`
- **Contracts/tests:** No SDK linked unless enabled
- **Verify:** (device)
- **Done when:** Sample IAP flow in a test store
- **Depends on:** M8.8


## Definition of done for a milestone

1. All steps' verify commands pass on Linux Mint, and `./run-tests.sh` ends with `ALL TESTS PASSED`.
2. CI is green (Linux tests, Windows/macOS compile + smoke).
3. 0 compiler warnings in the dev build; GNATprove flow is clean for SPARK_Mode packages (from M2).
4. DESIGN.md and the Appendix A parity rows for the milestone are updated; new strings are translated in all seven languages.
5. Robert has run the verify commands himself and confirmed.
