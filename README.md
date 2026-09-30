# Count-Level Effects in Emission Tomography Reconstruction: FBP vs MLEM

How does the number of detected counts affect image quality when reconstructing simulated 2D PET data with filtered back-projection (FBP) versus maximum-likelihood expectation maximisation (MLEM)?

<img width="1489" height="455" alt="image" src="https://github.com/user-attachments/assets/60c8ef60-7f2f-46af-8813-b07e75efd438" />
Figure1. the result of mean and sd. of 10 simulated noise experiments. 



## Key findings
- MLEM's advantage over FBP grows with count level.
- At 10⁴ counts, MLEM's lower NRMSE does not mean a better image.
- FBP noise follows Poisson statistics.

## Background
- **PET and count statistics:** positron annihilation, coincidence detection, lines of response; detected counts follow a Poisson distribution, so relative noise scales as 1/√N [3].
- **FBP:** analytic inversion of the Radon transform; fast, but does not model Poisson noise [2,4].
- **MLEM:** is based on Poisson statistical properties of PET data, and theoretically ensures that at each iteration the 'likelihood' either increases or remains unchanged. [6]

## Method

### Simulation
| Parameter | Value |
|---|---|
| Phantom | Shepp–Logan (scikit-image), resized to 128 × 128 pixels, zero outside the circular field of view |
| Projection angles | 180 over 0–179° (1° steps) |
| Total counts | 10⁴, 10⁵, 10⁶ |
| Noise | Poisson, applied to the noise-free sinogram after scaling it to the target total counts |
| Noise realisations | 10 per count level; FBP and MLEM reconstruct the same realisations (paired design) |
| Random seeds | `SEED = 0`. Realisation *r* at count level *i* uses `np.random.default_rng([0, i, r])` |

### Reconstruction
- **FBP:** did comparisons of five built-in filters on noise-free data. Multiply the remp filter by the Hahn window with different value of the Nyquist frequency. The validity of this filter was verified by omitting the window function and comparing the result with `iradon(filter_name=‘ramp’)`: the maximum relative error was 2.6 × 10⁻¹⁵.
- **MLEM:** implemented from scratch in NumPy (`src/mlem.py`), update rule:
  x⁽ᵏ⁺¹⁾ = x⁽ᵏ⁾ / (Aᵀ1) · Aᵀ( y / (A x⁽ᵏ⁾) )
  where A is forward projection (`radon`) and Aᵀ is unfiltered back-projection.
- **Validation:**
  - Adjoint test. For random x and y, ⟨Ax, y⟩ / ⟨x, Aᵀy⟩ = 114.54, constant to within 0.005% across draws. iradon(filter_name=None) is therefore the adjoint of radon up to a constant factor, and that factor cancels between the numerator and the sensitivity image Aᵀ1.
  - Monotonicity. On noise-free data, NRMSE decreased and the Poisson log-likelihood increased at every one of 100 iterations (figure).

### Evaluation metrics
- NRMSE: RMSE / RMS(truth) inside the field of view, compared against the ground-truth phantom.
- Contrast recovery (CR): CR = (hot/background − 1) in the reconstruction ÷ (hot/background − 1) in the truth. The hot region is the large 0.3-intensity ellipse in the upper half of the phantom. The background is the 0.2-intensity "brain" region, which gives a true contrast of 0.5. Both regions are eroded by 3 pixels to exclude partial-volume edge pixels (330 px and 3183 px after erosion).
- Background noise: standard deviation of pixel values within the uniform 0.2-intensity region.

## Results
| Counts | Method | Setting | NRMSE | Contrast recovery | Background SD |
|---|---|---|---|---|---|
| 10⁴ | FBP | Hann, cut-off 0.25 | 0.603 ± 0.004 | 0.91 ± 0.30 | 0.070 ± 0.006 |
| 10⁴ | MLEM | 5 iter | 0.586 ± 0.004 | 0.51 ± 0.19 | 0.081 ± 0.003 |
| 10⁴ | MLEM | 100 iter | 2.262 ± 0.038 | 0.99 ± 0.37 | 0.650 ± 0.014 |
| 10⁵ | FBP | Hann, cut-off 0.5 | 0.449 ± 0.004 | 1.00 ± 0.10 | 0.057 ± 0.002 |
| 10⁵ | MLEM | 15 iter | 0.381 ± 0.003 | 0.87 ± 0.09 | 0.069 ± 0.002 |
| 10⁵ | MLEM | 100 iter | 0.993 ± 0.014 | 0.92 ± 0.13 | 0.284 ± 0.008 |
| 10⁶ | FBP | Hann, cut-off 1.0 | 0.298 ± 0.001 | 1.01 ± 0.05 | 0.046 ± 0.001 |
| 10⁶ | MLEM | 30 iter | 0.213 ± 0.002 | 1.05 ± 0.05 | 0.038 ± 0.001 |
| 10⁶ | MLEM | 100 iter | 0.339 ± 0.005 | 1.01 ± 0.06 | 0.095 ± 0.002 |

<img width="1281" height="1004" alt="image" src="https://github.com/user-attachments/assets/6f544b17-be18-48f7-87f8-03d8f82e19b7" />

Figure2. One noise realisation (realisation 0 of the experiment) per count level, reconstructed with the same settings as the table. At 10⁴ counts neither method recovers the internal structure. At 100 iterations MLEM has fitted the noise at every count level. The per-image CR values (e.g. 1.24 for FBP at 10⁴) differ from the table means because single-realisation CR is dominated by noise at low counts.

## Limitations

- Inverse crime. The data were simulated with the same radon operator used for reconstruction, on the same 128 × 128 grid. Real data never match the reconstruction model exactly, so the MLEM results are optimistic. Simulating on a finer grid and reconstructing on a coarser one would remove this.
- Settings were chosen with knowledge of the ground truth. The "optimal" filter cut-off and iteration number were selected by minimum NRMSE on the same realisations used to report results. This is impossible with real data and slightly flatters both methods.
- The background SD mixes noise with blur. With strong smoothing (FBP cut-off ≤ 0.15), blurred edges from neighbouring structures leak into the eroded background region, so background SD increases as smoothing increases. A pixel-wise ensemble SD across realisations would separate noise from bias.
- The hot region is large (330 px), so contrast recovery is insensitive to resolution loss except under heavy smoothing. A small lesion would be a more demanding test.
- 2D simulation only; real PET is 3D.
- No attenuation, scatter, random coincidences or detector response modelled.
- Single synthetic phantom; results may not generalise to clinical images.

## Reproducing the results
 
```bash
git clone https://github.com/Windy0628/PET-Image-Reconstructions-FBP-vs-Iterative-Reconstruction-MLEM-OSEM-.git
cd PET-Image-Reconstructions-FBP-vs-Iterative-Reconstruction-MLEM-OSEM-
pip install -r requirements
jupyter notebook notebooks/main.ipynb
```
Or open `notebooks/main.ipynb` in Google Colab and select **Runtime → Run all**.
Expected runtime: about 5 minutes on the development machine, most of it in the 10-realisation experiment (Step 6). Runtime on Colab has not been measured.

## Repository structure
 
```
├── notebooks/
│   └── main.ipynb        # full pipeline: simulation → reconstruction → evaluation → figures
├── figures/              # generated plots (written by the notebook)
├── requirements.txt
└── README.md
```
## References

1. Shepp, L. A. & Vardi, Y. (1982). Maximum likelihood reconstruction for emission tomography. IEEE Transactions on Medical Imaging, 1(2), 113–122.
2. Bruyant, P. P. (2002). Analytic and iterative reconstruction algorithms in SPECT. Journal of Nuclear Medicine, 43(10), 1343–1358.
3. Cherry, S. R., Sorenson, J. A. & Phelps, M. E. (2012). Physics in Nuclear Medicine (4th ed.). Elsevier Saunders.
4. Kak, A. C. & Slaney, M. (1988). Principles of Computerized Tomographic Imaging. IEEE Press.
5. van der Walt, S. et al. (2014). scikit-image: image processing in Python. PeerJ, 2, e453.
6. Bohrium Encyclopedia. PET image reconstruction methods and corrections]. In Foundations of Medical Imaging. Bohrium SciencePedia. https://www.bohrium.com/sciencepedia/feynman/foundations_of_medical_imaging_undergraduate-PET_image_reconstruction_methods_and_corrections.


# TBC: further work about Ordered-Subsets Expectation-Maximization, OSEM
