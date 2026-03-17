# ODD Generation Plan for SpatialRust

## Instructions for the Drafting Agent

You are an ODD technical writer. Execute this plan to produce a complete ODD+2 protocol document for the SpatialRust model. Read these reference files before starting:

- `${CLAUDE_SKILL_DIR}/odd-protocol-ref.md` — ODD+2 protocol structure
- `${CLAUDE_SKILL_DIR}/odd-guidance-ref.md` — element guidance and checklists

Then read all source materials listed below and follow the section-by-section instructions.

**Critical writing directive:** Describe what the program does, not what you think the model does. Every equation, algorithm, and decision rule must come from the code. Do not simplify, abstract, or omit mechanisms that exist in the implementation.

Open the ODD document with: *"The model description follows the ODD (Overview, Design concepts, Details) protocol for describing individual- and agent-based models (Grimm et al. 2006), as updated by Grimm et al. (2020)."*

## Context

- **Model name:** SpatialRust — Coffee Leaf Rust Epidemic Model
- **Authors:** Manuela Vanegas Ferro
- **Model complexity:** Moderate-to-Complex (1 explicit agent type, ~17 submodels, 2D grid, extensive stochasticity)
- **ODD format:** Strict ODD+2 (Grimm et al. 2020)
- **Input quality:** Code + minimal docs (README only; all findings reverse-engineered from source code)
- **Delegation strategy:** Single agent — no sub-agents needed. All findings are CODE_VERIFIED and fully embedded in this plan.

## Source Materials

Read these files before drafting. They are the authoritative sources for all claims.

| File | What it contains | Priority |
|------|-----------------|----------|
| `src/ABM/MainSetup.jl` | Coffee agent definition, ALL parameter structs (Weather, CoffeePars, RustPars, MngPars, Books, Props), `init_spatialrust()` entry point, weather generation | **Essential** — read first |
| `src/ABM/MainStep.jl` | `step_model!` orchestration, `pre_step!`, `coffee_step!` phenology, `rust_step!` scheduling, `farmer_step!` | **Essential** |
| `src/ABM/RustGrowth.jl` | Rust lifecycle: growth ±fungicide, sporulation, germination ±rain, parasitism, senescence | **Essential** |
| `src/ABM/RustDispersal.jl` | Rain splash dispersal, wind dispersal, trajectory tracing, outside spore re-entry | **Essential** |
| `src/ABM/CoffeeSteps.jl` | Photosynthesis, vegetative growth, reproductive growth, regrowth | **Essential** |
| `src/ABM/CGrowerSteps.jl` | Harvest, pruning, inspection, fungicide application | **Essential** |
| `src/ABM/CreateABM.jl` | ABM initialization, tree placement, rust inoculation, 2-year warm-up | Important |
| `src/ABM/FarmMap.jl` | Farm layout generation: rows, shade placement, barriers | Important |
| `src/ABM/ShadeMap.jl` | Shade influence map generation | Important |
| `src/ABM/ShadeSteps.jl` | Shade logistic growth (single function) | Important |
| `src/SpatialRust.jl` | Module structure, type alias | Reference |
| `src/Runners.jl` | Simulation runners, data collection | Reference |
| `src/Metrics.jl` | Output metric definitions | Reference |

All source files are located relative to: `../spatialrust-model/`

## Terminology

Use these exact terms throughout the ODD. Never substitute alternatives.

| Term | Use This | NOT This | Definition in Model |
|------|----------|----------|---------------------|
| Coffee plant / coffee agent | coffee plant, coffee agent | tree, plant agent, individual | The single explicit agent type (`@agent Coffee GridAgent{2}`), representing one coffee plant at a fixed grid position |
| Shade tree | shade tree | shade agent, shade plant | Implicit entity represented as cells with value 2 in `farm_map`; no individual state variables |
| Farmer / grower | farmer, grower | manager, decision-maker | Implicit entity executing management actions at model level via scheduled rules |
| Lesion | lesion | infection, rust spot, pustule | A single rust infection site on a coffee plant, tracked by age, area, and sporulation status |
| Deposited spores | deposited spores | spore bank, inoculum pool | Ungerminated spores on a plant's leaf surface (`deposited` state variable) |
| New deposits | new deposits, newdeps | fresh spores, incoming spores | Spores arriving during the current time step (`newdeps`), merged into `deposited` at end of day |
| Sporulating | sporulating | spore-producing, infectious | A lesion whose `spores` flag is `true`, meaning it produces spores for dispersal |
| Exhausted plant | exhausted plant | dead plant, removed plant | A coffee plant with `storage < -10` due to parasitism; enters recovery countdown, removed from farm_map |
| ind_shade | ind_shade, individual shade level | shade intensity, canopy cover | Global dynamic shade level (0–max_shade) that scales the static shade_map to compute per-cell sunlight |
| shade_map | shade_map | shade grid, canopy map | Static matrix of shade influence (0–1) based on Euclidean distance to nearest shade tree |
| farm_map | farm_map | land use map, grid | Integer matrix: 0=empty, 1=coffee, 2=shade tree |
| Rain splash dispersal | rain splash dispersal | splash dispersal, rain dispersal | Local omnidirectional spore dispersal via rain droplet splash (`disperse_rain!`) |
| Wind dispersal | wind dispersal | aerial dispersal, long-range dispersal | Directional long-range spore dispersal via wind (`disperse_wind!`) |
| Outpour | outpour, outpour pool | spore reservoir, external pool | Vector tracking spores that exited the farm in 8 compass directions; decays daily, re-enters on windy days |
| Preventive fungicide | preventive fungicide effect | prophylactic | Fungicide effect on lesions younger than 15 days at time of application (stronger suppression) |
| Curative fungicide | curative fungicide effect | therapeutic | Fungicide effect on lesions older than 15 days at time of application (weaker suppression) |
| Vegetative phase | vegetative phase | growth phase | Days veg_d to rep_d-1 (default 1–134) when photosynthate is allocated to veg biomass and storage |
| Reproductive phase | reproductive phase | fruiting phase | Days rep_d+1 to 365 (default 136–365) when allocation shifts toward production |
| Resource commitment | resource commitment | fruit initiation | One-time event on day rep_d (default 135) when initial production is set from storage, veg, and sunlight |
| Warm-up | warm-up, pre-run | burn-in, spin-up | 730-day initialization period running only shade growth + coffee phenology + pruning, no rust or weather |
| ABC | ABC, Approximate Bayesian Computation | calibration, fitting | Method used to calibrate 14 rust parameters against empirical data (details not available from code) |

## Knowledge Base

### Element 1: Purpose and Patterns

**What to write:** A concise statement of the model's purpose followed by the patterns used to evaluate it. Do NOT describe the model here — only state what question it addresses and what criteria define success.

**Findings to encode:**

The model's purpose is to explore how the interaction between shade management, weather conditions (temperature, rainfall, wind), and fungicide strategies affects Coffee Leaf Rust (*Hemileia vastarix*) epidemic dynamics and coffee farm productivity. {INFERRED from code structure: shade has complex competing effects — reduces sunlight/photosynthesis, cools temperature, modifies rain splash distance, blocks wind dispersal — and the parameter sweep [source: scripts/ParameterRuns.jl:7-9] tests temperature × rain × wind combinations}

Higher-level purpose: explanation and prediction — understanding the mechanisms by which shade trees mediate rust epidemics and evaluating management strategies under different weather scenarios. {INFERRED}

Patterns the model should reproduce:
- Realistic disease progression: incidence and severity dynamics over annual cycles {INFERRED from output metrics [source: src/Metrics.jl:3-7]}
- Known rust biology encoded in the model:
  - ~5-month lesion senescence {CODE_VERIFIED [source: src/ABM/RustGrowth.jl:5, citing McCain & Hennen 1984]}
  - Rain wash-off of deposited spores {CODE_VERIFIED [source: src/ABM/MainSetup.jl:65, citing Avelino et al. 2020]}
  - Enhanced kinetic energy of rain splash under shade {CODE_VERIFIED [source: src/ABM/MainSetup.jl:77, citing Avelino et al. 2020 and Gagliardi et al. 2021]}
  - Deposited spore viability loss {CODE_VERIFIED [source: src/ABM/RustGrowth.jl:194, citing Nutman et al. 1963]}
  - 10% incidence threshold triggering management response {CODE_VERIFIED [source: src/ABM/MainSetup.jl:86, citing Cenicafé Boletín 36]}
- The model was calibrated using Approximate Bayesian Computation (ABC) — the `p_row` parameter is described as "parameter combination number (for ABC)" {CODE_VERIFIED [source: src/ABM/MainSetup.jl:37-38]}. Specific calibration data not available from code.

Output metrics: incidence (fraction of plants with active lesions), severity (mean total lesion area per plant), farm production (cumulative yield). {CODE_VERIFIED [source: src/Metrics.jl:3-7]}

**Writing instruction:** State the purpose in 2-3 sentences. List patterns as a numbered list. Note that calibration targets are not available from the source code. Include a note that the model's purpose was inferred from its structure and parameter sweep design, as no separate documentation was available.

---

### Element 2: Entities, State Variables, and Scales

**What to write:** Complete description of all entity types, their state variables (with types, units, ranges, initial values), and the model's spatial and temporal scales.

**Findings to encode:**

**Entity 1: Coffee plants** — the only explicit agent type, defined as `@agent Coffee GridAgent{2}` {CODE_VERIFIED [source: src/ABM/MainSetup.jl:4-20]}. Each occupies one cell of a 2D grid. State variables:

| State Variable | Type | Description | Units/Range | Initial Value |
|---|---|---|---|---|
| `sunlight` | Float64 | Fraction of light reaching plant: `1 - shade_map[pos] × ind_shade` | 0.0–1.0 | Computed from shade_map and ind_shade |
| `veg` | Float64 | Vegetative biomass | Dimensionless | 2.0 |
| `storage` | Float64 | Stored carbohydrates for growth and production | Dimensionless | `75.5 × exp(-5.5 × sunlight) + 2.2` |
| `production` | Float64 | Reproductive/fruit biomass | Dimensionless | 0.0 |
| `exh_countdown` | Int | Days remaining in exhaustion recovery | 0 (active) or 731 (exhausted) | 0 |
| `rust_gr` | Float64 | Individual rust growth rate | day⁻¹ | `Truncated(Normal(0.0867, 0.000867), 0, 0.13)` |
| `rusted` | Bool | Whether plant has any rust (deposited or lesions) | true/false | false |
| `newdeps` | Float64 | Newly deposited spores arriving this time step | Spore count | 0.0 |
| `deposited` | Float64 | Accumulated deposited (ungerminated) spores | Spore count | 0.0 |
| `n_lesions` | Int | Number of active lesions | 0–max_lesions (25) | 0 |
| `ages` | Vector{Int} | Age of each lesion | Days | Empty |
| `areas` | Vector{Float64} | Area of each lesion | Dimensionless, soft cap ~25.0 | Empty |
| `spores` | Vector{Bool} | Whether each lesion is sporulating | true/false | Empty |

{CODE_VERIFIED [source: src/ABM/MainSetup.jl:4-20 for definition, src/ABM/MainSetup.jl:22-31 for constructor]}

**Entity 2: Shade trees** — implicit. Represented as cells with value 2 in `farm_map`. No individual state variables. Their spatial influence is captured by the static `shade_map` (distance-based, computed at initialization) scaled by the dynamic global `ind_shade` level. {CODE_VERIFIED [source: src/ABM/ShadeMap.jl:9-33]}

**Entity 3: Farmer/Grower** — implicit. No state variables. Behavior is encoded as model-level actions driven by `MngPars` (management schedule parameters) and `Books` (runtime tracking). {CODE_VERIFIED [source: src/ABM/MainStep.jl:170-206]}

**Entity 4: Environment** — global. Consists of the `Weather` struct holding pre-generated daily time series: `rain_data` (Vector{Bool}), `wind_data` (Vector{Bool}), `temp_data` (Vector{Float64}, °C). {CODE_VERIFIED [source: src/ABM/MainSetup.jl:232-236]}

**Model-level tracking** (`Books` mutable struct) {CODE_VERIFIED [source: src/ABM/MainSetup.jl:319-335]}:

| Variable | Type | Description |
|---|---|---|
| `days` | Int | Day counter (annual cycle via mod 365) |
| `ticks` | Int | Simulation step counter (starts at 0, incremented before first use) |
| `ind_shade` | Float64 | Current global shade level |
| `temperature` | Float64 | Current day's temperature (°C) |
| `rain` | Bool | Current day's rain status |
| `wind` | Bool | Current day's wind status |
| `wind_h` | Float64 | Wind heading (0–360°), randomized each windy day |
| `fungicide` | Int | Days since last fungicide application (0 = none active, 1–30 = active) |
| `fung_count` | Int | Fungicide applications this year |
| `obs_incidence` | Float64 | Observed incidence from last inspection |
| `costs` | Float64 | Cumulative costs |
| `prod` | Float64 | Cumulative production |
| `inbusiness` | Bool | Simulation validity flag |
| `withinbounds` | Bool | Numerical stability flag |
| `shadeacc` | Float64 | Accumulated daily shade for annual averaging |

**Scales:**
- **Spatial:** 2D grid, default 100×100 cells, non-periodic boundaries, Chebyshev metric. Each cell holds at most one entity (coffee plant, shade tree, or empty). No explicit physical scale (e.g., meters per cell) is defined in the code. {CODE_VERIFIED [source: src/ABM/CreateABM.jl:4]}
- **Temporal:** 1 time step = 1 day. Typical simulation: 730 steps (2 years). Annual production cycle of 365 days. {CODE_VERIFIED [source: scripts/ParameterRuns.jl:11-12]}
- **Warm-up:** 2 years (730 days) before the simulation proper. {CODE_VERIFIED [source: src/ABM/CreateABM.jl:112-200]}

**Writing instruction:** Present entity types in order. Use a table for Coffee state variables. Describe shade trees and farmer as implicit entities with a short paragraph each. Include the Books variables as a "model-level state variables" subsection. State spatial and temporal scales explicitly. Note the absence of explicit physical spatial scale.

---

### Element 3: Process Overview and Scheduling

**What to write:** The exact sequence of processes executed each time step, the order of agent execution within each process, and any conditional or periodic processes.

**Findings to encode:**

Each time step (1 day), `step_model!` {CODE_VERIFIED [source: src/ABM/MainStep.jl:3-9]} executes five processes in **fixed sequential order**:

1. **pre_step!** {CODE_VERIFIED [source: src/ABM/MainStep.jl:14-35]}
   - Increment `days` and `ticks` counters
   - Read today's weather from pre-generated vectors indexed by `ticks`
   - Decay outpour spore pool: `outpour *= 0.9`
   - If windy: randomize wind heading `wind_h ~ Uniform(0, 360°)`

2. **grow_shades!** {CODE_VERIFIED [source: src/ABM/ShadeSteps.jl:1-3]}
   - Logistic growth: `ind_shade += rate × (1 - ind_shade/max_shade) × ind_shade`

3. **coffee_step!** {CODE_VERIFIED [source: src/ABM/MainStep.jl:37-84]}
   - All agents iterated in deterministic storage order (column-major from farm_map)
   - Phenological phase determined by `mod1(days, 365)`:
     - `veg_d ≤ day < rep_d` (default days 1–134): vegetative growth
     - `day == rep_d` (default day 135): vegetative growth + resource commitment
     - Otherwise (default days 136–365): reproductive growth
   - Handles wrap-around when `veg_d > rep_d` via sign negation of all three values
   - Exhausted plants: if `exh_countdown > 1`, decrement; if `== 1`, regrow

4. **rust_step!** {CODE_VERIFIED [source: src/ABM/MainStep.jl:86-164]}
   - Function variants selected from 8 combinations of {fungicide active?, rain?, wind?}
   - **Loop 1** — for each agent where `rusted == true` (lazy `Iterators.filter`):
     - **Disperse first**: if any lesion sporulating, compute effective spore area, then rain splash dispersal, then wind dispersal
     - Compute local temperature: `temp - temp_cooling × (1 - sunlight)`
     - If `n_lesions > 0`: grow/sporulate → check numerical bounds → germinate deposited spores → parasitize
     - If `n_lesions == 0`: germinate only
   - **Loop 2** — lazy filter **re-evaluated** (includes plants newly marked `rusted = true` by dispersal in Loop 1):
     - `update_rust!`: `deposited = deposited × viab_loss + newdeps; newdeps = 0`
   - If windy: `outside_spores!` — spores from outpour pool re-enter farm from upwind edge
   - **Key design**: `newdeps`/`deposited` separation prevents spores deposited this step from germinating until the next step

5. **farmer_step!** {CODE_VERIFIED [source: src/ABM/MainStep.jl:170-206]}
   - If harvest day (`mod1(days, 365) == harvest_day`): full harvest cycle
   - If prune day (matches `prune_sch`): prune shades
   - If inspection day (`days % inspect_period == 0`): inspect plants
   - Fungicide logic:
     - If active (`fungicide > 0`): increment counter; expire when > 30 (hardcoded)
     - If inactive, not harvest day, under max sprayings:
       - `:cal` → spray on scheduled days
       - `:cal_incd` → spray on schedule OR if `obs_incidence > incidence_thresh`
       - `:incd` → spray only when `obs_incidence > incidence_thresh`
       - `:flor` → spray at `rep_d + 60` and `rep_d + 105`
   - Accumulate `shadeacc += ind_shade`

**Warm-up** (`pre_run365!`) {CODE_VERIFIED [source: src/ABM/CreateABM.jl:112-200]}:
- 730 days (2 years) before simulation proper
- Runs ONLY: `grow_shades!` + `coffee_step!` + explicit `prune_shades!` at scheduled days
- Does NOT run: `pre_step!` (no weather read, no tick increment), `rust_step!`, `farmer_step!`
- Simplified harvest at days 365 and 730: only `production = 0.0` per plant (no lesion survival, no cost accounting)
- After completion: resets `days = 0`, `costs = 0`

**Agent execution order:** Deterministic. Agents stored in a `Vector{Coffee}` indexed by ID, added in column-major order from `findall(x -> x == 1, farm_map)`. The model does NOT use the Agents.jl built-in scheduler; `step_model!` manually iterates `model.agents`. {CODE_VERIFIED [source: src/ABM/CreateABM.jl:17-38, src/SpatialRust.jl:8-11]}

**Writing instruction:** Present the daily schedule as a numbered list with sub-steps. Include a scheduling diagram or pseudocode block showing the step_model! structure. Explicitly state the agent iteration order and the lazy filter re-evaluation mechanism in the rust step. Describe the warm-up as a separate subsection. This element was corrected after modeler review — ensure all details from the corrected version are included.

---

### Element 4: Design Concepts

**What to write:** Address each of the 11 design concepts. For concepts that do not apply, state "Not applicable" with brief justification.

#### 4.1 Basic principles

The model integrates three theoretical domains: (1) plant physiology — carbon assimilation via Michaelis-Menten (Monod) kinetics with allocation tradeoffs between vegetative growth, storage, and reproduction; (2) epidemiology — complete rust lifecycle including spore deposition, germination/infection (temperature and moisture dependent), lesion growth, sporulation, dispersal, parasitism, and senescence; (3) agroecology — shade trees as mediators of microclimate (temperature cooling), light availability (photosynthesis), and physical barriers (blocking wind dispersal, modifying rain splash kinetics). An ABM was chosen because spatial heterogeneity in shade cover, dispersal directionality (wind heading), individual plant state (storage depletion leading to exhaustion), and farm layout geometry all require individual-level representation. {INFERRED from code structure [source: src/ABM/CoffeeSteps.jl:6-14, src/ABM/RustGrowth.jl:1-35, src/ABM/RustDispersal.jl:1-52]}

**Writing instruction:** State the three theoretical domains and justify the ABM approach. Note that no explicit theoretical grounding is documented; the basis is inferred from the code.

#### 4.2 Emergence

Emergent results: epidemic dynamics (incidence and severity curves over time), spatial patterns of infection (driven by dispersal mechanics and farm layout), and production losses (from parasitism draining individual plant storage). Imposed (not emergent): weather (pre-generated time series), farm layout (fixed at initialization), management schedules (calendar-based pruning and fungicide timing). {INFERRED [source: src/Metrics.jl:3-7, src/ABM/MainSetup.jl:232-236]}

#### 4.3 Adaptation

Coffee plants do not adapt their behavior. The farmer shows limited reactive adaptation through incidence-triggered fungicide strategies: under `:incd` and `:cal_incd` strategies, the farmer sprays fungicide when observed incidence exceeds the 10% threshold (from Cenicafé Boletín 36). This is reactive (respond to observed conditions), not optimizing. {CODE_VERIFIED [source: src/ABM/MainStep.jl:186-201, src/ABM/MainSetup.jl:86]}

#### 4.4 Objectives

Not applicable. No agents pursue explicit objectives or attempt to optimize any measure. The farmer follows rule-based management strategies without utility maximization. {CODE_VERIFIED}

#### 4.5 Learning

Not applicable. No agents change their decision rules or adaptive traits over time based on experience. (Commented-out code references a genetic algorithm for strategy evolution, suggesting this was explored but is not active.) {CODE_VERIFIED [source: src/ABM/MainStep.jl:31-33]}

#### 4.6 Prediction

Not applicable. No agents estimate future conditions when making decisions. Fungicide application is either calendar-based or reactive to current observed incidence. {CODE_VERIFIED}

#### 4.7 Sensing

- Coffee plants sense their local sunlight level, computed from their position-specific shade_map value scaled by the current global ind_shade level. {CODE_VERIFIED [source: src/ABM/CoffeeSteps.jl:2-3]}
- Rust processes are implicitly affected by local temperature (reduced by shade cooling) and current weather (rain, wind). This is not modeled as deliberate sensing. {CODE_VERIFIED [source: src/ABM/MainStep.jl:141]}
- The farmer senses disease through periodic inspection: every `inspect_period` (default 7) days, a sample of `inspect_effort` (default 1%) of plants is inspected. Detection is imperfect — only lesions with area > 0.05 are visible, and detection probability depends on lesion size (area > 0.8 guarantees detection; otherwise `P = n_visible / 5`). {CODE_VERIFIED [source: src/ABM/CGrowerSteps.jl:45-75]}

#### 4.8 Interaction

- **Rust-mediated indirect interaction:** Infected plants disperse spores to nearby plants via rain splash (local, mean distance ~2.2 cells, omnidirectional) and wind (longer range, mean ~5.8 cells, directional based on wind heading). {CODE_VERIFIED [source: src/ABM/RustDispersal.jl:3-52]}
- **Shade-mediated indirect interaction:** Shade trees affect all coffee plants within their influence radius (shade_r=3) via the shade_map, which is static spatially but scaled by the dynamic global ind_shade. {CODE_VERIFIED [source: src/ABM/ShadeMap.jl:9-33]}
- **No direct agent-agent interaction:** Coffee plants do not communicate or compete. No density dependence between plants. {CODE_VERIFIED}
- **Farm boundary interaction:** Spores exiting the farm boundary enter the outpour pool (8 compass directions, decaying ×0.9 daily). On windy days, spores re-enter from upwind edges. {CODE_VERIFIED [source: src/ABM/RustDispersal.jl:160-267]}

#### 4.9 Stochasticity

Stochasticity serves three purposes in the model:

1. **Natural variability in biological processes:** Weather generation (Bernoulli for rain/wind, Normal for temperature), dispersal distances (Exponential), dispersal directions (Uniform), spore counts (Poisson), germination/infection (Bernoulli with computed probability), sporulation (Bernoulli), UV inhibition (Bernoulli), rain wash-off (Bernoulli), tree/shade blocking of dispersal (Bernoulli).

2. **Individual heterogeneity:** Per-plant rust growth rate (`rust_gr ~ Truncated(Normal(0.0867, 0.000867), 0, 0.13)`), resource commitment on transition day (`Normal(res_commit, 0.01)`), management costs (each cost drawn from `Normal(cost, 0.05 × cost)`).

3. **Imperfect monitoring:** Farmer inspection samples a random subset of plants, and lesion detection is probabilistic based on visibility.

{CODE_VERIFIED — distributions documented per submodel in Element 7}

**Writing instruction:** Present as a table mapping each stochastic process to its distribution, purpose, and the submodel where it occurs.

#### 4.10 Collectives

Not applicable. No formal collectives (groups with emergent behaviors or state). There is implicit spatial clustering from the initial rust introduction mechanism (when `ini_rusts ≥ 1`, clusters of radius 1 around random centers are created) and from local dispersal creating infection neighborhoods. {CODE_VERIFIED [source: src/ABM/CreateABM.jl:43-50]}

#### 4.11 Observation

Three metrics are collected from simulations {CODE_VERIFIED [source: src/Metrics.jl:3-7]}:

| Metric | Definition | Formula |
|--------|-----------|---------|
| Incidence | Fraction of plants with active lesions | `mean(n_lesions > 0)` over all agents |
| Severity | Mean total lesion area per plant | `mean(sum(areas))` over all agents |
| Production | Mean production biomass per plant | `mean(production)` over all agents |

In single-run mode: all three collected every step → daily time series. {CODE_VERIFIED [source: src/Runners.jl:12-28]}
In parameter sweep mode: final incidence and severity (just before last harvest), cumulative farm production ÷ 1000. {CODE_VERIFIED [source: src/Runners.jl:51-69]}

Additionally tracked internally but not directly output: `costs`, `obs_incidence`, `shadeacc`, `outpour` (8-element vector), `withinbounds`, `inbusiness`. {CODE_VERIFIED [source: src/ABM/MainSetup.jl:319-335]}

---

### Element 5: Initialization

**What to write:** Complete description of the model state at t=0 (after warm-up), including all initial values, spatial configuration, and variation between runs.

**Findings to encode:**

**Farm layout generation** {CODE_VERIFIED [source: src/ABM/FarmMap.jl:2-65, src/ABM/MainSetup.jl:140-151]}:
Three options controlled by `common_map`:
- `:none` (default): Custom layout — coffee rows every `row_d` (2) cells, plants every `plant_d` (1) cell, shade trees in regular (`shade_d=6` spacing) or random pattern, optional barrier rows (`barrier_rows=2`, `barriers=(1,0)` = one internal barrier set, no edge barriers)
- `:fullsun`: Coffee only, no shade trees
- `:regshaded`: Coffee + regularly spaced shade trees

Farm map: `Int` matrix (0=empty, 1=coffee, 2=shade tree), default 100×100. A user-supplied farm_map can replace the generated layout.

**Shade map** {CODE_VERIFIED [source: src/ABM/ShadeMap.jl:9-33]}: Static matrix computed once at initialization. For each cell, the shade value equals `1.0 - min_euclidean_distance_to_shade_tree / (2 × shade_r)²` if any shade tree is within `shade_r=3` radius. Shade tree cells = 1.0. Full-sun maps have all zeros.

**Coffee agents** {CODE_VERIFIED [source: src/ABM/CreateABM.jl:17-38]}: One agent placed at each cell where `farm_map == 1`, in column-major order. Initial values: `veg=2.0`, `storage = 75.5 × exp(-5.5 × sunlight) + 2.2` (inverse relationship: shadier plants start with more storage), `production=0.0`, `rust_gr ~ Truncated(Normal(0.0867, 0.000867), 0, 0.13)`, all rust state zeroed.

**2-year warm-up** {CODE_VERIFIED [source: src/ABM/CreateABM.jl:112-200]}:
- 730 days of `grow_shades!` + `coffee_step!` + scheduled `prune_shades!`
- No weather, no rust, no full farmer actions
- Simplified harvest at days 365 and 730 (only `production = 0.0`)
- After completion: `days = 0`, `costs = 0`
- Purpose: establish quasi-equilibrium shade and coffee biomass states before rust introduction

**Rust inoculation** (post warm-up) {CODE_VERIFIED [source: src/ABM/CreateABM.jl:52-110]}:
Controlled by `p_rusts` (default 0.01):
- `0 < p_rusts < 1`: Randomly infect `round(p_rusts × n_agents)` plants (scattered, uniform random)
- `p_rusts == 1`: One spatial cluster (radius 1, Chebyshev, around a random center excluding margins)
- `p_rusts > 1`: Multiple clusters (one per integer value of p_rusts)
- Each inoculated plant receives `1 + Binomial(max_lesions-1, 0.05)` lesions. Each lesion assigned random area in [0, 0.3]:
  - area < 0.01: counted as deposited spore only
  - 0.01 ≤ area < 0.28: non-sporulating lesion
  - area ≥ 0.28: sporulating lesion

**Weather** {CODE_VERIFIED [source: src/ABM/MainSetup.jl:352-383]}: Pre-generated for the entire simulation length. If external data provided: noise added (5% bit-flip for booleans, Normal(0,0.05) for temperature). If not provided: generated from probability parameters.

**Variation between runs:** Weather, individual `rust_gr`, initial rust placement, and management cost noise are all stochastic. Setting `seed > 0` in `init_spatialrust` makes runs fully reproducible via `Xoshiro(seed)`.

**Writing instruction:** Describe initialization in the order: farm layout → shade map → coffee agents → warm-up → rust inoculation → weather. Explicitly state all default parameter values. Note that initialization varies between runs due to stochasticity, and that seed control is available.

---

### Element 6: Input Data

**What to write:** Description of any external data that drives the model during simulation.

**Findings to encode:**

The model can accept three external time-varying input vectors {CODE_VERIFIED [source: src/ABM/MainSetup.jl:45-47]}:
- `rain_data::Vector{Bool}` — daily rain occurrence
- `wind_data::Vector{Bool}` — daily wind occurrence
- `temp_data::Vector{Float64}` — daily mean temperature (°C)

If provided, noise is still added: 5% bit-flip for boolean vectors, `Normal(0, 0.05)` for temperature {CODE_VERIFIED [source: src/ABM/MainSetup.jl:362-383]}. If empty (default), weather is generated stochastically from probability parameters (`rain_prob`, `wind_prob`, `mean_temp`).

A user-supplied `farm_map::Array{Int}` can also replace the generated farm layout {CODE_VERIFIED [source: src/ABM/MainSetup.jl:107]}.

No other external data changes during simulation. All management schedules, costs, and biological parameters are fixed at initialization.

**Writing instruction:** Brief section. State what data can be provided, how it's processed (noise), and that the default is stochastic generation. Note the farm_map option.

---

### Element 7: Submodels

**What to write:** For each submodel, provide: purpose, complete logic (equations/pseudocode), all parameters with values/units/sources, and any edge cases or boundary conditions. The reimplementability standard applies: a reader must be able to code each submodel from this description alone.

**Writing instruction for all submodels:** Use numbered equations where possible. Present parameters in tables with columns: Symbol, Value, Units, Description, Source. Use pseudocode for complex branching logic. Cite the source file and line numbers.

#### Submodel 7.1: Weather Generation

**Purpose:** Generate or process daily weather time series for the full simulation.

**Logic:**
```
IF rain_data provided:
  Apply 5% bit-flip noise: for each day, P(flip) = 0.05
ELSE:
  rain[t] ~ Bernoulli(rain_prob)

IF wind_data provided:
  Apply 5% bit-flip noise
ELSE:
  IF rain[t]:
    wind[t] ~ Bernoulli(wind_prob - 0.1)
  ELSE:
    wind[t] ~ Bernoulli(wind_prob + 0.1)

IF temp_data provided:
  temp[t] = temp_data[t] + Normal(0, 0.05)
ELSE:
  temp[t] ~ Normal(mean_temp, 0.5)
```

{CODE_VERIFIED [source: src/ABM/MainSetup.jl:352-383]}

Note: Wind probability is conditionally adjusted — lower when raining (0.1 reduction), higher when dry (0.1 increase). This creates negative correlation between simultaneous rain+wind events.

| Parameter | Default | Units | Description | Source |
|-----------|---------|-------|-------------|--------|
| `rain_prob` | 0.8 | probability | Daily rain probability | Model parameter |
| `wind_prob` | 0.7 | probability | Base daily wind probability | Model parameter |
| `mean_temp` | 22.0 | °C | Mean daily temperature | Model parameter |

---

#### Submodel 7.2: Shade Tree Growth

**Purpose:** Model shade canopy recovery between pruning events.

**Equation:**

```
Δ(ind_shade) = shade_g_rate × (1 - ind_shade / max_shade) × ind_shade
```

This is discrete logistic growth applied to the single global `ind_shade` variable. Growth is density-dependent, approaching `max_shade` asymptotically.

{CODE_VERIFIED [source: src/ABM/ShadeSteps.jl:1-3]}

| Parameter | Default | Units | Description | Source |
|-----------|---------|-------|-------------|--------|
| `shade_g_rate` | 0.015 | day⁻¹ | Shade growth rate | Model parameter |
| `max_shade` | 0.8 | dimensionless | Maximum shade level | Model parameter |

---

#### Submodel 7.3: Coffee Photosynthesis

**Purpose:** Compute daily gross carbon assimilation for a coffee plant. Shared by vegetative and reproductive growth submodels.

**Equations:**

```
photo_veg = veg × photo_frac

PhS = photo_const × (sunlight / (k_sl + sunlight)) × (photo_veg / (k_v + photo_veg))
```

Where `photo_const = f_avail × phs_max` (precomputed at initialization).

This is a product of two Michaelis-Menten (Monod) saturating response functions: one for light availability and one for photosynthetic tissue quantity. Output `PhS` represents the daily photosynthetic rate.

{CODE_VERIFIED [source: src/ABM/CoffeeSteps.jl:6-8]}

| Parameter | Default | Units | Description | Source |
|-----------|---------|-------|-------------|--------|
| `f_avail` | 0.5 | fraction | Fraction of daily assimilates available for allocation | Model parameter |
| `phs_max` | 0.2 | day⁻¹ | Maximum assimilation rate | Model parameter |
| `k_sl` | 0.05 | dimensionless | Half-saturation constant for sunlight | Model parameter |
| `k_v` | 0.2 | dimensionless | Half-saturation constant for photosynthetic tissue | Model parameter |
| `photo_frac` | 0.2 | fraction | Fraction of vegetative tissue that is photosynthetic | Model parameter |

---

#### Submodel 7.4: Vegetative Growth

**Purpose:** Allocate photosynthate to vegetative biomass and storage during the vegetative phase (default days 1–134).

**Equations:**

```
veg ← veg + phs_veg × PhS - μ_veg × veg
storage ← storage + phs_sto × PhS
veg ← max(veg, 0)
```

{CODE_VERIFIED [source: src/ABM/CoffeeSteps.jl:6-15]}

| Parameter | Default | Units | Description | Source |
|-----------|---------|-------|-------------|--------|
| `phs_veg` | 0.8 | fraction | Fraction of photosynthate converted to vegetative biomass | Model parameter |
| `μ_veg` | 0.01 | day⁻¹ | Rate of vegetative biomass loss | Model parameter |
| `phs_sto` | 0.6 | fraction | Fraction of photosynthate converted to storage | Model parameter |

Note: `phs_veg + phs_sto = 1.4 > 1.0` because they represent conversion efficiencies of different allocation streams, not fractions of the same pool.

---

#### Submodel 7.5: Reproductive Growth

**Purpose:** Allocate photosynthate during the reproductive phase (default days 136–365), balancing vegetative maintenance, fruit production growth, and storage.

**Logic:** Two regimes based on storage status:

```
frac_v = veg / (veg + production)

IF storage < 0 (stressed plant):
  Δprod = PhS - μ_prod × production
  IF Δprod > 0:
    remainder = Δprod - μ_veg × veg
    veg ← veg + min(remainder, 0)        # veg can only decline or stay
    storage ← storage + max(remainder, 0) # surplus recovers storage
  ELSE:
    veg ← veg - μ_veg × veg
    production ← production + Δprod       # production declines

ELSE (storage ≥ 0, healthy plant):
  d_prod = (1 - frac_v) × PhS - μ_prod × production
  IF d_prod < 0 (production declining):
    veg ← veg + phs_veg × frac_v × PhS - μ_veg × veg
    storage ← storage + 0.95 × d_prod    # 95% of loss absorbed by storage
    production ← production + 0.05 × d_prod  # 5% actual production loss
  ELSE (d_prod ≥ 0, production growing):
    veg ← veg + phs_veg × (d_prod + frac_v × PhS) - μ_veg × veg

veg ← max(veg, 0)
production ← max(production, 0)
```

{CODE_VERIFIED [source: src/ABM/CoffeeSteps.jl:17-55]}

| Parameter | Default | Units | Description | Source |
|-----------|---------|-------|-------------|--------|
| `μ_prod` | 0.01 | day⁻¹ | Rate of production biomass loss | Model parameter |

---

#### Submodel 7.6: Resource Commitment

**Purpose:** Set initial fruit production on the transition day (day `rep_d`, default 135).

**Equation:**

```
production ← max(0, Normal(res_commit, 0.01) × sunlight × veg × storage)
```

Combines sunlight availability, vegetative capacity, and stored energy to determine initial fruit production, with individual stochastic variation.

{CODE_VERIFIED [source: src/ABM/MainStep.jl:69-74]}

| Parameter | Default | Units | Description | Source |
|-----------|---------|-------|-------------|--------|
| `res_commit` | 0.25 | dimensionless | Scaling factor for resource commitment | Model parameter |
| `veg_d` | 1 | day of year | Start of vegetative growth | Model parameter |
| `rep_d` | 135 | day of year | Start of reproductive growth / transition day | Model parameter |

---

#### Submodel 7.7: Plant Exhaustion and Regrowth

**Purpose:** Handle coffee plants drained below viability by rust parasitism.

**Exhaustion** (triggered when `storage < -10` in parasitism):
```
production ← 0
exh_countdown ← exh_countdown (default 731 days, ~2 years)
Clear all rust: newdeps=0, deposited=0, n_lesions=0, empty ages/areas/spores
farm_map[pos] ← 0  (plant removed from map — no longer a dispersal target)
```

**Regrowth** (when `exh_countdown` reaches 1 during coffee_step!):
```
veg ← 2.0
storage ← 75.5 × exp(-5.5 × sunlight) + 2.2
exh_countdown ← 0
farm_map[pos] ← 1  (plant restored to map)
```

{CODE_VERIFIED [source: src/ABM/RustGrowth.jl:172-186 for exhaustion, src/ABM/CoffeeSteps.jl:57-65 for regrowth]}

| Parameter | Default | Units | Description | Source |
|-----------|---------|-------|-------------|--------|
| `exh_countdown` | 731 | days | Recovery time after exhaustion (~2 years) | Model parameter |

---

#### Submodel 7.8: Rust Lesion Growth (No Fungicide)

**Purpose:** Daily aging, senescence, growth, and sporulation of rust lesions when no fungicide is active.

**Step-by-step logic:**

```
STEP 1 — Senescence:
  Remove all lesions with age > 150 days
  (Reference: ~5-month lifespan, McCain & Hennen 1984)

STEP 2 — Aging:
  All lesion ages ← ages + 1

STEP 3 — Temperature modifier:
  temp_ampl_c = -(1 / temp_ampl²)    # precomputed: -(1/5.0367²) ≈ -0.03941
  temp_mod = temp_ampl_c × (local_temp - opt_temp)² + 1.0 - 0.1 × sunlight
  IF temp_mod ≤ 0: SKIP growth and sporulation (too cold/hot or too sunny)

STEP 4 — Sporulation (non-sporulating lesions only):
  spor_mod = temp_mod × rain_spo × (1 + rep_spo × production/(production + veg))
  FOR each lesion where spores[i] == false:
    P(sporulate) = areas[i] × spor_mod
    spores[i] ← Bernoulli(P(sporulate))

STEP 5 — Area growth:
  host_gro = 1 + rep_gro × production / max(storage, 1.0)
  growth_mod = rust_gr × temp_mod × host_gro
  area_gro = max(0, 1 - sum(areas) / 25.0)    # density-dependent cap
  areas ← areas + areas × (growth_mod × area_gro)
```

Where `rain_spo = 1.0` if rain, `pdry_spo` if dry. `local_temp = temperature - temp_cooling × (1 - sunlight)`.

Note: The `- 0.1 × sunlight` term in `temp_mod` represents UV inhibition of rust growth under direct sunlight. This term is ABSENT from the fungicide version (submodel 7.9).

{CODE_VERIFIED [source: src/ABM/RustGrowth.jl:4-35]}

| Parameter | Value | Units | Description | Source |
|-----------|-------|-------|-------------|--------|
| `rust_gr` | 0.0867 | day⁻¹ | Base rust area growth rate (per-plant, drawn from truncated Normal) | ABC calibration |
| `opt_temp` | 22.8716 | °C | Optimal temperature for rust growth | ABC calibration |
| `temp_ampl` | 5.0367 | °C | Temperature amplitude (determines growth range) | ABC calibration |
| `temp_cooling` | 1.0665 | °C | Temperature reduction per unit of shade effect | ABC calibration |
| `rep_spo` | 0.3926 | dimensionless | Effect of reproductive growth stage on sporulation | ABC calibration |
| `pdry_spo` | 0.7078 | probability | Probability modifier for sporulation without rain | ABC calibration |
| `rep_gro` | 0.01 | dimensionless | Effect of reproductive growth on rust area growth | ABC calibration |
| `spore_pct` | 0.3836 | fraction | Fraction of lesion area that produces spores | ABC calibration |

---

#### Submodel 7.9: Rust Lesion Growth (With Fungicide)

**Purpose:** Same as 7.8 but with fungicide effects modifying growth and sporulation.

**Differences from 7.8:**

1. **No UV inhibition in temperature modifier**: `temp_mod = temp_ampl_c × (local_temp - opt_temp)² + 1.0` (no `- 0.1 × sunlight` term)

2. **Per-lesion preventive/curative classification:**
   ```
   prev_cur[i] = (ages[i] - fday) < 15
   # If true: lesion was < 15 days old at time of spraying → preventive
   # If false: lesion was ≥ 15 days old → curative
   ```
   Where `fday` = current fungicide counter (days since last application).

3. **Modified sporulation probabilities:**
   ```
   spor_probs[i] = spor_mod × IF prev_cur[i] THEN fung_spor_prev ELSE fung_spor_cur
   ```

4. **Modified growth rates:**
   ```
   gro_mods[i] = growth_mod × IF prev_cur[i] THEN fung_gro_prev ELSE fung_gro_cur
   areas ← areas + areas × (gro_mods × area_gro)
   ```

5. **Active area reduction (first 15 days of application):**
   ```
   IF fday < 15:
     areas ← areas × fung_reduce    # 5% daily reduction
     Remove lesions where area < 0.00005
   ```

{CODE_VERIFIED [source: src/ABM/RustGrowth.jl:38-86]}

| Parameter | Value | Units | Description | Source |
|-----------|-------|-------|-------------|--------|
| `fung_inf` | 0.9 | modifier | Infection probability modifier under fungicide | Model parameter |
| `fung_gro_prev` | 0.3 | modifier | Growth rate modifier (preventive: 70% reduction) | Model parameter |
| `fung_gro_cur` | 0.6 | modifier | Growth rate modifier (curative: 40% reduction) | Model parameter |
| `fung_spor_prev` | 0.0 | modifier | Sporulation modifier (preventive: complete suppression) | Model parameter |
| `fung_spor_cur` | 0.5 | modifier | Sporulation modifier (curative: 50% reduction) | Model parameter |
| `fung_reduce` | 0.95 | rate | Daily area reduction factor during active period | Model parameter |

---

#### Submodel 7.10: Germination and Infection (Rain)

**Purpose:** Deposited spores attempt to germinate and penetrate leaf tissue on rainy days.

**Per deposited spore** (iterating while `sp ≤ deposited`):
```
1. UV inhibition check:
   IF Bernoulli(sunlight × light_inh):
     deposited ← deposited - 1  (spore killed)
     CONTINUE

2. Rain wash-off check:
   IF Bernoulli(sunlight × rain_washoff):
     deposited ← deposited - 1  (spore washed away)
     CONTINUE

3. Infection attempt (if n_lesions < max_lesions):
   temp_inf_p = 1 - 0.0137457 × (local_temp - 21.5)²
   wet_inf_p = 0.05 × (18.0 + 2.0 × sunlight) - 0.2
   host = 1 - rep_inf + rep_inf × production / (production + veg)
   infection_p = max_inf × temp_inf_p × wet_inf_p × host × fung_modifier

   IF Bernoulli(infection_p):
     deposited ← deposited - 1
     n_lesions ← n_lesions + 1
     Push: age=0, area=0.00005, spores=false
```

Where `fung_modifier = 1.0` (no fungicide) or `fung_inf = 0.9` (fungicide active).

Note: The `wet_inf_p` term represents leaf wetness duration in hours: base 18 hours with rain + 2× sunlight effect. The coefficient 0.0137457 converts the temperature term to a probability centered at 21.5°C.

{CODE_VERIFIED [source: src/ABM/RustGrowth.jl:93-131]}

| Parameter | Value | Units | Description | Source |
|-----------|-------|-------|-------------|--------|
| `light_inh` | 0.5168 | probability/unit sunlight | UV inactivation under full sunlight | ABC calibration |
| `rain_washoff` | 0.25 | probability/unit sunlight | Rain wash-off probability | Avelino et al. 2020 |
| `max_inf` | 1.0301 | probability | Maximum infection probability | ABC calibration |
| `rep_inf` | 0.2113 | dimensionless | Weight of reproductive growth on infection probability | ABC calibration |
| `max_lesions` | 25 | count | Maximum lesions per plant | Model parameter |

---

#### Submodel 7.11: Germination and Infection (No Rain)

**Purpose:** Same as 7.10 but for dry days.

**Differences:**
- No rain wash-off step (step 2 skipped)
- Leaf wetness term: `wet_inf_p = 0.05 × (12.0 + 2.0 × sunlight) - 0.2` (base 12 hours without rain, vs 18 with rain)

{CODE_VERIFIED [source: src/ABM/RustGrowth.jl:133-168]}

---

#### Submodel 7.12: Parasitism

**Purpose:** Rust drains host plant resources proportional to total lesion area.

**Equation:**
```
storage ← storage - rust_paras × sum(areas)

IF storage < -10:
  Trigger plant exhaustion (see Submodel 7.7)
```

{CODE_VERIFIED [source: src/ABM/RustGrowth.jl:172-186]}

| Parameter | Value | Units | Description | Source |
|-----------|-------|-------|-------------|--------|
| `rust_paras` | 0.0155 | per area unit per day | Resources drained per unit of total lesion area | ABC calibration |

---

#### Submodel 7.13: End-of-Day Rust Update

**Purpose:** Merge newly deposited spores and apply viability loss to existing deposits. Executed in Loop 2 of rust_step, after all agents have completed dispersal and growth.

**Logic:**
```
IF exh_countdown > 0:
  rusted ← false
ELSE:
  deposited ← deposited × viab_loss + newdeps
  newdeps ← 0
  IF deposited < 0.05:
    deposited ← 0
    IF n_lesions == 0:
      rusted ← false
```

{CODE_VERIFIED [source: src/ABM/RustGrowth.jl:190-204]}

| Parameter | Value | Units | Description | Source |
|-----------|-------|-------|-------------|--------|
| `viab_loss` | 0.75 | fraction/day | Daily viability retention of deposited spores | Nutman et al. 1963 |

---

#### Submodel 7.14: Rain Splash Dispersal

**Purpose:** Local stochastic spore dispersal via rain droplet splash.

**Logic:**
```
Effective spore area:
  spore_area = sum(areas[i] for sporulating lesions) × spore_pct × (1 + sunlight)

Number of splashed spores:
  n_splashed ~ Poisson(spore_area)

Distance modifier (U-shaped function of shade):
  d_mod = (4 - 4 × diff_splash) × (sunlight - 0.5)² + diff_splash
  # Maximum splash distance at full sun and full shade;
  # minimum at intermediate shade (sunlight = 0.5)

Per spore:
  distance ~ Exponential(rain_dst) × d_mod
  heading ~ Uniform(0, 360°)

  IF distance < 1.0:
    Self-deposition: rust.newdeps ← newdeps + 1
  ELSE:
    Trace trajectory in 0.5-cell increments from pos along heading:
      At each new cell visited:
        IF outside farm boundary → add to outpour[direction_code]
        IF cell is coffee (farm_map==1) AND Bernoulli(tree_block):
          Deposit on that plant (c.newdeps += 1, c.rusted = true)
        IF cell is shade (farm_map==2) AND Bernoulli(tree_block):
          Spore lost (absorbed by tree)
      IF reaches max distance without interception:
        IF final cell is coffee → deposit; ELSE → lost
```

{CODE_VERIFIED [source: src/ABM/RustDispersal.jl:3-26 (disperse_rain!), src/ABM/RustDispersal.jl:56-101 (splash trajectory)]}

| Parameter | Value | Units | Description | Source |
|-----------|-------|-------|-------------|--------|
| `rain_dst` | 2.2071 | cells (mean) | Mean rain splash dispersal distance | ABC calibration |
| `diff_splash` | 2.5 | multiplier | Shade-enhanced splash distance modifier | Avelino et al. 2020; Gagliardi et al. 2021 |
| `tree_block` | 0.4889 | probability | Probability that a tree (coffee or shade) blocks a rain-splashed spore | ABC calibration |
| `spore_pct` | 0.3836 | fraction | Fraction of sporulating lesion area that produces spores | ABC calibration |

---

#### Submodel 7.15: Wind Dispersal

**Purpose:** Long-range directional spore dispersal via wind.

**Logic:**
```
Spores lifted:
  n_lifted ~ Poisson(spore_area × (1 - 0.5 × shade_map[pos]))
  # Fewer spores lifted from shaded positions

Distance:
  w_distance ~ Exponential(wind_dst) × (1 + sunlight × diff_wind)
  # More open positions → farther dispersal

IF w_distance < 1.0:
  Self-deposition: n_lifted spores added to newdeps

ELSE:
  Per lifted spore:
    heading = wind_h + Uniform(0, 30°) - 15°     # wind_h ± 15°
    Trace trajectory via gust() in 0.5-cell increments:
      Blocking check at each cell:
        P(blocked) = shade_map[cell] × shade_block
      IF blocked:
        The NEXT cell encountered:
          IF coffee → deposit
          ELSE → spore lost
      IF outside boundary → add to outpour[direction_code]
      IF reaches max distance:
        IF final cell is coffee → deposit; ELSE → lost
```

Key difference from rain splash: wind dispersal is **directional** (follows wind heading ± 15°), shade blocks probabilistically based on shade_map value (not just tree presence), and once blocked the spore deposits on the next cell rather than the blocking cell.

{CODE_VERIFIED [source: src/ABM/RustDispersal.jl:28-52 (disperse_wind!), src/ABM/RustDispersal.jl:103-156 (gust trajectory)]}

| Parameter | Value | Units | Description | Source |
|-----------|-------|-------|-------------|--------|
| `wind_dst` | 5.7806 | cells (mean) | Mean wind dispersal distance | ABC calibration |
| `diff_wind` | 3.0 | multiplier | Openness-enhanced wind distance modifier | Pezzopane |
| `shade_block` | 0.7898 | probability | Probability that shade blocks a wind-dispersed spore (per cell, scaled by shade_map) | ABC calibration |

---

#### Submodel 7.16: Outside Spore Re-entry

**Purpose:** Spores that left the farm via dispersal can re-enter on windy days.

**Logic:**
- The `outpour` vector (8 elements) tracks spores that exited in each compass direction. It decays by ×0.9 daily (in pre_step!).
- On windy days, `outside_spores!` selects upwind directions based on wind heading (e.g., if wind blows from the east, spores from the eastern outpour pool re-enter from the east edge).
- For each spore attempting re-entry: starting position is a random point along the relevant farm edge. Trajectory is computed using the `gust()` function with `distance ~ Exponential(wind_dst) × (1 + diff_wind)`.
- The direction mapping uses 8 sectors (each 45°) centered on cardinal and ordinal directions, with adjacent sectors also contributing.

{CODE_VERIFIED [source: src/ABM/RustDispersal.jl:160-309]}

---

#### Submodel 7.17: Harvest

**Purpose:** Annual harvest, cost accounting, and production cycle reset.

**Logic:**
```
yearly_prod = sum(c.production for all agents)
costs ← costs + fixed_costs + yearly_prod × (other_costs × (1 - shadeacc/365 × mean(shade_map)) + 0.012)
shadeacc ← 0
fung_count ← 0

FOR each agent:
  production ← 0
  deposited ← deposited × lesion_survive
  surviving = floor(n_lesions × lesion_survive)
  Remove the OLDEST (first) (n_lesions - surviving) lesions
  IF surviving == 0 AND deposited < 0.05:
    deposited ← 0, rusted ← false
```

{CODE_VERIFIED [source: src/ABM/CGrowerSteps.jl:1-34]}

| Parameter | Value | Units | Description | Source |
|-----------|-------|-------|-------------|--------|
| `harvest_day` | 365 | day of year | Day of annual harvest | Model parameter |
| `lesion_survive` | 0.4338 | fraction | Proportion of lesions surviving to next cycle | ABC calibration |
| `fixed_costs` | 7600.6 | cost units | Annual production-independent costs | Model parameter |
| `other_costs` | 0.088 | cost/unit production | Production-dependent variable costs | Model parameter |
| `coffee_price` | 1.0 | price/unit | Coffee price | Model parameter |

---

#### Submodel 7.18: Shade Pruning

**Purpose:** Reduce shade level on scheduled pruning days.

**Logic:**
```
IF ind_shade > target:
  ind_shade ← target
ELSE:
  ind_shade ← ind_shade × 0.9
costs ← costs + tot_prune_cost
```

Where `tot_prune_cost = Normal(prune_cost, 0.05×prune_cost) × n_shades` (computed at initialization with noise).

{CODE_VERIFIED [source: src/ABM/CGrowerSteps.jl:36-43]}

| Parameter | Default | Units | Description | Source |
|-----------|---------|-------|-------------|--------|
| `prune_sch` | (74, 196) | days of year | Pruning schedule (third entry -1 = disabled) | Model parameter |
| `post_prune` | (0.3, 0.5) | ind_shade level | Target shade after each pruning event | Model parameter |
| `prune_cost` | 1.656 | cost/shade tree | Per-shade-tree pruning cost | Model parameter |

---

#### Submodel 7.19: Inspection

**Purpose:** Farmer samples plants, detects and removes visible lesions, updates observed incidence.

**Logic:**
```
Select n_inspected = floor(inspect_effort × n_coffees) random active plants

n_infected = 0
FOR each inspected plant:
  n_visible = count(areas[i] > 0.05)    # lesions with ~0.25cm diameter visible
  IF n_visible > 0 AND (max(areas) > 0.8 OR Bernoulli(n_visible / 5)):
    n_infected ← n_infected + 1
    Select up to rm_lesions lesions, weighted by visibility:
      weight[i] = areas[i] if areas[i] > 0.05, else 0
    Remove selected lesions (delete from ages, areas, spores vectors)
    Update n_lesions
    IF n_lesions == 0 AND deposited < 0.05: rusted ← false

obs_incidence ← n_infected / n_inspected
costs ← costs + tot_inspect_cost
```

Where `tot_inspect_cost = Normal(inspect_cost × (1 + 0.2×(rm_lesions-1)), noise) × n_inspected`.

{CODE_VERIFIED [source: src/ABM/CGrowerSteps.jl:45-75]}

| Parameter | Default | Units | Description | Source |
|-----------|---------|-------|-------------|--------|
| `inspect_period` | 7 | days | Inspection frequency | Model parameter |
| `inspect_effort` | 0.01 | fraction | Fraction of plants inspected each time | Model parameter |
| `rm_lesions` | 2 | count | Lesions removed per detected infected plant | Model parameter |
| `inspect_cost` | 0.006 | cost/coffee | Per-plant inspection cost | Model parameter |

---

#### Submodel 7.20: Fungicide Application

**Purpose:** Apply fungicide spray to the entire farm.

**Logic:**
```
costs ← costs + tot_fung_cost
fungicide ← 1                    # start counter (hardcoded, not using fung_effect param)
fung_count ← fung_count + 1
```

Where `tot_fung_cost = Normal(fung_cost, 0.05×fung_cost) × n_coffees`.

Fungicide remains active for 30 days (counter increments daily in farmer_step!, expires when > 30). Maximum `max_fung_sprayings` (3) applications per year.

**Four application strategies** (controlled by `fung_stratg`):
- `:cal` — Calendar-based: spray on days in `fungicide_sch` (default: days 91, 273)
- `:cal_incd` — Hybrid: spray on scheduled days OR when `obs_incidence > incidence_thresh`
- `:incd` — Incidence-only: spray only when `obs_incidence > incidence_thresh`
- `:flor` — Flowering-based: spray at `rep_d + 60` and `rep_d + 105`

{CODE_VERIFIED [source: src/ABM/CGrowerSteps.jl:79-83 (fungicide!), src/ABM/MainStep.jl:186-201 (strategy logic)]}

| Parameter | Default | Units | Description | Source |
|-----------|---------|-------|-------------|--------|
| `fungicide_sch` | (91, 273) | days of year | Calendar spray days (entries ≤ 0 disabled) | Model parameter |
| `fung_stratg` | `:cal` | symbol | Fungicide application strategy | Model parameter |
| `incidence_thresh` | 0.1 | fraction | Incidence threshold for reactive spraying | Cenicafé Boletín 36 |
| `max_fung_sprayings` | 3 | count/year | Maximum annual fungicide applications | Model parameter |
| `fung_cost` | 0.179 | cost/coffee | Per-plant fungicide cost | Model parameter |
| `fung_effect` | 30 | days | Duration of fungicide effect (defined but NOT used — hardcoded as 30 in farmer_step!) | Model parameter |

---

## Quality Standards

### Reimplementability

A competent modeler who has never seen the source code must be able to reimplement the model from the ODD alone. This means:
- All parameters specified with values, units, and ranges
- All decision rules precisely defined with exact conditions and outcomes
- All equations written with clear mathematical notation
- All algorithms described in pseudocode or step-by-step logic
- All boundary conditions and edge cases addressed (e.g., storage < -10 exhaustion, lesion area bounds check, max_lesions cap)
- The order of operations within each step must be explicit and unambiguous

### Confidence Annotations

Tag every factual claim with one of:
- `{CODE_VERIFIED}` — verified by reading source code `[source: file:line]`
- `{DOC_STATED}` — explicitly stated in documentation `[source: doc:section]`
- `{MODELER_CONFIRMED}` — confirmed by modeler during interview `[interview Q#]`
- `{INFERRED}` — reasonably inferred from code structure `[inference chain]`
- `{UNVERIFIABLE}` — cannot be verified from available sources `[reason]`

### Citation Format

- Code references: `[source: src/ABM/RustGrowth.jl:4-35]`
- Literature in code comments: cite as found (e.g., "McCain & Hennen, 1984")
- Interview references: `[interview Q4]`
- Inline in the ODD text, use standard academic citation format for literature references

### Writing Rules

1. Use exact terminology from the Terminology table above
2. Never paraphrase technical concepts into simpler language
3. Describe what the program does, not what you think the model does
4. Do not add mechanisms, behaviors, or details not found in the code
5. Where the modeler's design rationale is unknown (most of this model), note this briefly but do not speculate
6. Flag the UV inhibition asymmetry between grow_rust! and grow_f_rust! as a notable implementation detail
7. Flag the unused `fung_effect` parameter as a notable implementation detail
8. Note all open questions from the research phase in a final "Unresolved Questions" appendix

## Sub-agent Instructions

No sub-agents needed. Handle all sections directly. The model is moderate-to-complex but all findings are fully embedded in this plan with CODE_VERIFIED confidence.

## Traceability Matrix

The draft must also produce `lazyodd/draft/traceability-matrix.md` mapping every ODD claim to its source.

Format:
| ODD Section | Claim | Source | Confidence | Notes |
|-------------|-------|--------|------------|-------|

Include every equation, parameter value, and behavioral rule as a separate row.

## Output Specifications

- Primary document: `lazyodd/draft/odd.md`
- Traceability matrix: `lazyodd/draft/traceability-matrix.md`
- Format: Markdown with inline citations and confidence annotations
- Structure: Numbered sections following ODD+2 protocol exactly (7 elements, 11 design concepts)
- Open with the standard ODD protocol statement
- No separate rationale subsections (rationale was not available from the modeler for most elements)
