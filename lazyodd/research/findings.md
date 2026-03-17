# ODD Research Findings

## Model Overview
- **Name:** SpatialRust — Coffee Leaf Rust Epidemic Model
- **Authors:** Manuela Vanegas Ferro
- **Purpose:** Explore how the interaction between shade management, weather conditions (temperature, rainfall, wind), and fungicide strategies affects Coffee Leaf Rust (*Hemileia vastarix*) epidemic dynamics and coffee farm productivity
- **Input Sources:** All source files in `src/` directory (Julia/Agents.jl), `scripts/`, `README.md`, `Project.toml`
- **Input Quality Assessment:** Code + minimal docs
- **Model Complexity:** Moderate-to-Complex (1 explicit agent type, ~17 submodels, 2D grid, extensive stochasticity)
- **ODD Format Decision:** Strict ODD+2 (Grimm et al. 2020)

---

## Element 1: Purpose and Patterns

### Findings

**Specific purpose:** The model explores how the interaction between shade management, weather conditions (temperature, rainfall, wind), and fungicide strategies affects Coffee Leaf Rust epidemic dynamics and coffee farm productivity. Shade trees have complex, competing effects: they reduce sunlight (lowering photosynthesis), cool local temperature, modify rain splash dispersal distance, and block wind dispersal.

**Higher-level purpose:** Explanation and prediction — understanding mechanisms by which shade trees mediate rust epidemics and evaluating management strategies under different weather scenarios.

**Patterns the model should reproduce:**
- Realistic disease progression curves (incidence and severity over annual cycles)
- Known rust biology: ~5-month lesion senescence (McCain & Hennen 1984), rain wash-off effects (Avelino et al. 2020), kinetic energy amplification under shade (Gagliardi et al. 2021), spore viability loss (Nutman et al. 1963)
- 10% incidence threshold as management trigger (Cenicafé Boletín 36)
- Model was calibrated using ABC (Approximate Bayesian Computation) — `p_row` parameter described as "for ABC"

**Output metrics:** incidence (fraction infected), severity (mean lesion area), farm production (cumulative yield)

### Sources
- `src/ABM/MainSetup.jl:37-38` — `p_row` parameter "for ABC"
- `src/ABM/MainSetup.jl:65` — rain_washoff comment "Avelino et al., 2020"
- `src/ABM/MainSetup.jl:77` — diff_splash comment referencing Avelino et al. 2020 and Gagliardi et al. 2021
- `src/ABM/RustGrowth.jl:5` — senescence comment "McCain & Hennen, 1984"
- `src/ABM/MainSetup.jl:86` — incidence threshold "10% from Cenicafe's Boletin 36"
- `src/ABM/RustGrowth.jl:194` — viability loss comment "Nutman et al, 1963"
- `src/Metrics.jl:3-7` — metric definitions
- `scripts/ParameterRuns.jl:7-9` — parameter sweep over temperature, rain, wind

### Confidence
- Purpose: `INFERRED` — inferred from code structure, parameter sweep design, and the centrality of shade mechanisms
- Patterns: `CODE_VERIFIED` for literature references in comments; `INFERRED` for ABC calibration
- Metrics: `CODE_VERIFIED` — explicit in `src/Metrics.jl`

### Open Questions
- What specific empirical data was the model calibrated against?
- Are there published papers describing this model's results?
- What patterns did the model fail to reproduce?

---

## Element 2: Entities, State Variables, and Scales

### Findings

**Entity type 1: Coffee plants** — explicit agent (`@agent Coffee GridAgent{2}`)

| State Variable | Type | Description | Units/Range |
|---|---|---|---|
| `sunlight` | Float64 | Light reaching plant (1 - shade_map × ind_shade) | 0.0–1.0 |
| `veg` | Float64 | Vegetative biomass | arbitrary, init=2.0 |
| `storage` | Float64 | Stored carbohydrates | arbitrary, init from `75.5*exp(-5.5*sl)+2.2` |
| `production` | Float64 | Reproductive/fruit biomass | arbitrary, init=0.0 |
| `exh_countdown` | Int | Days remaining if plant exhausted by rust | 0 or 731 |
| `rust_gr` | Float64 | Individual rust growth rate | ~0.0867 |
| `rusted` | Bool | Whether plant has any rust | true/false |
| `newdeps` | Float64 | Newly deposited spores this step | count |
| `deposited` | Float64 | Accumulated deposited spores | count |
| `n_lesions` | Int | Number of active lesions | 0–25 |
| `ages` | Vector{Int} | Age in days of each lesion | days |
| `areas` | Vector{Float64} | Area of each lesion | ~0–25.0 |
| `spores` | Vector{Bool} | Whether each lesion is sporulating | true/false |

**Entity type 2: Shade trees** — implicit, cells with value 2 in `farm_map`. No individual state; influence captured by static `shade_map` and dynamic global `ind_shade`.

**Entity type 3: Farmer/Grower** — implicit, model-level actions driven by `MngPars` schedule parameters and `Books` tracking.

**Entity type 4: Environment** — global `Weather` struct with daily rain (Bool), wind (Bool), temperature (Float64) vectors.

**Model-level tracking** (`Books` mutable struct): days, ticks, ind_shade, temperature, rain, wind, wind_h (heading 0–360°), fungicide (days since application), fung_count, obs_incidence, costs, prod, inbusiness, withinbounds, shadeacc.

**Scales:**
- **Spatial:** 2D grid, default 100×100 cells, non-periodic, Chebyshev metric. Each cell holds one coffee OR one shade tree OR is empty.
- **Temporal:** 1 step = 1 day. Typical run: 730 steps (2 years). Annual cycle of 365 days.
- **Warm-up:** 2 years (730 days) of coffee-only growth before rust inoculation.

### Sources
- `src/ABM/MainSetup.jl:4-20` — Coffee agent definition
- `src/ABM/MainSetup.jl:232-349` — parameter structs (Weather, CoffeePars, RustPars, MngPars, Books, Props)
- `src/ABM/CreateABM.jl:4` — GridSpaceSingle configuration
- `scripts/ParameterRuns.jl:11-12` — typical run length

### Confidence
- All state variables: `CODE_VERIFIED`
- Units and ranges: `CODE_VERIFIED` for coded constraints, `INFERRED` for missing explicit units on biomass variables

### Open Questions
- What are the physical units of `veg`, `storage`, and `production`? (appear to be arbitrary/dimensionless)
- What does one grid cell represent in physical space? (no explicit spatial scale given)

---

## Element 3: Process Overview and Scheduling

### Findings

Each time step (1 day), `step_model!` executes five processes in fixed sequential order. The model does NOT use the Agents.jl built-in scheduler — it manually iterates `model.agents` (a `Vector{Coffee}`) in deterministic order (column-major order from farm map position).

**Daily schedule:**

1. **pre_step!** — Increment day/tick counters, read weather from pre-generated vectors (indexed by tick), decay outpour spores (×0.9), randomize wind heading if windy (Uniform 0–360°)

2. **grow_shades!** — Logistic growth of global ind_shade: `Δs = rate × (1 - s/max) × s`

3. **coffee_step!** — All agents iterated in storage order:
   - Days 1–134 (veg_d ≤ mod1(days,365) < rep_d): vegetative growth
   - Day 135 (== rep_d): vegetative growth + resource commitment to production
   - Days 136–365: reproductive growth
   - Handles wrap-around when veg_d > rep_d via sign negation
   - Exhausted plants (exh_countdown > 0) decrement or regrow

4. **rust_step!** — Function variants selected from 8 combinations of {fungicide?, rain?, wind?}:
   - **Loop 1** (all rusted agents, lazy filter): disperse FIRST (rain splash, wind), then compute local temperature, then grow/sporulate, then germinate deposited spores, then parasitize
   - **Loop 2** (lazy filter re-evaluated, includes newly-rusted plants): merge newdeps into deposited with viability loss
   - If windy: outside spore re-entry from outpour pool
   - Key: newdeps/deposited separation prevents new deposits from germinating same step

5. **farmer_step!** — Annual harvest, scheduled pruning, periodic inspection (every inspect_period days), fungicide logic (calendar/incidence-based/hybrid/flowering strategies)

**Warm-up (pre_run365!):** 730 days of grow_shades! + coffee_step! + scheduled pruning only. No pre_step! (no weather), no rust_step!, no farmer_step!. Simplified harvest (production=0 only). Resets days=0, costs=0 after completion.

### Sources
- `src/ABM/MainStep.jl:3-9` — step_model! orchestration
- `src/ABM/MainStep.jl:14-35` — pre_step!
- `src/ABM/ShadeSteps.jl:1-3` — grow_shades!
- `src/ABM/MainStep.jl:37-84` — coffee_step! with phenology phases
- `src/ABM/MainStep.jl:86-164` — rust_step! and rust_step_schedule
- `src/ABM/MainStep.jl:170-206` — farmer_step!
- `src/ABM/CreateABM.jl:112-200` — pre_run365! warm-up

### Confidence
- Process order and scheduling: `CODE_VERIFIED`
- Lazy filter re-evaluation in Loop 2: `CODE_VERIFIED`
- newdeps/deposited separation: `CODE_VERIFIED`

### Open Questions
- Why is the agent execution order deterministic (storage/column-major) rather than randomized?
- Is the UV inhibition asymmetry between grow_rust! and grow_f_rust! intentional?

---

## Element 4: Design Concepts

### 4.1 Basic principles

#### Findings
The model integrates plant physiology (Michaelis-Menten photosynthesis with allocation tradeoffs), epidemiology (full rust lifecycle with temperature/moisture dependence), and agroecology (shade trees as mediators of microclimate, dispersal, and light). An ABM was chosen because spatial heterogeneity in shade, dispersal directionality, individual plant state, and farm geometry all require individual-level representation.

#### Sources
- `src/ABM/CoffeeSteps.jl:6-14` — Michaelis-Menten photosynthesis
- `src/ABM/RustGrowth.jl:1-35` — temperature-dependent rust growth
- `src/ABM/RustDispersal.jl:1-52` — spatially explicit dispersal

#### Confidence
`INFERRED` — theoretical basis inferred from code structure; no documentation states explicit theoretical grounding.

#### Open Questions
- What specific theories or hypotheses motivated the model design?
- Why was an ABM chosen over spatially explicit compartmental models?

### 4.2 Emergence

#### Findings
Emergent: epidemic dynamics (incidence/severity curves), spatial infection patterns, production losses from parasitism. Imposed: weather (pre-generated), farm layout (fixed), management schedules.

#### Sources
- `src/Metrics.jl:3-7` — emergent metrics
- `src/ABM/MainSetup.jl:232-236` — imposed weather

#### Confidence
`INFERRED`

### 4.3 Adaptation

#### Findings
Coffee plants do not adapt. The farmer shows limited reactive adaptation through incidence-triggered fungicide strategies (`:incd`, `:cal_incd`): spray when observed incidence exceeds 10% threshold. No optimization.

#### Sources
- `src/ABM/MainStep.jl:186-201` — fungicide decision logic
- `src/ABM/MainSetup.jl:86` — incidence threshold

#### Confidence
`CODE_VERIFIED`

### 4.4 Objectives

#### Findings
No explicit objectives. No agent optimizes or satisfices. Farmer follows rule-based strategies.

#### Confidence
`CODE_VERIFIED`

### 4.5 Learning

#### Findings
No learning. No agents change decision rules over time. Commented-out GA code suggests evolutionary optimization was explored but is inactive.

#### Sources
- `src/ABM/MainStep.jl:31-33` — commented GA reintroduction code

#### Confidence
`CODE_VERIFIED`

### 4.6 Prediction

#### Findings
No prediction. Decisions are reactive or calendar-based, not anticipatory.

#### Confidence
`CODE_VERIFIED`

### 4.7 Sensing

#### Findings
- Coffee plants: sense local sunlight (position-dependent shade × global shade level)
- Rust: implicitly affected by local temperature (shade-cooled), rain, wind
- Farmer: imperfect sensing via periodic inspection — samples `inspect_effort` (1%) of plants every `inspect_period` (7) days, detection weighted by lesion visibility (area > 0.05)

#### Sources
- `src/ABM/CoffeeSteps.jl:2-3` — update_sunlight!
- `src/ABM/CGrowerSteps.jl:45-75` — inspect!

#### Confidence
`CODE_VERIFIED`

### 4.8 Interaction

#### Findings
- **Rust-mediated indirect**: infected plants disperse spores to neighbors via rain splash (local, ~2.2 cells mean) and wind (directional, ~5.8 cells mean)
- **Shade-mediated indirect**: shade trees affect all coffee within radius via shade map (static spatial, dynamic global level)
- **No direct agent-agent interaction**: no communication, no density dependence between plants
- **Farm boundary**: outpour pool creates attenuated directional spore re-entry on windy days

#### Sources
- `src/ABM/RustDispersal.jl:3-52` — dispersal mechanics
- `src/ABM/ShadeMap.jl:9-33` — shade influence

#### Confidence
`CODE_VERIFIED`

### 4.9 Stochasticity

#### Findings
Extensive stochasticity serving three purposes:

1. **Natural variability**: weather generation, dispersal distances/angles/counts (Poisson, Exponential, Uniform), germination/sporulation (Bernoulli), UV inhibition, rain washoff
2. **Individual heterogeneity**: per-plant rust_gr (Truncated Normal), resource commitment (Normal), management cost noise (Normal)
3. **Imperfect monitoring**: inspection sampling, visibility-weighted detection

Key distributions: Bernoulli (rain/wind/germination/sporulation/blocking), Normal (temperature/costs/rust_gr), Poisson (spore counts), Exponential (dispersal distances), Binomial (initial lesion counts), Uniform (wind heading/splash angle).

#### Sources
- Throughout all source files; see Element 7 submodels for specific distributions per process

#### Confidence
`CODE_VERIFIED`

### 4.10 Collectives

#### Findings
No formal collectives. Implicit spatial clustering from initial rust introduction (radius-1 clusters when `ini_rusts ≥ 1`) and local dispersal creating infection neighborhoods.

#### Sources
- `src/ABM/CreateABM.jl:43-50` — rusted_cluster

#### Confidence
`CODE_VERIFIED`

### 4.11 Observation

#### Findings
Three output metrics:
- `incidence` = mean(n_lesions > 0) across all agents
- `severity` = mean(sum(areas)) across all agents
- `production` = mean(production) across all agents

Single runs: daily time series. Parameter sweeps: final incidence, severity, cumulative farm production (÷1000).

Internal tracking: costs, obs_incidence, shadeacc, outpour, withinbounds, inbusiness.

#### Sources
- `src/Metrics.jl:3-7`
- `src/Runners.jl:12-28` — daily collection
- `src/Runners.jl:51-69` — sweep collection

#### Confidence
`CODE_VERIFIED`

---

## Element 5: Initialization

### Findings

**Farm layout:** Configurable via three options:
- Default (`:none`): Coffee rows (row_d=2, plant_d=1), regular shade (shade_d=6), barrier rows (barrier_rows=2, barriers=(1,0))
- `:fullsun`: Coffee only
- `:regshaded`: Coffee + regular shade

Farm map: Int matrix (0=empty, 1=coffee, 2=shade). Default 100×100.

**Shade map:** Static, computed once. Each cell gets inverse Euclidean distance to nearest shade tree within shade_r=3 radius. Shade tree cells = 1.0.

**Coffee agents:** One per coffee cell. veg=2.0, storage=75.5×exp(-5.5×sunlight)+2.2, production=0.0, rust_gr ~ Truncated(Normal(0.0867, 0.000867), 0, 0.13).

**2-year warm-up:** 730 days of shade growth + coffee phenology + scheduled pruning (no rust, no weather, no farmer actions beyond pruning). Simplified harvest (production reset only). After warm-up: days=0, costs=0.

**Rust inoculation (post warm-up):**
- 0 < p_rusts < 1: Random fraction of all plants (scattered)
- p_rusts = 1: One spatial cluster (radius 1)
- p_rusts > 1: Multiple clusters
- Each plant: 1 + Binomial(24, 0.05) lesions, random areas 0–0.3, categorized as deposited/non-sporulating/sporulating

**Weather:** Pre-generated for entire simulation. Stochastic or from input data with noise.

**Variation between runs:** Yes — weather, rust_gr, rust placement, cost noise all stochastic. Reproducible with seed > 0.

### Sources
- `src/ABM/FarmMap.jl:2-65` — farm layout generation
- `src/ABM/ShadeMap.jl:9-33` — shade map
- `src/ABM/CreateABM.jl:17-38` — tree placement
- `src/ABM/CreateABM.jl:52-110` — rust inoculation
- `src/ABM/CreateABM.jl:112-200` — warm-up
- `src/ABM/MainSetup.jl:33-228` — init_spatialrust

### Confidence
`CODE_VERIFIED`

### Open Questions
- Why 2-year warm-up specifically?
- What does the initial storage function `75.5*exp(-5.5*sl)+2.2` represent biologically?

---

## Element 6: Input Data

### Findings

The model can accept external time-varying input:
- `rain_data::Vector{Bool}` — daily rain occurrence
- `wind_data::Vector{Bool}` — daily wind occurrence
- `temp_data::Vector{Float64}` — daily temperature

If provided, noise is added (5% bit-flip for booleans, Normal(0,0.05) for temperature). If empty, weather generated stochastically from probability parameters.

A user-supplied `farm_map::Array{Int}` can replace the generated layout.

No other external data drives the model during simulation.

### Sources
- `src/ABM/MainSetup.jl:45-47` — input data parameters
- `src/ABM/MainSetup.jl:362-383` — noise functions

### Confidence
`CODE_VERIFIED`

### Open Questions
- None

---

## Element 7: Submodels

### Submodel: Weather Generation

**Purpose:** Generate or process daily weather time series.

**Logic:**
- Rain: Bernoulli(rain_prob) or input with 5% bit-flip noise
- Wind: Bernoulli(wind_prob ± 0.1 conditional on rain) or input with noise
- Temperature: Normal(mean_temp, 0.5) or input + Normal(0, 0.05)

**Parameters:**

| Parameter | Default | Description |
|-----------|---------|-------------|
| rain_prob | 0.8 | Daily rain probability |
| wind_prob | 0.7 | Base wind probability |
| mean_temp | 22.0 °C | Mean temperature |

**Sources:** `src/ABM/MainSetup.jl:352-383`
**Confidence:** `CODE_VERIFIED`

---

### Submodel: Shade Tree Growth

**Purpose:** Model shade canopy recovery between pruning events.

**Equation:** `ind_shade += rate × (1 - ind_shade/max) × ind_shade` (logistic)

**Parameters:**

| Parameter | Default |
|-----------|---------|
| shade_g_rate | 0.015 day⁻¹ |
| max_shade | 0.8 |

**Sources:** `src/ABM/ShadeSteps.jl:1-3`
**Confidence:** `CODE_VERIFIED`

---

### Submodel: Coffee Photosynthesis

**Purpose:** Compute daily carbon assimilation (shared by vegetative and reproductive growth).

**Equation:** `PhS = (f_avail × phs_max) × (sunlight/(k_sl + sunlight)) × (photo_veg/(k_v + photo_veg))`

Product of two Michaelis-Menten saturating functions for light and photosynthetic tissue.

**Parameters:**

| Parameter | Default | Description |
|-----------|---------|-------------|
| f_avail | 0.5 | Fraction available |
| phs_max | 0.2 | Max assimilation rate |
| k_sl | 0.05 | Half-saturation (light) |
| k_v | 0.2 | Half-saturation (veg) |
| photo_frac | 0.2 | Photosynthetic fraction of veg |

**Sources:** `src/ABM/CoffeeSteps.jl:6-8`
**Confidence:** `CODE_VERIFIED`

---

### Submodel: Vegetative Growth

**Purpose:** Allocate photosynthate during vegetative phase (default days 1–134).

**Equations:**
```
veg += phs_veg × PhS - μ_veg × veg
storage += phs_sto × PhS
veg = max(veg, 0)
```

**Parameters:**

| Parameter | Default |
|-----------|---------|
| phs_veg | 0.8 |
| μ_veg | 0.01 day⁻¹ |
| phs_sto | 0.6 |

**Sources:** `src/ABM/CoffeeSteps.jl:6-15`
**Confidence:** `CODE_VERIFIED`

---

### Submodel: Reproductive Growth

**Purpose:** Allocate photosynthate during reproductive phase (default days 136–365).

**Logic:** Two regimes based on storage status:
- **Stressed (storage < 0):** If PhS covers production maintenance (Δprod > 0), remainder goes to veg/storage recovery. Otherwise, both veg and production decline.
- **Healthy (storage ≥ 0):** Allocation between veg and production based on relative sizes (frac_v = veg/(veg+prod)). If production declining: 95% of loss from storage, 5% actual loss. If growing: surplus to veg.

**Sources:** `src/ABM/CoffeeSteps.jl:17-55`
**Confidence:** `CODE_VERIFIED`
**Open Questions:** What biological rationale drives the 95/5% split when production declines?

---

### Submodel: Resource Commitment

**Purpose:** Initial fruit production commitment on transition day (day rep_d).

**Equation:** `production = max(0, Normal(res_commit, 0.01) × sunlight × veg × storage)`

**Parameters:** res_commit = 0.25
**Sources:** `src/ABM/MainStep.jl:69-74`
**Confidence:** `CODE_VERIFIED`

---

### Submodel: Plant Exhaustion and Regrowth

**Purpose:** Handle plants drained by parasitism.

**Exhaustion (storage < -10):** production=0, exh_countdown=731, all rust cleared, farm_map[pos]=0.
**Regrowth (countdown reaches 1):** veg=2.0, storage=75.5×exp(-5.5×sunlight)+2.2, countdown=0, farm_map[pos]=1.

**Sources:** `src/ABM/RustGrowth.jl:172-186` (parasitize!), `src/ABM/CoffeeSteps.jl:57-65` (regrow!)
**Confidence:** `CODE_VERIFIED`

---

### Submodel: Rust Lesion Growth (No Fungicide)

**Purpose:** Daily growth, aging, senescence, and sporulation of lesions.

**Steps:**
1. **Senescence:** Remove lesions with age > 150 days (McCain & Hennen 1984)
2. **Aging:** All ages += 1
3. **Temperature modifier:** `temp_mod = -(1/temp_ampl²) × (T - opt_T)² + 1.0 - 0.1 × sunlight`
4. **Sporulation:** `P(sporulate) = area × temp_mod × rain_spo × (1 + rep_spo × prod/(prod+veg))`
5. **Area growth:** `areas += areas × rust_gr × temp_mod × (1 + rep_gro × prod/max(storage,1)) × max(0, 1 - Σareas/25)`

**Calibrated Parameters:**

| Parameter | Value | Source |
|-----------|-------|--------|
| rust_gr | 0.0867 | ABC |
| opt_temp | 22.87 °C | ABC |
| temp_ampl | 5.0367 °C | ABC |
| rep_spo | 0.3926 | ABC |
| pdry_spo | 0.7078 | ABC |
| rep_gro | 0.01 | ABC |
| spore_pct | 0.3836 | ABC |

**Sources:** `src/ABM/RustGrowth.jl:4-35`
**Confidence:** `CODE_VERIFIED`

---

### Submodel: Rust Lesion Growth (With Fungicide)

**Same as above except:**
1. No UV inhibition in temp_mod
2. Per-lesion preventive/curative classification: `ages - fday < 15`
   - Preventive: fung_gro_prev=0.3, fung_spor_prev=0.0
   - Curative: fung_gro_cur=0.6, fung_spor_cur=0.5
3. First 15 days: areas ×= fung_reduce (0.95), remove lesions < 0.00005

**Sources:** `src/ABM/RustGrowth.jl:38-86`
**Confidence:** `CODE_VERIFIED`
**Open Questions:** Is the omission of UV inhibition (0.1×sunlight) from temp_mod in the fungicide version intentional?

---

### Submodel: Germination/Infection (Rain)

**Purpose:** Deposited spores attempt germination and tissue penetration.

**Per deposited spore:**
1. UV inhibition: P(killed) = sunlight × light_inh (0.5168)
2. Rain washoff: P(washed) = sunlight × rain_washoff (0.25)
3. If survived: infection_p = max_inf × (1 - 0.0137457×(T-21.5)²) × (0.05×(18+2×sunlight)-0.2) × host × fung
4. New lesion: age=0, area=0.00005, spores=false

**Sources:** `src/ABM/RustGrowth.jl:93-131`
**Confidence:** `CODE_VERIFIED`

---

### Submodel: Germination/Infection (No Rain)

**Same except:** No washoff. Wetness term uses 12 instead of 18 base hours.

**Sources:** `src/ABM/RustGrowth.jl:133-168`
**Confidence:** `CODE_VERIFIED`

---

### Submodel: Parasitism

**Equation:** `storage -= rust_paras (0.0155) × sum(areas)`

If storage < -10: plant exhausted.

**Sources:** `src/ABM/RustGrowth.jl:172-186`
**Confidence:** `CODE_VERIFIED`

---

### Submodel: End-of-Day Rust Update

**Equation:** `deposited = deposited × viab_loss (0.75) + newdeps; newdeps = 0`

If deposited < 0.05: clear. If no lesions and no deposits: rusted = false.

**Sources:** `src/ABM/RustGrowth.jl:190-204`
**Confidence:** `CODE_VERIFIED`

---

### Submodel: Rain Splash Dispersal

**Purpose:** Local stochastic spore dispersal via rain.

**Logic:**
- spore_area = Σ(sporulating areas) × spore_pct × (1 + sunlight)
- n_splashed ~ Poisson(spore_area)
- Per spore: distance ~ Exp(rain_dst) × d_mod, heading ~ Uniform(0,360°)
- d_mod = (4-4×diff_splash)×(sunlight-0.5)² + diff_splash (U-shaped: more splash at extremes of shade)
- Trajectory traced in 0.5-cell increments with probabilistic blocking by trees (tree_block=0.4889)

**Parameters:**

| Parameter | Value | Source |
|-----------|-------|--------|
| rain_dst | 2.2071 cells | ABC |
| diff_splash | 2.5 | Avelino et al. 2020, Gagliardi et al. 2021 |
| tree_block | 0.4889 | ABC |

**Sources:** `src/ABM/RustDispersal.jl:3-26` (disperse_rain!), `src/ABM/RustDispersal.jl:56-101` (splash trajectory)
**Confidence:** `CODE_VERIFIED`

---

### Submodel: Wind Dispersal

**Purpose:** Long-range directional spore dispersal via wind.

**Logic:**
- n_lifted ~ Poisson(spore_area × (1 - 0.5×shade_map[pos]))
- distance ~ Exp(wind_dst) × (1 + sunlight × diff_wind)
- heading = wind_h ± Uniform(0,30°) - 15°
- Shade blocks probabilistically: P = shade_map[cell] × shade_block
- Once blocked, deposits on next cell encountered (if coffee)

**Parameters:**

| Parameter | Value | Source |
|-----------|-------|--------|
| wind_dst | 5.7806 cells | ABC |
| diff_wind | 3.0 | Pezzopane |
| shade_block | 0.7898 | ABC |

**Sources:** `src/ABM/RustDispersal.jl:28-52` (disperse_wind!), `src/ABM/RustDispersal.jl:103-156` (gust trajectory)
**Confidence:** `CODE_VERIFIED`

---

### Submodel: Outside Spore Re-entry

**Purpose:** Spores that left the farm return on windy days.

Outpour pool (8 compass directions) decays ×0.9 daily. On windy days, spores re-enter from upwind edge directions based on wind heading, using gust() trajectory function.

**Sources:** `src/ABM/RustDispersal.jl:160-309`
**Confidence:** `CODE_VERIFIED`

---

### Submodel: Harvest

**Purpose:** Annual harvest, accounting, cycle reset.

**Logic:**
- Sum production across all plants
- Add fixed_costs + production × (other_costs × (1 - avg_shade × mean(shade_map)) + 0.012)
- Reset fung_count, shadeacc
- Per plant: production=0, deposited ×= lesion_survive (0.4338), remove oldest non-surviving lesions

**Sources:** `src/ABM/CGrowerSteps.jl:1-34`
**Confidence:** `CODE_VERIFIED`

---

### Submodel: Shade Pruning

**Logic:** If ind_shade > target: set to target. Else: ind_shade ×= 0.9. Add pruning costs.

Default schedule: days 74, 196. Targets: 0.3, 0.5.

**Sources:** `src/ABM/CGrowerSteps.jl:36-43`
**Confidence:** `CODE_VERIFIED`

---

### Submodel: Inspection

**Purpose:** Sample plants, detect and remove visible lesions.

**Logic:**
- Sample n_inspected plants (inspect_effort=1% × n_coffees)
- Per plant: count visible lesions (area > 0.05)
- Detection: if any visible AND (max area > 0.8 OR rand() < n_visible/5)
- Remove up to rm_lesions=2, weighted by visibility
- Update obs_incidence = n_infected / n_inspected

**Sources:** `src/ABM/CGrowerSteps.jl:45-75`
**Confidence:** `CODE_VERIFIED`

---

### Submodel: Fungicide Application

**Logic:** Set fungicide=1, increment fung_count, add costs. Active for 30 days (hardcoded). Max 3/year.

**Four strategies:** :cal (calendar), :incd (incidence-triggered), :cal_incd (hybrid), :flor (flowering-based at rep_d+60 and rep_d+105)

**Sources:** `src/ABM/CGrowerSteps.jl:79-83`, `src/ABM/MainStep.jl:186-201`
**Confidence:** `CODE_VERIFIED`
**Open Questions:** The `fung_effect` parameter (30) is defined but not used — the code hardcodes 30 in farmer_step!. Is this an oversight?
