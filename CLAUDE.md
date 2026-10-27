# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This repo is a LaTeX document that records **updates to the methodology** in David Torres's dissertation (source: `/home/det12/projects/torres_dissertation`, built from `dissertation.tex`). "RaM" means **Remeshing and Mapping** (also written "remapping"), the Lie-group-preserving mesh-transfer workflow in dissertation Chapter 4 (`Chapter4/main.tex`, label `ch:rmrm`). Read the baseline below as "what the dissertation currently says". Updates in this document should state what changed relative to the dissertation and why. Do not cite specific dissertation chapters, sections or equation numbers in the document text; refer to "the dissertation" generally. (The chapter table below is only for finding the source material.)

## Build

Builds use `latexmk` with `pdflatex` (TeX Live 2024), run from the repo root:

```sh
latexmk -pdf main.tex      # build main.pdf
latexmk -c                 # remove intermediate files (keeps main.pdf)
latexmk -C                 # remove intermediates and main.pdf
```

Build artifacts and `main.pdf` are gitignored. There is no lint or test setup.

## Layout

- `main.tex` is the root and the only file that gets compiled (`article`, 0.5in margins, `pgfplots` with `compat=1.18` and `groupplots`, `subcaption`, `placeins`).
- `results/main.tex` is the "Current Results" section (`\input` from the root). It holds a booktabs status table (`table:current_results`) with columns Name / Status / Physics, listing Simple Box, Wave, Simple ShearBand, Sandia Fracture Challenge, Tophat and Torsion.
- `tikz/` and `packages/` are still empty. Content placed there must be wired into the root with `\input`.
- **Macros are not shared with the dissertation.** Its equations use macros from `torres_dissertation/GaryMacros.sty` (`\ep`, `\epdot`, `\defgrad`, `\Gc`, `\grad`, `\divergence`, `\diff`, `\body`, `\activeenergy`, `\inactiveenergy`), `packages/bold.tex` (`\bfa`…`\bfZ`) and `packages/davidsymbols.tex` (`\tqf`, `\state`, `\vel`, `\epref`, `\epdotref`, `\bbare`). It also needs `siunitx` (`\qty`, `\num`). Copy these into `packages/` here before pasting dissertation equations, or they will not compile.
- `\hl{...}` here is a math-mode red `\colorbox` for flagging math that needs review. The dissertation's `\hl` is a different, text-mode yellow/red highlight. Don't confuse the two when porting text.

## Dissertation structure (for cross-references)

| Ch. | File | Label | Content |
|---|---|---|---|
| 2 | `Chapter2/main.tex`, `discretization.tex`, `framed/*.tex` | `chap:methodology` | Variational model, constitutive choices, discretization, return mapping |
| 3 | `Chapter3/*.tex` | `chap:shear_bands` | MWE, tophat mesh convergence (`ss:meshconv`), 3D torsion; no RaM |
| 4 | `Chapter4/*.tex` | `ch:rmrm` | RaM methodology (`rmrm:methodology`) + RaM results |
| 5 | `Chapter5/main.tex` | `chap:conclusions` | Summary, limitations and future work |

## Baseline methodology: the model (Ch. 2)

- **Kinematics:** total Lagrangian, `F = F^e F^p`. The state is `{φ, F^p, ε̄^p, d, ∇d, T}`. The variational framework follows Hu et al. 2021 and Yang et al. 2006. The governing equations come from an inf-sup of a Lagrangian (energy rate + dual kinetic potential − Fourier potential); the supremum over T recovers the heat equation.
- **Energy:** `Ψ = W^e(F,F^p,d) + W^p(ε̄^p,d,T) + F(d,∇d)`. The dissipation is `W^{p*}` (viscoplastic) + `Λ` (fracture).
- **Elasticity (two options):**
  - Hencky (log strain) with a vol/dev split. Used in Ch. 3.
  - Compressible Neo-Hookean with a Heaviside split on `J`. Used in Ch. 4.
  - In both, only the active part is degraded by `g(d)`.
- **Plasticity:**
  - J2 with Johnson-Cook hardening in variational form. The Taylor-Quinney factor `𝒬` splits stored from dissipated work, and thermal softening is `S(T) = 1 − ((T−T0)/(Tm−T0))^m`.
  - Prandtl-Reuss flow, with `N^p ≈ √(3/2) dev τ/‖dev τ‖` assuming `R^e ≈ I`.
  - The low-triaxiality problem uses Voce hardening instead.
- **Fracture:**
  - AT-1 (`α(d)=d`, `c0=8/3`). Degradation `g(d) = (1−d)²(1−η)+η` is shared by the elastic and plastic energies.
  - Nucleation threshold: `ψ_c g_c = (3/16) Gc/l`.
- **EPPD driving force:** `Ξ = g'(d) w^e_A + g'(d) w^p`. Plastic energy is included because in shear bands it exceeds elastic energy by orders of magnitude, so an EPD (elastic-only) driver is insufficient.
- **Coalescence:** `Ĝc/Gc = g_c(ε̄^p) = (1−c1)/(exp((ε̄^p−c2)/c3)+1) + c1`. This is a sigmoid drop near `c2`, unlike the exponential decay in Hu et al. It delays Gc degradation so damage nucleates after the shear band forms.
- **Heat equation:** `ρ0 c_v Ṫ = −∇·h + δ_i + δ_t`, with Fourier conduction and a convective BC. Thermal conduction is what regularizes shear-band width.
- **Numerics:**
  - Code: RACCOON (MOOSE-based).
  - Solver: staggered fixed-point between mechanics and damage. Tolerances: sub-solves abs 1e-10 / rel 1e-7; alternating scheme abs 1e-8 / rel 1e-5.
  - Time integration: HHT-α with `α=−1/4`.
  - Return mapping #1: radial return on `F^p` with exponential update, used with Hencky in Ch. 3.
  - Return mapping #2: Borden et al. 2016, with state variable `b̄^e`, isochoricity re-enforced by scaling to `J^p=1` each step, used with CNH in Ch. 4.

## Baseline methodology: RaM (Ch. 4)

- **Transfer principle:** a variational L2 projection (Mota et al. 2013). The Lagrange-multiplier constraint gives `z̄ = λ_α M^{-1} ∫ λ_β z dV`. Only the internal-variable projection is used, and displacement is not re-solved.
- **Lie-group interpolation:** map the variable to its Lie algebra with `log`, interpolate additively, map back with `exp`. This preserves constraints such as `det F^p = 1` (SL(3)) and orthogonality of rotations.
- **Per-variable treatment:**
  - **ε̄^p:** plain polynomial interpolation.
  - **F ∈ GL(3)+:**
    - Polar-decompose `F = RU`.
    - `U` is interpolated via `log` (Sym+).
    - `R` is interpolated via `log(R^{1/2})`, which extends the admissible range to `(−2π, 2π]`.
    - The parts are then recombined.
    - The total F is the product of per-step F's back to the original reference `X0`: `F = F̂0 F̂1 … F̂n`.
  - **F-bar:** the volumetric-locking correction is re-applied with the **new** element volumes on restart.
  - **b̄^e ∈ SL(3):** Lie interpolation, with no special restart handling.
  - **Nodal variables (T, v, a, d, reference coords for etch lines):** plain interpolation. However, the gradient in the damage diffusion term must be chain-ruled back to `X0`.
  - **Density:** not remapped. It is rescaled as `ρ_n = ρ0 / det F_n`. This is a practical treatment to keep inertia sane, not mass-conservative.
- **Workflow (Python-orchestrated, Exodus II exchange):**
  1. MOOSE runs until the remesh criterion is met.
  2. MOOSE exports the fields.
  3. CUBIT remeshes.
  4. SIERRA performs the Lie-group mapping.
  5. MOOSE restarts on the new mesh.
  6. The cycle repeats.

  Quadrature-point ordering differs between MOOSE and SIERRA, so a reorder map is applied. It was derived by prescribing an analytic field on a test element.
- **Remesh trigger:** Frobenius norm of the incremental deformation gradient > **0.75** in any element. 0.5 is used for stress tests and for the tophat notch-tip phase. Refinement uses a **fixed size field** that is remapped as a state variable and does not evolve.
- **QA:**
  - Check continuity of reaction forces and local fields (d, ε̄^p, stress) across RaM events.
  - Under-resolved steep gradients produce jumps via the IP→node→IP projection, so the minimum element size is set by the smallest size that avoids jumps.
- **Known limitation:** Exodus file size. One rank-2 tensor at 8 IPs is 72 scalar fields per element. In-core RaM is noted as future work.
- **Triaxiality-dependent EPPD variant (Borden-based):**
  - `Ξ = g' w^e_A + g' ⟨w^p_T − w^p_0⟩+`, with `w^p_T = w^p/φ` and `φ = 1 + exp(−T²/(2σ_T²))`.
  - Triaxiality is `𝒯 = tr(τ)/(3‖s‖)`, and `ẇ^p_T ≥ 0` is enforced.
  - Parameters used: `w^p_0 = 200 mJ/mm³`, `σ_T = 0.1803`.
  - `φ` has no physical basis between `𝒯=0` and `𝒯=1`, and the dissertation says so.

## Baseline results and open issues worth tracking

- **Mesh convergence (Ch. 3 tophat):** requires `2δ/h ≈ 17` (δ = Dodd–Bai half-width), far finer than the usual `2l/h = 5` for phase-field. The Ch. 4 RaM tophat runs at only `2δ/h = 6.8` because of cost.
- **RaM validation problems in Ch. 4:**
  - 2.5D simple tension: all physics on, `det b̄^e = 1` preserved.
  - Thermal compression (Prakash et al.): no phase field, pseudo-contact release, HEX8.
  - Low-triaxiality (Gross & Ravi-Chandar, Al 6061-T6): Voce hardening, d capped ~0.9.
  - Full tophat: 140 RaM events, d > 0.85.
  - 3D torsion: TET10 with 4th-order quadrature, d plateaus ~0.85 because of solver difficulty at RaM restarts.
- **Open problems stated in the dissertation (candidate targets for updates):**
  - Base EPPD floods the low-triaxiality process zone with damage, which contradicts experiment. The Borden variant fixes the nucleation site but not the eventual saturated band.
  - The tophat base and triaxiality models predict qualitatively different crack paths, and this is unresolved.
  - Johnson-Cook accuracy at high rate is a concern.
  - Exodus-based data transfer limits scalability.
  - No 3D tophat with RaM and damage has been run yet.
  - Experimental validation is mostly qualitative.
  - Lode-angle dependence is missing, and strength-surface nucleation is suggested.
