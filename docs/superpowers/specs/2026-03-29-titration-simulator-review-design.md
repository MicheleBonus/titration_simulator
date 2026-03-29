# Titrations-Simulator: Review & Erweiterung — Design-Spec

**Datum:** 2026-03-29
**Zielgruppe:** Pharmaziestudierende
**Datei:** `sb_titration.html` (Single-file, Vanilla JS, PicoCSS, Plotly.js)
**Grundlage:** Asuero & Michałowski (2011), Kalka (2021), Ehlers-Skript

---

## Überblick

Der Simulator ist ein didaktisch ausgereiftes Werkzeug für Säure-Base-Titrationen mit Ladungsbilanzmethode (parametrisch, pH als freie Variable). Diese Spec beschreibt 6 Bugfixes und 7 Feature-Erweiterungen, die in einem einzigen Entwicklungszyklus umgesetzt werden.

---

## Teil 1: Bugfixes

### B1 — Ungültige CSS-Farbe „orangebrown"

**Problem:** `INDICATOR_DATA` enthält für Nitramin `colors: ["rgba(0,0,0,0)", "orangebrown"]`. `"orangebrown"` ist kein gültiger CSS-Farbname; der Browser-Canvas interpretiert ihn als Schwarz (`#000000`), wodurch der Farbgradient des Indikators falsch dargestellt wird.

**Fix:** Ersetzen durch `"saddlebrown"` (CSS-Standard, visuell passend zu Nitramin-Umschlagfarbe).

---

### B2 — Division-by-zero in plotBufferCapacity

**Problem:** Die statische Pufferkapazität wird berechnet als:
```js
const beta_static = dv_dph.map(v => Math.abs(1 / v));
```
Wenn `dv/dpH = 0` (an den Extremen der pH-Achse oder bei Sprüngen), entsteht `Infinity`. Plotly zeigt dann vertikale Artefakte im Plot.

**Fix:** Werte filtern: `Infinity`- und `NaN`-Einträge werden vor dem Plotten auf `null` gesetzt (Plotly rendert Lücken statt Artefakte).

---

### B3 — Φ-Achse: falscher x_end-Fallback

**Problem:** In `plotMainCurve` wird bei `usePhiAxis && x_eq.length === 0` der `plot_end_volume_ml`-Wert (in mL) als Φ-Achsenende verwendet:
```js
const x_end = usePhiAxis
    ? (x_eq.length > 0 ? Math.max(...x_eq) * 1.5 : plot_end_volume_ml)
    : plot_end_volume_ml;
```
Ein typischer `plot_end_volume_ml`-Wert von z.B. 75 als Φ-Achse wäre vollständig falsch.

**Fix:** Fallback auf `2.0` (dimensionsloser Φ-Wert, sinnvoll als Plot-Ende wenn keine ÄPs berechnet werden konnten).

---

### B4 — Rücktitration ignoriert pK_L-Slider

**Problem:** `runBackTitration` erstellt beide `TitrationCalculator`-Instanzen mit `pks: 14.0` hartcodiert, unabhängig vom Lösungsmittel-Slider.

**Fix:** `pks` aus dem globalen UI-State lesen (`parseFloat(ui.pksSlider.value)`).

---

### B5 — HTML-Tags in Plotly-Legendennamen

**Problem:** `formatSpeciesLabel` gibt Strings wie `H<sub>2</sub>A<sup>-</sup>` zurück. Diese werden als `name` in Plotly-Traces verwendet, wo HTML nicht gerendert wird — die Legende zeigt rohe Tags.

**Fix:** Separate Funktion `formatSpeciesLabelPlain(baseName, totalProtons, numProtonsOnSpecies, isAcid)` die Unicode-Zeichen statt HTML verwendet (z.B. `H₂A⁻` via Unicode-Subscript/Superscript-Zeichen). Die HTML-Variante (`formatSpeciesLabel`) bleibt für DOM-Inhalte erhalten.

Mapping:
- Subscript-Ziffern: `₀₁₂₃₄₅` (U+2080–U+2085)
- Superscript `+`: `⁺` (U+207A), `-`: `⁻` (U+207B), Ziffern: `²³⁴⁵`

---

### B6 — pKₐ-Sortierung ohne Nutzerhinweis

**Problem:** In `getParams()` werden pKₐ-Werte still sortiert (`pkas.sort((a,b) => a-b)`). Wenn ein Nutzer z.B. pKₐ₁ = 9.0 und pKₐ₂ = 4.0 eingibt, werden sie ohne Rückmeldung vertauscht.

**Fix:** Nach der Sortierung prüfen ob Umsortierung stattgefunden hat; wenn ja, kurzen `showSimulationNotice`-Hinweis zeigen: *„pKₐ-Werte wurden in aufsteigender Reihenfolge sortiert."* (Hinweis verschwindet bei nächster Simulation ohne Umsortierung).

---

## Teil 2: Feature-Erweiterungen

### F1 — Theoretischer pH am Äquivalenzpunkt

**Was:** In der ÄP-Tabelle (`equivalence-table`) unter Tab 1 wird für jeden Äquivalenzpunkt der theoretische pH-Wert angezeigt.

**Berechnung:** Der theoretische ÄP-pH wird aus der simulierten Kurve interpoliert (pH am Volumen des ÄP). Kein analytischer Näherungsausdruck — wir lesen den Wert direkt aus dem bereits berechneten `titration.phs`-Array via linearer Interpolation bei `eq_vols_l[i]`.

**UI:** Die `computeEquivalenceMassTable`-Funktion wird erweitert: jede ÄP-Karte zeigt zusätzlich `pH ≈ X.XX`.

---

### F2 — pH = pKₐᵢ für alle ½-ÄP-Annotationen

**Was:** Aktuell wird nur für den ersten Halb-ÄP annotiert (`½ ÄP 1 ≈ pKₐ₁ (Puffer)`). Alle weiteren ½-ÄP-Linien erhalten analoge Annotationen.

**Berechnung:** Für ½-ÄP `i` (0-indiziert) gehört pKₐ `comp.pkas[i]` der entsprechenden Komponente. Die Annotation lautet `½ ÄP ${i+1} ≈ pKₐ${i+1} = ${pka.toFixed(2)}`.

**UI:** Y-Position der Annotationen alternierend (`pks*0.20`, `pks*0.30`, ...) um Überlappung bei polyprotischen Systemen zu vermeiden.

---

### F4 — Gran-Plot (Sub-Tab in Tab 2)

**Was:** Tab 2 erhält eine Sub-Navigation mit zwei Ansichten: „Ableitungen" (bestehend) und „Gran-Plot" (neu).

**Theorie:** Der Gran-Plot linearisiert die Titrationskurve in der Nähe des Äquivalenzpunkts. Klassische Gran-Funktionen (Basen-Titrant in Säure-Analyt):
- Vor dem ÄP: `G₁(V) = (V₀ + V) · 10^(−pH)` gegen `V` — linear, Nullstelle = ÄP
- Nach dem ÄP: `G₂(V) = (V₀ + V) · 10^(pH − pK_L)` gegen `V` — linear, Nullstelle = ÄP

(V₀ = Analytvolumen; bei Säure-Titrant in Base-Analyt sind die Vorzeichen der Exponenten vertauscht.)

**UI:** Zwei Traces pro ÄP (vor/nach), dargestellt über den gesamten Titrationsbereich. Nullstellen-Schnittpunkte mit der X-Achse werden als Marker eingezeichnet. Annotation: berechnetes ÄP-Volumen aus linearer Regression der Gran-Funktion nahe dem ÄP (letztes 30% Segment vor/nach ÄP).

**Implementierung:** Die Sub-Navigation wird als zwei Buttons oberhalb des Plot-Containers realisiert (analoges Muster zu den Haupt-Tabs).

---

### F6 — Export-Funktion

**Was:** Jeder Plot-Container erhält eine Export-Leiste mit:
1. **SVG/PNG:** Via Plotly's eingebauter `Plotly.downloadImage()`-Funktion (bereits im Plotly-Bundle enthalten, kein extra Code nötig außer dem Button-Handler).
2. **CSV:** Export der Rohdaten (V_mL, pH, α_i für jede Spezies, β_statisch, β_dynamisch) als herunterladbarer Blob.

**UI:** Ein kleines `<details>`-Element unter jedem Plotly-Container mit zwei Buttons: `⬇ PNG` und `⬇ CSV`. Nur für den aktiven Tab sichtbar (oder immer sichtbar — einfacher zu implementieren).

---

### F7 — Speziesverteilung auf pH-Achse (X-Achsen-Option in Tab 3)

**Was:** Tab 3 erhält neben den bestehenden Y-Achsen-Radio-Buttons (`α`, `Konzentration`, `n̄`) eine neue **X-Achsen-Option**: `Volumen (mL)` (bestehend) oder `pH`.

**UI:** Zweite Radiobutton-Gruppe `X-Achse: ○ Volumen ○ pH`.

**Implementierung:** Bei Auswahl `pH` wird `titration.phs` statt `volumes_ml` als X-Array verwendet. Der pH-Bereich ist `[0, pks]`. Dies ergibt das klassische Bjerrum-Diagramm wie in Lehrbüchern.

**Hinweis:** Im pH-Modus wird kein `range`-Limit auf `plot_end_volume_ml` angewendet — die X-Achse zeigt den vollen pH-Bereich.

---

### F8 — pKₐ-Annotationen auf Pufferkapazitäts-Plot

**Was:** Im β-Plot (Tab 5) werden vertikale gestrichelte Linien bei pH = pKₐᵢ aller Analyt-Komponenten eingezeichnet, mit Beschriftung `pKₐᵢ = X.XX (Komp. Symbol)`.

**Begründung:** β-Peaks liegen theoretisch bei pH = pKₐ; die Annotation macht diesen Zusammenhang für Studierende sichtbar.

**Implementierung:** Für jede Komponente und jedes pKₐ: ein `shape` (vertikale Linie) und eine `annotation` in der Plotly-Layout.

---

### F9 — Lösungsmittelkontext

**Was:** Der nackte pK_L-Slider wird durch ein zweistufiges UI ersetzt:
1. **Lösungsmittel-Dropdown** mit Presets
2. **Numerisches Eingabefeld** für pK_L (bei "Benutzerdefiniert" editierbar, sonst read-only mit Preset-Wert)

**Presets:**

| Lösungsmittel | pK_L | Klasse |
|---|---|---|
| Wasser | 14.00 | Protisch |
| Methanol | 16.90 | Protisch |
| Ethanol | 19.10 | Protisch |
| Eisessig | 14.45 | Protisch |
| DMF | 29.40 | Polar-aprotisch |
| Acetonitril | 33.30 | Polar-aprotisch |
| DMSO | 35.00 | Polar-aprotisch |
| Benutzerdefiniert | — | — |

**Achsenbeschriftung:**
- Wasser (pK_L = 14.00): `pH`
- Alle anderen: `−lg a(SH₂⁺)` (Lyonium-Aktivität)

Die Beschriftung wird überall aktualisiert wo bisher `'pH'` hardcodiert steht: Y-Achsentitel aller Plots, Hover-Templates, Tab-Beschriftungen.

**Warnmeldungen:**
- Polar-aprotische Lösungsmittel: *„Hinweis: pKₐ-Werte aus wässrigen Tabellen sind nicht auf dieses Lösungsmittel übertragbar. Lösungsmittelspezifische pKₐ-Werte verwenden."*
- Bei Benutzerdefiniert + pK_L > 40: *„Hinweis: Für unpolare aprotische Lösungsmittel (Toluol, CHCl₃ etc.) ist das Charge-Balance-Modell nicht anwendbar. Die Titrationen sind in der Praxis möglich, aber durch Ionenpaarbildung und fehlende Autoprotolyse mit diesem Simulator nicht simulierbar."*

**Slider-Bereich:** Ersetzt durch `<input type="number">` mit Wertebereich 0–50 (kein Slider mehr — zu unhandlich für den Bereich 0–50).

**Bug B4 Fix:** `runBackTitration` liest `pks` aus demselben State.

---

## Architektur & Implementierungshinweise

- Alles bleibt in `sb_titration.html` (Single-file-Ansatz beibehalten)
- Keine neuen Abhängigkeiten
- `formatSpeciesLabelPlain` als neue Hilfsfunktion neben `formatSpeciesLabel`
- Lösungsmittel-State in `currentSimulationData` mitführen (für korrekte Achsenbeschriftung in allen Plots)
- Achsenbeschriftungs-Hilfsfunktion `getAxisLabel()` die zentral `'pH'` oder `'−lg a(SH₂⁺)'` zurückgibt — damit kein String-Replace-Chaos entsteht

---

## Nicht im Scope

- Zwitterionen / Aminosäuren (H_N A^{+Z}, Z ≠ 0)
- Preset-Systeme (Schnellauswahl vordefinierter Titrationen)
- Ionenstärke-Korrekturen / Aktivitätskoeffizienten
- Unpolare aprotische Lösungsmittel (Toluol, CHCl₃) — Modell nicht anwendbar
