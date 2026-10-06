# GeoFilter-ET

<p align="center">
  <strong>Acquisition-geometry-aware post-reconstruction filtering for cryo-electron tomography</strong>
</p>

<p align="center">
  An interpretable, training-free framework for using measured cryo-ET tilt geometry to identify direction-dependent Fourier support and guide conservative post-reconstruction attenuation.
</p>

<p align="center">
  <img alt="Python" src="https://img.shields.io/badge/Python-3.11+-blue">
  <img alt="Jupyter" src="https://img.shields.io/badge/Jupyter-Notebook-orange">
  <img alt="Status" src="https://img.shields.io/badge/status-research%20prototype-yellow">
  <img alt="License" src="https://img.shields.io/badge/license-TBD-lightgrey">
</p>

---

## Overview

**GeoFilter-ET** investigates a simple question:

> Can the *known acquisition geometry* of a cryo-electron tomography experiment tell us where Fourier-domain attenuation should be stronger or weaker?

Limited-angle cryo-ET does not sample Fourier space uniformly. Each projection contributes a central Fourier plane, while finite tilt range and angular spacing create strongly direction-dependent support. GeoFilter-ET converts the **actual measured tilt angles and tilt-axis orientation** into a three-dimensional Fourier reliability model and uses that information for post-reconstruction filtering.

The current implementation is deliberately conservative:

- it is **training-free**;
- it uses measured acquisition geometry rather than a learned image prior;
- it is designed for **post-reconstruction filtering**;
- it does **not** synthesize missing Fourier coefficients;
- it therefore does **not claim missing-wedge restoration**.

The repository accompanies the GeoFilter-ET methodological study and contains the experimental preparation, synthetic validation, matched-control analyses, and development history used to evaluate the approach.

---

## Why GeoFilter-ET?

A conventional isotropic low-pass filter treats all Fourier directions equally. In cryo-ET, however, those directions are not equally supported by the acquisition.

GeoFilter-ET models this explicitly.

For a Fourier coordinate \(\mathbf{k}\), the distance to each experimentally sampled central plane is calculated and the minimum distance is retained:

\[
D_{\min}(\mathbf{k}) =
\min_i \left|\mathbf{k}\cdot\mathbf{n}_i\right|
\]

where \(\mathbf{n}_i\) is the normal of the central Fourier plane corresponding to tilt angle \(i\).

A smooth geometry reliability is then defined as

\[
R_0(\mathbf{k}) =
\exp\left[
-\frac{1}{2}
\left(
\frac{D_{\min}(\mathbf{k})}{\sigma_F}
\right)^2
\right]
\]

followed by light three-dimensional smoothing.

The frozen geometry-aware transfer used during development is

\[
H_{\mathrm{geom}}(\mathbf{k}) =
\exp\left[
-\left(1-R(\mathbf{k})\right)q(\mathbf{k})^2
\right]
\]

where \(q\) is spatial frequency normalized to Nyquist.

The filter therefore attenuates **high-frequency, poorly supported directions more strongly**, while naturally preserving low frequencies.

---

## Acquisition-derived support criterion

One of the most reproducible findings of the study is that the Fourier-space distance can also be expressed as an angular distance to the nearest sampled central plane:

\[
\theta_{\mathrm{nearest}}(\mathbf{k}) =
\sin^{-1}
\left(
\frac{D_{\min}(\mathbf{k})}{|\mathbf{k}|}
\right)
\]

For the experimental acquisition used in development:

| Acquisition property | Value |
|---|---:|
| Number of tilts | 41 |
| Tilt range | −74.00° to +45.99° |
| Mean tilt spacing | 2.99975° |
| Nominal tilt axis | −95.89° |
| Equivalent unoriented axis | 84.11° |
| In-plane correction used | −5.89° |
| Working volume | 256 × 256 × 256 |

A Fourier direction is classified as **poorly supported** when its nearest-plane angular distance exceeds half the measured mean tilt spacing:

\[
\theta_{\mathrm{nearest}} > \frac{\Delta\theta}{2}
\]

For this acquisition,

\[
\frac{\Delta\theta}{2}=1.49988^\circ
\]

and approximately **30.76%** of Fourier directions inside the Nyquist sphere satisfy this criterion.

Importantly, this boundary is defined from **acquisition metadata**, not from ground-truth optimization.

---

## Key result

The strongest current result is not that GeoFilter-ET universally outperforms Gaussian denoising.

Instead, the study identifies a highly reproducible **direction-dependent filtering regime**.

In acquisition-defined poorly supported directions, geometry-aware attenuation produced lower spectral error than a fair Gaussian control across normalized spatial-frequency shells from \(q=0.4\) to \(1.0\). In normally supported directions, Gaussian filtering was better.

### Reproducibility across two independent SNR=10 noise realizations

| Frequency shell | Poor support — seed 314159 | Poor support — seed 271828 | Supported — seed 314159 | Supported — seed 271828 |
|---|---:|---:|---:|---:|
| 0.4–0.5 | 1.0180 | 1.0180 | 0.8996 | 0.8992 |
| 0.5–0.6 | 1.0749 | 1.0750 | 0.7653 | 0.7657 |
| 0.6–0.7 | 1.1582 | 1.1592 | 0.6855 | 0.6861 |
| 0.7–0.8 | 1.2502 | 1.2505 | 0.6549 | 0.6554 |
| 0.8–0.9 | 1.4002 | 1.4006 | 0.6510 | 0.6527 |
| 0.9–1.0 | 1.6606 | 1.6591 | 0.6886 | 0.6897 |

Values are **Gaussian spectral error / geometry-filter spectral error**.

- **> 1:** geometry-aware attenuation has lower error.
- **< 1:** Gaussian filtering has lower error.

Across \(q=0.4-1.0\), the ratios were approximately:

- **Poor support:** 1.11837 and 1.11833
- **Supported:** 0.75527 and 0.75577

The near-identical behavior across independent noise seeds suggests that the separation is primarily associated with **acquisition support**, rather than a particular random noise realization.

---

## Fair-control result

A central principle of this project is that candidate filters must not appear better merely because they smooth more strongly.

Filtering strength is therefore compared using

\[
\mathrm{SD}\left(V_{\mathrm{raw}}-V_{\mathrm{filtered}}\right)
\]

and Gaussian controls are numerically matched to the same removed-signal standard deviation.

The acquisition-selective **GeoFilter-ET v4** used:

- Gaussian transfer in normally supported directions;
- geometry-aware transfer in poor-support directions.

After **exact filtering-strength matching**, the Gaussian control remained slightly better globally:

| Dataset | Method | NRMSE ↓ | Pearson ↑ | Structure RMSE ↓ | Background RMSE ↓ |
|---|---|---:|---:|---:|---:|
| Seed 314159 | Matched Gaussian | **0.652549** | **0.757746** | **0.385153** | **0.028500** |
| Seed 314159 | GeoFilter-ET v4 | 0.652727 | 0.757593 | 0.385194 | 0.028515 |
| Seed 271828 | Matched Gaussian | **0.652631** | **0.757676** | **0.385176** | **0.028506** |
| Seed 271828 | GeoFilter-ET v4 | 0.652810 | 0.757522 | 0.385217 | 0.028522 |

This negative result is intentionally retained.

It shows that **correctly identifying where geometry matters is not sufficient to guarantee a globally superior filter**. The remaining problem is how strongly the acquisition support should modulate a radial denoising baseline.

---

## Experimental proof of concept

The experimental workflow was developed using **CryoET Data Portal dataset 10528, Position056**.

A 256³ subvolume was extracted from the experimental reconstruction and processed with the original adaptive formulation.

At equal removed-signal standard deviation, the initial GeoFilter formulation retained strong gradients better than the matched Gaussian control:

| Gradient subset | GeoFilter-ET | Matched Gaussian | Difference |
|---|---:|---:|---:|
| Top 5% | 95.68% | 91.45% | +4.23 pp |
| Top 2.5% | 96.14% | 91.70% | +4.44 pp |
| Top 1% | 96.94% | 92.22% | +4.71 pp |
| Top 0.5% | 97.69% | 92.78% | +4.90 pp |

These values are **proof-of-concept structural-preservation measurements**, not ground-truth accuracy measurements, because the experimental reference itself contains noise and reconstruction artifacts.

---

## Synthetic validation

The controlled synthetic benchmark uses a **256³ multi-component phantom** containing:

- a thin spherical membrane;
- a small solid sphere;
- a hollow tube;
- a curved membrane-like sheet.

A Gaussian point-spread function is applied to create a continuous reference volume.

The phantom is projected using the **exact experimental tilt geometry**, reconstructed with a controlled weighted back-projection workflow, and evaluated against known ground truth.

The synthetic design allows separate measurement of:

- global NRMSE;
- Pearson correlation;
- structure RMSE;
- background RMSE;
- component-specific RMSE;
- frequency-resolved error;
- acquisition-support-specific spectral error.

---

## Development history

GeoFilter-ET was not selected from a single successful experiment. Several candidate ideas were explicitly tested and rejected.

### 1. Structure-tensor adaptive confidence

The first formulation used a 3D structure tensor to identify locally coherent structures and blend the original reconstruction with the geometry-filtered volume.

Under SNR=10, the confidence correctly assigned high values to true structures but also generated many high-confidence background false positives.

### 2. Local amplitude significance

A smooth amplitude-significance gate improved confidence specificity, but the resulting filter still lost to a newly strength-matched Gaussian control.

### 3. Residual restoration

Residuals removed by the geometry filter were more reproducible inside true structures than in background. Selective residual restoration produced a small improvement over Fourier-only filtering, but again lost to a fair Gaussian control.

### 4. Global geometry/Gaussian hybrid

A pointwise minimum of geometry and Gaussian transfer functions initially appeared superior. Once filtering strength was matched exactly, the advantage disappeared.

### 5. Acquisition-selective v4

The half-spacing criterion provided a physically interpretable support partition and produced the strongest mechanistic result. However, the hard-switch implementation remained slightly inferior to the exactly strength-matched Gaussian globally.

These negative results are part of the project because they help distinguish **real acquisition-geometry information** from improvements caused simply by stronger smoothing.

---

## Repository layout

A recommended repository structure is:

```text
GeoFilter-ET/
├── README.md
├── LICENSE
├── CITATION.cff
├── requirements.txt
├── environment.yml
│
├── notebooks/
│   ├── 01_prepare_experimental_data.ipynb
│   ├── 02_synthetic_validation.ipynb
│   └── 03_figures_and_metrics.ipynb
│
├── src/
│   ├── geofilter.py
│   ├── geometry.py
│   ├── reconstruction.py
│   ├── metrics.py
│   └── utils.py
│
├── data/
│   └── README.md
│
├── results/
│   ├── figures/
│   └── tables/
│
└── examples/
```

Large experimental tomograms should **not** be committed directly to GitHub. The `data/README.md` should instead provide instructions for obtaining the original public dataset and reproducing the analyzed crop.

---

## Installation

Clone the repository:

```bash
git clone https://github.com/YOUR-USERNAME/GeoFilter-ET.git
cd GeoFilter-ET
```

Create a Conda environment:

```bash
conda create -n geofilter python=3.11 -y
conda activate geofilter
```

Install the core dependencies:

```bash
pip install numpy scipy matplotlib pandas scikit-image mrcfile jupyterlab
```

Register the environment with Jupyter:

```bash
python -m ipykernel install --user \
  --name geofilter \
  --display-name "GeoFilter-ET"
```

Then start Jupyter:

```bash
jupyter lab
```

---

## Quick start

The current analysis is notebook-driven.

Start with:

```text
notebooks/01_prepare_experimental_data.ipynb
```

The workflow prepares the experimental crop and acquisition geometry used by the subsequent GeoFilter-ET analysis.

For full reproducibility, run notebooks in numerical order after downloading the required experimental data described in `data/README.md`.

---

## Frozen development parameters

| Parameter | Value |
|---|---:|
| Working volume | 256³ |
| Fourier reliability width | \(1/N\) |
| Reliability smoothing | σ = 2 voxels |
| Geometry attenuation α | 1.0 |
| Initial tensor pre-smoothing | σ = 1 voxel |
| Initial tensor smoothing | σ = 2 voxels |
| Initial confidence exponent γ | 0.5 |
| Main development projection SNR | 10 |
| Development seeds | 314159, 271828 |
| Locked seed | 20261005 |
| v4 support boundary | half measured tilt spacing |

---

## Reproducibility philosophy

This project follows several rules intended to reduce optimistic method selection:

1. **Use exact acquisition angles**, not an approximate linearly spaced tilt series.
2. **Separate development and locked validation data.**
3. **Record random seeds.**
4. **Compare against newly strength-matched controls.**
5. **Retain negative experiments.**
6. **Do not optimize the acquisition-support boundary against ground truth.**
7. **Do not describe attenuation as missing-wedge restoration.**

---

## What GeoFilter-ET currently supports

GeoFilter-ET currently supports the scientific conclusion that:

> Experimental tilt geometry can identify Fourier directions in which anisotropic attenuation is more useful than isotropic Gaussian smoothing.

The present results **do not yet support** claims that GeoFilter-ET:

- restores the missing wedge;
- recovers unmeasured biological information;
- improves physical resolution;
- universally outperforms Gaussian filtering;
- outperforms modern self-supervised cryo-ET denoisers.

This distinction is intentional.

---

## Next validation steps

The next development stage should include:

- multiple synthetic phantoms and orientations;
- multiple projection SNRs;
- independent forward and reconstruction operators;
- CTF-aware simulation;
- dose-dependent noise;
- detector-response modeling;
- multiple experimental tomograms;
- directional FSC;
- segmentation and localization benchmarks;
- template-matching or subtomogram-analysis endpoints;
- comparison with representative modern self-supervised denoisers.

The locked synthetic realization should be evaluated only after the next transfer formulation is frozen.

---

## Citation

A manuscript describing GeoFilter-ET is currently being prepared.

If you use this repository before publication, please cite the repository:

```text
Gupta, K. K. GeoFilter-ET:
Acquisition-geometry-aware post-reconstruction filtering
for cryo-electron tomography.
GitHub repository, 2026.
```

A `CITATION.cff` file and archival DOI should be added when the first versioned release is deposited.

---

## Data availability

The experimental proof-of-concept uses a publicly available cryo-electron tomography dataset. The raw experimental tomogram is not redistributed in this repository.

The repository should contain:

- acquisition geometry;
- synthetic phantom generation;
- controlled forward projection;
- reconstruction code;
- filtering code;
- exact development seeds;
- evaluation scripts;
- figures and tabulated metrics.

See `data/README.md` for acquisition instructions.

---

## License

A software license should be selected before the first public release.

For an academic open-source research tool, **MIT** is a simple permissive option. If stronger copyleft requirements are preferred, **GPL-3.0** is an alternative.

---

## Author

**Dr. Krishna Kant Gupta**

GeoFilter-ET — acquisition-geometry-aware cryo-electron tomography filtering.

---

## Acknowledgement

GeoFilter-ET is being developed as a reproducible methodological study of how experimentally measured tilt geometry can guide conservative post-reconstruction processing in cryo-electron tomography.

---

<p align="center">
  <strong>GeoFilter-ET</strong><br>
  Geometry knows what the acquisition measured.
</p>
