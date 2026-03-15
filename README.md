# Energy Island System Optimization (MILP)

A techno-economic **Mixed Integer Linear Programming (MILP)** model for optimizing the generation mix and storage capacity of a renewable-based **energy island** or **isolated microgrid** system.

The model simultaneously determines optimal **investment decisions** (installed capacity of each technology) and **hourly operational dispatch** to meet system electricity demand at minimum total cost, minimum CO₂ emissions, or with a guaranteed presence of every selected technology.

---

## System Technologies

### Variable Renewable Generation

| Technology | Description |
|---|---|
| **Wind** | Hourly generation profile scaled to 1 MW installed capacity |
| **Solar PV** | Hourly generation profile scaled to 1 MW installed capacity |

### Dispatchable Generation

| Technology | Dispatch mode | Description |
|---|---|---|
| **Biomass** | Non-flexible (must-run) | Combustion of solid biomass; dispatch pinned to hourly profile × installed capacity |
| **Biogas** | Non-flexible (must-run) | Anaerobic digestion of organic waste; dispatch pinned to hourly profile × installed capacity |
| **Waste-to-Energy (WTE)** | Non-flexible (must-run) | Energy recovery from municipal solid waste; dispatch pinned to hourly profile × installed capacity |
| **Hydro** | Flexible dispatch | Run-of-river or reservoir hydropower; free to dispatch 0 → profile × capacity |
| **Geothermal** | Non-flexible (must-run) | Baseload generation; dispatch pinned to hourly profile × installed capacity |
| **Gas turbine** | Flexible backup | Fast-response backup; highest variable cost and emissions |

> **Non-flexible dispatch:** Biomass, Biogas, Geothermal, and WTE are forced to dispatch exactly at `profile(t) × capacity` each hour via a zero-curtailment constraint. Unused capacity in flexible dispatchable or gas technologies is standby reserve — not curtailment.

### Energy Storage

| Technology | Typical Duration | Description |
|---|---|---|
| **BESS** | 2–6 hours | Battery Energy Storage System; fast response, flexible siting |
| **PHS** | 6–12 hours | Pumped Hydro Storage; requires ≥ 300 m head difference |
| **Hydrogen** | 12–48+ hours | Long-duration storage via electrolysis and fuel cell |

---

## Model Features

- **Capacity expansion optimization** — simultaneous investment sizing across all technologies via Pyomo + GLPK
- **Hourly dispatch modeling** — full 8,760-hour annual resolution
- **Non-flexible dispatch constraint** — Biomass, Biogas, Geothermal, and WTE dispatch is pinned to `profile(t) × capacity` via a zero-curtailment equality constraint
- **Grid loss factor** — raw demand inflated by a configurable distribution loss factor (default 4%)
- **Capital Recovery Factor** — CAPEX annualised over asset lifetime at user-defined discount rate
- **Storage SoC tracking** — cyclic state-of-charge with round-trip efficiency losses; minimum SoC floor enforced as a hard constraint
- **Storage duration limits** — user-defined maximum energy-to-power ratio per technology
- **Renewable-only charging** — storage cannot charge from gas generation
- **Renewable curtailment** — modeled explicitly; zero curtailment enforced for non-flexible technologies
- **Three-objective optimization** — Lowest LCOE, Lowest CO₂ (via carbon shadow price), or Most Diversified
- **Interactive setup widget** — technology selection, storage duration, discount rate, currency symbol, and demand scaling configurable via ipywidgets
- **Configurable currency symbol** — accepts any string (€, $, £, RM, Rp, etc.); applied to all cost labels and exports
- **Demand scaling** — multiply demand CSV by a constant factor directly from the widget, without editing source data
- **PHS feasibility check** — project location elevation validated against 300 m minimum head requirement
- **Dual export** — hourly results to Excel (`.xlsx`) and structured JSON (`.json`) for dashboard consumption

---

## Optimization Formulation

### Objective — Lowest LCOE

Minimise total annualised system cost:

$$\min \sum_{g} \text{Cap}_g \cdot (K_g \cdot \text{CRF}_g + O\&M_g)
+ \sum_{s} \text{StorPow}_s \cdot (K_s^{\text{MW}} \cdot \text{CRF}_s + O\&M_s)
+ \sum_{s} \text{StorEne}_s \cdot K_s^{\text{MWh}} \cdot \text{CRF}_s
+ \text{fuel costs}
+ \text{curtailment penalty}
+ \text{storage cycling cost}$$

where the **Capital Recovery Factor** annualises investment over asset lifetime:

$$\text{CRF} = \frac{r(1+r)^n}{(1+r)^n - 1}$$

### Objective — Lowest CO₂

A carbon shadow price (CSP = 10,000 €/tCO₂) is added to the marginal cost of each emitting technology, heavily penalising CO₂-intensive dispatch.

### Objective — Most Diversified

Identical marginal costs to Lowest LCOE, with a mandatory minimum installed capacity (`DIVERSIFIED_MIN_MW`, default 1 MW) applied to every selected technology.

### Key Constraints

| Constraint | Description |
|---|---|
| Power balance | Generation + discharge + gas = gross demand + charge (every hour) |
| Capacity limits | `RenGen ≤ Cap × profile` (availability upper bound) |
| Non-flexible floor | `Curtail[tech, t] == 0` for Biomass, Biogas, Geothermal, WTE |
| Max capacity bounds | `Cap ≤ Max_Capacity_MW` (resource constraint) |
| Storage power limits | Charge and discharge ≤ power capacity |
| SoC dynamics | $\text{SoC}_{s,t} = \text{SoC}_{s,t-1} + \eta \cdot \text{Charge}_{s,t} - \text{Discharge}_{s,t} / \eta$ |
| SoC limits | 20% minimum, 100% maximum of energy capacity |
| Duration limits | Energy capacity ≤ power capacity × max hours |
| Cyclic SoC | End-of-year SoC = initial SoC (50% of energy capacity) |
| Renewable charging | Storage charge ≤ total renewable generation (no gas → storage) |
| Grid loss factor | $P_{\text{gross}}(t) = P_{\text{demand}}(t) \times (1 + \text{GLF})$ |

---

## Model Constants

All hard-coded assumptions are defined in a single dedicated cell — the **only cell** that needs editing to change global model assumptions.

| Constant | Value | Description |
|---|---|---|
| `HOURS_PER_YEAR` | 8760 | Full-year hourly resolution |
| `GRID_LOSS_FACTOR` | 0.04 | Distribution loss factor — 4% of delivered energy lost in the island grid |
| `SOC_MIN_FRACTION` | 0.20 | Minimum state-of-charge — prevents deep discharge damage |
| `SOC_INIT_FRACTION` | 0.50 | Initial and final SoC for cyclic boundary condition |
| `CURTAILMENT_PENALTY` | 1 €/MWh | Small penalty to discourage oversized VRE capacity with curtailment |
| `STORAGE_CHARGE_COST` | 5 €/MWh | Proxy cost to prevent unnecessary storage cycling |
| `CARBON_SHADOW_PRICE` | 10,000 €/tCO₂ | Shadow price applied in Lowest CO₂ objective mode |
| `DIVERSIFIED_MIN_MW` | 1 MW | Minimum installed capacity per technology in Most Diversified mode |

A user-configurable **Demand Scale %** (default 100%) and **currency symbol** are set in the Step 3 widget and are scenario-specific — they are not model constants.

---

## Repository Structure

```
energy-island-milp-optimization/
│
├── data/
│   ├── inputs/
│   │   ├── geographic_setup.csv      # Project location and PHS height check
│   │   └── resource_assessment.csv   # Techno-economic parameters per technology
│   │
│   └── time_series/
│       ├── demand.csv                # Hourly system demand (MW)
│       ├── wind_prod.csv             # Normalised wind generation profile [0–1]
│       ├── solar_prod.csv            # Normalised solar generation profile [0–1]
│       └── <tech>_prod.csv           # Profiles for other technologies
│
└── Energy_Island_MILP_Optimization.ipynb
```

---

## Requirements

```bash
pip install pyomo pandas numpy matplotlib plotly openpyxl ipywidgets
```

**GLPK Solver:**
```bash
# Ubuntu / Debian
sudo apt install glpk-utils

# macOS
brew install glpk

# Windows — https://winglpk.sourceforge.net/
```

---

## Input File Specifications

### `geographic_setup.csv`

| Column | Type | Description |
|---|---|---|
| `Name` | string | Location or project name |
| `Latitude` | float | Decimal degrees north |
| `Longitude` | float | Decimal degrees east |
| `Max_Height_m` | float | Maximum terrain elevation difference (m) — used for PHS feasibility check (≥ 300 m required) |

### `resource_assessment.csv`

| Column | Unit | Description |
|---|---|---|
| `Sources` | — | Technology name — must match Step 3 selections exactly |
| `Max_Capacity_MW` | MW | Maximum installable capacity (resource or land constraint) |
| `Investment_per_MW` | €/MW | Capital cost per MW of installed power capacity |
| `O&M_per_MW_yr` | €/MW/yr | Annual fixed operation and maintenance cost |
| `Lifetime` | years | Economic asset lifetime for CRF calculation |
| `CO2_per_MWh` | tCO₂/MWh | Direct CO₂ emissions per MWh generated |
| `Fuel_Cost` | €/MWh_fuel | Variable fuel cost (0 for renewables) |
| `Efficiency` | 0–1 | Generator thermal efficiency or storage one-way efficiency |
| `Merit_Order` | integer | Dispatch priority (lower = dispatched first) |
| `Storage_MWh` | €/MWh | Capital cost per MWh of energy storage capacity (0 for generators) |

> **Note:** `Efficiency` for generators is used only in the economic fuel cost calculation — the generation profiles already represent electrical output and must not be scaled by efficiency again.

---

## Usage

1. Prepare input CSV files in the `data/` folder
2. Open `Energy_Island_MILP_Optimization.ipynb`
3. Run cells sequentially:

| Step | Description |
|:---:|---|
| **1** | Import required libraries and configure global matplotlib style |
| **Constants** | Review and adjust model constants (loss factor, SoC limits, penalties, `DIVERSIFIED_MIN_MW`) |
| **2** | Load geographic project information — enter file path and click Load |
| **3** | Configure technologies, objective, storage duration, discount rate, currency symbol, and demand scaling |
| **4** | Load technology and resource parameter CSV — enter file path and click Load |
| **5** | Load hourly demand and generation time series — enter file paths and click Load |
| **6** | Build and solve the MILP optimization model |
| **7** | Export results to Excel (`.xlsx`) and dashboard JSON (`.json`) |
| **8** | Analyse and visualise results |

**Demand Scaling**

The **Demand Scale %** widget in Step 3 multiplies every hourly value of the loaded demand CSV by a constant factor:

| Setting | Scenario |
|---|---|
| `100%` | Baseline — demand loaded as-is (default) |
| `110%` | +10% demand growth |
| `90%` | −10% efficiency / conservation |
| `150%` | +50% electrification or EV uptake |

**Generation profile data sources:**
- [Renewables.ninja](https://www.renewables.ninja) — Wind and solar PV hourly profiles
- [Global Solar Atlas](https://globalsolaratlas.info) — Solar irradiance data
- [PVGIS](https://re.jrc.ec.europa.eu/pvg_tools/) — European Commission PV estimation tool
- [ERA5 / Copernicus](https://cds.climate.copernicus.eu) — Global climate reanalysis data
- [IRENA Resource Map](https://resourceirena.irena.org) — Renewable resource atlases

---

## Key Outputs

| Output | Description |
|---|---|
| Installed capacity | Optimal MW per generation technology |
| Storage sizing | Power (MW) and energy (MWh) capacity per storage technology |
| System LCOE | Levelised cost of electricity (currency/MWh) |
| Technology LCOE / LCOS | Per-technology levelised cost of generation and storage |
| LCOE cost breakdown | Stacked CAPEX / O&M / fuel decomposition per technology |
| Annual energy mix | Generation breakdown as share of total demand (donut chart with callout leader lines) |
| Dispatch charts | Stacked hourly generation in merit order with net and gross demand overlay |
| Capacity factors | Annual generation ÷ (installed capacity × 8,760 h) |
| Monthly capacity factor grid | Technology × month CF heatmap with annotated cell values |
| Demand heatmap | Average gross demand by hour-of-day × month (24 × 12 grid) |
| Load duration curve | Sorted annual dispatch showing peak-to-valley coverage |
| State-of-charge | Storage SoC profiles with capacity and minimum SoC reference bands |
| Residual load | Demand minus renewables — identifies storage and backup needs |
| Worst renewable week | Auto-detected most critical 168-hour window for system reliability |
| Energy Sankey | Interactive annual energy flow: generation → storage → Grid Bus → Load + Losses + Curtailment |
| CO₂ summary | Per-technology and total annual emissions with intensity (gCO₂/kWh) |
| Excel export | Full 8,760-hour hourly dispatch for all technologies |
| JSON export | Structured results file consumed by the interactive dashboard |

### Visualisation Methods (`ResultsVisualization`)

| Method | Chart type | Description |
|---|---|---|
| `summary()` | Text | Installed capacity of all generation and storage technologies |
| `calculate_lcoe()` | Text | Per-technology LCOE/LCOS, system LCOE, and annual CO₂ summary |
| `plot_installed_capacity()` | Bar chart | Installed MW per generation technology with value labels |
| `plot_storage_capacity()` | Side-by-side bars | Storage power (MW) and energy (MWh) capacity |
| `plot_energy_mix()` | Donut chart | Annual generation mix as % of net demand — callout leader lines with collision avoidance |
| `plot_capacity_factors()` | Horizontal bars | Achieved capacity factor; technology name printed inside bar, CF % to the right |
| `plot_lcoe_breakdown()` | Stacked horizontal bars | Annualised cost decomposed into CAPEX / O&M / fuel per technology |
| `plot_dispatch(hours)` | Stacked area | Hourly dispatch by technology with net and gross demand overlay lines |
| `plot_load_duration()` | Step chart | Sorted load duration curve with stacked generation breakdown |
| `plot_demand_heatmap()` | Heatmap | Average gross demand by hour-of-day (rows 0–23) × month (columns Jan–Dec) |
| `plot_monthly_cf()` | Heatmap grid | Monthly capacity factor per technology; CF % annotated in each cell |
| `plot_soc(hours)` | Line chart | Storage state of charge with capacity and 20% minimum SoC reference bands |
| `plot_residual(hours)` | Area chart | Residual load after renewables (deficit/surplus) |
| `plot_worst_residual_week()` | Combined | Auto-detects and plots dispatch, SoC, and residual for the hardest 168-hour window |
| `plot_energy_sankey()` | Sankey (Plotly) | Interactive annual energy flow: generation → storage → Grid Bus → Load + Losses + Curtailment |

Pass `hours` as a `(start, end)` tuple or a single integer (168-hour window from that hour):

```python
viz.plot_dispatch((1000, 1168))   # explicit window
viz.plot_dispatch(1000)           # 168-hour window from hour 1000
```

### Excel Export Columns (`optimization_results.xlsx`)

| Column pattern | Unit | Description |
|---|---|---|
| `Prod_{tech}_MW` | MW | Hourly generation dispatch per technology |
| `Discharge_{storage}_MW` | MW | Storage discharge (positive flow to grid) |
| `Charge_{storage}_MW` | MW | Storage charge (positive flow from grid) |
| `SOC_{storage}_MWh` | MWh | State of charge at end of each hour |
| `GrossDemand_MW` | MW | Demand including grid losses fed to the model |
| `NetDemand_MW` | MW | Raw consumer demand (before loss factor) |

### JSON Export Structure (`dashboard_results.json`)

Produced by `model.export_dashboard_json(filepath, geo=geo)`. Pass `geo=geo` to embed geographic data in the output.

| Key | Contents |
|---|---|
| `meta` | Objective, discount rate, currency, grid loss factor, technology lists |
| `geographic` | Project name, latitude, longitude, max height — only present when `geo` is passed |
| `capacities` | Installed MW (and MWh for storage) + `potential_mw` (from `Max_Capacity_MW`) per technology |
| `energy_mix` | Annual GWh, demand share (%), capacity factor, per-technology LCOE, CO₂, curtailment GWh |
| `lcoe_summary` | Per-technology LCOE/LCOS, annualised cost, `annualised_capex`, `annual_opex`, `annual_fuel_cost`, `total_capex` |
| `lcoe_summary._system` | System LCOE, total annualised cost, total demand, `total_annualised_capex`, `total_annual_opex`, `total_annual_fuel`, `total_capex` |
| `co2_summary` | Per-technology tCO₂/yr, total CO₂, emission intensity (gCO₂/kWh) |
| `hourly` | Full 8,760-row array with generation, storage charge/discharge/SoC, gross demand, and net demand |

---

## Disclaimer

Developed for educational and research purposes. Results should not be used as the sole basis for investment or planning decisions. The author makes no guarantees regarding the completeness or accuracy of the model outputs.

---

## References

- Hart, W.E. et al. (2017). *Pyomo – Optimization Modeling in Python*. Springer.
- IRENA (2023). *Renewable Power Generation Costs 2023*. International Renewable Energy Agency.
- IEA (2023). *World Energy Outlook 2023*. International Energy Agency.
- NREL (2023). *Annual Technology Baseline 2023*. National Renewable Energy Laboratory.
- Pfenninger, S. & Staffell, I. (2016). Long-term patterns of European PV output using 30 years of validated hourly reanalysis and satellite data. *Energy*, 114, 1251–1265.
- Staffell, I. & Pfenninger, S. (2016). Using bias-corrected reanalysis to simulate current and future wind power output. *Energy*, 114, 1224–1239.
- Anthropic (2025). *Claude AI Assistant* (claude.ai). Used to support model development, code review, and documentation.

---

*Author: Agus Samsudin — Energy Systems Modelling · Optimization · Renewable Energy*
