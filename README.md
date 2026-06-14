# LPBF Beam Shaping Study — 6 Beam Profiles + AMR

<p align="center">
  <img src="https://img.shields.io/badge/FreeFEM++-Simulation-blue?style=for-the-badge&logo=gnu&logoColor=white"/>
  <img src="https://img.shields.io/badge/AlSi10Mg-Material-darkgreen?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/3D-Transient%20Heat%20Transfer-orange?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/6%20Beam-Profiles-red?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/AMR-Sinh%20Graded%20Mesh-purple?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/ParaView-VTK%20Export-green?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/License-MIT-lightgrey?style=for-the-badge"/>
</p>

<p align="center">
  A fully 3D transient heat transfer simulation of <b>Laser Powder Bed Fusion (LPBF)</b>
  of AlSi10Mg for six different laser beam profiles, implemented in FreeFEM++.
  All six beams run sequentially on the same <b>AMR sinh-graded mesh</b>,
  producing melt pool geometry, peak temperature, and cooling rate for direct
  comparison across beam profiles.
</p>

<img width="1008" height="772" alt="gaussian las" src="https://github.com/user-attachments/assets/3d775956-5c73-432e-aab9-22a85d40df1a" />

---

## Physics

The simulation solves the **3D transient heat equation** at every timestep on a fixed graded mesh:

- **Conduction-dominated heat transfer** — `rho*Cp_eff(T)*dT/dt = div(k(T)*grad T) + Q_beam(x,y,z,t)`
- **Temperature-dependent properties** — `k(T)` and `Cp(T)` for AlSi10Mg with latent heat spike in mushy zone `[Tsol=831K, Tliq=867K]`
- **Moving laser source** — beam centre advances at `v_scan = 1000 mm/s` along x-axis, deposited as volumetric source over powder layer depth
- **Enthalpy method** — latent heat absorbed via `Cp_eff = Cp_l + L/(Tliq-Tsol)` in mushy zone
- **No fluid flow** — pure conduction model

---

## Six Beam Profiles

| # | Beam Shape | Formula | Key Parameter |
|---|---|---|---|
| 1 | **Gaussian** | `Ig = 2P/(pi*Rg^2) * exp(-2r^2/Rg^2)` | `Rg = 100 um` |
| 2 | **Elliptical-Gaussian** | `Ieg = 2P/(pi*a1*b1) * exp(-2((x/a1)^2+(y/b1)^2))` | `a1=55 um, b1=175 um` |
| 3 | **Top Hat** | `ITH = P/(pi*Rg^2)` inside `r < Rg` | `Rg = 100 um` |
| 4 | **Flat Top** | `IFT = g*P/a2^2` inside square `|x|,|y| < a2/2` | `a2 = 177.25 um` |
| 5 | **Ring Shaped** | `Id = P/(pi*Rm^2) * (r^2/Rm^2) * exp(-r^2/Rm^2)` | `Ri=150 um, Ro=175 um` |
| 6 | **AMB** | `IAMB = 20%*Ig + 80%*Id` | Core + outer ring |

All beams share the same total laser power **P = 400 W** for a fair comparison.

---

## Geometry

```
        z
        |
    z=0 +-------------------------------------------  Powder surface (laser hits here)
        |  AlSi10Mg powder + substrate
  z=-D  +-------------------------------------------  Bottom (heat sink, T = Tamb)

  x in [0,  5 mm]       scan direction  (laser traverses at 1000 mm/s)
  y in [-1.5, +1.5 mm]  across track    (sinh graded, fine at y=0)
  z in [-1.0, 0 mm]     depth           (exp graded, fine at z=0 surface)

Boundary labels (from cube() mapping):
  Label 1: z=-D  bottom  ->  fixed T = Tamb = 298 K  (heat sink)
  Label 2: z=0   surface ->  convective cooling h=15 W/m2/K + laser flux
  Labels 3-6: sides      ->  insulated (far-field, negligible flux)
```

---

## Material Properties — AlSi10Mg

| Property | Symbol | Value | Unit |
|---|---|---|---|
| Density (solid) | rho | 2670 | kg/m3 |
| Density (liquid) | rho_l | 2380 | kg/m3 |
| Specific heat (solid) | Cp_s | 900 | J/kg/K |
| Specific heat (liquid) | Cp_l | 1160 | J/kg/K |
| Latent heat | L | 560 000 | J/kg |
| Conductivity (solid, low T) | k | 130 | W/m/K |
| Conductivity (solid, high T) | k | 160 | W/m/K |
| Conductivity (liquid) | k_l | 95 | W/m/K |
| Solidus temperature | T_sol | 831 | K |
| Liquidus temperature | T_liq | 867 | K |

---

## Process Parameters

| Parameter | Symbol | Value | Unit |
|---|---|---|---|
| Laser power | P | 400 | W |
| Scanning speed | v_scan | 1000 | mm/s |
| Layer thickness | h | 30 | um |
| Preheat / ambient | T_amb | 298 | K |
| Track length | L_track | 5 | mm |
| Total scan time | t_total | 5 | ms |
| Timestep | dt | 10 | us |
| Steps per beam | Nstep | 500 | — |

---

## AMR Mesh Strategy

```
  y-direction: sinh grading (gradY = 2.5)
    Fine at y=0 (centreline) -> ~8:1 ratio vs y=+-1.5mm (edge)
    Resolves steep thermal gradient across narrow melt pool width

  z-direction: exp grading (gradZ = 2.0)
    Fine near z=0 (powder surface) where laser deposits energy
    Coarse at z=-1mm (heat has diffused, gradient small)

  x-direction: uniform (40 divisions over 5mm = 125 um/element)
    Beam radius Rg=100um adequately resolved by moving source
```

| Direction | Divisions | Fine element size | Coarse element size | Grading |
|---|---|---|---|---|
| X (scan) | 40 | 125 um | 125 um | Uniform |
| Y (across) | 20 | ~18 um | ~150 um | sinh, factor 2.5 |
| Z (depth) | 15 | ~15 um | ~130 um | exp, factor 2.0 |

Total: ~13 000 nodes, ~60 000 tetrahedra. Runs in under 5 minutes per beam on a desktop.

---

## Simulation Strategy

### Time Integration
- **Crank-Nicolson (theta = 0.5)** — second-order accurate, unconditionally stable
- `dt = 0.1 * Rg / v_scan = 10 us` — beam advances 0.1 beam-radii per step
- Matrix reassembled each step for temperature-dependent coefficients

### Heat Source Deposition
Surface flux `I(x,y)` [W/m2] converted to volumetric source [W/m3]:

```
Q(x,y,z,t) = I(x - x_tool(t), y) / dz_beam * exp(-(z/dz_beam)^2)
```

where `dz_beam = 2 * h_layer = 60 um` distributes energy into the subsurface.

### Melt Pool Diagnostics (measured at half-track x = 2.5 mm)

```
Pool width:   scan along y at z=0; find max y where T > Tliq
Pool depth:   scan along z at y=0; find max |z| where T > Tliq
Cooling rate: dT/dt ~ (Tliq-Tsol) * v_scan / L_mushy
              where L_mushy = mushy zone length along x behind beam
```

---

## Output Files

### Per Beam Shape — in `D:\freefem++\lpbf_beams\<BeamName>\`

| File | Description |
|---|---|
| `beam.pvd` | Master animation — open this in ParaView |
| `temp_0000.vtu` | Temperature field at t=0 |
| `temp_NNNN.vtu` | Temperature field at each saved timestep (20 frames total) |

### Global Summary — `D:\freefem++\lpbf_beams\`

| File | Description |
|---|---|
| `beam_comparison.csv` | One row per beam: Tpeak, width, depth, cooling rate |

### CSV Columns

| Column | Description | Unit |
|---|---|---|
| BeamShape | Beam profile name | — |
| PeakTemp_K | Maximum temperature across all timesteps | K |
| PoolWidth_um | Melt pool width at half-track | um |
| PoolDepth_um | Melt pool depth at half-track | um |
| CoolingRate_MKs | Cooling rate at solidification front | x10^6 K/s |
| LaserSpotSize_um | Characteristic beam dimension | um |

---

## Repository Structure

```
lpbf_beam_shapes.edp                  # Main FreeFEM++ simulation script
README.md                             # This file

D:\freefem++\lpbf_beams\
├── beam_comparison.csv               # Summary table — all 6 beams
├── Gaussian\
│   ├── beam.pvd                      # Animation collection
│   ├── temp_0000.vtu                 # Frame 0
│   └── temp_NNNN.vtu                 # ...
├── EllipGauss\
│   ├── beam.pvd
│   └── ...
├── TopHat\
├── FlatTop\
├── RingShaped\
└── AMB\
```

---

## How to Run

### Requirements
- FreeFEM++ v4.10 or later: https://freefem.org
- ParaView v5.x or later: https://www.paraview.org

### Step 1 — Run the simulation

```bash
FreeFem++ lpbf_beam_shapes.edp
```

The script will:
1. Build the 3D sinh/exp graded mesh once — shared across all 6 beams
2. For each beam shape in sequence:
   - Reset temperature to `T_amb = 298 K`
   - Run 500 timesteps with moving laser source
   - Measure pool geometry and cooling rate at half-track
   - Save 20 VTU frames and close PVD
3. Append one row to `beam_comparison.csv` per beam

Console output example:
```
==========================================
 LPBF BEAM SHAPING STUDY
 P=400W  v=1000mm/s
 Domain: 5x3x1mm
 Mesh: 13041 nodes  57600 tets
==========================================
 dt=10us  Nstep=500

--- BEAM: Gaussian ---
  Half-track: Tp=2980K  W=271um  D=112um  dTdt=1.94e6 K/s
  DONE: Tpeak=2980K  W=271um  D=112um

--- BEAM: RingShaped ---
  Half-track: Tp=1658K  W=381um  D=108um  dTdt=1.58e6 K/s
  DONE: Tpeak=1658K  W=381um  D=108um
...
==========================================
 RESULTS SUMMARY
==========================================
```

### Step 2 — Open in ParaView

1. `File > Open` → navigate to `D:\freefem++\lpbf_beams\Gaussian\`
2. Select `beam.pvd` → OK → Apply
3. Set colour field to `Temp_K`
4. Repeat for each beam folder

### Step 3 — Visualise

**Option A — Temperature field (recommended first view)**
```
Color by:   Temp_K
Colormap:   Inferno or Rainbow
Range:      298 K to 3000 K
Press Play  -> melt pool forms and trails behind the moving beam
```

**Option B — Melt pool shape**
```
Filters > Threshold
Scalars:    Temp_K
Minimum:    867  (liquidus Tliq)
-> Shows only the molten region
-> Compare width and depth across all 6 beams
```

**Option C — Side-by-side comparison**
```
Open beam.pvd for all 6 beams simultaneously
Layout -> 3x2 grid view
Align all to same timestep (half-track position)
-> Visual comparison of all beam profiles
```

**Option D — Post-process CSV in Python**
```python
import pandas as pd
import matplotlib.pyplot as plt

df = pd.read_csv('beam_comparison.csv')
df.plot(x='BeamShape', y='PeakTemp_K', kind='bar', title='Peak Temperature')
df.plot(x='BeamShape', y=['PoolWidth_um','PoolDepth_um'], kind='bar',
        title='Pool Geometry')
plt.show()
```

---

## Key Physics Insights

### Why Gaussian has the highest peak temperature
The Gaussian beam concentrates all 400 W into `Rg=100um`, giving peak power density
`2P/(pi*Rg^2) ~ 2.5e9 W/m2`. This localised heating drives temperature well above
the liquidus, creating a deep narrow melt pool. The large pool volume dissipates
heat slowly, resulting in a low cooling rate.

### Why the Ring beam reduces peak temperature
The ring distributes energy over a large annular area (`Rm = 162.5 um`). Peak
intensity is far lower. The resulting pool is wide and shallow — ideal for reducing
evaporation and keyhole porosity without sacrificing melt pool width.

### Why cooling rate differs between beams
Cooling rate `dT/dt = G * R` where `G` is thermal gradient and `R` is solidification
growth rate. A wider, shallower pool (ring beam) has smaller thermal gradients at its
trailing edge — lower cooling rate — leading to potentially coarser microstructure.
The Gaussian beam's concentrated pool maintains steeper gradients — faster cooling —
yielding a finer grain size.

### AMB advantage
With 80% power in the outer ring, AMB combines the surface preheating of the ring
(reduces thermal shock) with a small central Gaussian (maintains penetration).
Results are very similar to the pure ring beam, confirming that the outer ring
dominates the power distribution.

---

## Common Errors and Fixes

| Error | Cause | Fix |
|---|---|---|
| `Tmax = Tamb` always | `kEff`/`CpEff`/`rhoEff` zero before first matrix assembly | Initialise all property fields to solid values before the time loop |
| `UMFPACK singular matrix` | Zero coefficients in thermal matrix | Verify `rhoEff`, `CpEff`, `kEff` are nonzero at every node |
| Melt pool width = 0 | Pool sampling outside liquidus isotherm | Increase `Plaser` or verify `Tliq` value |
| Tpeak much lower than expected | Absorption depth `dzBeam` too large | Reduce `dzBeam` to `hLayer` (single layer thickness) |
| Simulation too slow | 500 steps x 6 beams on large mesh | Reduce `Nxm` to 30; increase dt factor from 0.1 to 0.2 |
| PVD XML corrupted | Mixed write/append mode | First frame uses plain `ofstream pv(bdir+"beam.pvd")`; all subsequent frames use `append` |
| Pool depth always 0 | z-sampling below mesh extent | Check `Ddomain` vs `dzStep * Nzm` — ensure scan reaches full depth |

---

## Extending the Model

| Extension | What to change |
|---|---|
| Add Marangoni flow | Add Brinkman-Stokes solve; couple `Temp` to velocity field |
| Keyhole porosity prediction | Track `T > T_evap` region; flag nodes as vapour cavity |
| Residual stress | Post-process: solve thermoelastic equilibrium on cooled mesh |
| Different alloys (Ti-6Al-4V, IN718) | Update `kAlSi`, `CpAlSi`, `rhoAlSi`, `Tsol`, `Tliq`, `Lfus` |
| Multi-layer deposition | Extend z domain; add new powder layer each pass |
| Powder absorptivity | Multiply `Tsource` by absorptivity `A(T)` (increases with temperature) |
| Finer mesh near beam | Increase grading: `gradY = 3.5`, `gradZ = 3.0` |
| Validate against experiment | Export `dT/dt` field at solidification front; compare to pyrometer data |

---

## Citation

```bibtex
@software{mishra2026lpbf,
  author    = {Mishra, A.},
  title     = {LPBF Beam Shaping Study -- 6 Beam Profiles + AMR},
  year      = {2026},
  publisher = {Zenodo},
  doi       = {10.5281/zenodo.20686586},
  url       = {https://doi.org/10.5281/zenodo.20686586}
}
```

Plain text citation:

> Mishra, A. (2026). *LPBF Beam Shaping Study — 6 Beam Profiles + AMR*. Zenodo. https://doi.org/10.5281/zenodo.20686586

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.20686586.svg)](https://doi.org/10.5281/zenodo.20686586)

---

## Author

**akshansh11**
GitHub: https://github.com/akshansh11

---

## License

<p>
<a rel="license" href="http://creativecommons.org/licenses/by-nc/4.0/">
<img alt="Creative Commons Licence" style="border-width:0"
  src="https://i.creativecommons.org/l/by-nc/4.0/88x31.png"/>
</a>
<br/>
This work is licensed under a
<a rel="license" href="http://creativecommons.org/licenses/by-nc/4.0/">
Creative Commons Attribution-NonCommercial 4.0 International License</a>.
</p>

You are free to:
- **Share** — copy and redistribute in any medium or format
- **Adapt** — remix, transform, and build upon the material

Under the following terms:
- **Attribution** — give appropriate credit and link to this repository
- **NonCommercial** — not for commercial use without permission

Copyright 2026. All rights reserved for commercial use.
