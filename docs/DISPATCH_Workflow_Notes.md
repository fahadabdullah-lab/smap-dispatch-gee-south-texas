# DISPATCH Soil Moisture Downscaling (GEE + Python/Colab)
Bahía Grande / South Bay (Laguna Madre, Texas)

This repo implements a DISPATCH-style soil moisture downscaling workflow using:
- SMAP L3 Enhanced soil moisture as the coarse constraint (~36 km)
- MODIS NDVI + MODIS LST at ~1 km to derive evaporative efficiency (SEE)
- DISPATCH triangle method to downscale to ~1 km soil moisture maps

---

## Notebook 01 — Setup, Auth, Config, AOI
**Notebook:** `01_setup_auth_aoi.ipynb`

### Purpose
- Install packages and initialize Earth Engine
- Define composite time window (10-day window)
- Draw AOI interactively (geemap)
- Validate dataset availability (sanity checks)
- (Optional) export config + AOI for reuse across notebooks

### Key variables
- `TEST_DATE` (center date, YYYY-MM-DD)
- `COMPOSITE_DAYS` (default 10)
- `start_date`, `end_date` (computed from `TEST_DATE`)
- `aoi` (Earth Engine geometry drawn by user)

### Composite window logic
Let `HALF_WINDOW = COMPOSITE_DAYS // 2`.

- `start_date = TEST_DATE - HALF_WINDOW`
- `end_date   = TEST_DATE + HALF_WINDOW + 1`  (end is exclusive; safer in GEE)

Example (COMPOSITE_DAYS=10):
- Center: 2021-08-15
- Start : 2021-08-10
- End   : 2021-08-21 (exclusive)
→ actual included dates: Aug 10 … Aug 20 (11 daily images)

### AOI drawing
- User draws polygon on a geemap map
- Convert to `ee.Geometry`:
  - `aoi = ee.Feature(ee.FeatureCollection(Map.draw_control.data).first()).geometry()`

### Export (optional but recommended)
- Save `dispatch_config.json` (TEST_DATE, COMPOSITE_DAYS)
- Save `dispatch_aoi.geojson` (AOI)

---

## Notebook 02 — Data Loading + 10-day Composites (SMAP + MODIS)
**Notebook:** `02_data_loading_composites.ipynb`

### Purpose
- Build stable, memory-safe composites over the 10-day window for:
  1) SMAP soil moisture (coarse constraint)
  2) MODIS LST (QC-filtered; triangle temperature axis)
  3) MODIS NDVI (triangle vegetation axis)
- Combine MODIS NDVI + LST into a single stack
- Sample pixels safely for triangle analysis (avoid large reduceRegion calls)
- Produce an NDVI–LST scatter plot diagnostic

---

### 2.1 SMAP Soil Moisture composite
**Dataset:** `NASA/SMAP/SPL3SMP_E/005`  
**Band:** `soil_moisture_am`  
**Scale:** coarse (~36 km)

Composite:
- `smap_composite = SMAP.filterDate(start_date, end_date).select('soil_moisture_am').mean().clip(aoi)`

Notes:
- Blocky appearance is expected because SMAP resolution is much coarser than the AOI.

---

### 2.2 MODIS LST composite (QC-filtered)
**Dataset:** `MODIS/061/MOD11A1`  
**Bands:** `LST_Day_1km`, `QC_Day`  
**Scale factor:** `LST (K) = DN * 0.02`  
**Convert to Celsius:** `LST(°C) = LST(K) - 273.15`

QC filter (conservative, commonly used):
- Mandatory QA bits (0–1) == 0 (good)
- Data quality bits (2–3) == 0 (good)

Composite:
- Apply QC mask + scaling per image
- `lst_composite_qc = mean(10-day window).clip(aoi)`

---

### 2.3 MODIS NDVI composite
**Dataset:** `MODIS/061/MOD09GA`  
**Bands:** `sur_refl_b01` (RED), `sur_refl_b02` (NIR)  
**Scaling:** reflectance = DN * 0.0001

NDVI:
- `NDVI = (NIR - RED) / (NIR + RED)`

Composite:
- `ndvi = mean(10-day window NDVI).clip(aoi)`

---

### 2.4 MODIS stack + safe sampling
Stack:
- `modis_stack = cat([NDVI, LST]).clip(aoi)`

Sampling (avoid reduceRegion on large areas):
- `sample_df` created from `modis_stack.sample(region=aoi, scale=1000, numPixels=20000, geometries=False)`
- In small AOIs, returned sample size may be far lower (e.g., ~150) due to masking and limited pixels.

Diagnostic plot:
- NDVI–LST scatter (triangle behavior check)

---

## Notebook 03 — DISPATCH Triangle + SEE + Downscaling (Core algorithm)
**Notebook:** `03_dispatch_triangle_downscaling.ipynb`

### Purpose
- Estimate triangle edges: `Tmin(NDVI)` and `Tmax(NDVI)`
- Estimate vegetation temperature `Tv` (from high NDVI “cold edge”)
- Compute Fractional Vegetation Cover (FVC)
- Compute soil temperature proxy `Ts`
- Compute Soil Evaporative Efficiency (SEE)
- Use SMAP as the coarse constraint to produce ~1 km soil moisture

---

### 3.1 Fractional Vegetation Cover (FVC)
Given constants:
- `NDVI_s` (bare soil NDVI) ~ 0.15
- `NDVI_v` (full vegetation NDVI) ~ 0.90

Equation:
- `FVC = (NDVI - NDVI_s) / (NDVI_v - NDVI_s)`
Clamp:
- `FVC ∈ [0, 1]`

---

### 3.2 Triangle edges: Tmin, Tmax, Tv
From the NDVI–LST scatter:
- `Tmin(NDVI)` = lower envelope (“cold edge”)
- `Tmax(NDVI)` = upper envelope (“hot edge”)

Typical practical approach:
- Bin NDVI (e.g., 0.02–0.05 bins)
- For each bin, compute:
  - `Tmin_bin` = low percentile (e.g., 5th–10th)
  - `Tmax_bin` = high percentile (e.g., 90th–95th)
- Fit a line or smooth function through these points:
  - `Tmin(NDVI)` and `Tmax(NDVI)`

Vegetation temperature `Tv`:
- estimated near high NDVI from the cold edge (or constant derived from high-NDVI bins)

---

### 3.3 Soil temperature proxy Ts
Equation:
- `Ts = (Tmodis - FVC * Tv) / (1 - FVC)`

Numerical stability:
- for `FVC → 1`, protect denominator:
  - use `(1 - FVC)` floor (e.g., max(1-FVC, 1e-3))

---

### 3.4 Soil Evaporative Efficiency (SEE)
Equation:
- `SEE = (Tmax - Ts) / (Tmax - Tmin)`
Clamp:
- `SEE ∈ [0, 1]`

Interpretation:
- SEE ~ 1 → wet soil / high evaporation efficiency
- SEE ~ 0 → dry soil / low evaporation efficiency

---

### 3.5 Soil moisture downscaling (conceptual)
SMAP provides coarse soil moisture `SMAP` at ~36 km.
DISPATCH uses SEE variability at ~1 km to distribute moisture within the coarse pixel while preserving the coarse constraint.

Common formulations include:
- A proxy transformation linking SEE to soil moisture scaling, then rescaling so that the coarse mean matches SMAP.

(Implementation details are handled in Notebook 03, with careful rescaling to maintain SMAP consistency.)

---

## Common stability rules (important)
- Avoid `reduceRegion()` over large AOIs at 1 km scale.
- Use `sample()` with `numPixels` + `tileScale` where needed.
- Clip early to AOI (`.clip(aoi)`).
- Keep native projections during compositing; only standardize during final export if needed.
- Use QC masks for LST to improve triangle stability.

---
