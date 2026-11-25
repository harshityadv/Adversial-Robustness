# Certified Robustness to Geometric Transformations and Adversarial Patches

A research project for evaluating whether image classifiers can be certified against a geometric transformation and an adversarial patch applied together. The notebooks compare joint robustness certificates with single-threat certificates, empirical attacks, and a PatchCleanser-style masked classifier.

> **Project status:** research notebooks and supplied reference documents. The primary notebook contains executable experiments; this repository does not include trained weights or downloaded datasets.

## Table of contents

- [Research question](#research-question)
- [Threat model](#threat-model)
- [Certification and evaluation](#certification-and-evaluation)
- [Experiments](#experiments)
- [Repository contents](#repository-contents)
- [Requirements](#requirements)
- [Run the notebook](#run-the-notebook)
- [Configuration](#configuration)
- [Generated artifacts](#generated-artifacts)
- [Reproducibility and limitations](#reproducibility-and-limitations)
- [Reference documents](#reference-documents)

## Research question

A classifier may be robust to a transformation when tested alone and robust to a patch when tested alone, yet fail when both changes occur together. This project measures that interaction directly. Its central comparison is between:

1. a **joint certificate**, which accounts for transformation and patch uncertainty at the same time; and
2. the **naive conjunction** of separate transformation-only and patch-only certificates.

The notebook records cases where both separate certificates hold but an attack against the combined threat succeeds (`naive_overclaim`). This lets the experiment quantify when separate guarantees do not establish a joint guarantee.

## Threat model

For each image, the geometric parameters are enclosed in a box

\[
\kappa = (\theta, \Delta x, \Delta y, s),
\]

where the notebook configuration sets rotation to `[-theta_deg, theta_deg]`, each translation to `[-tau, tau]` pixels, and scale to `[1 - sigma, 1 + sigma]`. The image is warped with bilinear sampling and zero padding.

The patch is a square of side `k` whose contents can take any values in `[0, 1]`. The notebook evaluates two application orders:

- **Order (a):** transform the image, then apply the patch in output coordinates.
- **Order (b):** patch the image first, then transform it, so the patch moves with the image.

For order (b), the notebook compares three certification bounds: `lemma`, `footprint`, and `weighted`. The effective output patch size for the lemma bound is computed from the configured patch size and the maximum rotation and scale. The separate `mask` training mode evaluates a PatchCleanser-style covering set of masks; its certificate applies to the defended, masked prediction procedure.

## Certification and evaluation

The geometric bounder computes per-pixel lower and upper bounds over a parameter box. It uses interval bounds on the bilinear warp and can optionally add a first-order Taylor model with a remainder bound for pixels that stay within a fixed bilinear cell. These image bounds feed interval bound propagation (IBP) through the classifier. Optional CROWN verification can be enabled for tighter bounds on difficult cases.

The certifier subdivides parameter boxes up to configured depth and node budgets. It first tries concrete counterexamples for failing cases and then refines the remaining failing boxes. The notebook also includes empirical attacks that search transformation parameters, patch positions, and patch contents.

Reported quantities include:

- clean accuracy on the evaluated test subset;
- certified accuracy for transformation-only, patch-only, joint, and mask-based defenses;
- `naive_both`, the accuracy certified by both separate threat models;
- empirical robust accuracy under the matching attack search;
- `naive_overclaim`, cases where separate certificates hold but the joint attack succeeds;
- soundness-bug counts, where an attack breaks an item declared certified; and
- runtime, search nodes, confidence intervals, and sensitivity results where applicable.

A certificate is a lower bound on robust accuracy. The attack result is an empirical estimate and therefore an upper bound on the robust accuracy under the searched attack procedure; it is not an exhaustive proof that no adversarial example exists.

## Experiments

The primary notebook runs the following studies:

| Study | Comparison |
| --- | --- |
| Training modes | Standard, transform-augmented, patch-augmented, joint, and mask-based models |
| Main evaluation | Clean, transform-only, patch-only, joint certification, and matching attacks |
| Multiple seeds | Mean and standard deviation of core metrics across seeds |
| Range study | Joint certification as rotation, translation, and scale ranges change |
| Geometric-only benchmark | Rotation-only certification and attack estimates without patch, translation, or scale |
| Search budget | Certification results as subdivision depth and node budgets change |
| Certified frontier | Maximum certified rotation range across patch sizes |
| Ablations | Sensitivity to patch size, rotation, and translation |
| Patch order and windows | Order-(b) bounds and free-window versus mask-window results |
| Internal checks | Geometry-bound sampling, IBP checks, order-(b) footprint checks, masks, Taylor bounds, and falsification checks |

The studies are implemented in notebook cells and controlled by flags described under [Configuration](#configuration). Some are computationally expensive and train additional models.

## Repository contents

```text
.
├── README.md
├── requirements.txt
├── optional-requirements.txt
├── .gitignore
├── docs/
│   ├── Certified Robustness to a Geometric Transformation Combined with an Adversarial.pdf
│   └── Results and Findings Certified Robustness to a Geometric Transformation and an A.pdf
├── notebooks/
│   ├── alpha_11.ipynb       # Primary experiment; defaults to Fashion-MNIST
│   └── alpha10.ipynb        # Earlier version; defaults to MNIST
└── outputs/                 # Runtime artifacts; contents are ignored by Git
```

`alpha_11.ipynb` is the recommended entry point. `alpha10.ipynb` is retained as the earlier MNIST-focused notebook for comparison. Both notebooks are included with their original saved cell outputs.

## Requirements

The core dependencies are listed in `requirements.txt`:

- Python 3
- PyTorch and Torchvision
- NumPy and pandas
- Matplotlib
- JupyterLab

Install them with:

```bash
python -m pip install -r requirements.txt
```

The CROWN path is optional and imports `auto_LiRPA` only when enabled. Install it separately if needed:

```bash
python -m pip install -r optional-requirements.txt
```

Choose the PyTorch build that matches your CPU or CUDA environment using the [official PyTorch installation selector](https://pytorch.org/get-started/locally/). Dependency versions are not currently pinned, so exact environments may differ across machines.

## Run the notebook

1. Clone or download this repository.
2. Install the core requirements.
3. Start JupyterLab from the repository root:

   ```bash
   jupyter lab
   ```

4. Open `notebooks/alpha_11.ipynb` and run the cells in order.

The default configuration uses Fashion-MNIST (`DATASET = "fmnist"`). Torchvision downloads the selected dataset under `data/` on first use. Set `DATASET` to `"mnist"` or `"cifar10"` to select another supported dataset. Set `SYNTHETIC = True` to use generated data instead of a downloaded dataset.

The default `QUICK = False` configuration runs the larger experiment. For a smaller exploratory run, set `QUICK = True` in the configuration cell. This reduces epochs and evaluation sample sizes, but it still runs enabled studies and can take time. To shorten it further, set unneeded `RUN_*` flags to `False` in their corresponding cells. Run all cells from the top after changing configuration.

The notebook selects available CUDA devices automatically and falls back to CPU. A compatible GPU is recommended for the full study; CPU execution may be slow.

## Configuration

The primary notebook's main defaults are:

| Setting | Default | Meaning |
| --- | ---: | --- |
| `DATASET` | `"fmnist"` | `"mnist"`, `"fmnist"`, or `"cifar10"` |
| `QUICK` | `False` | Full study; `True` uses reduced epochs and sample counts |
| `SYNTHETIC` | `False` | Downloaded dataset by default, generated data when enabled |
| `theta_deg` | `0.5` | Maximum absolute rotation in degrees |
| `tau` | `0.1` | Maximum absolute translation per axis, in pixels |
| `sigma` | `0.005` | Scale deviation around 1 |
| `k` | `2` | Patch side length in pixels |
| `order` | `"a"` | Patch order; the notebook also evaluates order (b) |
| `epochs` | `30` | Training epochs for the standard dataset configuration |
| `n_val` | `2000` | Validation examples |
| `n_eval` | `500` | Test examples in the main evaluation |
| `n_abl` | `300` | Test examples in study and ablation evaluations |
| `n_seeds` | `3` | Seeds in the multi-seed evaluation |
| `max_depth` | `8` | Maximum subdivision depth for certification |
| `max_nodes` | `600` | Maximum search nodes for certification |

The CIFAR-10 configuration changes some training and attack defaults. The notebook notes that CIFAR-10 has not been run in the supplied results, so treat it as an available configuration rather than a reproduced result.

## Generated artifacts

The notebook writes to `outputs/` when run locally and to `/kaggle/working` when it detects a Kaggle environment. Depending on which study flags are enabled, it can produce:

- model checkpoints (`.pt`);
- CSV tables for main results, seeds, calibration, range studies, geometric benchmarks, ablations, patch-order comparisons, frontiers, and mask-window comparisons;
- PNG plots;
- a JSON summary and text tables; and
- a ZIP bundle of the generated result tables and plots.

Downloaded datasets, checkpoints, and generated outputs are excluded from version control by `.gitignore`. Keep result files outside Git unless you intentionally want to publish a particular run; include the configuration and environment details alongside published results.

## Reproducibility and limitations

- The notebooks set random seeds, but GPU kernels can be nondeterministic. The primary notebook notes variation of about 0.015 in certified accuracy at 500 evaluated examples in repeated runs, with larger variation possible at wider ranges.
- Results depend on hardware, PyTorch and Torchvision versions, numerical precision, dataset, and enabled study flags. Save the notebook configuration with any results you report.
- The full experiment runs many certification and attack searches and may train additional models. QUICK mode is intended for exploration and should not be presented as the full evaluation.
- The interval/taylor bounds use floating-point computations; the notebook notes that floating-point rounding is not modeled in the bounds.
- The certified geometric range in this project is intentionally limited. The notebook frames its rotation-only study as a comparison point, not a claim to outperform dedicated geometric verifiers.
- A mask certificate is for the PatchCleanser-style defended classifier, not the raw classifier.
- The provided notebook outputs are historical artifacts. Re-run the notebooks to generate results for your own environment and settings.

## Reference documents

The supplied PDFs are retained under [`docs/`](docs/):

- [Certified Robustness to a Geometric Transformation Combined with an Adversarial Patch](docs/Certified%20Robustness%20to%20a%20Geometric%20Transformation%20Combined%20with%20an%20Adversarial.pdf)
- [Results and Findings: Certified Robustness to a Geometric Transformation and an Adversarial Patch](docs/Results%20and%20Findings%20Certified%20Robustness%20to%20a%20Geometric%20Transformation%20and%20an%20A.pdf)

These documents accompany the notebooks as project references. Use the notebook configuration and outputs as the executable record for a specific run.
