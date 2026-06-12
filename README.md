# IrrEM — Irrigation Extent Mapping Without Training Data

> Mapping 25 years of seasonal irrigation extent across the eastern Mediterranean using Landsat and OPTRAM soil moisture — no labeled training data required.

---

## Abstract

<!-- Paste manuscript abstract here -->

*Abstract placeholder — to be updated upon manuscript publication.*

---

## Links & Resources

| Resource | Link |
|---|---|
| **Manuscript DOI** | `https://doi.org/XXXX/XXXXXXXXX` *(placeholder)* |
| **Dataset on Zenodo** | `https://doi.org/10.5281/zenodo.XXXXXXX` *(placeholder)* |
| **Irrigation Explorer App (GEE)** | `https://code.earthengine.google.com/XXXXXXXXXXXXXXXX` *(placeholder)* |
| **Earth Engine Code Repository** | `https://code.earthengine.google.com/?accept_repo=XXXXXXXXXXXXXXXX` *(placeholder)* |

---

## Overview

IrrEM is a training-data-free pipeline for mapping seasonal irrigation extent at regional scale. It leverages the **OPTRAM** (Optical Trapezoid Model) to retrieve relative soil moisture from Landsat surface reflectance and detects irrigation events through temporal soil moisture change analysis. The method is designed for data-scarce regions where labeled irrigation maps are unavailable.

The pipeline was applied to the **eastern Mediterranean** (Turkey, Syria, Lebanon, Palestine/Jordan) over **2000–2025**, producing annual maps of three irrigation seasons at 30 m resolution using Landsat 4, 5, 7, 8, and 9.

---

## Methodology

### Core Concept

Rather than learning from labeled examples, IrrEM detects irrigation by identifying anomalous soil moisture increases that cannot be explained by recent rainfall. Soil moisture is estimated optically using the OPTRAM model — calibrated from the NDVI–STR spectral space of each Landsat footprint and year — then analyzed in both time and space to isolate irrigation signals.

### Processing Pipeline

```
Step 1 — OPTRAM Parameter Calibration
    Landsat 4–9 surface reflectance
        → Compute NDVI and STR (Shortwave Infrared Transformed Reflectance)
        → Bin pixels by NDVI; fit wet/dry edges via 5th/95th percentiles
        → Export 6 parameters per footprint × year: iw, id, sd, sw, vw, wv

Step 2 — Quality Control (SMAP Validation)
    OPTRAM-derived soil moisture vs. NASA SMAP SPL3SMP_E
        → Temporal matching (±5 days)
        → Pearson R and RMSE time-series per footprint

Step 3 — Irrigation Event Detection (per Landsat footprint)
    Landsat time-series → relative soil moisture (OPTRAM)
        → Forward differencing: dSM, dSM_local (2.5 km neighbourhood), d²SM
        → Flag irrigation: high dSM + low dSM_local + no recent rain
        → Three seasons: Summer (May–Sep), Supp1 (Oct–Nov), Supp2 (Mar–Apr)
        → Pack event counts into 16-bit GEE asset

Step 4 — Regional Compositing & Post-Processing
    Multi-footprint mosaics → quality mosaic (max valid observations)
        → Spatial denoising: connected-component majority filter (90 m kernel)
        → Seamless annual maps per season

Step 5 — Accuracy Assessment
    5a: Stratified random sampling (irrigated / non-irrigated, by country)
    5b: Manual labeling protocol using VHR imagery in Google Earth Pro
    5c: Confusion matrix → Overall Accuracy, F-score, User's & Producer's Accuracy

Step 6 — Public Visualization
    Interactive GEE web app: country/province filters, area charts,
    split-map comparison, 25-year time-series of irrigation extent
```

---

## Repository Structure

```
IrrEM_Irrigation-extent-mapping-without-training-data/
│
├── 01_CalculateSM_OptramParameters          # Calibrate OPTRAM wet/dry edge parameters
├── 02_QualityControl_ComparisonWithSMAP     # Validate soil moisture against NASA SMAP
├── 03_IrrigationMapping                     # Detect irrigation events (single footprint)
├── 03b_IrrigationMapping_Operational        # Batch processing over regional footprint grid
├── 04_Compositing_PostProcessing            # Mosaic, smooth, and export annual maps
├── 05a_AccuracyAssessment_SamplingDesign    # Generate stratified validation sample points
├── 05b_AccuracyAssessment_ResponseDesign    # Labeling protocol (SOP, no executable code)
├── 05c_FinalAccruacyAssessment              # Compute accuracy metrics from reference labels
└── 06_ExploringApp                          # Interactive GEE visualization app
```

All scripts are written in **Google Earth Engine JavaScript API**.

---

## Script Descriptions

### `01_CalculateSM_OptramParameters`
Computes the six OPTRAM parameters (`iw`, `id`, `sd`, `sw`, `vw`, `wv`) that define the wet and dry edges of the NDVI–STR space for a given Landsat footprint and time period. The script processes Landsat 4–9 surface reflectance, applies cloud and terrain masks, bins pixels by NDVI into five intervals (0.2–0.7), fits linear regressions at the 5th (dry) and 95th (wet) percentiles, and exports parameters as a CSV for use in subsequent steps.

**Key inputs:** Landsat C2 L2 surface reflectance, CHIRPS precipitation, SRTM terrain mask  
**Key outputs:** CSV with OPTRAM parameters per WRS path/row and year

---

### `02_QualityControl_ComparisonWithSMAP`
Validates OPTRAM-derived soil moisture estimates against independent NASA SMAP (SPL3SMP_E) observations. For each overlapping Landsat–SMAP acquisition pair (within ±5 days), the script computes Pearson correlation and RMSE, with optional Winsorization for outlier robustness. Results are visualized as a time-series chart and exported as CSV.

**Key inputs:** OPTRAM parameters (from Step 1), NASA SMAP SPL3SMP_E, CHIRPS precipitation  
**Key outputs:** R and RMSE time-series charts and CSV

---

### `03_IrrigationMapping`
Core irrigation detection script for a **single** Landsat WRS footprint and year. Converts Landsat imagery to relative soil moisture via OPTRAM, then applies forward differencing to compute soil moisture change (`dSM`) at pixel and neighbourhood scales. Irrigation events are flagged when soil moisture increases anomalously above the local background and no recent rainfall is recorded. Events are summed per season and packed into a 16-bit GEE image asset.

**Key inputs:** OPTRAM parameters, Landsat imagery, CHIRPS precipitation, ESA WorldCover crop mask  
**Key outputs:** 16-bit packed multi-band GEE asset (irrigation event counts × season)

---

### `03b_IrrigationMapping_Operational`
Batch version of `03` that loops over a grid of Landsat WRS footprints (Paths 170–175, Rows 34–38) to process an entire region in one run. Adds cross-sensor harmonization (Landsat 7 → 8, Roy et al. 2018 coefficients) and support for pre-calculated OPTRAM parameters loaded from exported assets.

**Key inputs:** Same as `03`, iterated over footprint grid  
**Key outputs:** One 16-bit packed GEE asset per footprint × year

---

### `04_Compositing_PostProcessing`
Assembles the per-footprint irrigation assets into seamless regional mosaics. Overlapping footprints are merged using a quality mosaic weighted by the number of valid Landsat observations. Spatial noise is reduced by identifying small connected patches (<50 pixels) and applying a majority filter within a 90 m kernel. Supplemental seasons 1 and 2 are merged into a unified supplemental layer.

**Key inputs:** All 16-bit packed assets from Steps 3/3b  
**Key outputs:** Smoothed annual irrigation extent maps (GEE assets) for 2000–2025

---

### `05a_AccuracyAssessment_SamplingDesign`
Creates a stratified random sample of validation points across irrigated and non-irrigated strata, stratified by country. For each point, the script extracts the 95th-percentile summer NDVI (June–July) for all available years and exports KMZ polygons for manual assessment in Google Earth Pro.

**Key inputs:** Irrigation extent maps, ESA WorldCover, country boundaries, Landsat NDVI  
**Key outputs:** KMZ files for Google Earth Pro, CSV with annual NDVI time-series per point

---

### `05b_AccuracyAssessment_ResponseDesign`
Standard Operating Procedure (SOP) document — no executable code. Provides step-by-step instructions for manually labeling validation points using VHR imagery in Google Earth Pro. Defines land-use categories: Rainfed (1), Low-confidence Irrigated (2), High-confidence Irrigated (3), Ambiguous (4), Not Agricultural (0), Overlapping (5), Trees/Orchards (6).

---

### `05c_FinalAccruacyAssessment`
Loads manual reference labels and the predicted irrigation maps, samples the prediction at each reference point, and computes a confusion matrix. Reports Overall Accuracy, F-score, User's Accuracy, and Producer's Accuracy. Supports three validation label sets and optional exclusion of low-confidence labels.

**Key inputs:** Predicted irrigation maps, reference feature collections (label sets 1–3)  
**Key outputs:** Console accuracy statistics per year

---

### `06_ExploringApp`
Interactive GEE web application for exploring the 25-year irrigation extent dataset. Features include country and province dropdowns, area calculation (km²) per season and year, a split-map viewer for before/after comparison, and a time-series chart of annual irrigation change. Designed for researchers, policy-makers, and the public.

**Key inputs:** Temporally smoothed irrigation maps, crop mask, country/province boundaries  
**Key outputs:** Deployed GEE web app

---

## Data Sources

| Dataset | Source | Role |
|---|---|---|
| Landsat 4/5/7/8/9 C2 L2 | USGS via GEE | Primary reflectance imagery |
| CHIRPS Daily Precipitation | UCSB/CHC | Rainfall masking |
| NASA SMAP SPL3SMP_E | NASA NSIDC via GEE | Soil moisture validation |
| SRTM DEM | USGS via GEE | Slope mask (exclude >5°) |
| ESA WorldCover v100 | ESA via GEE | Crop/urban/water masks |
| Country/Province Boundaries | — | Spatial stratification |

---

## Key Concepts

**OPTRAM** — The Optical Trapezoid Model (Sadeghi et al., 2017) estimates soil moisture from the triangular relationship between NDVI and STR (Shortwave Infrared Transformed Reflectance = (1 − SR²) / (2 × SR), where SR is shortwave infrared reflectance). Wet and dry boundary lines are derived statistically from the cloud of points in NDVI–STR space, yielding a continuous relative soil moisture index without any ground data.

**Training-data-free** — No labeled irrigation maps are used as training input anywhere in the detection pipeline. The irrigation signal emerges from physical soil moisture change thresholds, making the method transferable to any region with Landsat coverage.

**16-bit packing** — To minimize GEE asset storage, irrigation event counts for three seasons and their observation validity masks are encoded bitwise into a single 16-bit integer band per pixel.

---

## Citation

If you use this code or dataset, please cite:

> *Citation placeholder — to be updated upon manuscript publication.*
>
> DOI: `https://doi.org/XXXX/XXXXXXXXX` *(placeholder)*

Dataset:
> *Adrah, E. (2025). IrrEM — 25-year seasonal irrigation extent, eastern Mediterranean [Dataset]. Zenodo.*
>
> DOI: `https://doi.org/10.5281/zenodo.XXXXXXX` *(placeholder)*

---

## License

This code is released under the [MIT License](LICENSE). The associated dataset license is specified on Zenodo.

---

## Contact

**Esmaeel Adrah** — SenslandLab, Kent State University  
esmaeelad@gmail.com  
GitHub: [@Esmaeel-A](https://github.com/Esmaeel-A)
