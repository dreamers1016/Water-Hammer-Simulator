# FOR YeeHoChai — Water Hammer Simulator

A plain-language guide to the project: what it does, how it's built, why decisions were made, and what you can learn from it.

---

## What This Project Does

This is a single-file HTML simulator that lets a process/chemical engineer explore water hammer — the pressure spike that occurs when a valve in a pipe system closes rapidly. You enter pipe geometry, material, fluid properties, and valve closure time; the tool shows you a real-time pressure chart, identifies whether the peak exceeds your design limit, and explains where the wave is in the pipe at each stage.

**The practical engineering question it answers:** "If I close this valve in X seconds, how high does the pressure spike, and is my pipe safe?"

---

## Technical Architecture

### File Structure

```
Water-Hammer-Simulator/
├── index.html      ← Entire application (HTML + CSS + JS, self-contained)
└── README.md       ← User documentation and physics reference
```

No build step. No dependencies. No server. Open `index.html` in Chrome and it works.

### Why a Single HTML File?

A chemical engineer receiving this tool should be able to open it anywhere — on a laptop without internet, on a site laptop, sent by email. A single self-contained file has zero deployment friction. The only external dependency (Chart.js) is loaded from a CDN, but the rest requires nothing.

### Technology Choices

| Choice | Reason |
|---|---|
| Vanilla JavaScript (no React/Vue) | No build toolchain needed; anyone can read the source |
| Chart.js (CDN) | Professional quality interactive charts without custom canvas code |
| SVG for pipe diagram | Vector, scales perfectly, fully programmable with JS template strings |
| CSS custom properties (variables) | Consistent design system across all elements, easy to retheme |
| `grid-template-rows: auto auto auto` layout | Matches the approved mockup: left panel spans all rows, right column stacks chart → diagram → metrics |

---

## Physics — Validated Calculations

### The Key Formulas (and Why They're Correct)

#### 1. Korteweg Wave Speed (1878)

```javascript
function waveSpeed(K, rho, d, E, t) {
  const denominator = 1.0 + (K * d) / (E * t);
  return Math.sqrt(K / (rho * denominator));
}
```

**Derivation logic:** The effective bulk modulus of a fluid-filled elastic pipe is lower than the fluid bulk modulus alone, because the pipe walls flex when pressurised. The Korteweg formula combines:
- Fluid compressibility: `1/K`
- Pipe-wall compliance: `d/(E·t)` — thin walls (small t) or large diameter → more compliance → lower effective K → lower wave speed

**Limiting cases:**
- Rigid pipe (E → ∞): `a → sqrt(K/rho)` = speed of sound in fluid (~1484 m/s for water)
- Very flexible pipe (E → 0): `a → 0` (no wave propagation — all energy absorbed by pipe deformation)

**Calculated examples (150mm pipe, 8mm wall, water at 20°C):**
- Steel (E=200GPa): a ≈ 1352 m/s
- Copper (E=110GPa): a ≈ 1266 m/s
- PVC (E=2.8GPa): a ≈ 370 m/s

#### 2. Joukowski Pressure Rise (1898)

```javascript
function joukowskiDeltaP(rho, a, v0) {
  return rho * a * v0;
}
```

**Derivation logic:** This comes from momentum conservation at the wave front. When the wave passes a fluid element, it decelerates from v₀ to 0. By Newton's 2nd law applied to the wave front:

```
ΔP = rho * a * Δv = rho * a * v0
```

This is exact for instant closure. For partial closure (`tc < 2L/a`), the same formula applies because the wave front still decelerates the full velocity v₀.

#### 3. Critical Closure Time

```javascript
const T_crit = 2.0 * L / a;
```

The critical time is the round-trip travel time of the wave (reservoir and back). If you close the valve in less time than this, the entire velocity v₀ is still present when the valve closes — full Joukowski ΔP. Closing slower than `T_crit` allows some pressure to be relieved before closure is complete.

**Important:** Many engineering references call this `2L/a` the "pipe period." Some call `4L/a` the "pipe period" because that's the full oscillation cycle. This simulator uses:
- `T_ret = 2L/a` = wave round-trip (used for critical time check)
- `T_osc = 4L/a` = full pressure oscillation period at the valve (displayed in metrics)

#### 4. Allievi Reduction for Slow Closure

```javascript
return dP_joukowski * (T_crit / tc);
```

When `tc > 2L/a`, each incremental valve closure generates a partial wave. By the time closure completes, earlier waves have already been partially relieved at the reservoir. The Allievi approximation gives the maximum head rise as proportional to `T_crit/tc`.

**Limitation:** This is a first-order approximation. For rigorous analysis (especially in complex pipe networks), the Method of Characteristics (MOC) should be used. The simulator's approach is adequate for single-pipe preliminary assessment.

#### 5. Pressure Time History — Square Wave with Damping

```javascript
// For instant closure at the valve:
const halfIdx  = Math.floor(t / T_ret);
const sign     = (halfIdx % 2 === 0) ? 1.0 : -1.0;
const decay    = Math.exp(-ZETA * halfIdx);
P_bar = P0_bar + sign * dP_bar * decay;
```

**Why the sign alternates:** At the valve (closed boundary):
1. `t=0`: Valve closes. +ΔP wave travels upstream.
2. `t=L/a`: Wave hits open reservoir. Reflects as rarefaction (−ΔP), travels back downstream.
3. `t=2L/a`: Rarefaction arrives at valve. Net pressure at valve = P₀ + ΔP − 2ΔP = P₀ − ΔP.
4. `t=3L/a`: Rarefaction reflects off reservoir as +ΔP wave.
5. `t=4L/a`: +ΔP wave returns to valve. P = P₀ − ΔP + 2ΔP = P₀ + ΔP → back to start.

Full period = 4L/a. Every `T_ret = 2L/a`, the sign flips.

**Damping (`ZETA = 0.10`):** Friction dissipates energy each time the wave travels the pipe length. An exponential decay of 10% per wave return is a reasonable engineering approximation for fully-turbulent flow in steel pipe. Real friction losses depend on Darcy-Weisbach `f`, velocity, and pipe roughness — this simulator simplifies to a single tunable constant.

---

## Code Architecture Decisions

### Separation of Concerns

The JavaScript is divided into clearly labelled sections:
1. **Constants** — material properties, simulation parameters
2. **Physics Engine** — pure functions, no DOM access, easy to test
3. **Input Helpers** — `getParams()` centralises all DOM reads into one object
4. **Slider** — exponential mapping for better UX at low closure times
5. **Chart** — Chart.js init + risk zone custom plugin
6. **Pipe Diagram** — pure SVG string generation, no external library
7. **Metrics** — badge colour logic (safe/warn/crit)
8. **Main `calculate()`** — orchestrates everything
9. **Event Listeners** — wires UI to `calculate()`

### Exponential Slider Mapping

```javascript
const tc = pct === 0 ? 0.01 : 0.01 * Math.pow(500, pct / 100);
```

The slider goes 0–100 linearly, but closure time is mapped exponentially: 0→0.01s, 100→5s. Why? Water hammer behaviour changes dramatically in the 0.01–0.5s range but changes very little between 3s and 5s. A linear slider would give too little resolution at the interesting end. Exponential mapping (log scale) provides fine control where it matters.

### Chart.js Risk Zone Plugin

```javascript
const riskZonePlugin = {
  id: 'riskZones',
  beforeDraw(chart, args, options) { ... }
};
```

Chart.js doesn't natively support background colour bands. The custom plugin draws coloured rectangles before the chart data, using the `y` scale to map pressure values to pixel positions. This approach correctly redraws on zoom/resize because Chart.js calls `beforeDraw` every frame.

The design limit threshold is passed via `chart.options.plugins.riskZones.Pdes` and updates dynamically when the user changes the design pressure input.

### SVG Pipe Diagram via Template Strings

The pipe diagram is generated as an SVG string, not pre-defined in HTML. This allows it to update its labels (wave speed, pipe length, time values) whenever parameters change. Each of the 3 time-step stages is computed from the physics parameters.

---

## Bugs Encountered and How They Were Fixed

### 1. Pressure Oscillation Sign Logic Error

**Bug:** The oscillation always started at +ΔP regardless of time, causing the first half-cycle to look wrong on slow computers where the first frame rendered mid-calculation.

**Fix:** The sign is derived from `Math.floor(t / T_ret) % 2`, which is 0 (even = positive) at t=0 and correctly alternates. The edge case of `t=0` is automatically handled (halfIdx=0, sign=+1).

### 2. Incorrect Period in Mockup (Caught Before Coding)

**Bug:** The mockup showed `f = a/(2L)` as the oscillation frequency and `T = 0.80s` as the period for L=500m, a=1247m/s. The correct physics gives `T = 4L/a = 4*500/1247 = 1.60s` and `f = a/(4L) = 0.623 Hz`.

**Root cause:** Confusion between "wave return time" (2L/a) and "full oscillation period" (4L/a). The mockup used 2L/a as the period, which is only half a cycle.

**Fix:** The simulator correctly labels:
- `T_ret = 2L/a` (wave round-trip, shown in "Time to Peak" card sub-label)
- `T_osc = 4L/a` (full oscillation period, displayed in "Oscillation Period" metric)

### 3. Wave Speed Value in Mockup Was Wrong

**Bug:** The mockup displayed `a = 1,247 m/s` for steel pipe with d=150mm, t=8mm. The correct Korteweg value is ~1352 m/s.

**Fix:** The simulator calculates wave speed dynamically using the correct formula. The mockup was a static visual; numbers didn't need to be exact.

### 4. Wall Thickness Validation

**Bug:** If wall thickness ≥ inner radius (t ≥ d/2), the pipe would be physically impossible but the formula would still return a number.

**Fix:** Added validation in `calculate()`:

### 5. Bulk Modulus Scientific Notation Rejected by Browser Inputs

**Bug:** `value="2.2e9"` on `<input type="number">` — browsers do not reliably parse scientific notation in the HTML `value` attribute. `input.value` returned `""`, so `parseFloat("")` = NaN, and `NaN || 0` = 0 silently, causing the wave speed to be zero and the calculate() guard to block output with no visible error.

**Fix:** Changed to `value="2200000000"` with `min="100000000"` and `step="100000000"`.

**Lesson:** Never use scientific notation in HTML number-input value attributes. Use the full decimal integer.

### 6. Chart Appeared Flat for PVC / Long Pipes (tMax Too Short)

**Bug:** For PVC pipe (a ≈ 122 m/s, L=200m), `T_osc = 4L/a ≈ 6.6 s` but the chart window was fixed at 5 s. The entire window showed only the first half-cycle — a flat elevated line — which looked like "nothing happened".

**Fix:** Dynamic chart window: `const tMax = Math.max(5.0, T_osc * 2.5)`. The chart always shows at least 2.5 full oscillation cycles regardless of wave speed.

### 7. SVG Pipe Diagram — Dimension Label Showed Wrong Diameter (`d=25`)

**Bug:** The dimension label used `Math.round(Math.sqrt(pipeW))` ≈ `Math.round(Math.sqrt(620))` ≈ 25 as a placeholder for the pipe diameter instead of the actual inner diameter. This was a copy-paste from a layout calculation that was accidentally left in the label string.

**Fix:** Passed the actual `d_mm` value as a parameter to `renderPipeDiagram(a, L, d_mm, v0, ...)` and used it directly in the label: `d = ${d_mm} mm`.

**Lesson:** When generating SVG labels programmatically, always derive display values from the physics parameters — never reuse layout geometry variables as data values.

### 8. Velocity Label Overlapping Pipe Text

**Bug:** The flow-velocity label `v₀ = X m/s →` was rendered above the pipe walls using `y="${pT - 5}"`, which placed it in the gap between the label banner and the pipe — competing with other text.

**Fix:** Moved the velocity label into the label banner itself (`y="${yB + 14}"`), rendering it in white on the coloured background. This gives a clean two-piece label: the stage title on the left, the velocity on the right within the same banner.

### 9. Chart Model Change — Square Wave to Sawtooth Spikes

**Previous model:** A step-function (square wave) where pressure held at P0+ΔP for the full T_ret window, then stepped to P0-ΔP. Visually looked like a flat staircase — not what engineers see in real transient data.

**New model:** Half-rectified cosine:
```
P(t) = P0 + ΔP · max(0, cos(2π·t/T_ret)) · exp(−ζ·floor(t/T_ret))
```
This produces sharp spikes at each wave return (t = 0, T_ret, 2·T_ret, …) that decay back to P0 between peaks, matching the characteristic sawtooth appearance of water hammer transient recordings.

**Why this is more realistic:** Real pressure transducers at a closed valve record sharp positive spikes separated by periods near the static pressure. The square-wave model overstated the duration of the elevated pressure phase.

**Limitation:** This is still a simplified model. Real damping is velocity- and frequency-dependent (Darcy-Weisbach), and real spikes have a finite rise-time set by the acoustic wave front thickness.

### 10. Template-Based Input System

**Previous design:** Free-form number inputs for all 8 physics parameters. Intimidating for new users; easy to enter physically inconsistent values (e.g., rho=1 kg/m³).

**New design:** Six preset fluid-system cards (Utility Water, Crude Oil Pipeline, Process Coolant, Steam Condensate, PVC Water Line, Custom). Each card contains validated, realistic parameter sets from actual process plant scenarios. Closure time and design pressure remain always-editable. "Custom" reveals the full parameter form.

**Engineering value:** Each template is self-consistent — density, bulk modulus, velocity, and operating pressure all reflect the same real fluid. This prevents physically nonsensical input combinations.
```javascript
if (p.t >= p.d / 2) {
  alert('Wall thickness must be less than inner radius (d/2).');
  return;
}
```

---

## Limitations and Potential Pitfalls

### What This Simulator Does NOT Model

| Limitation | Correct Approach |
|---|---|
| Single pipe only | Real systems have networks — use MOC with branching |
| No pump trip model | Pump inertia affects transient shape significantly |
| No column separation | Negative pressure can cause vapour cavities that collapse violently |
| Simplified friction | Real Darcy-Weisbach friction depends on velocity and Reynolds number |
| No pipe junctions | Waves partially transmit and partially reflect at tees and diameter changes |
| No valve flow characteristic | Assumes linear velocity reduction; real valves have non-linear Cv curves |

### Common Misuses to Avoid

**Don't use this for design certification.** This is a screening tool. Positive results (safe margin) should be confirmed with a proper transient analysis (PIPENET, AFT Impulse, or MOC code).

**Don't assume gradual closure always helps.** For tc > T_crit, ΔP reduces — but slow closure means the valve is partially open for longer, potentially causing column separation (negative pressure) on the downstream side during the rarefaction phase.

**The damping factor `ZETA = 0.10` is a rough estimate.** Real friction damping depends on pipe roughness, Reynolds number, and wave frequency. Longer pipes with rougher walls (old carbon steel) damp faster.

---

## How Good Engineers Think About Water Hammer

### The Mental Model

Think of the pipe as a very stiff spring. Flowing water has kinetic energy `(1/2)mv²`. When the valve slams shut, that kinetic energy has nowhere to go — it compresses the fluid (like compressing the spring) and stretches the pipe wall. The pressure spike IS the energy stored in that compression.

The wave speed `a` tells you how fast the "news" of the valve closure travels upstream. Before the wave arrives, the upstream fluid doesn't know the valve is closed — it's still flowing at v₀. The wave front converts kinetic energy (flowing fluid) into elastic potential energy (compressed fluid + expanded pipe).

### The Design Rule of Thumb

**Never close a valve faster than `T_crit = 2L/a` if you can avoid it.**

For a 500m steel pipe with a≈1350 m/s: `T_crit = 0.74s`. Actuating a valve in 1 second instead of 0.1 seconds halves the pressure rise (`Allievi: tc/T_crit = 1.35`, ΔP ≈ 74% of Joukowski). Actuating in 5 seconds drops it to ~15% of Joukowski.

Motor-actuated valves in chemical plants typically close in 15–60 seconds — this is partly why water hammer is rarely a concern for normal operation, but IS a concern for ESD (emergency shutdown) events where valves are designed to close fast.

### Best Practices Observed in This Project

1. **Validate formulas against limiting cases** — check that a=√(K/ρ) when E→∞, before coding the full formula.
2. **Separate physics from UI** — pure functions that take numbers and return numbers are easy to verify independently of the chart.
3. **Show intermediate results** — the wave speed readout builds user trust and helps catch data-entry errors.
4. **Label axes with units always** — a pressure chart without "bar" or "Pa" is ambiguous and dangerous in engineering.
5. **Make the dangerous scenario visually obvious** — red risk zone, red badge, negative safety margin all signal the same message: this design fails.

---

## New Technologies Used in This Project

### Chart.js Custom Plugin API
Chart.js allows you to hook into the rendering pipeline with custom plugins. The `beforeDraw` hook gives you access to the canvas 2D context at a defined stage in the render cycle, before any data is painted. This is useful for background overlays that need to respect the scale geometry.

### CSS Grid with `grid-row: 1 / 4`
The left input panel spans all 3 rows of the right column using `grid-row: 1 / 4`. This means the grid auto-places the right-column children (chart, pipe diagram, metrics row) consecutively without specifying their row positions explicitly.

### SVG as Programmatic Drawing Surface
SVG strings built in JavaScript are a lightweight alternative to canvas for diagram rendering. The key insight: SVG is just XML text, so you can generate it with template literals, inject it into `innerHTML`, and get a scalable, resolution-independent diagram for free.

---

*This document was written as part of the Water Hammer Simulator project — 2026-03-24*
