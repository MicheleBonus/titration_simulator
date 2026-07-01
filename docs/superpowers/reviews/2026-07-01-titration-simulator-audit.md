<!-- Provenance: ultracode multi-agent audit (95 agents, 118 literature facts, 72 findings audited -> 67 verified). Sources: de Levie 1993, Harris S-B chapter, Ehlers script. Generated 2026-07-01. -->

# Titration Simulator — Final Improvement Report

**File:** `sb_titration.html` · **Branch:** `feature/simulator-improvements`
Findings below are de-duplicated across review dimensions, ranked most-severe first, and use the adversarially-corrected severities. Refuted or negligible items were dropped.

---

## Executive Summary

1. **CRITICAL — Buffer capacity is inverted.** `beta_static` is coded as `|1/(dV/dpH)| = |dpH/dV|`, the *reciprocal* of Van Slyke buffer capacity. It peaks at the **equivalence point** and is minimal at the **pKa** — the exact opposite of reality. Pharmacy students are taught that buffering is strongest at the equivalence jump. (`plotBufferCapacity` L1457–1460, CSV L1545–1546)
2. **HIGH (functional) — Tab-2 sub-tabs are dead.** `onclick="switchTab2(...)"` calls a closure-scoped function that is not global → `ReferenceError`; the **Gran-Plot** sub-view is unreachable for all users. (L255/257 vs L999)
3. **MEDIUM — Gran G₁ linearization is wrong for the default weak analyte.** The pre-ÄP branch uses the excess-strong-electrolyte form `(V0+V)·10^(∓pH)`, which is non-linear for a weak acid/base; the near-correct default result is an accident of window placement. (`granG` L1282–1291)
4. **MEDIUM/LOW — Pervasive silent input-validation gaps.** `c_titrant`/`v_analyte`/component conc/custom pKL/back-titration fields accept `0`/empty/negative → `Infinity`/`NaN`, blank or garbage plots with no feedback.
5. **LOW — Chemistry-data inconsistencies.** CO₂ correction uses pK₂=10.1 while the app's own carbonate default uses 10.33; the "diprotic acid" default `[2.15, 7.20]` is actually phosphoric acid's first two steps.
6. **MEDIUM/LOW — Accessibility gaps.** Dynamically added analyte inputs have no labels, all six Plotly charts are opaque to screen readers, and tab/selected-state ARIA is missing.

---

## A. Correctness & Scientific Bugs

| # | Severity | Item | Location | Problem & Fix |
|---|----------|------|----------|---------------|
| A1 | **CRITICAL** | `beta_static` computes the reciprocal of buffer capacity | `plotBufferCapacity` L1457–1460; CSV L1545–1546 | `dv_dph = numGradient(volumes, phs) = dV/dpH`, then `beta_static = |1/dv_dph| = |dpH/dV|`. Van Slyke β = `dC_b/dpH ∝ dV/dpH`, **not** its reciprocal. Numerically (0.1 M acetic acid + NaOH): coded β peaks at pH 8.73 (equivalence, 4.05e4) and troughs at pH 4.76 (pKa, 3.47e1) — inverted by ~1166×. Contradicts Harris (Gl. 8.19, max at pH=pKs), Ehlers (van Slyke, max at half-neutralisation), de Levie (buffer index = inverse slope ∝ dV/dpH). **Fix:** `beta_static = Math.abs(dv_dph) * params.c_titrant / params.v_analyte` (units mol·L⁻¹·pH⁻¹; peaks at pKa with 0.576·Ca = 0.0576 for 0.1 M, matching analytic Van Slyke to 4 sig figs). Apply identically to CSV export (bs2). |
| A2 | **HIGH** | Tab-2 sub-tab buttons inoperable | L255/257 `onclick="switchTab2('derivatives'/'gran')"`; `switchTab2` defined L999 inside `DOMContentLoaded` | Inline handlers run in global scope; `switchTab2` is closure-scoped and never assigned to `window` → `ReferenceError: switchTab2 is not defined`. The **Gran-Plot** feature is completely unreachable (mouse and keyboard). **Fix:** remove inline `onclick`, give buttons ids, and bind inside the closure: `getElementById('tab2-btn-gran').addEventListener('click', () => switchTab2('gran'))` (mirror the main-tab pattern at L1725-1734). |
| A3 | **LOW** | Indicator endpoint silently defaults to 0 mL when the curve never crosses the indicator midpoint pH | `plotMainCurve` L1071–1088 | `endpoint_vol` init to `0`, overwritten only on a crossing; if none (e.g. acid-range indicator on a weak-acid/strong-base titration), it reports "0.000 mL" + a bogus −100 % Indikatorfehler. **Fix:** init to `null`; if still null after the loop, render "Umschlagspunkt im erreichbaren pH-Bereich nicht erreicht" and suppress the error cards instead of computing against 0. |
| A4 | **LOW** | Species labels assign wrong charges to anionic (salt) bases | `formatSpeciesLabelPlain` L503 | Engine numerics are correct (Na₂CO₃ curve matches an independent charge balance to machine precision), but the labeler shows the default carbonate base as CO₃²⁻/HCO₃⁻/H₂CO₃ = 0/+1/+2. **Fix (display-only):** add an optional per-component reference-form charge `z0` (0 for neutral amines, −2 for carbonate, −3 for phosphate); base-mode charge = `z0 + numProtonsOnSpecies`. Engine untouched. |

---

## B. Numerical Accuracy

| # | Severity | Item | Location | Problem & Fix |
|---|----------|------|----------|---------------|
| B1 | **MEDIUM** | Gran pre-ÄP G₁ non-linear/window-dependent for weak analytes | `granG` L1282–1291; caption L268 | For a weak analyte the buffer-region Gran function is `V_titrant·10^(∓pH)` (de Levie eq 51; Harris Gl 10.5), not `(V0+V)·10^(∓pH)`. Coded form: R²=0.996 and Veq drifts −0.24 % to −49 % depending on window. **Fix:** choose the pre-ÄP factor by step chemistry — `fPre = strongExcess ? (V0+V) : (v − prevVeq)`, with a **symmetric** strong-excess test (`pKa < 0.5` for acid analytes, `pKa > pKL − 0.5` for base analytes). Reword caption: linearity in V holds rigorously only in the excess-strong-titrant region. |
| B2 | **MEDIUM** | Gran intermediate-ÄP zero-crossings biased | `plotGranFunction` L1260–1351; window L1267–1268 | Same excess-strong forms are applied around *every* ÄP; for an intermediate ÄP both sides are neighbouring buffer regions. Diprotic repro: post-ÄP1 G₂ zero = +1.04 % to +76 % vs exact 50.00 mL. **Fix:** restrict rigorous extrapolation to the first (strong-analyte) and final (excess-titrant) regions; label intermediate-ÄP lines "näherungsweise" or omit. Clamp windows so they never cross neighbouring ÄPs: `Math.min(Math.max((Veq−prevVeq)*0.35,1.0),(Veq−prevVeq)*0.9)`. |
| B3 | **MEDIUM** | y-axis unit label & comment for `beta_static` are wrong | L1494 (`L·mol⁻¹·pH⁻¹`), comment L1456 | Coded quantity is `|dpH/dV|` (units pH·L⁻¹, i.e. L⁻¹), not `L·mol⁻¹·pH⁻¹`; `|dΔ/dpH| ≈ 1/(dV/dpH)` is self-contradictory (dΔ/dpH ∝ dV/dpH). **Fix (with A1):** label `β statisch (Van Slyke) = dC_b/dpH (mol·L⁻¹·pH⁻¹)`; comment `β = dC_b/dpH ≈ (c_titrant/V0)·|dV/dpH|`. |
| B4 | **LOW** | `numGradient` has no zero-denominator guard | `numGradient` L448–456; consumers `findInflectionPoints`, `plotDerivatives`, `plotInflectionPoints` | On re-sorted `volumes` with equal/near-equal adjacent values (notably high-pKL non-aqueous solvents: DMF/MeCN/DMSO), division yields ±Infinity→NaN, spawning phantom/missing Wendepunkte and NaN table rows. (Buffer-capacity calls use `x = phs` and are **safe**.) **Fix:** guard denominator (`Math.abs(dx) < 1e-12 ? NaN : …`) at interior + both endpoints; skip non-finite pairs in `findInflectionPoints`. Preferred root fix: differentiate on the uniform pH grid and invert via chain rule (as `plotBufferCapacity` already does). |
| B5 | **LOW** | `beta_dynamic` peak displaced ~0.30 pH below pKa by dilution weight | `plotBufferCapacity` L1462–1466 | `dc/dV = c_titrant·V0/(V0+V)²` decreases with V, pulling the (valid, dilution-corrected) peak to pH 4.458 vs pKa 4.76. Offset scales with dilution. **Fix:** keep `beta_dynamic`; do not claim its peak marks the pKa. Optionally annotate the βV maximum ("liegt durch die Verdünnung leicht unterhalb des pKₐ"). |
| B6 | **LOW** | pH sweep floor of −1 truncates concentrated strong acids | `calculateCurve` L603 `linspace(-1, p.pks+1, 3000)` | For >10–12 M strong acid the true V=0 pH is below −1 (pinned to floor); symmetric strong-base case clips at `pks+1`. **Fix:** concentration-aware bounds widening **both** ends: `margin = Math.max(0, log10(maxConc)) + 1; ph_start = min(-1, -margin); ph_stop = max(pks+1, pks+margin)`. |
| B7 | **LOW** | Uniform pH sweep retains a large off-plot V→∞ tail; normalized βV anchored to it | `calculateCurve` L603, filter L641–646; `plotBufferCapacity` normalization | V-axis plots clip to `[0, plot_end]` so the tail is invisible (main curve **not** distorted). Real artifact: for weak *monoprotic* acids, `maxDynamic` in "normalized" mode anchors to a ~166×-equivalence tail point (0.1132→0.0568, 99 % shift). **Fix:** compute the normalization reference over `volume ≤ plot_end_volume_ml/1000` while still plotting the full curve. **Do NOT** cap V or bound the sweep below the titrant asymptote — those high-pH samples legitimately populate the H₂O/OH⁻ buffering region of the pH-axis plots. |

---

## C. Chemistry-Data Corrections

| # | Severity | Wrong → Correct | Location | Reference |
|---|----------|-----------------|----------|-----------|
| C1 | **LOW** | CO₂ constants `pK1=6.3, pK2=10.1` → **`6.35, 10.33`** | `CO2_PK1/PK2` L407–408 | Same H₂CO₃/HCO₃⁻ system is encoded as `[6.35, 10.33]` at `DEFAULT_PKAS.base[2]` (L715). pK₂=10.1 is below Harris (~10.33) / Ehlers (10.25). Unify to one shared constant. pK1 6.3→6.35 is cosmetic; the substantive change is pK₂. |
| C2 | **LOW** | Diprotic-acid default `[2.15, 7.20]` → a **real** diprotic acid, e.g. **oxalic `[1.25, 4.27]`** or maleic `[1.9, 6.1]` | `DEFAULT_PKAS.acid[2]` L714; also L1748 **and L1832** (page-load) | `[2.15, 7.20]` are H₃PO₄'s first two steps (Harris) — no real diprotic acid has this pair, and they reappear as the prefix of the triprotic default. Define one constant referenced at all **three** sites. Do not use carbonic acid (duplicates the base default). |
| C3 | **LOW** | Back-titration ignores product acid-base chemistry | `runBackTitration` L1642–1676 | Models leftover R1 as strong monoprotic → always a strong-strong curve; the liberated carbonate/HCO₃⁻ buffering (Ehlers §6.1.4.7) is dropped. Mass/purity output is stoichiometric and **correct**; only the curve shape is idealized. **Fix:** add a note that the curve assumes an acid-base-inert product (carbonate titrations require boiling off CO₂); optionally inject an optional "freigesetztes Produkt" component. Note: MgO/ZnO do *not* liberate CO₂. |

*(Species-label charges for anionic bases: see A4.)*

---

## D. UI/UX & Visual Design

| # | Severity | Item | Location | Fix |
|---|----------|------|----------|-----|
| D1 | **MEDIUM** | Fixed `1fr 2.5fr` sidebar grid overrides Pico's mobile stacking | `.grid` L23–26; inline `2fr 1fr 1fr` L229; `.grid-3-col` L71–75; mixture grid L735 | Make mobile-first: `.grid{grid-template-columns:1fr} @media(min-width:992px){…1fr 2.5fr}`. Same treatment for `.grid-3-col`, the tab-1 controls (replace inline style with a class), and the mixture inner grid. |
| D2 | **MEDIUM** | Opaque `plotly_dark` background clashes with Pico cards (floating grey rectangle) | All 8 `Plotly.newPlot` layouts (L1048 etc.) | Add a shared layout helper setting `paper_bgcolor`/`plot_bgcolor:'rgba(0,0,0,0)'` + Pico font tokens; **deep-merge** per-axis (`xaxis`/`xaxis2`/`yaxis`/`yaxis2`) so gridcolors survive the spread. |
| D3 | **LOW** | Benign info messages use red assertive "alert" styling | `#simulation-notice` L121–130, role L246; uses L867/954/887 | Add a `level` param to `showSimulationNotice`: sort-hint → `info` (`role=status`, polite, neutral), "keine Äquivalente" → `warn`, hard stop → `error` (red, assertive). |
| D4 | **LOW** | Live recompute on every keystroke; `Plotly.newPlot` (full teardown) flashes | wiring L763/784/1737-1739; `updateAllPlots` L1009 | Swap `newPlot`→`Plotly.react` (removes flash, biggest win); render only the active tab per recompute; debounce `input` events (~180 ms), keep `change`/select immediate. Sweep itself is ~4 ms — don't reduce point count. |
| D5 | **LOW** | Ad-hoc hardcoded Plotly colors (`lime`, `cyan`, `crimson`, `white`…) | L1096/1136/1479/1487/1587/1693/1697 etc. | Define one `PLOT` palette token object; replace named colors (neon lime/cyan especially). Recurring roles are already de-facto consistent (eq=crimson, curve=royalblue, inflection=orange) — tokenize them and desaturate the buffer neon. |
| D6 | **LOW** | Rücktitration tip uses non-existent `.pico-card` class → unstyled floating text | L1688 | Pico v2 has no `.pico-card`; use `<article role="note">` (matches L1648's error card), drop the wrong `role="alert"`. |
| D7 | **LOW** | Critical modeling guidance hidden only in native `title` tooltips (invisible on touch) | L194/197/240/312/330 | Surface strong-acid/base pKs guidance (15.74 / −1.74), Φ definition, and R1:Analyt stoichiometry as persistent `<small>` captions. (Defaults are pre-filled, so limited impact.) |
| D8 | **LOW** | Three different UI idioms for "switch sub-view" (buttons / radios / selects) | Tab2 L254-259, Tab3 L277-292, Tab1/5 selects | Standardize: reuse `.tab-buttons` for the Tab-2 panel switch; convert the 2-option config `<select>`s to Tab-3's `fieldset`+radio pattern. Keep `<select>` for long lists (indicators). |
| D9 | **LOW** | Emoji iconography (▶️/💡/⚠️/⬇) renders inconsistently (mixed variation selectors) | L334/569/1649/1688 | Drop leading glyphs (rely on labels) or use one inline-SVG set sized `1em`/`currentColor`; wrap any retained glyph in `aria-hidden`. |
| D10 | **LOW** | Metric numbers lack tabular figures → horizontal jitter on live update | `.metrics-grid p` L66–70 | Add `font-variant-numeric: tabular-nums;`; optionally split value/unit with a fixed-width `.val` span. |
| D11 | **LOW** | Read-only pKL field looks editable and label echoes its own value | L169–171; toggled L1759–1763 | Style `#pks[readonly]` distinctly (opacity/cursor/disabled bg); remove the `<span id="pks_value">` echo (and its JS refs at L686/1764/1780). |
| D12 | **enhancement** | Stock Pico azure primary — no visual identity | `:root` L12–16 | Override the **full** primary palette (`--pico-primary` *and* `--pico-primary-background`, hover, focus, inverse) scoped to dark theme, else buttons and active tab go two-tone. Optional. |
| D13 | **enhancement** | Remove-button hardcodes `crimson`, no hover/focus | `.remove-btn` L97–112 | `background: var(--pico-del-color)` + `:hover`/`:focus-visible` states. (Keyboard focus is *not* actually absent — Pico supplies a ring.) |

---

## E. Accessibility

| # | Severity | Item | Location | Fix |
|---|----------|------|----------|-----|
| E1 | **MEDIUM** | Dynamically added mixture inputs have no associated `<label>` | `addMixtureComponent` L733–748; `updateComponentPkas` L776–777 | Inputs get only class+data-id; sibling labels have no `for`. Assign unique ids and link (`for`/`htmlFor`); add `role="group" aria-label="Analytkomponente N"` to the wrapper. WCAG 1.3.1/3.3.2/4.1.2 (Level A). |
| E2 | **MEDIUM** | All six Plotly charts opaque to screen readers | plot containers L248/261/270/295/336/357/363 | Add `role="img"` + a per-render `aria-label` summarizing key figures (ÄP volumes/pH, pH range) after each `newPlot`; add sr-only data tables / `<details>` for Tabs 2/3/5 (which have **no** text alternative); give the CSV button `aria-label="Daten als CSV herunterladen"` and hide the ⬇ glyph. WCAG 1.1.1. |
| E3 | **MEDIUM** | Remove-component button's accessible name is "×" (title ignored) | L734 | `aria-label="Komponente entfernen"` (generic, not `${id}`) + `<span aria-hidden="true">×</span>` + `type="button"`; move focus to a stable element after removal. WCAG 4.1.2. |
| E4 | **MEDIUM** | Indicator transition band encoded only by translucent color gradient | `plotMainCurve` L1057–1067; metric card L1084–1088 | Add a metric card showing numeric range + color names (deColor incl. "farblos"), two dashed reference lines at `range[0]/[1]`, and a labeled band annotation. WCAG 1.4.1. |
| E5 | **LOW** | Tab nav lacks tab/tablist ARIA; selected state is color-only | nav L218–225; handler L1725–1734; CSS L39–41; Tab-2 sub-tabs | Add non-color active indicator (underline/weight) + `aria-selected`/`aria-current` toggled in the handler. Full APG tab pattern (roles + Arrow/Home/End roving tabindex) optional — but **do not** add `role="tab"` without the keyboard handler. WCAG 1.4.1/4.1.2. |
| E6 | **LOW** | Dynamically updated result regions are not live regions | `#indicator-metrics`, `#equivalence-table`, `#inflection-results`, `#back-titration-results`, `#solvent-warning` | Add `aria-live="polite"`/`role="status"`; keep containers rendered (hide via empty `textContent`, not `display:none`, which drops them from the a11y tree). WCAG 4.1.3. |
| E7 | **LOW** | Back-titration error not announced while the info tip uses assertive `role="alert"` | error `<article>` L1648 (no role) vs tip L1688 | Make `#back-titration-results` `role="status"`, wrap the error in `role="alert"`, and remove `role="alert"` from the static tip. |
| E8 | **LOW** | Heading hierarchy skips levels (h1→h3, no h2); metric-card labels are `<h3>` | h1 L143; h3 L150/228…; metric `<h3>` L1085-1087/1199-1205/1574/1684-1686 | Promote section/tab titles to h2 (cascade sub-levels down one, no new skips); change metric-card captions to `<dt>`/`<span class="metric-label">` and move `.metrics-grid h3` CSS accordingly. |

---

## F. Code Quality & Robustness

| # | Severity | Item | Location | Fix |
|---|----------|------|----------|-----|
| F1 | **LOW** | `c_titrant`/`v_analyte` accept 0/empty/NaN → Infinity/NaN plot ranges, blank plots | `getParams` L874/876; consumed L909/943-944 | Validate in `getParams` (`Number.isFinite && > 0`) returning `{valid:false, validationErrors:[…]}`. **Also** fix L887 which hardcodes the pKa message — surface `params.validationErrors.join(' ')` so the real error shows. |
| F2 | **LOW** | Component concentration `\|\| 0.1` silently turns 0 into 0.1 and passes negatives | `getParams` L839 | Validate rather than coerce (mirror the pKa validation at L843–853): reject `!Number.isFinite \|\| <= 0`. |
| F3 | **LOW** | Custom pKL unvalidated; negative → descending `linspace`, garbage/empty curve, inverted y-axis | pks input L170; used L603, axis L1047 | Validate `pkl > 0` centrally in `getParams`; clamp y-axis `range:[0, Math.max(pks,1)]`; extend the existing >40 warning to the low/negative branch. |
| F4 | **LOW** | Back-titration inputs unvalidated: empty MM / zero stoich → NaN mass/purity | `runBackTitration` L1625–1644, 1679 | Add a finiteness/positivity guard (include `stoich_ratio` in the `.every(Number.isFinite)` check — the naive `<1` test lets empty slip through); render an "Ungültige Eingabe" card + `Plotly.purge`. |
| F5 | **LOW** | `beta_dynamic` finite-guarded in CSV but not in plotted trace; `maxDynamic` unguarded | plot L1466/1469 vs CSV L1549 | Mirror `beta_static`'s guard: `map(v => (v===0\|\|!isFinite(v)) ? null : Math.abs(v))`; `maxDynamic = Math.max(...filtered, EPSILON)`; normalize with null pass-through. Prevents a single NaN wiping the whole normalized trace. Root-cause is the `v_analyte=0` (0/0) path (see F1). |
| F6 | **LOW** | Plotly loaded from unpinned/deprecated `plotly-latest.min.js` | L10 | The alias is frozen at EOL v1.58.5 (not "rolling latest"). Pin an explicit version (e.g. `plotly-2.35.2.min.js` + SRI; retest all tabs) and add a `typeof Plotly === 'undefined'` guard with a friendly message. For offline exams, vendor Plotly/Pico/MathJax locally. |
| F7 | **LOW** | Dead code: `const Z` (L623), `this.species_fractions` (L673), `n_protons` in `currentSimulationData` (L958/966) | as noted | Delete all three (recomputed/unused). Keep the unrelated component-level `comp.n_protons`. |
| F8 | **LOW** | `phi_values` fallback holds mL magnitudes when `phiIsValid` is false (misleading, latent) | `runFullSimulation` L948–950 | Set the invalid branch to `null` (sole consumer at L1031 is already gated by `usePhiAxis`), so any future unguarded read fails loudly instead of plotting mL as Φ. |
| F9 | **enhancement** | Dead `const alpha = h_plus - oh_minus` + debatable non-aqueous axis label | `calculateCurve` L610; `getAxisLabel` L517 | Remove the unused `alpha` (or use it inline for the numerator/denominator). Keep `−lg a(SH₂⁺)` — pH is itself an activity quantity, so it's consistent; optionally add a "γ=1, a≡c" note. |

---

## G. Feature & Didactic Enhancements

### Quick wins (data already available)

| # | Severity | Item | Location | Note |
|---|----------|------|----------|------|
| G1 | **MEDIUM** | Preset system for canonical Ph.-Eur. titrations | init L1814–1833; new `<select>` in sidebar | `PRESETS` map + `applyPreset(key)`: HCl, Essigsäure (4.76), H₃PO₄ (1.96/7.21/12.32), Na₂CO₃/HCl, Borsäure (±Mannitol), ASS-Rücktitration. Defer Glycin until zwitterion support. Set control values directly (avoid firing the mode `change` handler that wipes components). |
| G2 | **LOW** | Titratability badge (ΔpK ≥ 8 criterion) per ÄP | `computeEquivalenceMassTable` L1193–1208; ÄP lines L1095–1098 | Show "visuell / nur potentiometrisch / nicht direkt titrierbar" (Ehlers pKL−pKs≥8); gray out non-titratable ÄP lines. **Correct the base formula:** margin = `pKa_conj` itself (=pKL−pKb), *not* `pKL−pKa_conj` (that misclassifies ammonia). |
| G3 | **LOW** | Auto indicator recommendation per ÄP | indicator logic L1069–1092; `INDICATOR_DATA` | `recommendIndicator(phEP)`: filter indicators bracketing the ÄP pH; **rank by actual interpolated volume error**, not just |midpoint−phEP| (bracketing + midpoint alone mis-picks H₃PO₄ pT1 and carbonate EP1). Show "Empf.: …" per card. |
| G4 | **enhancement** | Analytic ÄP-pH formulas beside the interpolated value | `computeEquivalenceMassTable` L1193–1208 | Display textbook approximations (weak acid `½pKL+½pKa+½logC`; amphiprotic `½(pKa_i+pKa_{i+1})`; strong+strong `½pKL`). Use `params.pks` (not hardcoded 7) for non-aqueous solvents; detect strong steps to avoid feeding pKa=−1.74 into the weak formula. |
| G5 | **enhancement** | Overlay analytic Van Slyke β as a validation curve | `plotBufferCapacity` L1446–1539 | Thin dotted reference from the alphas; would have caught A1. Use the diluted formal conc and de Levie's exact `(m−n)²` coupling (not independent per-pKa) for close-pKa acids; generalize the water term to `10^-pKL`. |
| G6 | **enhancement** | Differentiate/fan out the two overlapping Gran ÄP annotations | `plotGranFunction` L1324–1342 | Label `G₁→…` / `G₂→…`, stack both above the axis (not `ay:+30`, which clips below y=0); merge when the two zeros agree within tolerance. |
| G7 | **enhancement** | Indikatorfehler as an interval over the Umschlagsbereich | `plotMainCurve` L1069–1088 | Interpolate V at `range[0]`/`range[1]`, report ΔV and draw a vertical band; guard when either bound has no crossing. |

### Larger / optional

| # | Severity | Item | Note |
|---|----------|------|------|
| G8 | **enhancement** | Multi-curve overlay ("Kurve fixieren") | Store `{volumes_ml, phi_values, phs, label}` in a `frozenCurves[]`, push as grey dashed traces; highest didactic value for strong-vs-weak / concentration / solvent comparisons. |
| G9 | **enhancement** | Printable Ph.-Eur. protocol | Hidden report `<div>` + `Plotly.toImage` PNG + `@media print`; persist indicator/back-titration results onto `currentSimulationData` first. |
| G10 | **enhancement** | State persistence (localStorage) + shareable `?state=` URL | Serialize **raw** control values (dropdown, indicator, tab), gate the hardcoded default (L1832) behind "no restored state". |
| G11 | **enhancement** | Light/dark theme toggle | `data-theme` fixed to dark; drive Plotly template from a variable; requires the transparent-background fix (D2) to look right. |
| G12 | **enhancement** | Temperature dependence of Kw/pKw (water only) | Single input feeding `pks`; fix `getAxisLabel`'s `|pKL−14|<0.005` water test so a T-shifted pKw still shows "pH". |
| G13 | **enhancement** | Ionic-strength / activity switch (Davies) + NaCl salt-effect demo | Cap I ≤ ~0.5 M (Davies validity); for the real ~1-unit H₃PO₄ pT2 shift at 10–15 % NaCl, ship a preset with literature conditional pKa' values rather than relying on Davies at high I. |
| G14 | **enhancement** | Hidden self-test harness (`#selftest`) | Assert strong+strong ÄP=½pKL, half-ÄP=pKa, amphiprotic=½(pK1+pK2), Σαᵢ=1, charge-balance residual≈0. The Σα=1 / mass-balance checks are near-trivial by construction — prefer the end-to-end and residual checks. |

---

## Verified Correct (checked and sound)

- **Charge-balance engine** reproduces independent solvers to machine precision: Na₂CO₃/HCl carbonate curve, acetic-acid and diprotic titrations, back-titration mass/purity (stoichiometric, unaffected by the product-chemistry limitation).
- **`beta_dynamic` = |dc/dpH|** (cyan trace) is the correct dilution-corrected buffer capacity and peaks in the buffer region (only mislabeled relative to pKa — see B5).
- **Gran post-ÄP G₂** `(V0+V)·10^(pH−pKL)` is the correct excess-strong-titrant form (de Levie eq 52).
- **Main titration curve** is not distorted by the V-tail — every V-axis plot clips to `[0, plot_end_volume_ml]`.
- **Buffer-capacity `numGradient` calls** (`x = phs`) are safe from the zero-denominator bug (distinct linspace pH values).
- **`averageCharge` base branch** correctly bakes in monovalent counter-cations (engine numerics right for salt bases despite the label bug).
- **Non-aqueous axis label** `−lg a(SH₂⁺)` is a consistent generalization of pH (pH is itself an activity quantity).
- **Native `<button>` tabs** are keyboard-operable (Tab/Enter/Space) and inactive panels are correctly removed from the a11y tree via `display:none` — only the ARIA selected-state/labeling is missing.
- **Contrast/target-size** of the remove button (24 px, ~5:1) meet WCAG minimums; only its accessible name is wrong.