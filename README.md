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

| Clip | Date | Flow condition | Notes |
|---|---|---|---|
| [MAX_0102](results/MAX_0102/) | 2024-11-15 | Low / base flow | Test clip (2 s); breach spillway cascade |

*Additional clips will be added as processing continues.*

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

### MAX_0321 and MAX_0322 (June 2025): georeferencing failure

These clips were the top-ranked candidates in the inventory assessment based on surface texture
quality. However, SIFT feature matching against the orthophoto produced only **5–6 RANSAC
inliers** (vs. 177 for MAX_0102), yielding a degenerate homography. As a result, the projected
frame was a narrow wedge and only ~19 / 7676 PIV cells fell within the valid domain.

**Root cause:** these clips show an almost entirely water-dominated scene — only a thin sliver of
exposed rock is visible on one edge. The orthophoto was acquired at lower flow when rock ledges,
banks, and bare sediment were exposed; SIFT relies on stable land-surface features to tie the
video frame to the map. When the frame is dominated by moving, featureless water, there are no
stable correspondences to match. Frames 0, 20, 50, 100, 200, and 300 were all tested; all gave
5–6 inliers, confirming this is a scene composition problem, not a frame selection issue.

**Decision:** MAX_0321 and MAX_0322 are skipped pending either (a) manual GCPs placed on the
visible rock edge, or (b) an alternative georeferencing approach (lab-method known dimensions
once the scene width is measured).

**Note from field review (Andy Wickert):** MAX_0321 appears to be an oblique shot of the
right-hand side of the MAX_0102 domain. The viewer notes "I am not sure if there is enough
information here to analyze it correctly" even with manual GCPs.

### High-flow clips from June–July 2024

Clips DJI_0586, DJI_0658, DJI_0672, DJI_0860 were acquired 5–47 days after the dam failure
during very high and receding flow. The orthophoto (acquired later at low flow) shows the site
in a very different state: exposed rock, bare sediment, low water. SIFT feature matching may
struggle because the high-water scene masks the ground-surface features that the orthophoto shows.

These clips are processed on a best-effort basis; the `georeference_debug.png` output should be
inspected before accepting results. If inlier count is low (<20), results should not be used
without manual GCP verification.

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

Clips selected for LSPIV span the full post-failure hydrograph:

| Clip | Date | Flow regime |
|---|---|---|
| DJI_0586_RAPIDAN_nadir | 2024-06-28 | Very high — ~5 days post-failure |
| DJI_0658_RAPIDAN_nadir | 2024-06-30 | High / actively receding |
| DJI_0672_RAPIDAN_nadir | 2024-06-30 | High / actively receding (79 s, stationary) |
| DJI_0860_RAPIDAN_nadir | 2024-07-09 | Moderate-high |
| MAX_0015_nadir | 2024-08-06 | Moderate / low |
| MAX_0102 | 2024-11-15 | Base flow |
| DJI_0022 | 2025-05-15 | Elevated (spring) |
| DJI_0023 | 2025-05-15 | Elevated (spring) |
| DJI_0024 | 2025-05-15 | Elevated (spring) |
| MAX_0321 | 2025-06-24 | Elevated | Skipped — georeferencing failure (see Processing notes) |
| MAX_0322 | 2025-06-24 | Elevated | Skipped — georeferencing failure (see Processing notes) |

Clips from 2025-05-15 include both a cascade and calmer pool; the cascade zone
is masked and only the pool is used for velocity retrieval.

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
- Quality filters applied: signal-to-noise ratio > 6.0, cross-correlation > 0.5,
  speed > 0.02 m/s (removes true-zero noise cells)
- Noisiness (land/water) mask: CV < 100% (speed std < mean speed); see Processing notes
- `*_utm_all.png` figures show all PIV cells before the noisiness mask is applied
