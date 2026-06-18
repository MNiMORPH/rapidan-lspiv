# Rapidan Dam LSPIV — Surface Velocity Measurements

Large-Scale Particle Image Velocimetry (LSPIV) surface velocity fields derived
from drone video of the Blue Earth River at Rapidan Dam (Martin County, MN),
spanning the June 2024 dam failure through post-failure channel adjustment.

**Collaborators:** Andy Wickert (UMN), Zach Hilgendorf (MSU Mankato)

---

## Background

The Rapidan Dam on the Blue Earth River failed on **June 23, 2024** during an
extreme flood event. This repository documents LSPIV-derived surface velocity
fields from drone video collected before, during, and after the failure, covering
a wide range of flow conditions and geomorphic states.

LSPIV measures surface velocity by tracking the movement of natural surface
tracers (turbulence patterns, foam streaks, floating debris) between consecutive
video frames using cross-correlation. All velocities are georeferenced to
UTM Zone 15N (EPSG:32615).

---

## Clip inventory and assessment

[`inventory/assessment.json`](inventory/assessment.json) contains assessments of
all 79 drone video clips reviewed. Fields per clip:

| Field | Values | Meaning |
|---|---|---|
| `clip` | string | Clip name (matches results/ subdirectory) |
| `date` | string | Acquisition date folder from Drive |
| `suitable` | bool | Whether the clip is/was attempted for LSPIV |
| `camera_motion` | stationary / minor / significant / excessive | Motion between early and late frames |
| `notes` | string | Detailed assessment |

Representative mid-clip frames for all 79 clips are in
[`inventory/frames/`](inventory/frames/).

Clips with `_nadir` suffix in `camera_motion` notes are subclips trimmed from the
original to isolate the near-nadir hover segment.

---

## Results

Results are organized by clip name under `results/`. Each subdirectory contains:

| File | Description |
|---|---|
| `velocity.tif` | 10-band GeoTIFF: u, v, speed, speed\_std, speed\_cv\_pct, s2n, corr, land\_mask, R, G, B |
| `velocity.nc` | Full velocity field (u, v, speed, std, CV, S/N, correlation) as NetCDF |
| `velocity.gpkg` | Velocity vectors as GeoPackage (view in QGIS/ArcGIS) |
| `frame_utm.tif` | Mid-clip video frame georeferenced to UTM |
| `velocity_utm.png` | Colored quiver plot on georeferenced frame (CV-masked) |
| `velocity_utm_all.png` | Same, unmasked (all PIV cells) |
| `velocity_raster_utm.png` | Speed raster overlaid on frame (CV-masked) |
| `velocity_raster_utm_all.png` | Same, unmasked |
| `velocity_raster_arrows_utm.png` | Speed raster with direction arrows (CV-masked) |
| `velocity_raster_arrows_utm_all.png` | Same, unmasked |
| `velocity_std_utm.png` | Temporal speed std deviation — flow unsteadiness (CV-masked) |
| `velocity_std_utm_all.png` | Same, unmasked |
| `velocity_cv_utm.png` | Coefficient of variation — normalized unsteadiness (CV-masked) |
| `velocity_cv_utm_all.png` | Same, unmasked |
| `PIVquiverFrame.png` | Vector quiver on raw video frame (pixel coordinates) |
| `PIVquiverFiltered.png` | Quality-filtered quiver on raw frame |
| `georeference_debug.png` | SIFT feature matching used for georeferencing |

Each figure type has a paired `*_utm.png` (CV-masked) and `*_utm_all.png` (unmasked) variant.
All figures use an identical fixed layout so the same geographic region falls at the same pixel
position across all outputs — enabling direct toggle-comparison in an image viewer.

### Processed clips

| Clip | Date | Flow condition | Velocity points | Notes |
|---|---|---|---|---|
| [MAX_0102](results/MAX_0102/) | 2024-11-15 | Low / base flow | 1,727 | Breach spillway cascade |
| [MAX_0015_nadir](results/MAX_0015_nadir/) | 2024-08-06 | Moderate | 11,672 | Near-nadir hover; strong coherent flow |
| [DJI_0022](results/DJI_0022/) | 2025-05-14 | Elevated (spring) | 8,021 | DJI nadir; broad flow domain |
| [DJI_0023](results/DJI_0023/) | 2025-05-14 | Elevated (spring) | 6,295 | DJI nadir; companion clip to DJI_0022 |
| [MAX_0177](results/MAX_0177/) | 2025-03-03 | High / flood | 1,768 | March 2025 flood event |
| [MAX_0178](results/MAX_0178/) | 2025-03-03 | High / flood | 1,716 | March 2025 flood event; companion clip |

---

## Processing notes and decisions

### Noisiness mask: CV-based land/water discrimination

The primary criterion for classifying a PIV cell as "water" (coherent flow) vs. "land or noise"
is the **coefficient of variation** (CV) of speed:

```
CV = speed_std / speed × 100%
```

Default threshold: CV < 100% (cells where temporal std exceeds mean speed are treated as
non-water and masked in the `*_utm.png` outputs).

**Why CV instead of DSM elevation?**  
The waterfall site has a single-tier cascade where water and rock are at essentially the same
elevation — a DSM cut cannot separate them. CV captures whether a cell is making *coherent*
directional motion: stationary land produces near-zero mean speed but finite noise std → very
high CV; flowing water has bounded temporal variation relative to its mean speed → low CV.

The DSM elevation filter remains available as a secondary mask (see config `dsm` / `water_elev_m`)
but is not needed at this site.

### Condition-matched orthomosaics: unlocking SIFT for additional clips

The original SIFT georeferencing attempt (against a single November 2024 low-flow
orthophoto) succeeded only for MAX_0102 and failed for every other clip. The root
cause was a scene-composition mismatch: at higher flow stages, exposed rock and sediment
are submerged, and the video frame and orthophoto show the site in visually incompatible
states — too few stable correspondences for reliable feature matching.

**Resolution (June 2026):** Zach Hilgendorf (MSU Mankato) generated SfM orthomosaics
from the drone video itself, one per distinct acquisition event. Each orthomosaic
captures the site at the *same* flow condition as the video clip, providing the shared
visual features that SIFT requires. This unlocked georeferencing for five additional clips.

The pipeline now supports per-clip orthophoto overrides via `config.yaml`:

```yaml
orthophotos:
  MAX_0015_nadir: data/orthophotos/2024_08Aug06.tif
  DJI_0022: data/orthophotos/2025_05May14.tif
  ...
```

Clips not listed fall back to the global `orthophoto:` entry.

### SIFT georeferencing outcomes

| Clip | Date | SIFT orthophoto used | Outcome |
|---|---|---|---|
| MAX_0102 | 2024-11-15 | Global (2024-11 low flow) | ✓ Processed (177 inliers) |
| MAX_0015_nadir | 2024-08-06 | 2024_08Aug06 (condition-matched) | ✓ Processed |
| DJI_0022 | 2025-05-14 | 2025_05May14 (condition-matched) | ✓ Processed |
| DJI_0023 | 2025-05-14 | 2025_05May14 (condition-matched) | ✓ Processed |
| MAX_0177 | 2025-03-03 | 2025_03Mar03 (condition-matched) | ✓ Processed |
| MAX_0178 | 2025-03-03 | 2025_03Mar03 (condition-matched) | ✓ Processed |
| MAX_0094 | 2024-07-03 | 2024_07Jul03 (condition-matched) | ✗ 8 RANSAC inliers — see below |
| DJI_0024 | 2025-05-14 | 2025_05May14 (condition-matched) | ✗ Near-failure — see below |
| DJI_0036_nadir | 2024-09-25 | (stabilization failed before SIFT) | ✗ Stabilization crash |

### MAX_0094 (July 3, 2024): SIFT failure despite condition-matched orthomosaic

Even with a condition-matched orthomosaic (acquired the same day), SIFT produced
only **8 RANSAC inliers** for MAX_0094. The resulting homography was degenerate;
0 of 493 PIV cells fell within the valid domain after quality filtering.

Root cause: the July 3 clip shows near-peak post-failure flow — turbulent whitewater
dominates essentially the entire frame. The orthomosaic shows the site at the same
high flow, but turbulent water surfaces have no stable texture for feature matching.
There are no stable rock, sediment, or structure features shared between the video
frame and the orthomosaic.

Path forward: manual GCPs from fixed structures visible in both video and map (rock
ledge edges, concrete fragments), or SfM-derived pixel↔UTM correspondences extracted
directly from Zach's Metashape project.

### DJI_0024 (May 14, 2025): oblique angle limits PIV domain

DJI_0024 georeferenced successfully but only **66 PIV velocity points** were recovered
(vs. ~6,000–8,000 for companion clips DJI_0022 and DJI_0023). Inspection of the
`georeference_debug.png` and PIV output reveals a steep oblique camera angle: most of
the rectified domain maps to land. Results were not committed — 66 points is
insufficient for a meaningful velocity field.

### DJI_0036_nadir (September 25, 2024): stabilization failure

The Stabilo video stabilizer found only 5–7 feature inliers per consecutive frame pair
(vs. a typical ~50–100). The accumulated sub-frame offsets required a crop of −277 × −301 px
on a 1920 × 1080 frame — a physically impossible crop — causing a non-zero exit code and
no output file.

Root cause: the high-flow scene has too little stable texture in overlapping frame regions
for robust homography estimation. The clip has been moved to `data/raw_pending/` pending
either a bypass of stabilization (run PIV directly on the raw video) or a different
stabilizer configuration.

Zach is also generating a condition-matched orthomosaic for this clip. Once that is
available, processing can proceed with `pipeline.stabilize: false` in config.

---

### MAX_0321 and MAX_0322 (June 2025): withdrawn from inventory

These clips were originally top-ranked candidates based on surface texture quality. However,
SIFT matching against the November 2024 orthophoto yielded only 5–6 RANSAC inliers (degenerate
homography); only ~19 / 7,676 PIV cells fell within the valid domain.

**June 2026 update:** Zach Hilgendorf reviewed the raw clips and withdrew them from the
processing inventory — they were determined not to be useful for this analysis. They have been
removed from `data/raw/` and will not be reprocessed.

### High-flow clips from June–July 2024 (pending orthomosaics)

Clips DJI_0586 (2024-06-28), DJI_0658 (2024-06-30), and DJI_0672 (2024-06-30) were acquired
5–7 days after the dam failure during very high and rapidly receding flow. They are held in
`data/raw_pending/` pending condition-matched orthomosaics from Zach, which have not yet been
generated for these dates. Once orthomosaics are available, processing will follow the same
workflow used for the June 2026 batch above.

### Memory management for long clips

The frame normalization step (temporal mean subtraction) loads all frames as float32 arrays —
~32 MB per frame at 3836×2102 px. For a 20-second clip (~600 frames) this would require ~20 GB,
exceeding available RAM. The pipeline processes videos in 50-frame chunks to keep peak RAM at
~1.6 GB regardless of clip length. Per-chunk normalization is equivalent to global normalization
for a stationary camera with stable illumination.

### Paired output figures and fixed layout

Every geographic figure is produced in two variants:
- **`*_utm.png`** — CV-masked (non-water cells hidden)
- **`*_utm_all.png`** — unmasked (all PIV cells, including land/noise)

All figures use a fixed subplot layout (`subplots_adjust`) and a manually placed colorbar axes,
with no `bbox_inches="tight"` rescaling. This ensures every figure in a clip's directory places
the same geographic region at the same pixel position, so the masked and unmasked variants can
be toggled between directly in an image viewer without any shift.

### Background frame

The UTM-georeferenced background image (`frame_utm.tif`) is extracted from the **middle frame**
of the clip. For short clips this is ~1 s in; for longer clips it avoids the first/last frames
which may have higher motion blur during settling.

---

## Camera configurations

[`camera_configs/`](camera_configs/) contains the georeferencing configuration
JSON files (generated by the LSPIV pipeline) for each processed clip. These record
the camera intrinsics, ground control point correspondences, and bounding box used
for the velocity field.

---

## Clip temporal coverage

Clips span the full post-failure hydrograph from June 2024 through March 2025:

| Clip | Date | Flow regime | Status |
|---|---|---|---|
| DJI_0586_RAPIDAN_nadir | 2024-06-28 | Very high — ~5 days post-failure | Pending — no orthomosaic yet |
| DJI_0658_RAPIDAN_nadir | 2024-06-30 | High / actively receding | Pending — no orthomosaic yet |
| DJI_0672_RAPIDAN_nadir | 2024-06-30 | High / actively receding | Pending — no orthomosaic yet |
| MAX_0094 | 2024-07-03 | Very high / turbulent whitewater | Failed — SIFT (8 inliers, featureless water surface) |
| MAX_0015_nadir | 2024-08-06 | Moderate | **Processed** — 11,672 pts |
| DJI_0036_nadir | 2024-09-25 | Moderate | Failed — stabilization crash |
| MAX_0102 | 2024-11-15 | Base flow | **Processed** — 1,727 pts |
| MAX_0177 | 2025-03-03 | High / flood | **Processed** — 1,768 pts |
| MAX_0178 | 2025-03-03 | High / flood | **Processed** — 1,716 pts |
| DJI_0022 | 2025-05-14 | Elevated (spring) | **Processed** — 8,021 pts |
| DJI_0023 | 2025-05-14 | Elevated (spring) | **Processed** — 6,295 pts |
| DJI_0024 | 2025-05-14 | Elevated (spring) | Not committed — only 66 pts (oblique angle) |

See Processing notes for full diagnosis of failures and pending clips.

---

## Pipeline

LSPIV processing uses the
[lspiv-rapidan](https://github.com/umn-earth-surface/lspiv-rapidan) pipeline
(pyOpenRiverCam / pyORC backend, Snakemake orchestration). Contact Andy Wickert
for access to the raw video and pipeline configuration.

Georeferencing uses SIFT feature matching against Zach Hilgendorf's SfM
orthophoto and digital surface model of the site.

---

## Data notes

- Coordinate reference system: **UTM Zone 15N (EPSG:32615)**
- All velocity magnitudes in **m/s**
- `_nadir` clips are trimmed subclips isolating the near-nadir hover segment
  of a longer moving shot
- Quality filters applied: signal-to-noise ratio > 1.0 (OpenPIV peak2mean; scale differs
  from peak2peak used in older runs — effectively permissive at this threshold),
  cross-correlation > 0.5, speed > 0.02 m/s (removes true-zero noise cells)
- Noisiness (land/water) mask: CV < 100% (speed std < mean speed); see Processing notes
- `*_utm_all.png` figures show all PIV cells before the noisiness mask is applied
