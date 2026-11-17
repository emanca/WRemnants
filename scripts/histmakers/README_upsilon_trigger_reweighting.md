# Upsilon trigger selections and kinematic reweighting

The Upsilon calibration-input histmaker can apply trigger-motivated offline
selections and direct, category-specific shape weights in
`(ptll, yll, cosThetaStarll)`. The feature is opt-in; the existing default
selection and weights are unchanged.

## Workflow

First produce the unweighted data and simulation inputs for one selection:

```bash
python3 scripts/histmakers/dimuon_resonances_calinput.py \
  --resonance upsilon --era 2016PostVFP --triggers inclusive \
  --upsilonTriggerSelection muon_eta \
  --makeUpsilonVReweightInputs -j 16 -o OUTPUT_UNWEIGHTED
```

Build one correction file from the resulting HDF5 file:

```bash
python3 scripts/corrections/make_upsilon_v_reweighting.py \
  OUTPUT_UNWEIGHTED/dimuon_resonances_calinput_upsilon_2016PostVFP.hdf5 \
  upsilon_v_weights_muon_eta.hdf5
```

Then rerun the histmaker with the same selection and apply that file:

```bash
python3 scripts/histmakers/dimuon_resonances_calinput.py \
  --resonance upsilon --era 2016PostVFP --triggers inclusive \
  --upsilonTriggerSelection muon_eta \
  --upsilonVReweightFile upsilon_v_weights_muon_eta.hdf5 \
  -j 16 -o OUTPUT_WEIGHTED
```

Use a separate correction file for each selection scheme. The loader rejects
a file if its recorded selection, category labels, axes, or binning do not
match the requested selection. Data are never weighted. In simulation, the
same kinematic weight multiplies the nominal and systematic histograms.

## NanoV15 Upsilon inputs

For NanoV15, pass `--nanoVersion v15 --nanoV15Inputs inputs.json`. The JSON
contains these four lists; use files or directories for the desired production:

```json
{
  "data": ["/path/to/NanoV15/MuOnia"],
  "pt0to8": ["/path/to/NanoV15/MC_Pt0to8"],
  "pt8to13": ["/path/to/NanoV15/MC_Pt8to13"],
  "pt13toInf": ["/path/to/NanoV15/MC_Pt13toInf"]
}
```

Use the complete inventory. Directories are searched recursively; if two
production tasks have the same output basename, keep both distinct full
paths. The histmaker applies the 2016 post-VFP golden JSON to data and uses
the stored CVH refit in data and ideal-CVH refit in simulation. It does not
apply v9 nominal smearing or pixel-multiplicity corrections to NanoV15.

The default category selection is `muon_eta`; the default state assignment
is `pdg` (the nearest common Upsilon ancestor of the two matched muons).
Use `--stateAssignment legacy` only for a generator-dimuon-mass comparison.
The V-map input is made before the state split. For the response production,
add `--fitMuonScaleAndResolution --etaBins 24 --scale_A 1 --scale_e 1 --scale_M 1`
to the weighted histmaker command. It books the scale (`A`, `e`, `M`) and
resolution (`a`, `b`, `c`) variations separately for each of the three states.
NanoV15 output names include the input version, state assignment, and category
selection, so use the matching filename when building the V map.

## Selections and categories

`inclusive_or` retains one category requiring the Dimuon8-barrel or Dimuon13
HLT path and reconstructed dimuon pT above 8 GeV. The other choices make two
exclusive categories. Their barrel route is selected using both-muon eta
(`muon_eta`), dimuon rapidity (`dimuon_rapidity`), or their combination with
the dimuon delta-eta requirement (`combined_geometry`). Barrel events must
pass the Dimuon8-barrel path, its full offline geometry, and dimuon pT above
8 GeV. Events outside the chosen barrel route must pass Dimuon13 and have dimuon, leading-muon,
and subleading-muon pT above 13, 13, and 6 GeV, respectively. Derive fresh
weights if moving from an older selection with raised thresholds.

## Weight definition and fallback

Weights are derived independently in every retained category as

```text
(data bin / supported data yield) / (MC bin / supported MC yield).
```

Only bins with positive data and simulation content enter this calculation.
No background subtraction, clipping, smoothing, interpolation, or
regularization is applied. Bins without common support, including flow bins,
receive unit weight. The payload records the fallback fraction and weight
summary for every category; weighted histmaker output also contains
`upsilonVReweightCoverage`, split by category and usable/fallback status. A
common pT axis starts at 8 GeV, so bins below the 13 GeV high-category
threshold are unreachable for that category and remain at unit weight.
