# Upsilon trigger selections and kinematic reweighting

The Upsilon calibration-input histmaker can apply trigger-motivated offline
selections and direct, category-specific shape weights in
`(ptll, yll, cosThetaStarll)`. For legacy inputs, the trigger selection is
opt-in. For NanoV15, `muon_eta` is the default. V weighting is opt-in for both.

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
The V-map input is made before the state split. To make the default NanoV15
calibration input, run from the repository root after `source setup.sh`:

```bash
nano=(--resonance upsilon --era 2016PostVFP --nanoVersion v15 --nanoV15Inputs inputs.json)
stem=dimuon_resonances_calinput_upsilon_2016PostVFP_v15_states_pdg

python3 scripts/histmakers/dimuon_resonances_calinput.py \
  "${nano[@]}" --makeUpsilonVReweightInputs -j 16 -o CALINPUT
python3 scripts/corrections/make_upsilon_v_reweighting.py \
  "CALINPUT/${stem}_upsilonSel_muon_eta.hdf5" CALINPUT/upsilon_v_weights.hdf5
python3 scripts/histmakers/dimuon_resonances_calinput.py \
  "${nano[@]}" --upsilonVReweightFile CALINPUT/upsilon_v_weights.hdf5 \
  --fitMuonScaleAndResolution --etaBins 24 --scale_A 1 --scale_e 1 --scale_M 1 \
  -j 16 -o CALINPUT
```

The last command books separate scale (`A`, `e`, `M`) and resolution (`a`,
`b`, `c`) responses for each state. Use its HDF5 output for the
[tensor and fit commands](../rabbit/README.md#nanov15-upsilon-fit).

### Optional MC slice stitching

The default uses unit MC event weights before V weighting. To stitch the
three generator slices by cross section, pass
`--upsilonStitchingFile normalizations.json` both when making the V-map input and when applying the
map in the response production. The JSON has one entry for each of `pt0to8`,
`pt8to13`, and `pt13toInf`, with numeric `effective_xsec_pb` and
`generated_sumw` fields. Supply the effective cross section after the
generator filter and the generated sum of weights for that same production,
before any reconstruction or analysis selection. Do not use the sum of
selected weights or apply the generator filter efficiency twice.

The histmaker multiplies MC `genWeight` by `effective_xsec_pb / generated_sumw`
for each slice before deriving or applying the V weights. Data, selections,
and input files are unchanged. A map derived with a different set of factors
or without stitching cannot be used on stitched MC. Stitched output is named
with `upsilonStitched` and includes `Y_mc_unweighted` for the tensor's MC
occupancy cut. The tensor does not apply cross sections a second time.
Use the produced `upsilonStitched` filenames in the V-map and tensor commands.

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
