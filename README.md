# Lung CT classification project

Predict per-nodule reader malignancy ratings from LIDC-IDRI and LNDb CT images. The targets are reader assessments, not confirmed clinical diagnoses. Detailed decisions and limitations are in [methodology](docs/methodology.md).

## Setup

Run commands from `project/`:

```sh
uv sync
```

`uv sync` installs the project's Python package in editable mode. Use `.venv/bin/python` as the notebook kernel. All processing writes stay under `project/`; downloaded datasets in `data/` are read-only.

## Notebook workflow

1. `01_data_quality_and_annotations.ipynb`: XML/pylidc verification, reader grouping, labels, and manual annotation review.
2. `02_ct_preprocessing_and_crops.ipynb`: HU and geometry inspection, followed by reader-union contour crops with a 6 mm CT margin per face and unchanged mask foreground.
3. `03_feature_extraction.ipynb`: contour radiomics, verified configuration caches, and extraction-quality summaries.
4. `04_model_validation.ipynb`: feature-count, target, and hyperparameter comparisons on selection patients with assessment figures, then the 5-class contour model validated once with its fairness audit and interpretability (comparison with reader characteristics, grouped permutation importance, PCA, and SHAP), and finally the 3-class triage model, chosen on selection patients and validated once.
5. `05_4mm_pipeline.ipynb`: competition notebook, independent of notebooks 02 to 04. It extracts 4 mm and 2 mm sphere features (optional crop regeneration, 0 mm filter context), compares ExtraTrees hyperparameters one at a time on selection patients, validates once, and writes `predictions.csv`.

`final_solution.ipynb` was removed at the user's request. The five notebooks above are the active workflow.

Notebook 05 normally reuses saved crops and verified feature tables. Set `REGENERATE_CROPS=True` only when rebuilding from source CT data. Stop after extraction for a feature-only run. To model existing exports, run setup and configuration, then continue at the modeling section. Settings have one definition in each notebook's configuration cell.

## Distinct model pipelines

Notebook 02 includes a geometry-policy cell that excludes four audited LIDC scans and repairs 0267 and 0572 while retaining source files and recovery copies. It can update existing contour crops without a full CT rerun. Rerun notebook 03 after applying it. See [methodology](docs/methodology.md#generated-tables) for scope and provenance.

- Notebook 04 uses `fit_contour_model` and `evaluate_contour_holdout` in `modeling_feng.py`: median imputation, correlation filtering, information-gain selection, scaling, PCA, and tree regression. Optional training-only SMOTE and target mappings are supported.
- Notebook 05 uses `fit_sphere_model` and `evaluate_sphere_holdout` in `modeling.py`: fitting-only constant/correlation filtering, median imputation, StandardScaler, dataset-specific SMOTE, and ExtraTrees regression. It defaults to the unrounded reader mean as its training target.

`evaluation.py` shares metrics, threshold calibration, patient bootstrap intervals, and the holdout procedure. `experiments.py` owns run snapshots, comparison runs, provenance, and figures. Historical saved model class names remain loadable through compatibility aliases.

Both workflows use frozen fitting, threshold, selection, and validation patient groups. Threshold patients calibrate cut-points; selection patients choose a configuration; validation patients are scored once. Selection-only comparisons never predict validation patients. The final refit excludes every threshold patient.

Configure split fractions in `config/contour_holdout.json` and `config/sphere_holdout.json`. Use `config/scan_processing.json` for CT geometry and the contour margin, and `config/radiomics.yaml` for radiomics. The sphere context is configured separately in notebook 05. Patient manifests are created on the first run and reused without reassignment.

## Commands

```sh
uv run python -m lung_project review
uv run python -m lung_project rebuild-lidc --refresh-derived
uv run python -m lung_project audit-lndb
uv run python -m lung_project audit-alignment
uv run python -m lung_project refresh-characteristics
uv run python -m lung_project refresh-contour-split
```

Use `uv run python -m lung_project <command> --help` for options. `review` covers LIDC membership and malignancy, then LNDb findings with wide rating disagreement. Review decisions are saved immediately in `state/annotation_decisions.json`. `refresh-contour-split` targets the current contour holdout and archives previous assignments before applying explicit review exclusions.

## Outputs and reproducibility

- `outputs/prepared/`: canonical annotations, crops, radius feature CSVs, contour configuration caches, and the hash-checked `stages.json` registry.
- `outputs/images/`: standardized/native crops, review assets, and generated figures.
- `outputs/experiments/<run>/`: configuration and input snapshots, provenance, scores, predictions, and fitted models. Existing run directories cannot be overwritten.
- `outputs/reports/`: processing audits and patient manifests.
- `state/`: human decisions and recovery archives.
- `config/references/`: checksum-pinned publisher annotations and the official test candidate list.
- `deliverables/`: team video link, ethics/regulation document, and professors' self-evaluation when prepared.

Notebook 05 saves draft predictions inside its run directory. `EXPORT_SUBMISSION=True` writes `predictions.csv` after checking official candidate ordering, preserved source columns, and integer predictions from 1 to 5. This flag is editable in the configuration cell. Rows without image predictors are explicitly audited as training-median fallbacks.

Current prepared tables, images, historical experiments, decisions, recovery records, and `outputs/6mmimages/` are preserved. `list_contour_settings()` distinguishes available and missing historical cache exports. Its availability report does not delete or rewrite historical metadata.

See [organization](docs/organization.md) for locations and migration guidance, and [history](docs/history/code_cleanup_20261015.md) for the cleanup record. LNDb-derived outputs remain subject to the source dataset terms.

## Checks

```sh
uv run python -m unittest discover -s tests
```

The project tests use synthetic fixtures and temporary output directories. They do not depend on external quickscripts or full downloaded CT datasets.
