# Water Hammer Simulator

An interactive, browser-based simulator for modelling pressure transients (water hammer) in pipe systems. Built for chemical and process engineers to assess surge risk and evaluate valve closure strategies.

![Water Hammer Simulator](https://img.shields.io/badge/Tool-Chemical%20Engineering-c0392b?style=flat-square) ![HTML](https://img.shields.io/badge/Stack-HTML%20%2F%20JS%20%2F%20Chart.js-1a5276?style=flat-square)

## Live Demo

Open `index.html` in any modern browser — no installation, no server required.

---

## Features

| Feature | Description |
|---|---|
| Wave speed calculator | Korteweg (1878) formula accounting for fluid + pipe elasticity |
| Pressure transient chart | Real-time pressure vs time at the valve location |
| Comparison overlay | Instant closure vs gradual closure curves side by side |
| Risk zones | Color-coded safe / warning / critical bands on the chart |
| Material toggle | Steel / PVC / Copper with correct Young's moduli |
| Live slider | Closure time 0.01 s → 5 s updates chart in real time |
| Wave propagation diagram | 3-stage SVG showing where the pressure wave is at each time step |
| Safety margin | Calculated vs user-defined design pressure limit |
| ChE explanation panel | Physics background and process plant applications |

---

## Input Parameters

| Parameter | Default | Notes |
|---|---|---|
| Pipe Length | 500 m | Full pipe length from reservoir to valve |
| Inner Diameter | 150 mm | Internal pipe diameter |
| Wall Thickness | 8 mm | Pipe wall thickness (affects wave speed) |
| Pipe Material | Steel | Sets Young's modulus E |
| Flow Velocity | 2.5 m/s | Pre-closure fluid velocity |
| Bulk Modulus | 2.2 × 10⁹ Pa | Water at ~20°C |
| Fluid Density | 998 kg/m³ | Water at ~20°C |
| Static Pressure | 2 bar | Operating pressure before closure |
| Valve Closure Time | 0.01 s | Slider: instant to 5 s |
| Design Pressure Limit | 40 bar | Pipe rated pressure for safety margin |

---

## Physics

### Wave Speed — Korteweg (1878)

```
a = sqrt( K / (rho * (1 + K*d / (E*t))) )
```

Where:
- `K` = Fluid bulk modulus [Pa]
- `rho` = Fluid density [kg/m³]
- `d` = Inner diameter [m]
- `E` = Young's modulus of pipe material [Pa]
- `t` = Wall thickness [m]

This is the Korteweg formula for acoustic wave speed in an elastic-walled pipe. Rigid pipe (large E) → wave speed approaches `sqrt(K/rho)` (speed of sound in fluid). Flexible pipe (small E, e.g. PVC) → slower wave speed → lower peak pressure.

**Typical values:**
| Material | E (GPa) | a (m/s, 150mm × 8mm pipe) |
|---|---|---|
| Steel | 200 | ~1350 |
| Copper | 110 | ~1260 |
| PVC | 2.8 | ~370 |

### Joukowski Pressure Rise (1898)

For instantaneous valve closure (or `tc < 2L/a`):

```
ΔP = rho * a * v0
```

### Allievi Reduction for Gradual Closure

When closure time `tc > T_crit = 2L/a`:

```
ΔP_grad = rho * a * v0 * (2L/a) / tc
```

The critical time `T_crit = 2L/a` is the wave round-trip travel time. Closing the valve in less than this time produces full Joukowski pressure regardless of how fast you close it.

### Pressure Time History at Valve

For instant closure, the valve experiences a square wave:

| Time Range | Pressure |
|---|---|
| 0 to 2L/a | P₀ + ΔP (compression wave travelling upstream) |
| 2L/a to 4L/a | P₀ − ΔP (rarefaction returns after reservoir reflection) |
| 4L/a to ... | Repeats with exponential friction damping |

Period = **4L/a**, frequency = **a/(4L)**

For gradual closure (`tc > 2L/a`), pressure builds smoothly to `ΔP_grad` over `tc`, then undergoes smaller damped oscillations.

---

## Chemical Engineering Applications

### Where Water Hammer Occurs in Process Plants

**Cooling Water Networks**
Emergency shutdown (ESD) valves or pump trips can create severe surges in cooling water headers. The combination of long pipe runs and high velocities makes this a common design concern.

**Steam Condensate Lines**
Steam collapse creates a vacuum pocket that then fills violently with condensate — a steam hammer variant. Check valves slam shut during steam trap failures.

**Chemical Dosing Systems**
Rapid solenoid valves in dosing skids generate hammer in small-bore lines, cracking compression fittings and instrument connections over time (fatigue failure).

**Crude Oil and Product Pipelines**
Block valves on long transmission lines can generate surges exceeding 100 bar if operated too quickly. Pipeline operators define minimum closure times in their operating procedures.

**Fire Water Systems**
Deluge valve actuation on demand releases large flow suddenly — the downstream pressure transient can exceed the system MAWP.

### Mitigation Strategies

| Strategy | Mechanism | When to Use |
|---|---|---|
| Slow valve closure (tc > 2L/a) | Reduces ΔP via Allievi reduction | Most practical first step |
| Surge vessel / air vessel | Absorbs compression energy | Long pipelines, pump trips |
| Pressure relief valve | Limits peak pressure | Safety-critical systems |
| Surge anticipating valve | Opens on upstream surge detection | Large pump stations |
| Variable-speed pumps | Controlled ramp-down eliminates sudden stop | Motor-driven pump systems |
| Flexible (PVC/GRP) pipe | Lower E → lower wave speed → lower ΔP | Where pressure rating allows |

---

## References

- Joukowski, N. (1898). *Über den hydraulischen Stoss in Wasserleitungsrohren.* Mémoires de l'Académie Impériale des Sciences de St.-Pétersbourg.
- Korteweg, D.J. (1878). *Über die Fortpflanzungsgeschwindigkeit des Schalles in elastischen Röhren.* Annalen der Physik.
- Streeter, V.L. & Wylie, E.B. (1978). *Fluid Transients.* McGraw-Hill.
- Chaudhry, M.H. (2014). *Applied Hydraulic Transients* (3rd ed.). Springer.

---

## License

MIT License — free to use and adapt.
