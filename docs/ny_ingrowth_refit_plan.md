
## Stress test addendum (2026-09-10, v5.1.2)

A boundary-value stress test of the whole model (all SUNY.* functions plus Ingrowth.FUN, and 100-year
runs on seven synthetic stress stands) found one concrete extrapolation danger relevant to this refit
plan: `Ingrowth.FUN`'s IPH grows exponentially once ClimateSI exceeds roughly 40 m, reaching into the
millions of trees per hectare per year well before CSI=60. Every other function and stand-level run
stayed numerically well-behaved across the full grid.

This bears directly on the CSI work above. The figshare CSI raster (`CurrentSI_m.tif`) has a reported
maximum of 48.9 m, inside the blow-up zone. If the extent problem is ever fixed and that CSI source is
substituted for the SICOND proxy, any pixel landing above roughly 30 to 40 m must be checked against
this failure mode before the resulting ingrowth numbers are trusted; a runtime warning was added in
v5.1.2 (`warning()` when ClimateSI > 30) as a tripwire, not a fix. Refitting the occurrence and rate
models on real NY remeasurement data, per the sequence above, should also report the empirical range of
ClimateSI in the fitting data, so any future CSI source can be checked for staying inside that fitted
range rather than being fed to the model blind.
