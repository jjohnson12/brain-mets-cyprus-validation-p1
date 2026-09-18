# SPIRS-P1 Cyprus longitudinal validation

Analysis code for the manuscript-focused external evaluation of SPIRS-P1 on the longitudinal Cyprus/PROTEAS brain-metastasis MRI dataset.

This repository contains the audited analysis layer used to establish cohort denominators, relink the released reference segmentations, rescore cross-sectional segmentation and lesion detection, estimate patient-clustered confidence intervals, and evaluate longitudinal tumor-burden and modified RANO-BM endpoints. It intentionally excludes training code, model weights, source imaging, derived patient-level data, internal infrastructure, and unrelated UCSF analyses.

## Scope

The release reproduces the manuscript analyses from an existing SPIRS-P1 prediction workspace. It does **not** run model inference. The required inputs are obtained separately:

- Cyprus/PROTEAS dataset version used for the manuscript: [Zenodo record 17253793](https://doi.org/10.5281/zenodo.17253793)
- SPIRS-P1 tracked prediction label maps and the analysis manifest produced by the longitudinal inference workflow

Use of the imaging dataset remains subject to its repository terms. No imaging, clinical tables, reference masks, predictions, or patient-level result files are included here.

## Manuscript denominator logic

The analysis distinguishes three nested cross-sectional denominators:

1. **186** released postcontrast T1-weighted studies.
2. **170** studies with a corresponding released reference segmentation mask. The 16 absent masks are not imputed.
3. **169** tumor-core-positive studies used for cross-sectional segmentation and lesion-detection analysis. One of the 170 masks contains edema only and no tumor core.

For whole-burden longitudinal analysis, all 170 reference-mask studies are retained, including the zero-core study. Chronologically consecutive available-reference observations yield **125 pairs** across **45 case units**; **116** are strictly adjacent in the original timepoint order and nine bridge one or more studies without a reference mask.

## Repository layout

```text
.
├── scripts/                 # Audited manuscript analysis scripts
├── docs/EXPECTED_RESULTS.md # Locked denominators and headline results
├── environment.yml          # Reproducible Python environment
├── CITATION.cff             # GitHub and citation-tool metadata
├── .zenodo.json             # Metadata used by Zenodo's GitHub integration
├── LICENSE
└── README.md
```

## Installation

```bash
conda env create -f environment.yml
conda activate brain-mets-cyprus-validation
```

The scripts were developed with Python 3.11. They are CPU analyses; model inference is outside the scope of this repository.

## Expected analysis workspace

The commands below use:

```bash
export CYPRUS_ARCHIVES=/path/to/cyprus_archives
export ANALYSIS_ROOT=/path/to/analysis_workspace
export REPO_ROOT=/path/to/brain-mets-cyprus-validation-p1
```

`$ANALYSIS_ROOT/dataset.csv` is the pre-existing anonymized analysis manifest. The analysis scripts expect columns including `AnonPatientID`, `AnonStudyID`, `Timepoint`, `TimepointOrder`, and prediction-path fields documented in the script headers. Tracked prediction ID maps are expected under `$ANALYSIS_ROOT/04_tracked` unless `--tracked-root` is supplied.

Paths written into a local manifest should refer only to the user's own analysis workspace. Do not commit manifests or generated outputs.

## Reproduction order

### 1. Audit released archive denominators

```bash
python "$REPO_ROOT/scripts/audit_cyprus_denominators_zips.py" \
  --zip-root "$CYPRUS_ARCHIVES" \
  --out-dir "$ANALYSIS_ROOT/denominator_audit"
```

### 2. Relink source reference masks

This step performs case-insensitive timepoint matching, extracts only the required tumor masks, preserves the input manifest, and writes a corrected manifest.

```bash
python "$REPO_ROOT/scripts/relink_cyprus_ground_truth.py" \
  --dataset "$ANALYSIS_ROOT/dataset.csv" \
  --zip-root "$CYPRUS_ARCHIVES" \
  --output-root "$ANALYSIS_ROOT/reference_masks_relinked" \
  --out-dataset "$ANALYSIS_ROOT/dataset_relinked.csv" \
  --summary "$ANALYSIS_ROOT/relink_summary.txt"
```

### 3. Recompute cross-sectional performance

```bash
python "$REPO_ROOT/scripts/rescore_cyprus_relinked_v2.py" \
  --dataset "$ANALYSIS_ROOT/dataset_relinked.csv" \
  --out-dir "$ANALYSIS_ROOT/rescore_relinked"
```

Definitions are fixed to Cyprus tumor-core labels 1+2, 26-connected components, one-to-one Hungarian matching, and lesion detection at Jaccard ≥0.10.

### 4. Compute patient-clustered confidence intervals

```bash
python "$REPO_ROOT/scripts/patient_cluster_bootstrap_cross_sectional_v3_direct.py" \
  --output-dir "$ANALYSIS_ROOT" \
  --dataset-csv "$ANALYSIS_ROOT/dataset_relinked.csv" \
  --out-dir "$ANALYSIS_ROOT/bootstrap_clustered_cross_sectional" \
  --n-bootstrap 10000 \
  --seed 20260809
```

The bootstrap samples the 40 underlying patients with replacement and keeps all studies and lesions belonging to each sampled patient. Split case units from the same patient remain in the same cluster. The script refuses to bootstrap unless the locked point estimates are reproduced.

### 5. Recompute whole-burden longitudinal change

```bash
python "$REPO_ROOT/scripts/rescore_longitudinal_volume_change_relinked_v2.py" \
  --output-dir "$ANALYSIS_ROOT" \
  --dataset-csv "$ANALYSIS_ROOT/dataset_relinked.csv" \
  --out-dir "$ANALYSIS_ROOT/volume_change_relinked170"
```

This is a case-unit total tumor-core burden analysis—not a per-lesion volume-change analysis. It pairs chronologically consecutive available-reference observations and separately records strict adjacency.

### 6. Run the reference-anchored modified RANO-BM analysis

Keep the RANO scripts together because the v4 analysis imports shared functions from `ranobm_endpoint.py`.

```bash
python "$REPO_ROOT/scripts/ranobm_endpoint_v4_reference_anchored.py" \
  --output-dir "$ANALYSIS_ROOT" \
  --dataset-csv "$ANALYSIS_ROOT/dataset_relinked.csv" \
  --out-dir "$ANALYSIS_ROOT/ranobm_out_relinked_v4"

python "$REPO_ROOT/scripts/finalize_ranobm_v4_primary_v2.py" \
  --v4-dir "$ANALYSIS_ROOT/ranobm_out_relinked_v4" \
  --out-dir "$ANALYSIS_ROOT/ranobm_out_relinked_v4_final"

python "$REPO_ROOT/scripts/audit_ranobm_new_lesions_v4.py" \
  --output-dir "$ANALYSIS_ROOT" \
  --dataset-csv "$ANALYSIS_ROOT/dataset_relinked.csv" \
  --out-dir "$ANALYSIS_ROOT/ranobm_new_lesion_audit_v4"
```

The primary trajectory analysis is reference-anchored: missing reference components are not assumed to represent disappearance, while model absence at a reference-positive follow-up is scored as 0 mm. A reference lesion requires at least two positive reference observations, and entry detection failures are reported separately.

## Locked analysis choices

- Cyprus tumor core: labels 1 (necrotic core) + 2 (enhancing tumor); edema label 3 excluded.
- Lesion components: 26-connectivity for the cross-sectional analysis.
- Detection: one-to-one Hungarian assignment with Jaccard ≥0.10.
- Lesion-volume strata: `<0.05`, `0.05–<0.5`, `0.5–<4`, and `≥4` mL.
- Bootstrap: 10,000 patient-level resamples, percentile 95% confidence intervals, seed `20260809`.
- Modified lesion-level RANO-BM: measurable diameter ≥10 mm; improvement ≥30% decrease from baseline; progression ≥20% increase from nadir plus ≥5 mm absolute increase.
- Whole-burden categories: CR at decrease to zero; PR at ≥65% decrease; PD at ≥40% increase plus >0.10 mL absolute increase; SD otherwise; NEW from zero to nonzero burden.

See [docs/EXPECTED_RESULTS.md](docs/EXPECTED_RESULTS.md) for the manuscript-locked validation targets.

## Privacy and release boundaries

The repository contains code only. Case-unit values such as `P01` are pseudonymous identifiers defined by the released research dataset and are used solely to reproduce clustering and audit logic. No names, dates of birth, medical-record numbers, contact details, credentials, local user paths, or institutional compute paths are included.

Generated manifests and outputs may contain dataset case-unit identifiers and local paths. They are ignored by default and should be reviewed separately before any distribution.

## Intended use

This software is for research and reproducibility only. It is not a medical device and is not intended for clinical diagnosis, treatment selection, or autonomous screening.

## Citation and archival release

The citation metadata for this software is provided in [`CITATION.cff`](CITATION.cff). Zenodo-specific release metadata are provided in [`.zenodo.json`](.zenodo.json); when both files are present, Zenodo's GitHub integration uses `.zenodo.json`.

Version 1.0 is the manuscript-associated frozen analysis release. After the GitHub `v1.0` release is archived, cite the version-specific Zenodo DOI shown on the release record. The DOI is intentionally not hard-coded here before Zenodo mints it.

## License

Released under the [MIT License](LICENSE).
