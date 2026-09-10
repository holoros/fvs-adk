# FVS-ADK

An R implementation of the Adirondack variant of the Forest Vegetation Simulator: an
individual-tree growth-and-yield model for the mixed-species forests of the Adirondacks
region of New York, USA.

## Source

Weiskittel, A. and Kuehne, C. (2016). Development and evaluation of an individual tree
growth and yield model for the mixed species forest of the Adirondacks Region of New
York, USA. *Forest Ecosystems* 3:26. https://doi.org/10.1186/s40663-016-0086-3

Correction: Weiskittel, A. and Kuehne, C. (2019). Correction to: Development and
evaluation of an individual tree growth and yield model for the mixed species forest of
the Adirondacks Region of New York, USA. *Forest Ecosystems*.
https://doi.org/10.1186/s40663-019-0182-2

The paper's equations were fit on remeasurement data from five calibration forests:
Dubuar Memorial Forest (DMF), Huntington Wildlife Forest (HWF), Pack Demonstration
Forest, Pack Experimental Forest (PEF), and Shirley Forest (SF).

## Model components

`R/AdirondackGY.r` implements, per species or species group:

| Function | What it predicts | Paper table |
|---|---|---|
| `SUNY.HT` | Total height from DBH | -- |
| `SUNY.BHT` | Crown base height | Table 5 |
| `SUNY.dDBH` | Annual diameter increment | Table 6 |
| `SUNY.dHT` | Annual height increment | Table 7 |
| `SUNY.surv` / `SUNY.survOCP` | Annual survival probability | Table 8 |
| `Ingrowth.FUN` | Ingrowth occurrence and rate | not in the 2016 paper -- see below |

`AdirondackGYOneStand` drives a full stand projection: growth, mortality, and ingrowth
applied to a tree list for one annual cycle.

Climate site index (CSI) is a base-age-50 site index derived from downscaled climate
variables (Weiskittel et al. 2011b), used in place of a field-measured site index.

## Ingrowth submodel: provenance note

`Ingrowth.FUN` uses the logistic-occurrence-plus-rate structure of Li et al. (2011,
*Canadian Journal of Forest Research* 41:2077-2089): a logit model for probability of
ingrowth occurrence (PI) and a log-linear model for ingrowth rate (IPH), gated by a
fitted cutpoint.

**This submodel is not part of the 2016 Weiskittel and Kuehne paper.** That paper cites
Li et al. (2011) exactly once, as a direction for future work, and does not implement or
report it. The coefficients in the NLME branch of `Ingrowth.FUN` have not been traced to
any published source; treat them as a provisional, unpublished parameterization. See
`docs/ny_ingrowth_refit_plan.md` for the plan to replace them with a proper NY-specific
fit once an empirical benchmark is identified.

At a minimum DBH threshold of 10 cm, the fitted occurrence cutpoint (`PI >= CutPoint`) is
not reached across the 300 New York FIA test stands used to validate this release, so
ingrowth does not fire under the model's own gate at that threshold. An expected-value
sensitivity run (using the fitted probability continuously instead of the hard cutpoint) changed 50-year basal area, density, and QMD by under a
quarter of a percent for that stand set, so the gate's effect is small for typical NY
conditions but is a real structural property of the submodel worth knowing before
relying on it elsewhere.

## Known open issue: survival coefficients for Balsam fir and Spruce

The full line-by-line audit against the paper's Table 8 (see the v5.1.1 commit message
and the `AUDIT FLAG` comments in `SUNY.surv` and `SUNY.survOCP`) found that for Balsam
fir and the Spruce group (red, white, black), the extracted table places the nonzero
mid-magnitude survival coefficient in the sqrt(BA) slot with BAL at zero, while the code
has it the other way around: BAL nonzero, sqrt(BA) at zero. This could not be resolved
because clean access to the raw PDF table was not available during the audit. It is
flagged in the code rather than silently changed. Anyone with the original PDF or the
underlying fit output should check this before treating those two groups' survival
predictions as fully verified.

## Known extrapolation hazard: ClimateSI above ~30-40 m

A stress test found that `Ingrowth.FUN`'s predicted ingrowth
rate (IPH) grows exponentially once `ClimateSI` exceeds roughly 40 m, reaching into the
millions of trees per hectare per year. This is almost certainly outside the domain the
untraced ingrowth coefficients were fit on. `Ingrowth.FUN` now emits a `warning()`
whenever `ClimateSI > 30`; this is a tripwire, not a clamp, since no verified safe upper
bound exists. This matters concretely for any future climate-derived CSI source: the
figshare climate site index dataset evaluated for a future release (Weiskittel 2025,
[10.6084/m9.figshare.28509299.v1](https://doi.org/10.6084/m9.figshare.28509299.v1)) has a
reported maximum of 48.9 m, inside the blow-up zone. Check any new CSI source against
this boundary before trusting downstream ingrowth output. All other model components
(height, crown base, diameter increment, height increment, survival) were stress-tested
across a wide boundary-value grid and 100-year runs on seven synthetic extreme stands
with no NaN/Inf, no impossible negative growth, no crashes, and no numerical blow-up. See
the v5.1.2 commit message for the full grid specification.

## Usage

```r
source("R/AdirondackGY.r")

# One annual growth cycle for a tree list `tr` (columns: STAND, PLOT, TREE, id, SP, DBH,
# HT, CR, HCB, EXPF) and a stand list carrying CSI:
stand <- list(STAND = "example", CSI = 15)   # CSI in meters, base age 50
ops <- list(cyclen = 1, INGROWTH = "Y", MinDBH = 10, mortModel = "Adirondack", verbose = FALSE)
result <- AdirondackGYOneStand(tr, stand, ops)
```

## Species codes

Species are coded by short internal codes (e.g. `SM` sugar maple, `BF` balsam fir, `EH`
eastern hemlock, `RS`/`WS`/`BS` red/white/black spruce). See the `SPP ==` branches in each
`SUNY.*` function for the full list.

## Version history

- **v5.1.2** (2026-09-10) -- Stress-tested; added a `ClimateSI` extrapolation warning in
  `Ingrowth.FUN`. No coefficient changed.
- **v5.1.1** (2026-09-10) -- Full line-by-line audit against the 2016 paper and its 2019
  erratum. Relabeled the ingrowth submodel's provenance; flagged (not changed) a possible
  Balsam fir / Spruce survival coefficient swap. No coefficient changed from v5.1.0.
- **v5.1.0** -- Prior working version (dependency cleanup, `make_adk_tree()` /
  `make_fvs_tree()` / `make_fvs_regen()` R-FVS data handoff helpers, trees-per-hectare
  correction, `AdirondackGYOneStand()` returns a data frame). See the source file's own
  changelog header for the full history before v5.1.0.

## Citing this code

If you use this model, cite the source paper (Weiskittel and Kuehne 2016, with its 2019
correction) above. This repository documents modifications and audit findings made to
the R implementation; it is not itself a citable methods reference for the underlying
equations.

## Contact

Aaron Weiskittel, University of Maine, School of Forest Resources.
aaron.weiskittel@maine.edu
