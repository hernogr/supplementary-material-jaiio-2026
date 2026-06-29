# Landsat Volcano Monitoring Pipeline

Automated pipeline for downloading, processing, and analyzing Landsat satellite imagery to construct long-term surface temperature and spectral index time series at active volcanoes. Implemented at the crater lake of **Tupungatito volcano** (Mendoza, Argentina), generating a calibrated LST time series spanning 2000–2025 from 837 Landsat scenes.

This pipeline is described in:

> Rosenfeld, H.G., Toyos, G.P., Agusto, M.R., & Cabral, J.B. (2026). *Pipeline for the construction of a multitemporal dataset of surface temperature and spectral indices from satellite sensors for volcano monitoring.* [Conference paper]

The pipeline code is under active development and is not yet publicly available. The calibrated dataset and this documentation are provided as supplementary material to support reproducibility of the results presented in the paper.

corresponding author: Rosenfeld, H.G. --- email: hrosenfeld@gl.fcen.uba.ar

---

## Repository structure

```
├── src/
│   ├── fetcher.py        # USGS M2M API requests and file download with retry logic
│   ├── processor.py      # Per-scene analysis: masking, extraction, and index computation
│   └── analytics.py      # Spectral index and LST calculations
├── main.py               # Entry point: scene search, download orchestration, threading
├── lago_tup.gpkg         # Region of interest (crater lake polygon, GeoPackage)
├── requirements.txt
└── README.md
```

---

## How it works

### 1. Scene retrieval (`fetcher.py`, `main.py`)
The pipeline connects to the [USGS Machine-to-Machine (M2M) API](https://m2m.cr.usgs.gov/) using personal credentials. It queries Landsat 7 (ETM+) and Landsat 8–9 (OLI/TIRS) Collection 2 Level-2 scenes that intersect the region of interest, filters by maximum cloud cover (default: 30%), and skips scenes already processed. Downloads run in parallel threads with retry logic and file integrity validation via `rasterio`.

**Downloaded bands per scene:**
- `ST_B6` (L7) / `ST_B10` (L8–9): Surface Temperature
- `QA_PIXEL`: Quality Assurance flags
- `ST_QA`: Surface Temperature uncertainty
- `SR_B2/B3` (Green), `SR_B4/B5` (NIR), `SR_B5/B6` (SWIR1), `SR_B7` (SWIR2)
- `MTL.txt`: Scene metadata (used for acquisition time)

### 2. Scene analysis (`processor.py`)
For each scene, the lake polygon is reprojected to the scene's UTM CRS. Two spatial regions are extracted:

- **Lake pixels**: cropped to the crater lake boundary.
- **Background ring**: an annular buffer from 60 m to 300 m outside the lake, used to compute a surroundings reference temperature.

QA_PIXEL flags are applied to mask cloud, cirrus, and snow-contaminated pixels (bits 2, 3, 5). Scenes with fewer than 3 valid lake pixels are discarded and logged to a separate invalid-scenes CSV with a documented reason.

### 3. Spectral index computation (`analytics.py`)
SR bands are converted from DN to surface reflectance (scale: `× 0.0000275 − 0.2`). LST is derived from the ST band (`DN × 0.00341802 + 149.0 − 273.15`, result in °C). Computed indices:

| Index | Formula | Purpose |
|-------|---------|---------|
| NDSI | (Green − SWIR1) / (Green + SWIR1) | Background snow cover detection |
| NDWI | (Green − NIR) / (Green + NIR) | Water extent |

Background snow masking: pixels in the background ring with NDSI > 0.4 and LST < 2°C are excluded. If more than 50% of the background is snow-covered, the thermal anomaly (lake − background) is flagged as invalid for that scene.

### 4. Inter-sensor calibration
A systematic LST bias between Landsat 7 and Landsat 8–9 was estimated from near-contemporaneous scene pairs — scenes from both sensors acquired on the same day (i.e., within a 24-hour acquisition window) over the same target. For each pair, the difference `diff_lst = lst_l89 − lst_l7` was computed using `lst_max` values.

**Calibration dataset (`Intersensor_Calibration_Pairs.csv`):** 93 valid pairs spanning 2013–2023, structured as:

| Column | Description |
|--------|-------------|
| `fecha_l7` | Acquisition date of the Landsat 7 scene |
| `fecha_l89` | Acquisition date of the Landsat 8–9 scene |
| `dias_entre` | Days between acquisitions (always 0: same day) |
| `lst_l7` | Maximum LST from the L7 scene [°C] |
| `lst_l89` | Maximum LST from the L8-9 scene [°C] |
| `diff_lst` | LST difference: L8-9 − L7 [°C] |

The median offset across all pairs is **−3.732°C** (SD = 5.0°C), statistically significant against zero (one-sample t-test). This offset is applied to all Landsat 7 `lst_max` values to produce the calibrated column `lst_max_cal`. Landsat 8–9 values are kept unchanged.

```python
# Calibration applied per scene
lst_max_cal = lst_max + offset  if sensor == 'L7'  else lst_max
# where offset = median(diff_lst) = -3.732°C
```

The effect of calibration on the LST distribution (all 837 valid scenes, evaluated on `lst_max`) was assessed using skewness and excess kurtosis:

| | Skewness | Excess kurtosis |
|-|----------|-----------------|
| Before calibration | −0.346 | −0.749 |
| After calibration  | −0.285 | −0.679 |

Reduced absolute skewness and kurtosis approaching zero after calibration indicate improved inter-sensor consistency. Note that the KDE comparison uses all 837 scenes (including those that become negative after the offset is applied) to preserve distributional consistency between pre- and post-calibration datasets. The final time series uses only the 826 scenes with `lst_max_cal ≥ 0°C`.

---

## Running the pipeline

```bash
python main.py -u YOUR_USGS_USERNAME -t YOUR_USGS_TOKEN
```

Configure paths at the top of `main.py`:

```python
vector        = r"path/to/lago_tup.gpkg"    # ROI polygon
path          = r"path/to/output"            # Temporary TIF storage
csv_filename  = r"path/to/output.csv"        # Valid scenes output
csv_invalidos = r"path/to/invalidos.csv"     # Discarded scenes log
```

The pipeline is **resumable**: it reads existing CSVs on startup and skips already-processed scenes. TIFs are deleted after processing to save disk space.

---
## Region of Interest: `lago_tup.gpkg`

This file was used as a reference for the position of the crater lake. It consists of a 3×3 grid of 30 m pixels, covering areas most frequently inundated by water over the entire study period. This approach reduces sensitivity to subtle water-level variations and fumarolic activity within the lake.
These coordinates correspond approximately to the Tupungatito volcano study area (WGS84 geographic coordinates):
North: -33.3689
South: -33.4554
West: -69.8871
East: -69.7622
---

## Output dataset: `Final_Valid_Data_Tupungatito_Landsat.csv`

This is the final, **publication-ready dataset**. It contains only scenes that passed all quality filters (sufficient valid lake pixels and a pragmatic LST threshold based on a 0 °C proxy), with Landsat 7 values corrected by the inter-sensor calibration offset.

**826 scenes after post-calibration filtering, from 837 valid pre-calibration scenes, 2000–2025.** This row count matches the n=826 time series reported in the paper (Fig. 3b).

**Filtering logic:**
1. Initial filter: 854 → 837 scenes with `lst_max ≥ 0°C` (using maximum, not mean, LST as the quality criterion)
2. Post-calibration filter: 837 → 826 scenes, excluding those with `lst_max_cal < 0°C` (physically implausible after sensor calibration)

### Column reference

| Column | Units | Description |
|--------|-------|-------------|
| `scene_id` | — | USGS Landsat scene identifier |
| `sensor` | — | Sensor family: `L7` (Landsat 7 ETM+) or `L8-9` (Landsat 8/9 OLI-TIRS) |
| `acquisition_datetime` | UTC | Scene center date and time from MTL metadata |
| `lst_max` | °C | Maximum lake surface temperature (raw, uncalibrated) |
| `lst_mean` | °C | Mean lake surface temperature across valid pixels |
| `lst_std` | °C | Standard deviation of lake LST across valid pixels |
| `valid_px` | count | Number of lake pixels that passed QA masking |
| `ndsi_mean` | — | Mean NDSI over valid background pixels (snow index; higher = more snow/ice) |
| `ndwi_mean` | — | Mean NDWI over valid lake pixels (water index; > 0 indicates open water) |
| `water_fraction` | 0–1 | Fraction of valid lake pixels with NDWI > 0 |
| `water_pixels` | count | Number of valid pixels with NDWI > 0 |
| `lst_max_cal` | °C | **Calibrated maximum LST** — L7 corrected by −3.7°C offset; L8–9 unchanged. Primary variable for time-series analysis. |

> **Note on NDSI:** NDSI (Normalized Difference Snow Index) data is used as a proxy for estimating the percentage of snow cover in the crater lake background, in order to assess potential anomalies of the lake relative to its surrounding environment. This analysis is intended as future work and is not included in the present study.

> **Note on valid_px:** The crater lake at Tupungatito spans approximately 3×3 Landsat pixels (~100–120 m diameter). A minimum of 3 valid pixels is required for a scene to be included. Scenes with fewer valid pixels after QA masking are logged in the separate invalid-scenes file with the reason (e.g., `enmascarado_por_qa(1/9_validos)`).

---

## Platform

The pipeline is implemented in Python 3.9+ and runs on Windows and Linux. See `requirements.txt` for dependencies. No commercial software or proprietary platforms are required; all data are freely accessible via the USGS M2M API with a registered account.

---

## Notes and limitations

- **ST uncertainty**: The USGS ST product (Collection 2 Level-2) has an estimated uncertainty of up to 2–3 K under clear-sky conditions.
- **Inter-sensor calibration**: The −3.7°C offset (SD = 5.0°C) reflects a systematic bias and a non-trivial spread. Individual Landsat 7 scenes should be interpreted with this uncertainty in mind.
- **LST threshold (0 °C proxy)**: Scenes are filtered using a maximum LST threshold of 0 °C as a quality-control criterion. This threshold is a pragmatic approximation rather than a measured freezing point. In an acidic, hydrothermally-influenced crater lake, the true freezing point may differ from 0 °C; this threshold will be refined in future work with direct chemical characterization of the lake and ground-truth validation data.
- **Background snow masking**: High NDSI in the background ring can reflect sulfurous volcanic deposits in addition to snow. This is a documented methodological limitation.
- **Landsat 7 SLC-off**: From May 2003 onward, Landsat 7 images have scan line corrector failure (striping). The small lake footprint (≤9 pixels) means some scenes may have reduced valid pixel counts as a result.

---

## Discarded scenes: `Discarded_Data_Landsat_Tupungatito.csv`

Scenes that were downloaded and processed but did not meet quality criteria are logged here with a documented reason. This file covers 277 scenes discarded during processing.

| Column | Description |
|--------|-------------|
| `scene_id` | USGS Landsat scene identifier |
| `sensor` | Sensor family: `L7` or `L8-9` |
| `acquisition_datetime` | Scene center date and time (UTC) |
| `motivo_invalidez` | Discard reason (see below) |

**Discard reasons:**

| Reason | Count | Description |
|--------|-------|-------------|
| `enmascarado_por_qa(0/9_validos)` | 253 | All lake pixels masked by QA (total cloud/snow cover) |
| `enmascarado_por_qa(2/9_validos)` | 10 | Too few valid pixels after QA masking (< 3 required) |
| `enmascarado_por_qa(1/9_validos)` | 5 | Too few valid pixels after QA masking (< 3 required) |
| `ndwi_negativo` | 9 | Mean NDWI < 0 over lake pixels (no open water detected) |

This file does not include discarded data due to thermal criteria as it is explained in the article.

---

## Supplementary material — file list

| File | Description |
|------|-------------|
| `Final_Valid_Data_Tupungatito_Landsat.csv` | Final calibrated dataset (826 valid scenes) |
| `Discarded_Data_Landsat_Tupungatito.csv` | Discarded scenes with documented reason (277 scenes) |
| `Intersensor_Calibration_Pairs.csv` | Inter-sensor calibration pairs (93 same-day pairs) |
| `requirements.txt` | Python dependencies |
| `README.md` | This file |
