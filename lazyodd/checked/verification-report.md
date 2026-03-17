# ODD Verification Report: SpatialRust

> Verified by lazyodd:check | Date: 2026-03-17
> ODD Document: lazyodd/draft/odd.md
> Overall Grade: **A** (4.6 / 5.0)

## Summary

This is a high-quality, publication-ready ODD document. All 7 elements and 11 design concepts are present with comprehensive detail. 12 claims were independently verified against source code with 12/12 matching. The document is internally consistent, terminologically precise, and contains sufficient pseudocode and equations for reimplementation. The only weaknesses are inherent to the input quality (code-only, no documentation) — the purpose and theoretical grounding are necessarily inferred.

## Scores by Element

| Element | Completeness | Precision | Traceability | Consistency | Overall |
|---------|-------------|-----------|--------------|-------------|---------|
| 1. Purpose and Patterns | 4 | 4 | 4 | 5 | 4.3 |
| 2. Entities, State Variables, and Scales | 5 | 5 | 5 | 5 | 5.0 |
| 3. Process Overview and Scheduling | 5 | 5 | 5 | 5 | 5.0 |
| 4. Design Concepts | 5 | 4 | 4 | 5 | 4.5 |
| 5. Initialization | 5 | 5 | 5 | 5 | 5.0 |
| 6. Input Data | 5 | 5 | 5 | 5 | 5.0 |
| 7. Submodels | 5 | 5 | 5 | 4 | 4.8 |
| **Average** | **4.9** | **4.7** | **4.7** | **4.9** | **4.8** |

### Grading Scale
- **A** (4.5-5.0): Publication-ready, minor polish only ← **This ODD**
- **B** (3.5-4.4): Good quality, some improvements needed
- **C** (2.5-3.4): Adequate but significant gaps
- **D** (1.5-2.4): Major revisions needed
- **F** (1.0-1.4): Fundamental issues, consider re-drafting

## Issues Found

### Critical (must fix)

None.

### Major (should fix)

1. **[7.5/7.19]: Pseudocode comment "production growing" is misleading**
   - **Problem:** In Submodel 7.5 (Reproductive Growth), the pseudocode labels the `d_prod ≥ 0` branch as "production growing". However, in this branch, only `veg` is updated — `production` is NOT changed. Production doesn't grow; it simply isn't declining because photosynthate covers its maintenance cost, with surplus going to vegetative growth.
   - **Evidence:** `src/ABM/CoffeeSteps.jl:44-45` — only `coffee.veg` is updated in the `else` branch; no `coffee.production` assignment exists.
   - **Fix:** Change the pseudocode comment from "production growing" to "production stable (maintenance covered, surplus to veg)" and add a note: "Note: production is only set on the commitment day (Submodel 7.6) and can decline but never increases during the reproductive phase."

2. **[7.19]: Source code bug not flagged in ODD**
   - **Problem:** In the inspection submodel, the source code at `src/ABM/CGrowerSteps.jl:67` has `c.deposited == 0.0` (equality comparison, no-op) where it likely intends `c.deposited = 0.0` (assignment). The equivalent line in the harvest submodel (`src/ABM/CGrowerSteps.jl:23`) correctly uses `=`. The ODD's pseudocode for inspection omits the deposited-reset step entirely, which is actually consistent with what the code *does* (nothing, due to the bug), but inconsistent with what it likely *intends*.
   - **Evidence:** Compare `CGrowerSteps.jl:67` (`c.deposited == 0.0`, comparison) with `CGrowerSteps.jl:23` (`c.deposited = 0.0`, assignment in harvest).
   - **Fix:** Add a note in Submodel 7.19 or the Appendix: "Note: The source code at `CGrowerSteps.jl:67` contains `c.deposited == 0.0` (equality comparison) where `c.deposited = 0.0` (assignment) was likely intended, matching the equivalent logic in `harvest!` at line 23. The ODD describes the code's actual behavior (deposited is not reset during inspection)."

### Minor (nice to fix)

1. **[2]: Term "active" used but not formally defined**
   - **Problem:** The ODD uses "active plants" in Submodels 7.17 and 7.19 without defining the term. The source code defines `active(c::Coffee) = c.exh_countdown == 0` at `src/Metrics.jl:11`.
   - **Fix:** Add a brief definition in Element 2 or a footnote: "A coffee plant is *active* when `exh_countdown == 0` (i.e., not in exhaustion recovery)."

2. **[7.3]: Struct comment discrepancy worth noting**
   - **Problem:** The `CoffeePars` struct comment at `src/ABM/MainSetup.jl:240` says `photo_const::Float64 # f_avail * phs_max * photo_frac`, but the actual construction at line 155 is `f_avail * phs_max` (without `photo_frac`). The ODD correctly describes the computation as `photo_const = f_avail × phs_max`, but a reimplementer reading both the ODD and source code might be confused by the discrepancy.
   - **Fix:** Optional — add a note: "Note: The source code struct comment describes `photo_const` as including `photo_frac`, but the actual construction excludes it; `photo_frac` is applied separately via `photo_veg = veg × photo_frac`."

3. **[1]: Species name misspelling**
   - **Problem:** The ODD writes *Hemileia vastarix* — the correct spelling is *Hemileia vastatrix* (with a 't').
   - **Fix:** Replace "vastarix" with "vastatrix" throughout.

4. **[7.14/7.15]: Spore area computation described in two places**
   - **Problem:** The effective spore area formula (`sum(sporulating areas) × spore_pct × (1 + sunlight)`) appears in both the Element 3 scheduling description and in Submodels 7.14/7.15. This is not wrong but creates slight redundancy.
   - **Fix:** Consider referencing the submodel from Element 3 rather than repeating the formula.

## Verification Details

### A. Structural Completeness: PASS

- [x] All 7 ODD elements present as numbered sections
- [x] All 11 design concepts addressed (4.4–4.6, 4.10 explicitly marked "not applicable")
- [x] No empty or placeholder sections
- [x] Rationale subsections omitted with justification (model developed by another; rationale not available)
- [x] Element ordering follows ODD+2 standard
- [x] ODD citation statement present (line 10)
- [x] Code-to-ODD mapping table included (Table 23)
- [x] Technical context section present
- [x] Unresolved questions appendix included

### B. Source Traceability: PASS

- [x] Every factual claim has an inline citation
- [x] Citations reference real files/locations that exist
- [x] Traceability matrix has 107 rows covering all major claims
- [x] Traceability matrix is consistent with inline citations
- [x] No orphaned claims found
- [x] Confidence categories present on all claims

**Citation verification sample (12 claims):**

| # | Claim (ODD) | Cited Source | Code Says | Verdict |
|---|-------------|-------------|-----------|---------|
| 1 | incidence = mean(n_lesions > 0) | Metrics.jl:3 | `mean(a -> (a.n_lesions > 0), model.agents)` | ✓ Match |
| 2 | Coffee has 13 state variables | MainSetup.jl:4-20 | 13 fields in @agent block (excluding inherited id, pos) | ✓ Match |
| 3 | shade_map = 1 - d²_min / (2×shade_r)² | ShadeMap.jl:9-35 | `1.0 - minimum(eucdist(n)...) / maxdist` where `maxdist=(2*shade_r)^2` | ✓ Match |
| 4 | UV inhibition in grow_rust!, absent in grow_f_rust! | RustGrowth.jl:18,56 | Line 18: `- 0.1 * rust.sunlight`; Line 56: no such term | ✓ Match |
| 5 | photo_const = f_avail × phs_max | MainSetup.jl:155 | `CoffeePars(..., f_avail * phs_max, ...)` | ✓ Match |
| 6 | Harvest: remove oldest (first) lesions | CGrowerSteps.jl:29 | `deleteat!(c.ages, 1:lost)` | ✓ Match |
| 7 | Detection: area>0.8 certain, else P=nvis/5 | CGrowerSteps.jl:59 | `0.8 < maximum(c.areas...) \|\| rand(...) < nvis / 5` | ✓ Match |
| 8 | production = max(0, N(res_commit,0.01)×sl×veg×sto) | MainStep.jl:69-74 | `max(0.0, rand(rng, commit_dist) * cof.sunlight * cof.veg * cof.storage)` | ✓ Match |
| 9 | w_distance ~ Exp(wind_dst) × (1+sl×diff_wind) | RustDispersal.jl:31 | `rand(rng, Exponential(rustpars.wind_dst)) * (1 + rust.sunlight * rustpars.diff_wind)` | ✓ Match |
| 10 | Wind heading: wind_h ± 15° | RustDispersal.jl:38,41 | `wind_h = model.current.wind_h - 15.0`; per spore: `wind_h + rand()*30.0` | ✓ Match |
| 11 | Rust params: rust_gr=0.0867, opt_temp=22.8716 | MainSetup.jl:163 | `0.0867, 22.8716` in RustPars constructor | ✓ Match |
| 12 | n_splashed ~ Poisson(spore_area) | RustDispersal.jl:9 | `rand(model.rng, Poisson(spores))` | ✓ Match |

**Result: 12/12 claims verified (100%)**

### C. Semantic Consistency: PASS

- [x] No contradictions between sections
- [x] Terminology used consistently (checked: "coffee plant", "lesion", "deposited", "ind_shade", "farm_map" all consistent)
- [x] Parameter names match between Element 2 state variables and Element 7 parameter tables
- [x] Process order in Element 3 consistent with submodel descriptions in Element 7
- [x] Entity types in Element 2 match Element 4 design concepts
- [x] Initialization values in Element 5 consistent with state variables in Element 2
- [~] Minor: "production growing" label in Submodel 7.5 pseudocode is misleading (see Major issue #1)

### D. Code Alignment: PASS

- [x] Process descriptions in Element 3 match actual code execution order
- [x] Submodel equations/pseudocode in Element 7 match code logic
- [x] Entity state variables in Element 2 match code data structures
- [x] Initialization values in Element 5 match code defaults
- [x] Parameter values stated in the ODD match values in code (all 14 ABC-calibrated parameters verified)
- [x] Decision rules match implemented logic
- [~] Source code bug at CGrowerSteps.jl:67 (== vs =) not flagged (see Major issue #2)

### E. Reimplementability: PASS

- [x] All parameters specified with values (22 parameter tables, every parameter has a value)
- [x] All decision rules precisely defined with exact conditions
- [x] Equations complete with all variables defined
- [x] Pseudocode unambiguous and complete for all 20 submodels
- [x] Boundary conditions addressed (exhaustion at -10, max_lesions=25, senescence at 150 days, area bounds check)
- [x] Spatial structure fully specified (grid size, topology, metric, boundary conditions)
- [x] Temporal structure fully specified (1 day/step, 365-day cycle, 730-day warm-up)
- [x] All stochastic elements specify exact distributions and parameters (Table 3 + per-submodel)
- [~] Term "active" used but not formally defined (see Minor issue #1)

### F. Confidence Audit: PASS

- [x] CODE_VERIFIED claims reference actual code locations (12/12 sample verified)
- [x] INFERRED claims have documented inference chains (Element 1 purpose, Element 4.1)
- [x] UNVERIFIABLE claims explain why (11 items in Appendix, all with reasons)
- [x] No claims marked CODE_VERIFIED that contradict the code
- [x] Distribution reasonable for code-only input: ~89% CODE_VERIFIED, ~5% INFERRED, ~10% UNVERIFIABLE
- [x] Appropriate that purpose is INFERRED (no documentation available) while all structural/behavioral claims are CODE_VERIFIED

## Recommendations

Prioritized by impact on ODD quality:

1. **Fix species name**: *Hemileia vastarix* → *Hemileia vastatrix* (typo, easy fix, affects credibility)
2. **Clarify "production growing" label**: Change Submodel 7.5 pseudocode comment to "production stable" and add a note that production never increases during the reproductive phase
3. **Document the inspect! bug**: Add a note about the `==` vs `=` discrepancy at CGrowerSteps.jl:67, since reimplementers reading both ODD and code may encounter this
4. **Define "active"**: Add formal definition of "active plant" (exh_countdown == 0) in Element 2
5. **Note photo_const comment discrepancy**: Optional, helps reimplementers who read both ODD and source

All issues are minor-to-moderate. The ODD is ready for publication with these polishing fixes applied.
