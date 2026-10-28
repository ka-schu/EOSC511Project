# EOSC511Project

A 2-D (x–z) finite-difference model of **methane (CH₄) and salinity transport in a glacial fjord**, built for UBC EOSC 511. Freshwater carrying methane enters the domain at a glacier wall. The model then tracks how the methane spreads through diffusion and (optionally) advection, and how it is removed by microbial oxidation and by evasion to the atmosphere through the surface mixed layer.

## Repository structure

```
EOSC511Project/
├── source/
│   └── glacier_v3.py          # Current model (diffusion, sinks, optional advection)
├── archive/
│   ├── glacier.py             # Earlier version: diffusion + sinks (ML, alpha as arguments)
│   └── glacier_v2.py          # Earlier version: diffusion, alpha hard-coded
├── scripts/
│   └── euler2d.py             # Standalone 2-D advection test (Euler upstream / Lax-Wendroff)
├── notebooks/
│   ├── glacier_perturbations.ipynb       # Sensitivity experiments (ML, alpha, input depth, advection)
│   ├── ScientificSignificanceKate.ipynb  # Sensitivity runs + comparison with observed salinity
│   └── DiffusionModel_Jose.ipynb         # Advection–diffusion run + animation
├── RunModel.ipynb             # Main entry point: run glacier_v3 with motion and plot/animate
├── DiffusionModel.ipynb       # Diffusion-only runs (steady vs. with sinks), timing test
├── data/
│   ├── Mittiesalinity.mat     # Observed salinity profile (variable S1)
│   └── salinity_depth.mat     # Depths matching the observed profile (variable depth)
├── figures/                   # Saved model output figures (PDF/PNG)
├── README.md
└── README.txt.rtf
```

`__pycache__/` and `.ipynb_checkpoints/` are build artifacts and are listed in `.gitignore`.

## The model: `source/glacier_v3.py`

The main function is:

```python
glacier(ngridx, ngridz, dt, zinput, T, ML, alpha, motion=False, steady=True)
```

| Argument | Meaning | Recommended value |
|---|---|---|
| `ngridx`, `ngridz` | Grid points in x and z | 50, 20 |
| `dt` | Time step [s] | 60 |
| `zinput` | Depth of the freshwater/methane input at the glacier wall [m] | 0 (other depths are unstable when `motion=True`) |
| `T` | Total run time [s] | e.g. 5·86400 (5 days) |
| `ML` | Mixed-layer thickness for atmospheric evasion [m] | ≥ dz (e.g. 20) |
| `alpha` | Methane oxidation rate [s⁻¹] | 0.4/86400 (0.4 day⁻¹) |
| `motion` | `True` adds advection (u, w) driven by the density/pressure gradient and inflow | |
| `steady` | `True` = no sinks; `False` = oxidation + evasion | |

**Returns:** `C, S` when `motion=False`, or `C, S, u, w` when `motion=True`. Each output is a `(time, x, z)` array. `C` is methane [nM], offset by a 3.7 nM atmospheric-equilibrium background. `S` is salinity [PSU].

**Domain and physics:**
- 20 km long and 200 m deep, on an Arakawa B-grid (C and S at grid points, u and w between them).
- The glacier wall inputs C₀ = 10 nM methane and S₀ = 0 PSU. The ambient salinity is 33 PSU. The inflow is Q = 5×10⁻⁴ m/s.
- Time stepping uses forward Euler. Diffusion uses centred second-order differences. Advection uses an upstream scheme.
- Density comes from a linearized equation of state. The vertical velocity w is diagnosed from continuity.
- Boundary conditions: zero-gradient at the surface, bottom and open-ocean boundaries, with the source fixed at the glacier wall.

The older versions in `archive/` cover diffusion only (no `motion` branch is implemented). `glacier_v2.py` takes no `ML` or `alpha` arguments.

## Running the notebooks

Each notebook's first cell adds `source/` and `archive/` to `sys.path`, so `import glacier_v3` and `import glacier` / `glacier_v2` work. Data is loaded from `data/` and figures are saved to `figures/`. Paths are relative to the notebook's own folder, so run each notebook from where it sits:

```python
import sys
sys.path += ['source', 'archive']        # notebooks at the repo root
sys.path += ['../source', '../archive']  # notebooks in notebooks/
```

Most `savefig` calls are commented out. Uncomment them to regenerate the figures.

**Dependencies:** `numpy`, `scipy`, `matplotlib`, `cmocean`, `mat4py` (for the `.mat` data), and `ffmpeg` / `ffmpeg-python` (for animations and `euler2d.py`).

## Experiments and outputs

| Figure (`figures/`) | Content | Produced in |
|---|---|---|
| `SteadyDiffusion.pdf`, `diffusionsteady.pdf` | Diffusion-only methane and salinity fields, no sinks | `DiffusionModel.ipynb` |
| `SinksDiffusion.pdf` | Diffusion with oxidation + evasion | `DiffusionModel.ipynb` |
| `fig.pdf` | Methane profile at a distance from the glacier | `DiffusionModel.ipynb` |
| `ss_ML_sensitivity.pdf`, `ML_sensitivity.pdf` | Sensitivity to mixed-layer thickness (ML = 20–60 m) | `glacier_perturbations.ipynb` |
| `Max_ML.pdf` | Mixed layer thick enough to suppress the plume (80 m vs. 200 m) | `glacier_perturbations.ipynb`, `ScientificSignificanceKate.ipynb` |
| `ss_alpha_sensitivity.pdf`, `alpha_sensitivity.pdf` | Sensitivity to oxidation rate α (0, 0.4, 4, 40 day⁻¹) | `glacier_perturbations.ipynb`, `ScientificSignificanceKate.ipynb` |
| `ss_z_input_methane.pdf`, `ss_z_input_salinity.pdf` | Effect of input depth (20, 60, 100, 180 m) | `glacier_perturbations.ipynb`, `ScientificSignificanceKate.ipynb` |
| `AdvDiff.pdf`, `Diffusion-Advection.pdf` | Advection–diffusion run: C, S, u, w | `RunModel.ipynb`, notebooks in `notebooks/` |
| `Salinity_Methane_Depth_Profile.pdf/.png` | Modelled methane and salinity profiles vs. observed salinity (`data/*.mat`) | `glacier_perturbations.ipynb`, `ScientificSignificanceKate.ipynb` |

`RunModel.ipynb` and `DiffusionModel_Jose.ipynb` also build an HTML5 animation of how the advection–diffusion run evolves over time.

## Known limitations

- The advective (`motion=True`) case is only stable for `zinput = 0`.
- Several constants are marked as placeholders in the code (e.g. the input methane concentration `C0` and the inflow velocity `u0`).
