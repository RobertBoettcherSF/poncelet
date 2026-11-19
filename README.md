# Poncelet

Poncelet is a commercial desktop game engine for Steam, with its core written in **Ada**.

**Status:** design draft. There is no code yet.

## Overview
- **Platforms:** Windows, macOS, and Linux desktop. Games ship on Steam.
- **Core in Plain Ada with contracts** (`Pre`/`Post`, type invariants, ranged subtypes, assertions), written so
  that selected components can later be proven in **SPARK** without refactoring.
- **Thin platform layer:** SDL3 (window, input, audio, GPU via Vulkan/Metal/D3D12), FreeType/HarfBuzz for text,
  and Steamworks via its flat C API.
- **2D first.** An entity/component world, a deterministic fixed-step simulation, and data-driven game content.
- **Templates:** existing games can be folded in as starter templates for new projects.
- **UI languages (7):** English, German, French, Russian, Simplified Chinese, Spanish, Italian.
  Identifiers are English internally, and all user-visible text goes through string IDs.

## Documents
- [Design document](docs/DESIGN.md)
- [Name check](docs/NAME_CHECK.md)

## Licence
Source-available for viewing only. No reuse is permitted. See [LICENSE](LICENSE).

Copyright (c) 2026 Robert Boettcher. All rights reserved.
