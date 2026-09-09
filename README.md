# PASTAM: Passive Sensor Tracking and Analysis for MATLAB

A MATLAB toolbox for tracking centroid wander in fixed-pattern dot arrays and computing image-plane scintillation indices from checkerboard calibration targets. Designed for atmospheric turbulence characterisation, FSOC boundary-layer anemometry, and adaptive optics.

---

## Requirements

- MATLAB R2019b or later
- Image Processing Toolbox
- Computer Vision Toolbox

---

## Installation

```matlab
addpath(genpath('src'));
addpath('classes');
```

---

## Complete Pipeline

```matlab
% 1. Load a raw image sequence
A = initPastamFromImageSequence('data', '*.raw');

% 2. Track centroids — checkerboard mode
TM = centroidwander(A, 26, 26, 'median', 0.01, ...
    'polarity', 'both', 'stat_threshold', 0.4, 'dilate_mask', 3, 'verbose', 1);

% 3. Impose grid order (column-major, upper-left origin)
[order, TM] = gridSortTM(TM, 10, 10);

% 4. Inject into pastam; TM can be cleared after this
A = injectTracksIntoPastam(A, TM);

% 5. Wander radius for the scintillation masks
wc    = A.OptimalCentroids;
mu    = mean(wc, 3, 'omitnan');
d     = wc - mu;
r     = sqrt(d(:,1,:).^2 + d(:,2,:).^2);
se_rm = ceil(max(r(:), [], 'omitnan')) + 3;   % peak excursion + margin

% 6. Image-plane scintillation
A = scintillationFromPastam(A, se_rm, 'Ib_mode', 'local', ...
    'disk_bg', 5, 'dW', 26, 'dh', 26, 'verbose', 1);

% 7. Visualise and detect threshold
thresh = showScintillation(A);
```

---

## Repository Structure

```
dots-centroid-wander/
├── src/
│   ├── centroidwander.m            main tracking function
│   ├── ThresholdMask.m             frame-to-frame detection and matching
│   ├── analyzeTrackingQuality.m    quality metrics and plots
│   ├── gridSortTM.m                column-major grid reordering of TM
│   ├── injectTracksIntoPastam.m    TM → pastam field transfer
│   ├── scintillationFromPastam.m   image-plane scintillation index
│   ├── showScintillation.m         comparative visualisation
│   └── init/
│       ├── initPastamFromImageSequence.m   numbered image files + .raw
│       ├── initPastamFromFITS.m
│       ├── initPastamFromVideo.m
│       ├── initPastamFromArray.m
│       └── initPastamSynthetic.m
├── classes/
│   └── pastam.m                    data container class
├── examples/
│   ├── pipeline.m                  complete end-to-end example
│   ├── example_basic.m
│   ├── example_quality_analysis.m
│   └── example_synthetic.m
├── docs/
│   ├── improvements.md
│   └── temporal_validation.md
├── tests/
│   └── test_tracking.m
├── data/
│   └── README.md
├── README.md
├── CHANGELOG.md
├── LICENSE
└── .gitignore
```

---

## The `pastam` Class

`pastam` is a value class that carries the raw image stack and all derived products through the pipeline.

| Field | Populated by | Content |
|---|---|---|
| `Info` | `initPastam*` | metadata struct |
| `Primary` | `initPastam*` | `H×W×T uint16` array or `VideoReader` |
| `Centroids` | `injectTracksIntoPastam` | `N×2×T` geometric centroids `[x y]` |
| `OptimalCentroids` | `injectTracksIntoPastam` | `N×2×T` intensity-weighted centroids `[x y]` |
| `Boxes` | `injectTracksIntoPastam` | `N×4×T` bounding boxes `[x y w h]` |
| `Tracks` | `injectTracksIntoPastam` | archival grid-sorted `TM` table |
| `Scintillation` | `scintillationFromPastam` | struct — see that function |

Because `pastam` is a value class, every function that modifies it must be called as `A = f(A, ...)`.

---

## Function Reference

### Initialisation

#### `initPastamFromImageSequence(folder, pattern, ...)`

Loads a numbered image sequence into `A.Primary` as a `H×W×T uint16` array.

Supports standard formats via `imread` (TIFF, PNG, JPEG, …) and camera-native **12-bit RAW** files (headerless `uint16`, row-major, no embedded metadata).

```matlab
A = initPastamFromImageSequence('data', '*.raw');
A = initPastamFromImageSequence('frames', '*.tif', 'frame_range', [1 500]);
A = initPastamFromImageSequence('data', '*.raw', 'raw_size', [800 800], 'raw_bits', 12);
```

| Parameter | Default | Description |
|---|---|---|
| `method` | `'array'` | `'array'` loads into memory; `'video'` writes an MJ2 archive |
| `frame_range` | all | `[start end]` 1-based indices into the sorted file list |
| `raw_size` | `[800 800]` | sensor dimensions `[H W]` for `.raw` files |
| `raw_bits` | `12` | effective bit depth; upper bits are masked |

---

### Tracking

#### `centroidwander(A, dW, dh, stat, thres, ...)`

Tracks spot centroids through the sequence using adaptive thresholding and frame-to-frame bounding-box matching. Returns `TM`, a table with one row per frame.

```matlab
% Standard bright-on-dark (Shack-Hartmann)
TM = centroidwander(A, 3, 3, 'median', 0.01);

% Checkerboard: bright dots on dark squares + dark dots on white squares
TM = centroidwander(A, 26, 26, 'median', 0.01, ...
    'polarity', 'both', 'stat_threshold', 0.4, 'dilate_mask', 3, 'verbose', 1);
```

**Required arguments**

| Argument | Description |
|---|---|
| `dW` | edge exclusion half-width in pixels (columns) |
| `dh` | edge exclusion half-height in pixels (rows) |
| `stat` | threshold statistic: `'gaussian'`, `'mean'`, or `'median'` |
| `thres` | bounding-box overlap ratio for frame-to-frame matching (0–1) |

**Key optional parameters**

| Parameter | Default | Description |
|---|---|---|
| `polarity` | `'bright'` | `'bright'` standard; `'dark'` complement path; `'both'` checkerboard — detects bright dots on dark squares and dark dots on white squares using morphological square-core masks |
| `stat_threshold` | `0.5` | `adaptthresh` sensitivity: fraction of the local statistic a pixel must exceed to be classified foreground. Controls mask size independently of `thres`. Raise to consolidate split detections; lower to recover faint spots |
| `dilate_mask` | `0` | disk radius in pixels to enlarge the **stored** mask after centroid computation. Grows `TM.mask` and the bounding box without moving the weighted centroid |
| `verbose` | `1` | `0` silent; `1` header + periodic progress + summary; `2` full per-frame detail |
| `TM_init` | `[]` | pre-existing table to continue from |
| `edge_overlap` | `0.1` | overlap ratio for edge exclusion |
| `expected_spots` | auto | expected spot count for validation warnings |
| `show_debug` | `false` | visualisation at `debug_frames` |

**TM table columns**

| Column | Content |
|---|---|
| `id` | `N×1 string` spot identifiers |
| `bbox` | `N×4 double` bounding boxes `[x y w h]` |
| `centroid` | `N×2 double` geometric centroids `[x y]` |
| `wcentroid` | `N×2 double` intensity-weighted centroids `[x y]` |
| `mask` | `N×1 cell` sparse logical masks per spot |
| `anomaly` | table of corrections applied this frame |

Anomaly types: `noisy_detection`, `false_positive_removal`, `fusion`, `split`, `missing`, `count_mismatch`, `out_of_bounds`.

---

#### `gridSortTM(TM, ncols, nrows)`

Reorders TM so spot 1 is the upper-left dot, numbering runs top-to-bottom down each column, then left-to-right across columns (column-major, upper-left origin).

Because `centroidwander` stores rows in detection order rather than ID order, this function first aligns every frame to the frame-1 ID set, then applies a single column-major permutation derived from the frame-1 centroid geometry.

```matlab
[order, TM] = gridSortTM(TM, 10, 10);
```

After this call `TM.wcentroid{t}(k,:)` is the same physical dot in every frame `t`.

---

#### `analyzeTrackingQuality(TM, ...)`

Computes quality metrics and generates a 9-panel diagnostic figure. Returns a `stats` struct.

```matlab
stats = analyzeTrackingQuality(TM);
stats = analyzeTrackingQuality(TM, 'expected_spots', 100, 'plot_results', true);
```

Quality score (0–100) weights: spot-count stability 30 %, anomaly rate 30 %, trajectory continuity 20 %, displacement consistency 20 %.

---

### Pastam Population

#### `injectTracksIntoPastam(A, TM)`

Collapses the per-frame cell arrays of a grid-sorted `TM` into `N×2×T` and `N×4×T` arrays and stores them in `A`. `TM` may be cleared after this call; `A.Tracks` keeps the archival copy.

```matlab
A = injectTracksIntoPastam(A, TM);
clear TM
```

---

### Scintillation

#### `scintillationFromPastam(A, se_rm, ...)`

Computes the image-plane scintillation index over the checker edges. Two indices are always produced:

$$\sigma_i^2 = \langle i^2 \rangle - \langle i \rangle^2, \quad i = \frac{I - B}{B} \quad \text{(Charnotskii)}$$

$$\sigma_I^2 = \frac{\mathrm{Var}_t(I)}{\langle I \rangle^2} \quad \text{(classical)}$$

Both are evaluated on the same edge band, stored in `A.Scintillation.sigma2` and `A.Scintillation.sigma2_classic`.

`se_rm` (required) is the disk radius shared by the bright-square zone erosion and the edge-band dilation. It must cover the full dot wander and is computed externally from `A.OptimalCentroids`.

```matlab
A = scintillationFromPastam(A, se_rm);
A = scintillationFromPastam(A, se_rm, 'Ib_mode', 'local', 'disk_bg', 5, ...
    'dW', 26, 'dh', 26, 'tau', 0.03, 'verbose', 1);
```

**Background estimation — `B = (I_bright + I_dark) / 2`**

Following Charnotskii (eq. 4–5), the step-edge object has constant background $B$ = midpoint between the bright and dark sides. Three modes estimate this:

| `Ib_mode` | Method |
|---|---|
| `'local'` (default) | Tile `⟨I⟩` into non-overlapping `local_window × local_window` blocks; `B = (max + min)/2` per block. Tracks the illumination gradient across the board |
| `'zonal'` | One `B` per bright/dark zone pair, from the mean of the eroded square interior on `⟨I⟩`. Automatically collapses to `'global'` when the relative spread across band pixels is below `Ib_tol` |
| `'global'` | Single `B` = mean of the zonal values, forced |

All geometry (zone masks, edge band, dot exclusion, boundary strip) is derived from the time-averaged image `⟨I⟩`, not from individual frames.

**Key optional parameters**

| Parameter | Default | Description |
|---|---|---|
| `Ib_mode` | `'local'` | background estimation mode (see above) |
| `local_window` | `se_rm + disk_bg` | block side length in px for `'local'` mode |
| `Ib_tol` | `0.05` | relative spread threshold for auto-collapse of `'zonal'` to `'global'` |
| `disk_bg` | `5` | morphological opening radius to recover bright/dark squares |
| `min_edge_area` | `60` | minimum Sobel edge fragment retained by `bwareafilt` |
| `dW`, `dh` | `30` | frame-boundary exclusion in pixels |
| `tau` | `0.03` | reference threshold stored in `A.Scintillation.tau` — not applied |

**`A.Scintillation` struct fields**

| Field | Content |
|---|---|
| `sigma2` | `H×W` Charnotskii index |
| `sigma2_classic` | `H×W` classical index |
| `mask` | `H×W` logical evaluation band |
| `B_bright`, `B_dark` | per-zone background means |
| `B_applied` | `H×W` background actually used |
| `B_compare` | spread statistics and global/zonal verdict |
| `Ib_mode_used` | mode string actually applied |
| `mean_image` | `H×W` time-averaged `⟨I⟩` (raw counts) |
| `se_rm_radius` | scalar |
| `tau`, `E_Omega` | reference threshold and excess-pixel fraction |

---

#### `showScintillation(A, ...)`

Produces a 2×3 comparative figure and returns suggested thresholds.

```
Row 1:  sigma_i^2 full map | sigma_i^2 histogram | sigma_i^2 thresholded map
Row 2:  sigma_I^2 full map | sigma_I^2 histogram | sigma_I^2 thresholded map
```

Both histograms are log-y. The two noise peaks (bright and dark square populations) are marked; the suggested threshold is placed `peak_sigma` estimated widths beyond the second peak, in the tail where signal lives.

```matlab
thresh = showScintillation(A);
thresh = showScintillation(A, 'peak_sigma', 1, 'nbins', 100);
thresh = showScintillation(A, 'clim_charnotskii', [0 0.02]);
```

Returns `thresh.charnotskii` and `thresh.classical`.

---

## Algorithm Notes

### Checkerboard detection (`polarity = 'both'`)

The `'both'` path in `ThresholdMask` builds two non-intersecting square-core masks from `⟨I⟩` using morphological opening (`disk_bg`) and erosion. Bright dots on dark squares are detected in the original frame; dark dots on white squares are detected in the arithmetic complement `1 − imgn`. Both populations go through the same adaptive-threshold pipeline and are merged before bounding-box matching.

### Spot ordering after tracking

`centroidwander` assigns IDs in detection order, not grid order. Row `k` of `TM` frame `t` is not the same dot as row `k` of frame `t+1`. `gridSortTM` resolves this by aligning every frame to the frame-1 ID set via `ismember`, then applying a column-major permutation from the frame-1 centroid geometry.

### Scintillation two-pass structure

Pass 1 accumulates `⟨I⟩ = mean(stack, 3)`. All masks are computed from `⟨I⟩`. Pass 2 computes `i = (I − B)/B` via implicit broadcast over the stack dimension and takes `var(i_stack, 1, 3)` for `σ_i^2` and `var(stack, 1, 3) / meanI.^2` for `σ_I^2`. Both operations are fully vectorised; no frame loop exists in the array-backed case.

---

## Citation

```
@software{pastam2025,
  author = {Dario G. Perez},
  title  = {PASTAM: Passive Sensor Tracking and Analysis for MATLAB},
  year   = {2025},
  url    = {https://github.com/atsol-pucv/dots-centroid-wander}
}
```

## License

MIT License 2025. Contact: dario.perez@pucv.cl
