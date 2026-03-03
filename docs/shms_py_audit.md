SHMS_py DISPATCH Audit
Source

Original implementation:
SHMS_py (Morteza Khazaei)

Python 2.7 GDAL-based implementation of DISPATCH downscaling algorithm.

Core Workflow in SHMS_py

Read MODIS reflectance (Red, NIR)

Read MODIS LST (MOD11A1)

Read SMAP soil moisture (36 km)

Reproject to WGS84

Compute NDVI

Compute Fractional Vegetation Cover (FVC)

Estimate vegetation temperature (Tv)

Estimate soil temperature (Ts)

Compute Soil Evaporative Efficiency (SEE)

Compute mean SEE at SMAP scale

Estimate soil moisture parameter (SMp)

Apply DISPATCH disaggregation equation

Key Equations
NDVI

NDVI = (NIR - Red) / (NIR + Red)

Fractional Vegetation Cover (FVC)

FVC = (NDVI - NDVIs) / (NDVIv - NDVIs)

Where:

NDVIs = 0.15 (bare soil)

NDVIv = 0.9 (pure vegetation)

Soil Temperature

Ts = (Tmodis - FVC * Tv) / (1 - FVC)

Soil Evaporative Efficiency (SEE)

SEE = (Tsmax - Ts) / (Tsmax - Tsmin)

Soil Moisture Parameter

SMp = (π * SMAP) / arccos(1 - 2 * SEE_mean)

DISPATCH Disaggregation

SM_1km = SMAP_rescaled
  + (SMp / π) * sqrt(-SEE² + SEE)
  × (SEE_1km - SEE_mean)

Limitations in SHMS_py

Python 2.7

No cloud masking

Manual resampling

Hardcoded degree-based pixel spacing

No QA filtering

No projection control

Improvements in This Project

Google Earth Engine implementation

Terra + Aqua fusion option

Proper QA masking

Clean reprojection

Daily-first scalable workflow

South Texas AOI focus
