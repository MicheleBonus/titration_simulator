# Titrations-Simulator: Bugfixes & Features — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix 6 confirmed bugs and implement 7 didactic/UX features in `sb_titration.html`.

**Architecture:** Single HTML file (vanilla JS, PicoCSS, Plotly.js). All changes stay in this file. New helper functions are added to the HILFSFUNKTIONEN section (~line 373). UI changes go in the HTML body. Plot functions go in their existing locations. No new files or dependencies.

**Tech Stack:** Vanilla JavaScript ES6, Plotly.js (CDN), PicoCSS v2 (CDN), MathJax v3 (CDN)

---

## File Map

| What | Location in `sb_titration.html` |
|---|---|
| Indicator color data | ~line 336 (`INDICATOR_DATA`) |
| Helper functions | ~line 373 (`HILFSFUNKTIONEN`) |
| `TitrationCalculator` class | ~line 472 |
| `getParams()` | ~line 730 |
| `runFullSimulation()` | ~line 773 |
| `plotMainCurve()` | ~line 894 |
| `plotDerivatives()` | ~line 1061 |
| `plotSpecies()` | ~line 1095 |
| `plotBufferCapacity()` | ~line 1167 |
| `plotInflectionPoints()` | ~line 1239 |
| `runBackTitration()` | ~line 1302 |
| Event listeners / init | ~line 1400 |
| HTML: Tab 2 | ~line 239 |
| HTML: Tab 3 | ~line 244 |
| HTML: Tab 5 | ~line 299 |
| HTML: Sidebar (pKL slider) | ~line 158 |

---

## Task 1: Add helper functions (formatSpeciesLabelPlain, getAxisLabel, downloadCSV)

These utilities are used by multiple later tasks. Add them first so subsequent tasks can reference them.

**Files:** Modify `sb_titration.html` — add to HILFSFUNKTIONEN section after `formationFunction` (~line 457)

- [ ] **Step 1: Add three helper functions after `formationFunction`**

Find the line:
```js
        // CO₂ formation function m̄ for H₂CO₃ (Eq. [35])
```

Insert BEFORE that line:

```js
        // Unicode subscript/superscript maps for Plotly trace names (no HTML in Plotly legends)
        const SUB = ['₀','₁','₂','₃','₄','₅','₆','₇','₈','₉'];
        const SUP_DIGIT = ['⁰','¹','²','³','⁴','⁵','⁶','⁷','⁸','⁹'];

        function formatSpeciesLabelPlain(baseName, totalProtons, numProtonsOnSpecies, isAcid) {
            const charge = isAcid ? (numProtonsOnSpecies - totalProtons) : numProtonsOnSpecies;
            let chargeStr = '';
            if (charge === 1) chargeStr = '⁺';
            else if (charge === -1) chargeStr = '⁻';
            else if (charge > 1) chargeStr = (SUP_DIGIT[charge] || String(charge)) + '⁺';
            else if (charge < -1) chargeStr = (SUP_DIGIT[-charge] || String(-charge)) + '⁻';
            let protonStr = '';
            if (numProtonsOnSpecies === 1) protonStr = 'H';
            else if (numProtonsOnSpecies > 1) protonStr = 'H' + (SUB[numProtonsOnSpecies] || numProtonsOnSpecies);
            return `${protonStr}${baseName}${chargeStr}`;
        }

        // Returns axis label depending on solvent: 'pH' for water, lyonium notation otherwise
        function getAxisLabel(pks) {
            return Math.abs(parseFloat(pks) - 14.00) < 0.005 ? 'pH' : '−lg\u2009a(SH₂⁺)';
        }

        // Download an array-of-arrays as CSV
        function downloadCSV(filename, rows) {
            const csv = rows.map(r => r.map(v => (v === null || v === undefined) ? '' : v).join(',')).join('\n');
            const blob = new Blob([csv], { type: 'text/csv;charset=utf-8;' });
            const url = URL.createObjectURL(blob);
            const a = Object.assign(document.createElement('a'), { href: url, download: filename });
            document.body.appendChild(a);
            a.click();
            document.body.removeChild(a);
            URL.revokeObjectURL(url);
        }

```

- [ ] **Step 2: Verify in browser**

Open `sb_titration.html`. Open browser console (F12). Type:
```js
console.log(formatSpeciesLabelPlain('A', 2, 2, true));  // Expected: H₂A
console.log(formatSpeciesLabelPlain('A', 2, 1, true));  // Expected: HA⁻
console.log(formatSpeciesLabelPlain('A', 2, 0, true));  // Expected: A²⁻
console.log(getAxisLabel(14.00));                        // Expected: pH
console.log(getAxisLabel(16.90));                        // Expected: −lg a(SH₂⁺)
```

- [ ] **Step 3: Commit**

```bash
git add sb_titration.html
git commit -m "feat: add formatSpeciesLabelPlain, getAxisLabel, downloadCSV helpers"
```

---

## Task 2: B1 — Fix Nitramin indicator color

**Files:** Modify `sb_titration.html` — `INDICATOR_DATA` constant (~line 363)

- [ ] **Step 1: Replace invalid color**

Find:
```js
            "Nitramin": { range: [10.8, 13.0], colors: ["rgba(0,0,0,0)", "orangebrown"] },
```

Replace with:
```js
            "Nitramin": { range: [10.8, 13.0], colors: ["rgba(0,0,0,0)", "saddlebrown"] },
```

- [ ] **Step 2: Verify in browser**

Select „Nitramin" from the Indikator dropdown in Tab 1. The gradient band at pH 10.8–13.0 should appear brown (not black).

- [ ] **Step 3: Commit**

```bash
git add sb_titration.html
git commit -m "fix: replace invalid 'orangebrown' CSS color with 'saddlebrown' for Nitramin"
```

---

## Task 3: B2 — Fix division-by-zero in buffer capacity plot

**Files:** Modify `sb_titration.html` — `plotBufferCapacity()` (~line 1179)

- [ ] **Step 1: Filter Infinity/NaN from beta_static**

Find:
```js
            const beta_static = dv_dph.map(v => Math.abs(1 / v));
```

Replace with:
```js
            const beta_static = dv_dph.map(v =>
                (v === 0 || !isFinite(v)) ? null : Math.abs(1 / v)
            );
```

- [ ] **Step 2: Verify in browser**

Go to Tab 5 (Pufferkapazität). The β-Kurve (statisch) should not show vertical spikes at the edges of the pH range. The plot should be smooth.

Also test with a strong acid/strong base system: set pKa of analyte to −2 (starke Säure) and observe the buffer plot — no Infinity artefacts.

- [ ] **Step 3: Commit**

```bash
git add sb_titration.html
git commit -m "fix: filter Infinity/NaN from static buffer capacity to prevent plot artefacts"
```

---

## Task 4: B3 — Fix Φ-axis x_end fallback

**Files:** Modify `sb_titration.html` — `plotMainCurve()` (~line 915)

- [ ] **Step 1: Fix fallback value**

Find:
```js
            const x_end = usePhiAxis
                ? (x_eq.length > 0 ? Math.max(...x_eq) * 1.5 : plot_end_volume_ml)
                : plot_end_volume_ml;
```

Replace with:
```js
            const x_end = usePhiAxis
                ? (x_eq.length > 0 ? Math.max(...x_eq) * 1.5 : 2.0)
                : plot_end_volume_ml;
```

- [ ] **Step 2: Verify in browser**

Switch X-Achse to „Titrationsgrad Φ". The axis should end at a sensible Φ value (~1.5 × last equivalence Φ), not at ~75 (mL value).

- [ ] **Step 3: Commit**

```bash
git add sb_titration.html
git commit -m "fix: use dimensionless fallback (2.0) for phi-axis x_end when no equivalence points"
```

---

## Task 5: B5 — Fix HTML tags in Plotly legend (use formatSpeciesLabelPlain)

**Files:** Modify `sb_titration.html` — `runFullSimulation()` (~line 813)

- [ ] **Step 1: Switch label generation to plain Unicode**

Find:
```js
                const labels = Array.from({length: n_k + 1}, (_, i) =>
                    formatSpeciesLabel(comp.symbol, n_k, n_k - i, comp.is_acid));
```

Replace with:
```js
                const labels = Array.from({length: n_k + 1}, (_, i) =>
                    formatSpeciesLabelPlain(comp.symbol, n_k, n_k - i, comp.is_acid));
```

- [ ] **Step 2: Verify in browser**

Go to Tab 3 (Speziesverteilung). For a diprotic acid (default: H₂A), the legend should show:
- `H₂A` (not `H<sub>2</sub>A`)
- `HA⁻` (not `HA<sup>-</sup>`)
- `A²⁻` (not `A<sup>2-</sup>`)

- [ ] **Step 3: Commit**

```bash
git add sb_titration.html
git commit -m "fix: use Unicode subscripts/superscripts in Plotly legend names instead of raw HTML"
```

---

## Task 6: B6 — Warn when pKₐ values are silently reordered

**Files:** Modify `sb_titration.html` — `getParams()` (~line 742)

- [ ] **Step 1: Capture order before sort, warn if changed**

Find in `getParams()` inside the `componentDivs.forEach` loop:
```js
                pkas.sort((a, b) => a - b);
                components.push({ symbol, concentration, n_protons: pkas.length, pkas, is_acid: isAcid });
```

Replace with:
```js
                const unsortedPkas = [...pkas];
                pkas.sort((a, b) => a - b);
                if (pkas.some((v, i) => v !== unsortedPkas[i])) {
                    anyPkaReordered = true;
                }
                components.push({ symbol, concentration, n_protons: pkas.length, pkas, is_acid: isAcid });
```

Then find the line BEFORE the `componentDivs.forEach` call (just before it begins):
```js
            componentDivs.forEach((div, compIdx) => {
```

Insert before it:
```js
            let anyPkaReordered = false;
```

Then find at the end of `getParams()`, just before the `return { valid: true, ... }`:
```js
            return {
                valid: true,
```

Insert before it:
```js
            if (anyPkaReordered) {
                showSimulationNotice('Hinweis: pKₐ-Werte wurden in aufsteigender Reihenfolge sortiert.');
            }
```

- [ ] **Step 2: Verify in browser**

Enter two pKₐ values out of order (e.g., pKₐ₁ = 9.0, pKₐ₂ = 4.0). A notice should appear: „Hinweis: pKₐ-Werte wurden in aufsteigender Reihenfolge sortiert." Correcting the order makes the notice disappear.

- [ ] **Step 3: Commit**

```bash
git add sb_titration.html
git commit -m "fix: notify user when pKa values are reordered during validation"
```

---

## Task 7: F9 — Solvent context (dropdown, dynamic axis label, pKL range 0–50)

This is the largest structural change to the UI. It replaces the pKL range slider with a solvent dropdown + number input.

**Files:** Modify `sb_titration.html` — HTML sidebar (~line 158) + `ui` object (~line 564) + event listeners (~line 1429)

- [ ] **Step 1: Replace slider HTML in sidebar**

Find and replace this block entirely:
```html
                    <label for="pks">Lösungsmittel pK<sub>L</sub>: <span id="pks_value">14.00</span></label>
                    <input type="range" id="pks" min="0" max="30" value="14.0" step="0.01" title="Die Autoprotolysekonstante des Lösungsmittels (z.B. ist pKL für Wasser pKw = 14.00).">
```

Replace with:
```html
                    <label for="solvent_select">Lösungsmittel</label>
                    <select id="solvent_select">
                        <option value="14.00|protic" selected>Wasser (pK<sub>L</sub> = 14.00)</option>
                        <option value="16.90|protic">Methanol (pK<sub>L</sub> = 16.90)</option>
                        <option value="19.10|protic">Ethanol (pK<sub>L</sub> = 19.10)</option>
                        <option value="14.45|protic">Eisessig (pK<sub>L</sub> = 14.45)</option>
                        <option value="29.40|polar-aprotic">DMF (pK<sub>L</sub> = 29.40)</option>
                        <option value="33.30|polar-aprotic">Acetonitril (pK<sub>L</sub> = 33.30)</option>
                        <option value="35.00|polar-aprotic">DMSO (pK<sub>L</sub> = 35.00)</option>
                        <option value="custom|custom">Benutzerdefiniert</option>
                    </select>
                    <label for="pks">pK<sub>L</sub>: <span id="pks_value">14.00</span></label>
                    <input type="number" id="pks" min="0" max="50" value="14.0" step="0.01"
                           readonly title="Autoprotolysekonstante des Lösungsmittels. Bei 'Benutzerdefiniert' editierbar.">
                    <small id="solvent-warning" style="color: var(--pico-del-color); display:none;"></small>
```

- [ ] **Step 2: Add `solventSelect` to the `ui` object**

Find:
```js
            pksSlider: document.getElementById('pks'),
            pksValue: document.getElementById('pks_value'),
```

Replace with:
```js
            pksSlider: document.getElementById('pks'),
            pksValue: document.getElementById('pks_value'),
            solventSelect: document.getElementById('solvent_select'),
            solventWarning: document.getElementById('solvent-warning'),
```

- [ ] **Step 3: Replace pks slider event listener**

Find:
```js
        ui.pksSlider.addEventListener('input', () => {
            ui.pksValue.textContent = parseFloat(ui.pksSlider.value).toFixed(2);
            runFullSimulation();
        });
```

Replace with:
```js
        function updateSolventUI() {
            const [pklStr, solventClass] = ui.solventSelect.value.split('|');
            const isCustom = solventClass === 'custom';
            const isAprotic = solventClass === 'polar-aprotic';
            const pklVal = isCustom ? parseFloat(ui.pksSlider.value) : parseFloat(pklStr);

            if (!isCustom) {
                ui.pksSlider.value = pklVal;
                ui.pksSlider.readOnly = true;
            } else {
                ui.pksSlider.readOnly = false;
            }
            ui.pksValue.textContent = parseFloat(ui.pksSlider.value).toFixed(2);

            if (isAprotic) {
                ui.solventWarning.textContent = 'Hinweis: pKₐ-Werte aus wässrigen Tabellen gelten nicht für dieses Lösungsmittel. Lösungsmittelspezifische pKₐ-Werte verwenden.';
                ui.solventWarning.style.display = 'block';
            } else if (isCustom && parseFloat(ui.pksSlider.value) > 40) {
                ui.solventWarning.textContent = 'Hinweis: Für unpolare aprotische Lösungsmittel (Toluol, CHCl₃ etc.) ist der Charge-Balance-Ansatz nicht anwendbar. Die Titrationen sind in der Praxis möglich, lassen sich aber mit diesem Modell nicht simulieren (Ionenpaarbildung, keine Autoprotolyse).';
                ui.solventWarning.style.display = 'block';
            } else {
                ui.solventWarning.style.display = 'none';
            }
            runFullSimulation();
        }

        ui.solventSelect.addEventListener('change', updateSolventUI);
        ui.pksSlider.addEventListener('input', () => {
            ui.pksValue.textContent = parseFloat(ui.pksSlider.value).toFixed(2);
            if (parseFloat(ui.pksSlider.value) > 40) {
                ui.solventWarning.textContent = 'Hinweis: Für unpolare aprotische Lösungsmittel (Toluol, CHCl₃ etc.) ist der Charge-Balance-Ansatz nicht anwendbar.';
                ui.solventWarning.style.display = 'block';
            } else {
                ui.solventWarning.style.display = 'none';
            }
            runFullSimulation();
        });
```

- [ ] **Step 4: Update all hardcoded `'pH'` axis labels in plot functions**

In each plot function, replace the hardcoded `'pH'` strings used for **y-axis titles and hover labels** with `getAxisLabel(params.pks)`. Changes needed:

In `plotMainCurve`:
```js
// Find:
                yaxis: { title: 'pH', range: [0, params.pks] },
// Replace with:
                yaxis: { title: getAxisLabel(params.pks), range: [0, params.pks] },
```

```js
// Find (in hovertemplate):
                hovertemplate += "<b>pH: %{y:.2f}</b><br><br>[H₃O⁺]: %{customdata[0]:.2e} M<br>[OH⁻]: %{customdata[1]:.2e} M<br><extra></extra>";
// Replace with:
                hovertemplate += `<b>${getAxisLabel(params.pks)}: %{y:.2f}</b><br><br>[H₃O⁺]: %{customdata[0]:.2e} M<br>[OH⁻]: %{customdata[1]:.2e} M<br><extra></extra>`;
```

In `plotInflectionPoints`:
```js
// Find:
                yaxis: { title: 'pH', side: 'left' },
// Replace with:
                yaxis: { title: getAxisLabel(params.pks), side: 'left' },
```

In `plotSpecies` (the nbar layout):
```js
// Find:
                    xaxis: { title: 'Volumen zugegebener Maßlösung (mL)', range: [0, plot_end_volume_ml] },
                    yaxis: { title: 'n̄ (mittlere Protonenzahl)' },
// (no pH label here, skip)
```

In `plotBufferCapacity`:
```js
// Find:
                xaxis: { title: 'pH', range: [0, params.pks] },
// Replace with:
                xaxis: { title: getAxisLabel(params.pks), range: [0, params.pks] },
```

In `plotDerivatives`:
```js
// Find:
                yaxis: { title: 'ΔpH/ΔV' },
                yaxis2: { title: 'Δ²pH/ΔV²' },
// Replace with (using dynamic label):
                yaxis: { title: `Δ(${getAxisLabel(params.pks)})/ΔV` },
                yaxis2: { title: `Δ²(${getAxisLabel(params.pks)})/ΔV²` },
```

Note: `params` is not in scope of `plotDerivatives` directly — check the destructuring at the top of that function and add `params` if missing:
```js
        function plotDerivatives() {
            const { volumes_ml, titration, eq_vols_l, plot_end_volume_ml, params } = currentSimulationData;
```

- [ ] **Step 5: Verify in browser**

1. Default (Wasser): all axis labels show „pH", no warning.
2. Select „DMF": pKL input shows 29.40 (read-only), warning appears about non-aqueous pKa values, axis labels show „−lg a(SH₂⁺)".
3. Select „Benutzerdefiniert": pKL input becomes editable. Enter 42 → warning about unpolare aprotische Lösungsmittel.
4. Return to Wasser: labels back to „pH", no warning.

- [ ] **Step 6: Commit**

```bash
git add sb_titration.html
git commit -m "feat: replace pKL slider with solvent dropdown, dynamic axis labels, and aprotic solvent warnings (F9)"
```

---

## Task 8: B4 — Fix hardcoded pKL in back-titration

**Files:** Modify `sb_titration.html` — `runBackTitration()` (~line 1339)

This is easy now that Task 7 is done: `ui.pksSlider.value` reads the correct pKL regardless of solvent.

- [ ] **Step 1: Replace hardcoded pks**

Find (appears twice, in `params_total` and `params_leftover` via spread):
```js
                pks: 14.0,
```

The first occurrence is in `params_total` definition. Replace:
```js
                pks: parseFloat(ui.pksSlider.value),
```

The `params_leftover` uses `...params_total` (spread), so it inherits `pks` automatically. Verify the spread:
```js
            const params_leftover = {
                ...params_total,
                components: [{ ... }],
            };
```
This is correct — no second change needed.

- [ ] **Step 2: Also fix the hardcoded `range: [0, 14]` in the back-titration plot layout**

Find in `runBackTitration()`:
```js
                yaxis: { title: 'pH', range: [0, 14] },
```

Replace with:
```js
                yaxis: { title: getAxisLabel(parseFloat(ui.pksSlider.value)), range: [0, parseFloat(ui.pksSlider.value)] },
```

- [ ] **Step 3: Verify in browser**

Switch to Tab 4 (Rücktitration). Select DMF as solvent (pKL = 29.40). Run the back-titration — the plot y-axis should now reach 29.40, not 14.

- [ ] **Step 4: Commit**

```bash
git add sb_titration.html
git commit -m "fix: back-titration now reads pKL from solvent selector instead of hardcoded 14.0 (B4)"
```

---

## Task 9: F1 + F2 — Theoretical EP pH + pKₐᵢ for all half-EP annotations

**Files:** Modify `sb_titration.html` — `runFullSimulation()`, `computeEquivalenceMassTable()`, `plotMainCurve()`

- [ ] **Step 1: Add interpolatePH helper after the existing helper functions**

After `getAxisLabel` (added in Task 1), add:

```js
        function interpolatePH(volumes_ml, phs, target_vol_ml) {
            for (let i = 0; i < volumes_ml.length - 1; i++) {
                if (volumes_ml[i] <= target_vol_ml && volumes_ml[i + 1] >= target_vol_ml) {
                    const frac = (target_vol_ml - volumes_ml[i]) / (volumes_ml[i + 1] - volumes_ml[i]);
                    return phs[i] + frac * (phs[i + 1] - phs[i]);
                }
            }
            return null;
        }
```

- [ ] **Step 2: Store pKa for each half-EP in runFullSimulation**

In `runFullSimulation()`, find where `half_eq_vols_l` is built:
```js
                // Half-equivalence points
                let cumHalf = cumulative_vol - n_k * step_vol;
                for (let i = 0; i < n_k; i++) {
                    half_eq_vols_l.push(cumHalf + (i + 0.5) * step_vol);
                }
```

Replace with:
```js
                // Half-equivalence points
                let cumHalf = cumulative_vol - n_k * step_vol;
                for (let i = 0; i < n_k; i++) {
                    half_eq_vols_l.push(cumHalf + (i + 0.5) * step_vol);
                    half_eq_pkas.push(comp.pkas[i]);
                }
```

Find where `half_eq_vols_l` is declared (just before the `params.components.forEach` loop):
```js
            const eq_vols_l = [];
            const half_eq_vols_l = [];
```

Replace with:
```js
            const eq_vols_l = [];
            const half_eq_vols_l = [];
            const half_eq_pkas = [];
```

Add `half_eq_pkas` to `currentSimulationData`:
```js
            currentSimulationData = {
                titration, params, volumes_ml, n_protons, plot_end_volume_ml,
                eq_vols_l, half_eq_vols_l, species_labels, species_concentrations,
                all_species_labels, all_species_concentrations, phi_values,
                total_moles_titratable, inflection_points, phiIsValid
            };
```

Replace with:
```js
            currentSimulationData = {
                titration, params, volumes_ml, n_protons, plot_end_volume_ml,
                eq_vols_l, half_eq_vols_l, half_eq_pkas, species_labels, species_concentrations,
                all_species_labels, all_species_concentrations, phi_values,
                total_moles_titratable, inflection_points, phiIsValid
            };
```

- [ ] **Step 3: Update computeEquivalenceMassTable to show pH at EP**

Find the function signature:
```js
        function computeEquivalenceMassTable(params, eq_vols_l) {
```

Replace with:
```js
        function computeEquivalenceMassTable(params, eq_vols_l, volumes_ml, phs) {
```

Find the function call in `plotMainCurve`:
```js
            computeEquivalenceMassTable(params, eq_vols_l);
```

Replace with:
```js
            computeEquivalenceMassTable(params, eq_vols_l, volumes_ml, titration.phs);
```

Replace the function body:
```js
        function computeEquivalenceMassTable(params, eq_vols_l, volumes_ml, phs) {
            const totalEq = eq_vols_l.length;
            const totalVol = eq_vols_l.length > 0 ? eq_vols_l[eq_vols_l.length-1]*1000 : 0;
            const nComp = params.components.length;
            const tableDiv = document.getElementById('equivalence-table');
            let html = `
                <div><h3>Äquivalenzpunkte</h3><p>${totalEq}</p></div>
                <div><h3>Komponenten</h3><p>${nComp}</p></div>
                <div><h3>Gesamt-ÄP-Volumen</h3><p>${totalVol.toFixed(2)} mL</p></div>`;
            eq_vols_l.forEach((v, i) => {
                const ph = interpolatePH(volumes_ml, phs, v * 1000);
                const phStr = ph !== null ? ph.toFixed(2) : '—';
                html += `<div><h3>ÄP ${i+1} — ${getAxisLabel(params.pks)}</h3><p>${phStr}</p></div>`;
            });
            tableDiv.innerHTML = html;
        }
```

- [ ] **Step 4: Update half-EP annotations in plotMainCurve to use pKₐᵢ**

In `plotMainCurve`, find:
```js
            const { titration, params, volumes_ml, plot_end_volume_ml, eq_vols_l, half_eq_vols_l,
                    species_labels, species_concentrations, phi_values, inflection_points,
                    total_moles_titratable, phiIsValid } = currentSimulationData;
```

Replace with:
```js
            const { titration, params, volumes_ml, plot_end_volume_ml, eq_vols_l, half_eq_vols_l, half_eq_pkas,
                    species_labels, species_concentrations, phi_values, inflection_points,
                    total_moles_titratable, phiIsValid } = currentSimulationData;
```

Then find and replace the half-EP annotation block entirely:
```js
            if (x_half.length > 0) {
                layout.annotations.push({
                    x: x_half[0], y: params.pks*0.20,
                    text: '½ ÄP 1 ≈ pKₐ₁ (Puffer)',
                    showarrow: false,
                    font: { size: 12, color: 'lightgray' },
                    xanchor: 'left', yanchor: 'bottom'
                });
            }
```

Replace with:
```js
            const halfAnnotYLevels = [0.20, 0.30, 0.12, 0.38, 0.08];
            x_half.forEach((xv, i) => {
                const pka = half_eq_pkas[i];
                const pkaStr = pka !== undefined ? `= ${pka.toFixed(2)}` : '';
                layout.annotations.push({
                    x: xv,
                    y: params.pks * (halfAnnotYLevels[i % halfAnnotYLevels.length]),
                    text: `½ ÄP ${i+1} ≈ pKₐ${i+1} ${pkaStr}`,
                    showarrow: false,
                    font: { size: 11, color: 'lightgray' },
                    xanchor: 'left', yanchor: 'bottom'
                });
            });
```

- [ ] **Step 5: Verify in browser**

1. Default diprotic acid (A, pKa1=2.15, pKa2=7.20): ÄP-Tabelle now shows „ÄP 1 — pH: ~4.3" and „ÄP 2 — pH: ~8.9" (approximate values depending on concentrations).
2. Both half-EP dashed lines are annotated: „½ ÄP 1 ≈ pKₐ₁ = 2.15" and „½ ÄP 2 ≈ pKₐ₂ = 7.20".

- [ ] **Step 6: Commit**

```bash
git add sb_titration.html
git commit -m "feat: show theoretical pH at equivalence points, annotate all half-EP lines with pKa values (F1, F2)"
```

---

## Task 10: F8 — pKₐ annotations on buffer capacity plot

**Files:** Modify `sb_titration.html` — `plotBufferCapacity()` (~line 1215)

- [ ] **Step 1: Add pKₐ shapes and annotations to buffer plot layout**

In `plotBufferCapacity()`, find the layout definition:
```js
            const layout = {
                title: isNormalized
                    ? 'Pufferkapazität (normiert): β/βmax und βV/βVmax'
                    : 'Pufferkapazität: β (statisch) und βV (dynamisch, Gl. [93])',
                xaxis: { title: getAxisLabel(params.pks), range: [0, params.pks] },
                yaxis: { title: yAxisTitle, type: yScale },
                template: 'plotly_dark',
                legend: { yanchor: 'top', y: 0.95, xanchor: 'right', x: 0.95 }
            };
```

Replace with:
```js
            const pkaShapes = [];
            const pkaAnnotations = [];
            params.components.forEach((comp) => {
                comp.pkas.forEach((pka, i) => {
                    if (pka >= 0 && pka <= params.pks) {
                        pkaShapes.push({
                            type: 'line', x0: pka, y0: 0, x1: pka, y1: 1,
                            xref: 'x', yref: 'paper',
                            line: { color: 'rgba(255,165,0,0.6)', dash: 'dot', width: 1.5 }
                        });
                        pkaAnnotations.push({
                            x: pka, y: 0.97, xref: 'x', yref: 'paper',
                            text: `pKₐ${i+1}(${comp.symbol})`,
                            showarrow: false, xanchor: 'center', yanchor: 'top',
                            font: { size: 10, color: 'orange' },
                            bgcolor: 'rgba(0,0,0,0.4)', borderpad: 2
                        });
                    }
                });
            });

            const layout = {
                title: isNormalized
                    ? 'Pufferkapazität (normiert): β/βmax und βV/βVmax'
                    : 'Pufferkapazität: β (statisch) und βV (dynamisch, Gl. [93])',
                xaxis: { title: getAxisLabel(params.pks), range: [0, params.pks] },
                yaxis: { title: yAxisTitle, type: yScale },
                template: 'plotly_dark',
                legend: { yanchor: 'top', y: 0.95, xanchor: 'right', x: 0.95 },
                shapes: pkaShapes,
                annotations: pkaAnnotations
            };
```

- [ ] **Step 2: Verify in browser**

Go to Tab 5. For the default diprotic acid (pKa1=2.15, pKa2=7.20): two vertical orange dotted lines should appear at pH 2.15 and 7.20, each labeled „pKₐ1(A)" and „pKₐ2(A)". The β-peaks should align with these lines.

- [ ] **Step 3: Commit**

```bash
git add sb_titration.html
git commit -m "feat: add pKa annotations to buffer capacity plot to show peak-pKa relationship (F8)"
```

---

## Task 11: F7 — pH-axis option in species distribution (Tab 3)

**Files:** Modify `sb_titration.html` — HTML Tab 3 (~line 244) + `plotSpecies()` (~line 1095) + `ui` object + event listeners

- [ ] **Step 1: Add X-axis radio buttons to Tab 3 HTML**

Find the existing fieldset in Tab 3:
```html
                    <fieldset>
                        <legend>Y-Achse anzeigen als:</legend>
                        <input type="radio" name="species_plot_type" value="fraction" id="radio-fraction" checked>
                        <label for="radio-fraction">Anteile der Spezies (α)</label>
                        <input type="radio" name="species_plot_type" value="concentration" id="radio-concentration">
                        <label for="radio-concentration">Molare Konzentration</label>
                        <input type="radio" name="species_plot_type" value="nbar" id="radio-nbar">
                        <label for="radio-nbar" title="Bjerrum-Funktion: n̄ = mittlere Protonenzahl am Anion; α_i ist der Anteil von H_{q-i}L (Gl. [10])">Bjerrum-Funktion (n&#772;)</label>
                    </fieldset>
```

Replace with:
```html
                    <div style="display:grid; grid-template-columns:1fr 1fr; gap:1rem;">
                        <fieldset>
                            <legend>Y-Achse anzeigen als:</legend>
                            <input type="radio" name="species_plot_type" value="fraction" id="radio-fraction" checked>
                            <label for="radio-fraction">Anteile der Spezies (α)</label>
                            <input type="radio" name="species_plot_type" value="concentration" id="radio-concentration">
                            <label for="radio-concentration">Molare Konzentration</label>
                            <input type="radio" name="species_plot_type" value="nbar" id="radio-nbar">
                            <label for="radio-nbar" title="Bjerrum-Funktion: n̄ = mittlere Protonenzahl am Anion; α_i ist der Anteil von H_{q-i}L (Gl. [10])">Bjerrum-Funktion (n&#772;)</label>
                        </fieldset>
                        <fieldset>
                            <legend>X-Achse:</legend>
                            <input type="radio" name="species_x_axis" value="volume" id="radio-x-volume" checked>
                            <label for="radio-x-volume">Volumen (mL)</label>
                            <input type="radio" name="species_x_axis" value="ph" id="radio-x-ph">
                            <label for="radio-x-ph" title="Klassisches Bjerrum-Diagramm: Speziesverteilung gegen pH">pH-Achse (Bjerrum)</label>
                        </fieldset>
                    </div>
```

- [ ] **Step 2: Add event listener for x-axis radio buttons**

Find:
```js
        ui.speciesPlotType.forEach(radio => radio.addEventListener('change', plotSpecies));
```

Replace with:
```js
        ui.speciesPlotType.forEach(radio => radio.addEventListener('change', plotSpecies));
        document.getElementsByName('species_x_axis').forEach(radio => radio.addEventListener('change', plotSpecies));
```

- [ ] **Step 3: Update plotSpecies to support pH x-axis**

In `plotSpecies()`, find where x_data is used for the standard species distribution traces.

Find at the top of `plotSpecies`:
```js
        function plotSpecies() {
            const { volumes_ml, titration, species_labels, species_concentrations, plot_end_volume_ml, params,
                    all_species_labels, all_species_concentrations } = currentSimulationData;
```

Replace with:
```js
        function plotSpecies() {
            const { volumes_ml, titration, species_labels, species_concentrations, plot_end_volume_ml, params,
                    all_species_labels, all_species_concentrations } = currentSimulationData;
            const xMode = (document.querySelector('input[name="species_x_axis"]:checked') || {value:'volume'}).value;
            const usePhAxis = xMode === 'ph';
            const x_data_species = usePhAxis ? titration.phs : volumes_ml;
            const x_title_species = usePhAxis ? getAxisLabel(params.pks) : 'Volumen zugegebener Maßlösung (mL)';
            const x_range_species = usePhAxis ? [0, params.pks] : [0, plot_end_volume_ml];
```

Then in the nbar layout, replace:
```js
                    xaxis: { title: 'Volumen zugegebener Maßlösung (mL)', range: [0, plot_end_volume_ml] },
```
with:
```js
                    xaxis: { title: x_title_species, range: x_range_species },
```

And update nbar traces to use `x_data_species`:
```js
                        data.push({
                            x: x_data_species, y: titration.nbar_values[k],
```

In the standard species distribution section, replace each occurrence of `volumes_ml` in trace `x` fields and layout `xaxis` with `x_data_species` / `x_title_species` / `x_range_species`:

```js
                        data.push({
                            x: x_data_species, y: y_source[i], mode: 'lines',
```

```js
                xaxis: { title: x_title_species, range: x_range_species },
```

- [ ] **Step 4: Verify in browser**

1. Tab 3, X-Achse „Volumen" (default): identical to before.
2. Switch to „pH-Achse (Bjerrum)": species curves now plotted against pH (0–14). For the default diprotic acid, the crossover between H₂A and HA⁻ should occur at pH ≈ pKa1 = 2.15, and HA⁻/A²⁻ at pH ≈ 7.20. This matches the textbook Bjerrum diagram.

- [ ] **Step 5: Commit**

```bash
git add sb_titration.html
git commit -m "feat: add pH-axis option for species distribution plot (Bjerrum diagram) in Tab 3 (F7)"
```

---

## Task 12: F4 — Gran plot sub-tab in Tab 2

**Files:** Modify `sb_titration.html` — HTML Tab 2 (~line 239) + new `plotGranFunction()` + `updateAllPlots()` + event listeners

- [ ] **Step 1: Redesign Tab 2 HTML with sub-navigation**

Find and replace the Tab 2 article entirely:
```html
                <article id="tab2" class="tab-content">
                    <h3>Ableitungsdiagramme zur Endpunkterkennung</h3>
                    <div id="plot-derivatives" style="width:100%; height:450px;"></div>
                </article>
```

Replace with:
```html
                <article id="tab2" class="tab-content">
                    <h3>Endpunktanalyse</h3>
                    <div style="display:flex; gap:0.5rem; margin-bottom:1rem;">
                        <button id="tab2-btn-derivatives" onclick="switchTab2('derivatives')"
                                style="flex:1;">Ableitungsdiagramme</button>
                        <button id="tab2-btn-gran" class="outline" onclick="switchTab2('gran')"
                                style="flex:1;">Gran-Plot</button>
                    </div>
                    <div id="tab2-panel-derivatives">
                        <div id="plot-derivatives" style="width:100%; height:450px;"></div>
                    </div>
                    <div id="tab2-panel-gran" style="display:none;">
                        <p style="font-size:0.9em; color:var(--pico-muted-color);">
                            Linearisierung der Titrationskurve (Gran 1952).
                            G₁ = (V₀+V)·10<sup>−p[H]</sup> (vor ÄP) und
                            G₂ = (V₀+V)·10<sup>p[H]−pK<sub>L</sub></sup> (nach ÄP)
                            sind jeweils linear in V; die Nullstellen geben das Äquivalenzvolumen.
                        </p>
                        <div id="plot-gran" style="width:100%; height:430px;"></div>
                    </div>
                </article>
```

- [ ] **Step 2: Add switchTab2 function and tab2Mode state**

Before `updateAllPlots()`, add:

```js
        let tab2Mode = 'derivatives';

        function switchTab2(mode) {
            tab2Mode = mode;
            document.getElementById('tab2-panel-derivatives').style.display = mode === 'derivatives' ? 'block' : 'none';
            document.getElementById('tab2-panel-gran').style.display = mode === 'gran' ? 'block' : 'none';
            document.getElementById('tab2-btn-derivatives').classList.toggle('outline', mode !== 'derivatives');
            document.getElementById('tab2-btn-gran').classList.toggle('outline', mode === 'derivatives');
            if (mode === 'gran') plotGranFunction();
            else plotDerivatives();
        }
```

- [ ] **Step 3: Add linear regression helper**

Add after `interpolatePH` (Task 9, Step 1):

```js
        function linReg(xs, ys) {
            const n = xs.length;
            if (n < 2) return null;
            const sx = xs.reduce((a, b) => a + b, 0);
            const sy = ys.reduce((a, b) => a + b, 0);
            const sxy = xs.reduce((a, x, i) => a + x * ys[i], 0);
            const sxx = xs.reduce((a, x) => a + x * x, 0);
            const det = n * sxx - sx * sx;
            if (Math.abs(det) < 1e-12) return null;
            const m = (n * sxy - sx * sy) / det;
            const b = (sy - m * sx) / n;
            const zero = Math.abs(m) > 1e-12 ? -b / m : null;
            return { m, b, zero };
        }
```

- [ ] **Step 4: Add plotGranFunction**

Add after `plotDerivatives()`:

```js
        function plotGranFunction() {
            const { titration, params, volumes_ml, plot_end_volume_ml, eq_vols_l } = currentSimulationData;
            if (!titration || volumes_ml.length < 4 || eq_vols_l.length === 0) {
                Plotly.purge('plot-gran');
                document.getElementById('plot-gran').innerHTML = '<p>Nicht genügend Daten für Gran-Plot.</p>';
                return;
            }

            const V0_ml = params.v_analyte * 1000;
            const pKL = params.pks;
            const isAcidAnalyte = params.analyte_is_acid;
            const colors = ['royalblue', 'tomato', 'mediumseagreen', 'orange', 'violet'];
            const data = [];
            const shapes = [];
            const annotations = [];

            eq_vols_l.forEach((eq_v_l, eqIdx) => {
                const Veq = eq_v_l * 1000;
                const prevVeq = eqIdx > 0 ? eq_vols_l[eqIdx - 1] * 1000 : 0;
                const nextVeq = eqIdx < eq_vols_l.length - 1 ? eq_vols_l[eqIdx + 1] * 1000 : plot_end_volume_ml;
                const color = colors[eqIdx % colors.length];

                // Window: use 30% of surrounding segment on each side
                const windowBefore = Math.max((Veq - prevVeq) * 0.35, 1.0);
                const windowAfter = Math.max((nextVeq - Veq) * 0.35, 1.0);

                const beforeIdx = volumes_ml.reduce((acc, v, i) => {
                    if (v >= Veq - windowBefore && v < Veq) acc.push(i);
                    return acc;
                }, []);
                const afterIdx = volumes_ml.reduce((acc, v, i) => {
                    if (v > Veq && v <= Veq + windowAfter) acc.push(i);
                    return acc;
                }, []);

                // Gran functions:
                // isAcidAnalyte (base titrant → acid): G1 = (V0+V)*10^(-pH), G2 = (V0+V)*10^(pH-pKL)
                // !isAcidAnalyte (acid titrant → base): G1 = (V0+V)*10^(pH-pKL), G2 = (V0+V)*10^(-pH)
                const granG = (idx, isBefore) => {
                    const v = volumes_ml[idx];
                    const ph = titration.phs[idx];
                    const f = V0_ml + v;
                    if (isAcidAnalyte) {
                        return isBefore ? f * Math.pow(10, -ph) : f * Math.pow(10, ph - pKL);
                    } else {
                        return isBefore ? f * Math.pow(10, ph - pKL) : f * Math.pow(10, -ph);
                    }
                };

                const g1x = beforeIdx.map(i => volumes_ml[i]);
                const g1y = beforeIdx.map(i => granG(i, true));
                const g2x = afterIdx.map(i => volumes_ml[i]);
                const g2y = afterIdx.map(i => granG(i, false));

                // Traces for data points
                if (g1x.length > 0) {
                    data.push({
                        x: g1x, y: g1y, mode: 'markers', name: `G₁ (vor ÄP ${eqIdx+1})`,
                        marker: { color, size: 5 }, legendgroup: `ep${eqIdx}`
                    });
                }
                if (g2x.length > 0) {
                    data.push({
                        x: g2x, y: g2y, mode: 'markers', name: `G₂ (nach ÄP ${eqIdx+1})`,
                        marker: { color, size: 5, symbol: 'diamond' }, legendgroup: `ep${eqIdx}`
                    });
                }

                // Linear regression + extrapolation lines
                const reg1 = linReg(g1x, g1y);
                const reg2 = linReg(g2x, g2y);

                if (reg1 && reg1.zero !== null && isFinite(reg1.zero)) {
                    const xA = Math.max(0, g1x[0] - windowBefore * 0.5);
                    const xB = reg1.zero;
                    data.push({
                        x: [xA, xB], y: [reg1.m*xA+reg1.b, 0],
                        mode: 'lines', line: { color, dash: 'dot', width: 1.5 },
                        name: `Extrap. G₁(ÄP ${eqIdx+1})`, showlegend: false, legendgroup: `ep${eqIdx}`
                    });
                    annotations.push({
                        x: reg1.zero, y: 0, text: `ÄP ${eqIdx+1}<br>${reg1.zero.toFixed(2)} mL`,
                        showarrow: true, arrowhead: 2, ax: 0, ay: -30,
                        font: { size: 10, color }, bgcolor: 'rgba(0,0,0,0.5)', borderpad: 2
                    });
                }
                if (reg2 && reg2.zero !== null && isFinite(reg2.zero)) {
                    const xA = reg2.zero;
                    const xB = g2x[g2x.length-1] + windowAfter * 0.5;
                    data.push({
                        x: [xA, g2x[g2x.length-1]], y: [0, reg2.m*g2x[g2x.length-1]+reg2.b],
                        mode: 'lines', line: { color, dash: 'dash', width: 1.5 },
                        name: `Extrap. G₂(ÄP ${eqIdx+1})`, showlegend: false, legendgroup: `ep${eqIdx}`
                    });
                }

                // Vertical line at known EP
                shapes.push({
                    type: 'line', x0: Veq, y0: 0, x1: Veq, y1: 1,
                    xref: 'x', yref: 'paper',
                    line: { color: 'crimson', dash: 'dash', width: 1 }
                });
            });

            const gLabel = isAcidAnalyte
                ? 'G₁=(V₀+V)·10⁻ᵖᴴ / G₂=(V₀+V)·10ᵖᴴ⁻ᵖᴷᴸ'
                : 'G₁=(V₀+V)·10ᵖᴴ⁻ᵖᴷᴸ / G₂=(V₀+V)·10⁻ᵖᴴ';

            Plotly.newPlot('plot-gran', data, {
                title: `Gran-Plot — ${gLabel}`,
                xaxis: { title: 'Volumen zugegebener Maßlösung (mL)', range: [0, plot_end_volume_ml] },
                yaxis: { title: 'Gran-Funktion G(V)' },
                template: 'plotly_dark',
                shapes, annotations,
                legend: { yanchor: 'top', y: 0.95, xanchor: 'left', x: 0.05 }
            }, { responsive: true });
        }
```

- [ ] **Step 5: Update updateAllPlots to include Gran**

Find:
```js
        function updateAllPlots() {
            plotMainCurve();
            plotDerivatives();
            plotSpecies();
            plotBufferCapacity();
            plotInflectionPoints();
        }
```

Replace with:
```js
        function updateAllPlots() {
            plotMainCurve();
            if (tab2Mode === 'gran') plotGranFunction();
            else plotDerivatives();
            plotSpecies();
            plotBufferCapacity();
            plotInflectionPoints();
        }
```

- [ ] **Step 6: Verify in browser**

1. Tab 2 shows two buttons: „Ableitungsdiagramme" (active) and „Gran-Plot".
2. Click „Gran-Plot": G₁ and G₂ data points appear as scatter, with dotted/dashed extrapolation lines. For default diprotic acid/NaOH, two ÄP should be identified by the extrapolation lines. Zero crossings should be close to the theoretical ÄP volumes.
3. Switch back to „Ableitungsdiagramme": derivatives plot still works.

- [ ] **Step 7: Commit**

```bash
git add sb_titration.html
git commit -m "feat: add Gran plot sub-tab to Tab 2 for linearised endpoint detection (F4)"
```

---

## Task 13: F6 — CSV export buttons

**Files:** Modify `sb_titration.html` — each tab's article element + `plotMainCurve()` / `plotBufferCapacity()` / `plotSpecies()`

Strategy: add a single export `<div>` under each plot container, populated via JS after each plot update. Use `Plotly.downloadImage` for PNG (Plotly modebar already provides this, so we just add a redundant button for discoverability) and `downloadCSV` for data.

- [ ] **Step 1: Add an export helper that attaches CSV button to a plot div**

Add after `downloadCSV` helper (Task 1):

```js
        function attachCSVExport(plotDivId, getRowsFn) {
            const plotEl = document.getElementById(plotDivId);
            if (!plotEl) return;
            // Remove previous export bar if present
            const prev = plotEl.parentElement.querySelector('.export-bar');
            if (prev) prev.remove();
            const bar = document.createElement('div');
            bar.className = 'export-bar';
            bar.style.cssText = 'text-align:right; margin-top:0.4rem;';
            const btn = document.createElement('button');
            btn.className = 'secondary outline';
            btn.style.cssText = 'width:auto; padding:0.25rem 0.7rem; font-size:0.82em;';
            btn.textContent = '⬇ CSV';
            btn.addEventListener('click', () => {
                const { rows, filename } = getRowsFn();
                downloadCSV(filename, rows);
            });
            bar.appendChild(btn);
            plotEl.parentElement.appendChild(bar);
        }
```

- [ ] **Step 2: Attach CSV export after plotMainCurve**

At the end of `plotMainCurve()`, before the closing `}`, add:

```js
            attachCSVExport('plot-main', () => {
                const { titration, params, volumes_ml, species_labels } = currentSimulationData;
                const axLabel = getAxisLabel(params.pks);
                const header = ['V_mL', axLabel, '[H3O+]_M', '[OH-]_M',
                    ...species_labels.map(l => `[${l}]_M`)];
                const rows = [header];
                volumes_ml.forEach((v, i) => {
                    const h = Math.pow(10, -titration.phs[i]);
                    const oh = Math.pow(10, titration.phs[i] - params.pks);
                    const specConcs = (currentSimulationData.species_concentrations || []).map(sc => sc[i] !== undefined ? sc[i].toExponential(4) : '');
                    rows.push([v.toFixed(4), titration.phs[i].toFixed(4),
                        h.toExponential(4), oh.toExponential(4), ...specConcs]);
                });
                return { rows, filename: 'titration_curve.csv' };
            });
```

- [ ] **Step 3: Attach CSV export after plotBufferCapacity**

At the end of `plotBufferCapacity()`, before the closing `}`, add:

```js
            attachCSVExport('plot-buffer', () => {
                const { titration, params } = currentSimulationData;
                const axLabel = getAxisLabel(params.pks);
                const header = [axLabel, 'beta_static', 'beta_dynamic'];
                // Recompute the same values (already computed above in this function scope — but we're outside it)
                // So recompute here using currentSimulationData
                const dv_dph2 = numGradient(titration.volumes, titration.phs);
                const bs2 = dv_dph2.map(v => (v === 0 || !isFinite(v)) ? null : Math.abs(1 / v));
                const c2 = titration.volumes.map(v => (params.c_titrant * v) / (params.v_analyte + v));
                const dc2 = numGradient(c2, titration.phs);
                const bd2 = dc2.map(v => Math.abs(v));
                const rows = [header, ...titration.phs.map((ph, i) => [
                    ph.toFixed(4),
                    bs2[i] !== null ? bs2[i].toExponential(4) : '',
                    bd2[i].toExponential(4)
                ])];
                return { rows, filename: 'buffer_capacity.csv' };
            });
```

- [ ] **Step 4: Verify in browser**

1. In Tab 1, a small „⬇ CSV" button appears below the titration curve plot. Clicking downloads `titration_curve.csv` with columns V_mL, pH (or −lg a(SH₂⁺)), [H3O+], [OH-], species concentrations.
2. In Tab 5, a „⬇ CSV" button appears below the buffer capacity plot.
3. Plotly's own camera icon (top-right of each plot) still works for PNG download.

- [ ] **Step 5: Commit**

```bash
git add sb_titration.html
git commit -m "feat: add CSV export buttons to titration curve and buffer capacity plots (F6)"
```

---

## Self-Review

**Spec coverage check:**

| Requirement | Task |
|---|---|
| B1: orangebrown | Task 2 ✓ |
| B2: buffer infinity | Task 3 ✓ |
| B3: phi fallback | Task 4 ✓ |
| B4: hardcoded pKL back-titration | Task 8 ✓ |
| B5: HTML in Plotly legend | Task 5 ✓ |
| B6: pKa sort warning | Task 6 ✓ |
| F1: EP theoretical pH | Task 9 ✓ |
| F2: half-EP pKa annotations | Task 9 ✓ |
| F4: Gran plot | Task 12 ✓ |
| F6: Export (CSV + PNG via modebar) | Task 13 ✓ |
| F7: pH-axis in species | Task 11 ✓ |
| F8: pKa annotations on buffer | Task 10 ✓ |
| F9: Solvent dropdown + dynamic label + warnings | Task 7 ✓ |

**Placeholder scan:** No TBDs or incomplete steps found.

**Type consistency:**
- `half_eq_pkas` introduced in Task 9 Step 2, used in Task 9 Step 4 ✓
- `getAxisLabel(params.pks)` — `params.pks` is always `parseFloat(ui.pksSlider.value)` ✓
- `downloadCSV` defined Task 1, used Task 13 ✓
- `linReg` defined Task 12 Step 3, used Task 12 Step 4 ✓
- `interpolatePH` defined Task 9 Step 1, used Task 9 Step 3 ✓
- `attachCSVExport` defined Task 13 Step 1, used Task 13 Steps 2 & 3 ✓
- `switchTab2` references `tab2Mode`, `plotGranFunction`, `plotDerivatives` — all defined before use ✓

**Ordering note:** Tasks must be executed in order 1→13 because:
- Task 1 defines helpers used by Tasks 5, 7, 9, 10, 11, 12, 13
- Task 7 changes the pKL input element; Task 8 must run after Task 7
- Task 9 Step 2 adds `half_eq_pkas`; Task 9 Step 4 consumes it (same task, fine)
