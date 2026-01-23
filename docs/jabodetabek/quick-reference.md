# Quick Reference Guide

At-a-glance reference for the Jabodetabek Property Valuation v4 feature engineering pipeline.

---

## Table of Contents

- [Pipeline Overview](#pipeline-overview)
- [Final Feature List](#final-feature-list)
- [Feature Categories](#feature-categories)
- [Target Variable](#target-variable)
- [Data Input Summary](#data-input-summary)
- [Common Use Cases](#common-use-cases)
- [Key Formulas](#key-formulas)

---

## Pipeline Overview

**15-Step Feature Engineering Process**

```
1. Data Loading & Spatial Setup
2. Data Preprocessing & Target Variable Completion
3. CRS Reprojection (EPSG:4326 → EPSG:32748)
4. Spatial Distance Feature Engineering
5. Spatial Zone Marking (Binary Features)
6. Additional Data Cleaning & De-duplication
7. Accessibility Feature Engineering
8. Land-use / Area Reclassification Logic
9. Parcel Position Scoring Feature
10. Ownership / Certificate Adjustment Feature
11. Parcel Shape Scoring Feature
12. Economic Index Feature (PDRB ADHB)
13. Log Transform Features
14. Outlier Removal (District-Class Quantile Filtering)
15. Final Feature Generation for Modeling
```

---

## Final Feature List

**Total: 42 Model-Ready Features**

### Distance Features (4)
```python
ln_distance_to_big_city          # Log distance to major city centers
ln_distance_to_small_city        # Log distance to activity centers
ln_distance_to_road              # Log distance to any road
ln_distance_to_main_road         # Log distance to main roads only
```

### Core Continuous (2)
```python
ln_hpm_sertifikat                # Log price per m² (ownership-adjusted)
ln_luas_tanah                    # Log land area
```

### Accessibility & Roads (2)
```python
is_accessible                    # Binary: parcel accessible from road
lebar_jalan_new                  # Road width (0 if not accessible)
```

### Zone/Location Flags (4)
```python
is_the_most_premium              # Binary: Menteng/Kebayoran/Pondok Indah
is_premium_area_2                # Binary: BSD/SGS/Lippo Karawaci/etc.
is_pantai_indah_kapuk            # Binary: PIK/PIK2
is_rural                         # Binary: Rural area
```

### City Dummies (6)
```python
is_jakarta_selatan               # Binary: South Jakarta
is_jakarta_pusat                 # Binary: Central Jakarta
is_jakarta_barat                 # Binary: West Jakarta
is_jakarta_utara                 # Binary: North Jakarta
is_jakarta_timur                 # Binary: East Jakarta
is_bekasi_kota                   # Binary: Bekasi City
```

### Land-use Flags (8)
```python
is_premium                       # Binary: Premium residential
is_industri                      # Binary: Industrial
is_komersial                     # Binary: Commercial
is_hijau                         # Binary: Green/vacant land
is_sstn                          # Binary: Slum + Slum-to-Normal
is_nnp                           # Binary: Normal + Normal-Premium
is_komersial_jakarta             # Binary: Commercial in DKI Jakarta
is_komersial_non_jakarta         # Binary: Commercial outside DKI Jakarta
```

### Scoring Features (2)
```python
score_posisi                     # Numeric: Parcel position score by land-use
score_bentuk                     # Numeric: Parcel shape score by land-use & size
```

### Economic Index (1)
```python
index_pdrb_adhb                  # Numeric: PDRB-based economic index (base=100 at 2010)
```

### Supporting Features (4 - used in engineering)
```python
reclassify_num                   # Numeric land-use code (1-6)
reclassify_nn                    # Normalized land-use label
sisi                             # Effective parcel side length (meters)
jarak_sisi                       # Accessibility distance threshold
```

---

## Feature Categories

### By Data Type

| Type | Count | Features |
|------|-------|----------|
| **Continuous (Log-transformed)** | 6 | ln_distance_*, ln_hpm_sertifikat, ln_luas_tanah |
| **Binary (0/1)** | 22 | is_* flags |
| **Ordinal/Numeric** | 3 | score_posisi, score_bentuk, index_pdrb_adhb |
| **Supporting** | 4 | reclassify_num, reclassify_nn, sisi, jarak_sisi |

### By Feature Domain

| Domain | Count | Examples |
|--------|-------|----------|
| **Geospatial** | 8 | Distance features, zone flags |
| **Administrative** | 6 | City dummies |
| **Land-use** | 9 | Land-use flags + reclassify |
| **Physical Property** | 5 | Accessibility, road width, shape/position scores |
| **Economic** | 2 | Price (target), PDRB index |

---

## Target Variable

### ln_hpm_sertifikat

**Log-transformed, ownership-adjusted price per square meter**

**Computation**:
```
Step 1: Compute base HPM
  harga_setelah_diskon = harga_penawaran × (1 - diskon/100)

  If jenis_objek == 1 (land):
    hpm = harga_setelah_diskon / luas_tanah

  If jenis_objek == 2 (land + building):
    hpm = (harga_setelah_diskon - kemungkinan_transaksi_bangunan) / luas_tanah

Step 2: Adjust for ownership type
  hpm_sertifikat = hpm × adjustment_factor

  Where adjustment_factor depends on:
    - kepemilikan (certificate type)
    - reclassify (land-use category)

Step 3: Log transform
  ln_hpm_sertifikat = log(hpm_sertifikat)
```

**Valid Range (before log)**:
- 100,000 ≤ hpm_sertifikat ≤ 100,000,000 IDR/m²

---

## Data Input Summary

### Primary Dataset
**File**: `jbdtb_100_1.xlsx`
- **20 required columns**
- Property observations with coordinates, pricing, and characteristics

### External Data (6 files)
1. Road network GeoDataFrame
2. `mark_4.geojson` - Premium zones
3. `rural_2.geojson` - Rural areas
4. Big city centers POI
5. Small city centers POI
6. `pdrb_adhk (1).xlsx` - PDRB economic data

### Coordinate Systems
- **Input**: EPSG:4326 (WGS84)
- **Processing**: EPSG:32748 (UTM Zone 48S)

---

## Common Use Cases

### 1. Property Valuation Modeling

**Primary features**:
- `ln_hpm_sertifikat` (target)
- `ln_luas_tanah` (key predictor)
- Distance features (location value)
- Land-use flags (market segment)
- City dummies (regional effects)

**Model types**:
- Linear regression (on log-transformed features)
- Gradient boosting (XGBoost, LightGBM)
- Random forest

---

### 2. Market Segmentation

**Segmentation features**:
- `reclassify_num` / `reclassify_nn` (land-use type)
- `is_the_most_premium`, `is_premium_area_2` (premium tier)
- `is_rural` (urban vs rural)
- City dummies (geographic market)

**Analysis**:
- Price distributions by segment
- Feature importance by segment
- Market trends by land-use

---

### 3. Spatial Analysis

**Key features**:
- `ln_distance_to_big_city` (centrality)
- `ln_distance_to_main_road` (accessibility)
- `is_accessible` (road connectivity)
- Zone flags (strategic location)

**Applications**:
- Distance-decay analysis
- Accessibility impact on pricing
- Premium zone value uplift
- Transit-oriented development effects

---

### 4. Temporal Analysis

**Time-series features**:
- `index_pdrb_adhb` (economic growth proxy)
- `tahun_pengambilan_data` (observation year)
- `ln_hpm_sertifikat` (price trends)

**Analysis**:
- Price appreciation rates
- Economic index correlation
- Year-over-year changes

---

## Key Formulas

### Accessibility

```python
# Effective parcel side length
sisi = f(bentuk_tapak, sqrt(luas_tanah), panjang, lebar, ratios)

# Accessibility threshold
jarak_sisi = sisi + 0.5 × lebar_jalan_di_depan

# Accessibility flag
is_accessible = (distance_to_road ≤ jarak_sisi) OR (is_pantai_indah_kapuk == 1)

# Adjusted road width
lebar_jalan_new = lebar_jalan_di_depan if is_accessible else 0
```

### Commercial Influence

```python
# Commercial radius
rad_komersial = lebar_jalan_di_depan × 2.5

# Commercial area flag
area_komersial = (distance_to_main_road < rad_komersial) AND
                 (kondisi_wilayah_sekitar_reclass == "komersial")
```

### PDRB Economic Index

```python
# Base year: 2010
index_pdrb_adhb = (PDRB_city_year / PDRB_city_2010) × 100
```

### Ownership Adjustment

```python
# Adjustment factors
HM + Category A (residential):  hpm × 0.995
HM + Category B (commercial):   hpm × 1.005
HP/HPL/HGU (any category):      hpm × 1.005
Other (non-HM/HGB):             hpm × 1.01
HGB:                            hpm × 1.0 (no adjustment)
```

### Outlier Removal

```python
# Grouping
group = str(reclassify_num) + wadmkd

# Per-group fences (district × land-use class)
lower_fence = quantile(ln_hpm_sertifikat, 0.05)
upper_fence = quantile(ln_hpm_sertifikat, 0.95)

# Keep records within fences
keep = (ln_hpm_sertifikat >= lower_fence) AND (ln_hpm_sertifikat <= upper_fence)
```

---

## Land-use Encoding

### Numeric Codes (reclassify_num)

| Code | Land-use Category | Short Code (reclassify_nn) |
|------|-------------------|----------------------------|
| 1 | hijau | hijau |
| 2 | slum+slum-to-normal | sstn |
| 3 | normal+normal-premium | nnp |
| 4 | premium | premium |
| 5 | industri | industri |
| 6 | komersial | komersial |

---

## Shape Factors

Heuristic adjustments for parcel side length estimation:

| Shape | Factor |
|-------|--------|
| persegi (square) | 1.0 |
| persegi panjang (rectangle) | 1.3 |
| trapesium (trapezoid) | 1.2 |
| ngantong (pocket) | 1.4 |
| letter l (L-shaped) | 1.5 |
| tidak beraturan (irregular) | 1.1 |
| kipas (fan) | 1.2 |

---

## Data Quality Filters

### Range Filters (Applied in Step 2.2)

```python
100,000 ≤ hpm ≤ 100,000,000           # IDR per m²
30 ≤ luas_tanah ≤ 10,000              # m²
2 ≤ lebar_jalan_di_depan ≤ 80         # m
2017 ≤ tahun_pengambilan_data ≤ 2025  # Year
```

### Coordinate Quality (Step 6.2)

Suspicious coordinate clusters (30m buffer):
- (106.8305723, -6.2184799)
- (106.8280988, -6.2184283)

### De-duplication (Step 6.3)

- Group by: `(latitude, longitude)`
- Sort by: `tahun_pengambilan_data` (descending)
- Keep: Most recent observation per location

---

## Performance Tips

### Feature Selection

**High-importance features** (typically):
1. `ln_luas_tanah` (land area)
2. `ln_distance_to_big_city` (centrality)
3. `is_the_most_premium` (premium location)
4. Land-use flags (`is_premium`, `is_komersial`, etc.)
5. `score_posisi` (position quality)
6. City dummies (regional effects)

**May have lower importance**:
- `ln_distance_to_small_city` (may correlate with big_city)
- `is_premium_area_2` (less exclusive than most_premium)
- `is_rural` (if few rural observations)

### Model Recommendations

**For interpretability**:
- Linear regression on log-transformed features
- Coefficients directly interpretable as elasticities

**For accuracy**:
- Gradient boosting (XGBoost, LightGBM, CatBoost)
- Can handle feature interactions
- May capture non-linear relationships

**For robustness**:
- Ensemble models
- Cross-validation by city or land-use
- Separate models by major segments

---

[← Back to Overview](index.md) | [← Data Requirements](data-requirements.md)
