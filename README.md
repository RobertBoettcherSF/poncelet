# Poncelet

Poncelet is a commercial desktop game engine **and editor** for Steam, with its core written in **Plain Ada**.

**Headline differentiators** (besides everything being Plain Ada): first-class **projective geometry**, **fractals**,
and **density**. These three pillars are core features of the first release, not add-ons and not post-launch work.
The engine also aims to match Godot's capability in **2D and 3D**.

**Status:** design draft. There is no code yet.

## Overview
- **Platforms:** Windows, macOS, and Linux desktop. Games ship on Steam.
- **Core in Plain Ada with contracts** (`Pre`/`Post`, type invariants, ranged subtypes, assertions), written so
  that selected components can later be proven in **SPARK** without refactoring.
- **Thin platform layer:** SDL3 (window, input, audio, GPU via Vulkan/Metal/D3D12), FreeType/HarfBuzz for text,
  and Steamworks via its flat C API.
- **2D and 3D from the start:** a batched 2D renderer and a clustered forward+ PBR 3D renderer, glTF import, our own Ada
  2D physics, and Jolt-backed 3D physics behind an Ada API. 2D ships first, then 3D on the same architecture.
- **The editor is the first product:** sold on Steam as software, localised in all seven languages, bundled with template packs.
- **Signature features beyond Godot (all in the first release):** **projective geometry** (homogeneous math,
  perspective guides, 2.5D and anamorphic cameras, portals and mirrors, AR calibration), **fractals** (seeded
  noise, L-systems, IFS, shader nodes, Mandelbulb raymarching, infinite zoom), and **density** (fields for fog,
  scattering and heat maps, SPH fluids, destructible voxels, dual contouring). See [Design document §19](docs/DESIGN.md).
- **AI-tool friendly (core feature):** text formats with JSON Schemas, a headless `poncelet` CLI with JSON output,
  and a built-in local MCP server, so users drive it with their own AI tools. No bundled AI model and no stored AI keys.
- **Template packs** (`.ponpack`): existing games folded in as starter templates for new projects.
- **UI languages (7):** English, German, French, Russian, Simplified Chinese, Spanish, Italian.
  Identifiers are English internally, and all user-visible text goes through string IDs.

## Documents
- [Design document](docs/DESIGN.md)
- [Implementation plan](docs/IMPLEMENTATION_PLAN.md)
- [Name check](docs/NAME_CHECK.md)

## Licence
Source-available for viewing only. No reuse is permitted. See [LICENSE](LICENSE).

Copyright (c) 2026 Robert Boettcher. All rights reserved.
