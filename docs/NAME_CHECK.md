# Name check (2026-11-19)

**Decision: Poncelet was chosen by Robert Boettcher on 2026-11-19.** Package prefix `Pc`, repo `poncelet`.

Method: GitHub repository search (`<name> engine`, `<name> game engine`, name-only), Steam store search
(`store.steampowered.com/api/storesearch`), plus web searches for "<name> engine/game engine".
This is **not a trademark clearance**. Before committing, run a search at EUIPO (TMview) and USPTO in
class 9 (software) and class 41/42.

| Candidate | Game engine / game-dev tool conflicts | Other notable conflicts | Risk |
|---|---|---|---|
| **Foucault** | None found. No GitHub engine, nothing on Steam. Only unrelated repos (`foucault/nvfancontrol`, a TUI note app, a telescope tool). | Strongly associated with philosopher Michel Foucault (searchability, some find him polarising). Léon Foucault (Foucault pendulum, a nice physics/determinism metaphor). | **Low** |
| **Fermat** | None (only a hobby "Fermata" audio repo). Nothing on Steam. | Fermat is a computer-algebra system. "Fermat's Last Theorem" dominates search. | **Low** |
| **Poncelet** ✅ CHOSEN | None. Nothing on Steam. Only tiny repos (a Poncelet porism demo, a ballistics lib). | Jean-Victor Poncelet (projective geometry, very apt for a renderer). Obscure, so very searchable. Harder to pronounce for non-French speakers. | **Low** |
| Cauchy | None for engines. Nothing on Steam. | Many maths libs (`CauchyDB`, Cauchy erasure codes). | Low–medium |
| Carnot | `IvEd123/Carnot_Engine` (hobby C++ engine, 1★). | "Carnot engine" is a thermodynamics term, so it's nearly unsearchable. CarnotEngine (a crypto tool). | Medium |
| Fourier | No engine. | Fourier Intelligence (robotics). "Fourier" saturates DSP and audio libraries. | Medium |
| Laplace | `Laplace-Engine` (hobby Haskell game engine). **Steam game "Laplace：拉普拉斯的神子".** | Laplace transform noise. | Medium |
| Lagrange | No engine. **Steam: "Star Hunter – Infinite Lagrange", "Lagrange Front Idle"**. "Infinite Lagrange" is a large NetEase game. | Lagrange (a Gemini browser), Lagrange points. | Medium–high (game market) |
| Galois | No engine, but **"Galois" is an existing puzzle game** (GPL, Linux/Mac/Win). | **Galois, Inc.**, a well-known formal-methods company. That's the same high-assurance/SPARK world we'd market to, so it's confusing. | High |
| Descartes | Hobby repo `Descartes_Engine`. | **Descartes Systems Group** (listed software company, trademarks in software classes), Descartes Labs. | High |
| Ampère | No engine. | **NVIDIA Ampere** GPU architecture and **Ampere Computing** (CPUs). Both are in the hardware/graphics space. | High |
| Coriolis | None for engines. | **"Coriolis – The Third Horizon"** (Free League RPG line, also on Fantasy Grounds/Steam DLC). | Medium–high |

## Recommendation (as made before the decision)
1. **Foucault**: Robert's first choice, and the cleanest result. Pitch it via Léon Foucault's pendulum
   (deterministic, provable motion fits the SPARK story). Repo/package prefix `foucault` / `Fc`.
2. **Poncelet** ✅ **chosen**: very clean and geometry-themed. Package prefix `Pc`.
3. **Fermat**: clean in games. Short, well known, easy to pronounce in all seven languages.

Avoid: Galois, Descartes, Ampère (company/trademark conflicts in software or hardware), and Lagrange/Laplace
(existing Steam games).
