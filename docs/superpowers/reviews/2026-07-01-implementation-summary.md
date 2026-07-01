# Titration Simulator — Implementation Summary (2026-07-01)

Companion to `2026-07-01-titration-simulator-audit.md`. Records what was implemented from the audit, how it was verified, and what remains as future work.

**Verification:** a jsdom harness boots the real `sb_titration.html` (Plotly stubbed) so tests exercise the actual in-file code; numeric fixes were TDD'd red→green. A Playwright/Chromium pass screenshots every tab and confirms rendering. Final state: **16/16 jsdom tests pass; all 6 tabs + Gran sub-tab + back-titration render in Chromium with 0 console/page errors.** (Test harness lives in the session scratchpad, not committed — the app stays single-file.)

## Shipped

### Correctness & scientific bugs
- **A1 (CRITICAL)** — static buffer capacity was inverted (`|1/(dV/dpH)|`). Now `β = (c_titrant/V₀)·|dV/dpH|` (mol·L⁻¹·pH⁻¹), peaks at pKₐ; verified vs analytic Van Slyke (0.0576 at pKₐ for 0.1 M; ratio β(pKₐ)/β(eq) = 1165×). Fixed plot, CSV export, y-axis label (B3).
- **A2 (HIGH)** — dead Gran-plot sub-tab (`onclick` → closure-scoped `switchTab2` → ReferenceError). Rebound via `addEventListener`; Gran plot now reachable.
- **A3** — indicator endpoint no longer reports a fake "0.000 mL" / −100 % when the transition pH is unreachable; shows a "nicht erreicht" message.
- **A4** — species labels now charge anionic salt bases correctly (carbonate → CO₃²⁻/HCO₃⁻/H₂CO₃) via a unified `z_L + protons` model + an editable per-base charge field.

### Numerical accuracy
- **B1/B2** — Gran pre-ÄP form corrected to the weak-buffer `(V−V_prev)·10^(∓pH)`; excess-region G₂ drawn only for the final ÄP; windows clamped. Monoprotic zero-crossings now exact.
- **B5** — clarified β vs βV maxima (βV sits slightly below pKₐ from dilution).
- **B6** — pH sweep widened by `log10(maxConc)` so concentrated strong acids/bases start at V=0.
- **B7 + follow-up** — buffer plot now computed/scaled over the titration region (V ≤ plot_end), so the analyte pKₐ peaks are visible instead of being swamped by the excess-titrant tail (caught by a browser screenshot after A1).
- **B4** — investigated and found **non-reproducible** (volumes are strictly monotonic); no code added.

### Chemistry data
- **C1** CO₂ pK₂ 10.1 → 10.33 (unified with the carbonate default). **C2** fake diprotic default `[2.15,7.20]` (H₃PO₄'s first steps) → maleic acid `[1.90,6.07]`, referenced via `DEFAULT_PKAS` at all sites. **C3** back-titration note that the curve assumes an acid-base-inert product.

### Robustness (F1–F9)
- Input validation for c_titrant / v_analyte / custom pKL / titrant pKs / component concentration and all back-titration fields, surfacing the real error text. Dead code removed. `phi_values` fallback → null. **Plotly pinned to 2.35.2 with SRI + a load guard** (was the unpinned EOL `latest` alias).

### UI/UX & visual identity (D1–D13)
- New "analytical instrument" identity: lab-slate palette, **phenolphthalein-rose primary**, IBM Plex + Space Grotesk type, and a **pH-spectrum gradient signature** (header rule + active-tab underline). Mobile-first responsive grids. All plots themed transparent and drawn via `Plotly.react` (no flash); debounced inputs. Notice severity levels (info/warn/error). Softer plot colours, tabular figures, read-only field styling, fixed `.pico-card`, cleaned emoji.

### Accessibility (E1–E8)
- Labelled dynamic inputs (`for`/`id` + `role=group`), accessible remove button, `aria-current` tabs, live result regions, `role=img` + descriptive `aria-label` on every chart (+ CSV button label), non-colour indicator cue (dashed boundary lines + range card), corrected heading hierarchy (h1→h2→h3).

### Didactic features
- **G1** one-click presets (HCl, Essigsäure, H₃PO₄, Citronensäure, Na₂CO₃, Ammoniak, Borsäure+Mannitol). **G2** per-ÄP titratability badge. **G3** per-ÄP indicator recommendation (ranked by real volume error). **G8** "Kurve fixieren" comparison overlay (up to 4 curves).

## Deferred (future enhancements — all "enhancement" severity)
- **G4** analytic ÄP-pH formulas beside the interpolated value (risky to get right for all polyprotic/base cases; the exact interpolated pH is already shown and is more accurate).
- **G5** analytic Van Slyke overlay · **G6** Gran annotation fan-out · **G7** indicator error as an interval band.
- **G9** printable Ph.-Eur. protocol · **G10** state persistence + shareable URL · **G11** light/dark toggle (needs a light palette) · **G12** temperature-dependent Kw · **G13** ionic-strength/activity (Davies) · **G14** in-app self-test harness.
- **D8** unify the three sub-view control idioms (buttons/radios/selects) — cosmetic consistency.
