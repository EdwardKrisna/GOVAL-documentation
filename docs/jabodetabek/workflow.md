# Feature Engineering Workflow

Complete step-by-step documentation of the Jabodetabek Property Valuation v4 feature engineering pipeline.

---

## Table of Contents

1. [Data Loading & Spatial Setup](#1-data-loading--spatial-setup)
2. [Data Preprocessing & Target Variable](#2-data-preprocessing--target-variable-completion)
3. [CRS Reprojection](#3-crs-reprojection-for-metric-distance-calculations)
4. [Spatial Distance Features](#4-spatial-distance-feature-engineering)
5. [Spatial Zone Marking](#5-spatial-zone-marking-binary-features)
6. [Additional Data Cleaning](#6-additional-data-cleaning--de-duplication)
7. [Accessibility Feature Engineering](#7-accessibility-feature-engineering)
8. [Land-use Reclassification](#8-land-use--area-reclassification-logic)
9. [Parcel Position Scoring](#9-parcel-position-scoring-feature)
10. [Ownership Adjustment](#10-ownership--certificate-adjustment-feature)
11. [Parcel Shape Scoring](#11-parcel-shape-scoring-feature)
12. [Economic Index Feature](#12-economic-index-feature-pdrb-adhb-normalization)
13. [Log Transform Features](#13-log-transform-features)
14. [Outlier Removal](#14-outlier-removal)
15. [Final Feature Generation](#15-final-feature-generation-for-modeling)

---

## 1. Data Loading & Spatial Setup

### Objective
Load raw tabular dataset and convert it into spatial point data for geospatial feature generation.

### Inputs
- **Raw dataset**: `jbdtb_100_1.xlsx`
- **Road network** (GeoDataFrame): `road`
- **Administrative/zone polygons**:
  - Premium marking polygons: `mark_4.geojson`
  - Rural polygons: `rural_2.geojson`
- **Macro data**: PDRB table: `pdrb_adhk (1).xlsx` (sheet: `adhb`)
- **Reference POIs** (constructed manually):
  - Big city center points (example: BEJ)
  - Small city/activity center points (malls, stadium, botanical garden, etc.)

### Process
1. Read Excel and convert to GeoDataFrame using longitude and latitude with CRS EPSG:4326
2. Create a subset of roads `main_road_jabodetabek` based on major road classes:
   - `primary`, `secondary`, `motorway`, `trunk` + their `_link` variants
3. Load polygon layers (`mark`, `rural`) and ensure CRS is consistent for spatial operations

### Output
- `df` as point GeoDataFrame (initially EPSG:4326)
- `main_road_jabodetabek` as filtered road GeoDataFrame
- POI GeoDataFrames: `bc_new_gdf` and `sc_new_gdf`

---

## 2. Data Preprocessing & Target Variable Completion

### Objective
Ensure essential numeric variable `hpm` (harga per meter) exists and remove invalid/outlier records.

### 2.1 Compute Missing HPM

**Feature created**: `hpm`

**Logic**:

Discounted price:
```
harga_setelah_diskon = harga_penawaran × (1 - diskon/100)
```

If `jenis_objek == 1` (land):
```
hpm = harga_setelah_diskon / luas_tanah
```

If `jenis_objek == 2` (land + building):
```
hpm = (harga_setelah_diskon - kemungkinan_transaksi_bangunan) / luas_tanah
```

**Guardrails**:
- If `luas_tanah ≤ 0` → NaN

**Cleaning action**:
- Fill `hpm` only for rows where it is missing
- Drop rows where `hpm` remains NaN

### 2.2 Rule-based Filtering (Outlier Control)

**Objective**: Keep records within realistic ranges.

**Filters applied**:
- `100,000 ≤ hpm ≤ 100,000,000`
- `30 ≤ luas_tanah ≤ 10,000`
- `2 ≤ lebar_jalan_di_depan ≤ 80`
- `2017 ≤ tahun_pengambilan_data ≤ 2025`

### Output
A cleaned subset of `df` with valid `hpm` and reasonable ranges.

---

## 3. CRS Reprojection for Metric Distance Calculations

### Objective
Convert all spatial layers into a projected CRS so distance features are in meters.

### CRS Conversion

**All layers converted**: EPSG:4326 → EPSG:32748

Layers converted:
- `df`
- `road`
- `main_road_jabodetabek`
- `bc_new_gdf`
- `sc_new_gdf`
- `mark` and `rural`

**Reason**: Distance in geographic CRS (lat/lon) is not in meters; projected CRS is required for correct Euclidean distance.

---

## 4. Spatial Distance Feature Engineering

### Objective
Generate continuous proximity variables (in meters) to key spatial objects using spatial index nearest queries.

### Method
Use spatial index (`sindex.nearest`) to compute nearest distance from each property point to:
1. Big city centers (`bc_new_gdf`)
2. Small city/activity centers (`sc_new_gdf`)
3. Any road (`road`)
4. Main roads only (`main_road_jabodetabek`)

### Features Created
- `distance_to_big_city`
- `distance_to_small_city`
- `distance_to_road`
- `distance_to_main_road`

### Output
`df` enriched with 4 continuous distance features.

---

## 5. Spatial Zone Marking (Binary Features)

### Objective
Add categorical/binary features indicating whether a property falls inside predefined strategic zones.

### 5.1 Premium Zone Flags (Polygon Intersection)

Polygons sourced from `mark_4.geojson`, using marking labels.

**Features created**:

#### 1. Most premium core areas
- `marking ∈ {menteng, kebayoran, pondok indah}`
- **Feature**: `is_the_most_premium` (0/1)
- **Logic**: Point intersects polygon group

#### 2. Secondary premium areas (excluding PIK group)
- `marking ∈ {segara city, harapan indah, bsd, sgs, lippo karawaci}`
- **Feature**: `is_premium_area_2` (0/1)

#### 3. PIK specific flag
- `marking ∈ {pik, pik2}`
- **Feature**: `is_pantai_indah_kapuk` (0/1)

### 5.2 Rural Flag (Spatial Join "Within")
- Remove `index_right` column (to avoid conflicts)
- Spatial join `df` with rural polygons and mark if point is inside rural polygon
- **Feature created**: `is_rural` (0/1)

### Output
`df` enriched with zone-based binary indicators.

---

## 6. Additional Data Cleaning & De-duplication

### Objective
Remove incomplete labels, remove suspected wrong-coordinate records, and keep the most recent record per coordinate.

### 6.1 Remove Missing Critical Fields

Records are removed if any of these are missing:
- `predicted_class`
- `kondisi_wilayah_sekitar`

**Action**: `dropna(subset=[...])`

### 6.2 Remove Wrong-Coordinate Cluster (Buffer Exclusion)

**Objective**: Exclude points suspected to be in an incorrect location (GPS/data entry issue), defined by known coordinate anchors.

**Process**:
1. Define suspicious coordinates:
   - (106.8305723, -6.2184799)
   - (106.8280988, -6.2184283)
2. Convert to GeoDataFrame (EPSG:4326) → reproject to EPSG:32748
3. Create 30-meter buffer around those points
4. Remove any record where property point lies within buffer union

### 6.3 De-duplicate by Lat/Long Keeping Newest Observation

**Objective**: If multiple observations exist at the same location, keep the latest record.

**Process**:
1. Sort descending by `tahun_pengambilan_data`
2. Drop duplicates by `(latitude, longitude)`, keep first (newest)

### Output
- A filtered dataset excluding likely coordinate errors
- One row per coordinate pair (latest year retained)

---

## 7. Accessibility Feature Engineering

### Objective
Create a binary indicator capturing whether a parcel is "accessible" from roads, considering parcel geometry proxy and road width; with a special rule for PIK.

### 7.1 Shape Factor Definition (Heuristic Geometry Adjustment)

A lookup factor used to adjust estimated parcel side length depending on tapak shape (`bentuk_tapak`):

| Shape | Factor |
|-------|--------|
| persegi | 1.0 |
| persegi panjang | 1.3 |
| trapesium | 1.2 |
| ngantong | 1.4 |
| letter l | 1.5 |
| tidak beraturan | 1.1 |
| kipas | 1.2 |

### 7.2 Numeric Coercion and Normalization

**Objective**: Ensure consistent numeric types and handle missing.

**Conversions applied**:
- `luas_tanah`: numeric, invalid → NaN → clipped to ≥ 0
- `distance_to_road`: numeric
- `panjang_maks_kebelakang`: numeric, missing → 0
- `lebar_frontage`: numeric, missing → 0
- `lebar_jalan_di_depan`: numeric, missing → 0
- `bentuk_tapak`: fill empty string, normalize to lowercase/strip

### 7.3 Intermediate Geometry Proxy Features

**Features created**:

#### sqrt_luas
```
sqrt_luas = sqrt(luas_tanah)
```

#### ratio_luas
Compares land area vs rectangle approximation (panjang_maks_kebelakang × lebar_frontage):
- If either side is 0 → inf
- Else: `ratio = max(luas_tanah, p×l) / min(luas_tanah, p×l)`

#### ratio_sisi
Aspect ratio of sides:
- If min side is 0 → inf
- Else: `ratio = max(panjang, lebar) / min(panjang, lebar)`

### 7.4 Compute "Sisi" (Effective Side Length)

**Feature created**: `sisi`

**Method**: `compute_sisi(row)` uses:
- `bentuk_tapak`
- `sqrt_luas`
- `panjang_maks_kebelakang`, `lebar_frontage`
- `ratio_luas`, `ratio_sisi`

**Heuristic rule summary**:

If panjang or lebar missing (0):
- If `persegi` → `sqrt_luas`
- Else → `sqrt_luas × shape_factor(bentuk) × 2`

If sides exist:
- If `ratio_luas ≤ 1.5`:
  - If `persegi` and `ratio_sisi ≤ 1.5` → `sqrt_luas`
  - Else → `panjang`
  - If non-persegi → `panjang`
- If `ratio_luas > 1.5`:
  - If `persegi` → `sqrt_luas`
  - Else → `sqrt_luas × shape_factor(bentuk) × 2`

### 7.5 Accessibility Decision Variables

**Features created**:

#### jarak_sisi
```
jarak_sisi = sisi + 0.5 × lebar_jalan_di_depan
```

#### is_accessible (0/1)

**Accessibility rule**:

A record is accessible if:
- `distance_to_road ≤ jarak_sisi`, OR
- `is_pantai_indah_kapuk == 1` (PIK override)

Therefore:
- `is_accessible = 1` if either condition true, else 0

---

## 8. Land-use / Area Reclassification Logic

### Objective
Harmonize/clean land-use labels and generate a numeric land-use class used downstream.

### 8.1 Reclassify "kondisi_wilayah_sekitar" into Standard Groups

**Feature created**: `kondisi_wilayah_sekitar_reclass`

**Mapping groups**:

| Group | Source Categories |
|-------|-------------------|
| Hijau | Hijau, Kosong Pertanian |
| Perumahan | Perumahan, Perumahan Sederhana, Menengah, Mewah |
| Komersial | Komersial, Pemerintahan, Komersial Menengah/UKM/Primer |
| Campuran | Campuran |
| Industri | Industri, Industri/Perdagangan…, Industri/Pergudangan Besar |
| Lainnya | Lainnya, Dekat Sungai/Parit, Rawan Bencana, Dekat TPU |

Anything else → "Lainnya"

### 8.2 Filter Invalid Empty Prediction Label

**Action**: Remove rows where `prediction_class == ""`

### 8.3 Commercial Influence Zone (Derived from Road Width)

**Features created**:

#### rad_komersial
```
rad_komersial = lebar_jalan_di_depan × 2.5
```

#### area_komersial (0/1)

**Rule**:
```
area_komersial = 1 if:
  - distance_to_main_road < rad_komersial AND
  - kondisi_wilayah_sekitar_reclass == "komersial"
Else 0.
```

### 8.4 Final Rule-based Reclassification (Multi-source Reconciliation)

**Features created**:
- `reclassify` (string)
- `reclassify_num` (integer code)

**Inputs used per row**:
- `kondisi_wilayah_sekitar_reclass` (context label)
- `predicted_class` (AGGA / your baseline class)
- `prediction_class` (model predicted class)
- `area_komersial` (commercial override indicator)

**Normalization**:
- Convert "Green" → "Hijau" (for both AGGA and model labels)

**Core decision logic** (high-level):
- If context is **Campuran/Lainnya/empty** → use AGGA (`predicted_class`)
- If context is **Industri**:
  - Allow predicted to stay (Industri/Komersial)
  - If predicted Perumahan → revert to AGGA
  - If predicted Hijau → force Industri
- If context is **Hijau**:
  - If predicted Industri/Komersial → force Hijau
  - If predicted Hijau → keep
  - If predicted Perumahan → revert to AGGA
- If context is **Perumahan** → always AGGA
- If context is **Komersial**:
  - If `area_komersial == 1` → force Komersial
  - Else if `area_komersial == 0`:
    - If AGGA ≠ Hijau → AGGA
    - If AGGA == Hijau → Komersial

**Post-process**:
- Lowercase `reclassify`, replace "green" → "hijau"

### 8.5 Encode to Numeric Land-use Class

**Mapping** (`rn_mapping`):

| Land-use | Numeric Code |
|----------|--------------|
| hijau | 1 |
| slum+slum-to-normal | 2 |
| normal+normal-premium | 3 |
| premium | 4 |
| industri | 5 |
| komersial | 6 |

**Feature created**: `reclassify_num = rn_mapping[reclassify]`

**Final cleaning**:
- Remove rows where `reclassify_num` is invalid/zero-like
- Drop rows where `reclassify` is missing

---

## 9. Parcel Position Scoring Feature

### Objective
Convert categorical `posisi_tapak` (site position) into a numeric score that differs by land-use class.

### 9.1 Define Scoring Rules by Class

A nested scoring dictionary assigns `posisi_tapak` → score depending on `reclassify` category:

**Classes covered explicitly**:
- `premium`
- `slum+slum-to-normal`
- `komersial`
- `hijau`

**Extended by inheritance**:
- `industri` uses the same scoring as `komersial`
- `normal+normal-premium` uses the same scoring as `slum+slum-to-normal`

### 9.2 Apply to Dataframe

**Feature created**: `score_posisi`

**Method**:
1. Normalize keys:
   - `reclassify` → lower()
   - `posisi_tapak` → title()
2. Lookup score in the dictionary
3. If not found → NaN

### Output
`df['score_posisi']` is a numeric ordinal feature representing location advantage/disadvantage based on land-use context.

---

## 10. Ownership / Certificate Adjustment Feature

### Objective
Adjust `hpm` into `hpm_sertifikat` to reflect price impacts due to different land title types (`kepemilikan`), with different direction by land-use group.

### 10.1 Normalize Text Labels

Temporary normalized fields:
- `reclassify_norm = reclassify.strip().lower()`
- `kepemilikan_norm = kepemilikan.strip().lower()`

### 10.2 Create Adjusted Price Metric

**Feature created**: `hpm_sertifikat`

**Initialization**: `hpm_sertifikat = hpm` (base)

### 10.3 Define Land-use Groups

**Category A** (residential-like / premium spectrum):
- `slum+slum-to-normal`, `normal+normal-premium`, `premium`

**Category B** (commercial/industry/green spectrum):
- `komersial`, `hijau`, `industri`

### 10.4 Define Ownership Groups

**HM group**:
- hak milik (hm)

**HP/HPL/HGU group**:
- hak pakai (hp), hak pengelolaan (hpl), hak guna usaha (hgu)

**HGB group** (no adjustment):
- hak guna bangunan (hgb), hgb diatas hpl

### 10.5 Apply Rule-based Multipliers

**Adjustments applied** for rows in Category A or B:

1. **HM + Category A** → 0.995
   - `hpm_sertifikat = hpm × 0.995`

2. **HM + Category B** → 1.005
   - `hpm_sertifikat = hpm × 1.005`

3. **HP/HPL/HGU** (any category) → 1.005
   - `hpm_sertifikat = hpm × 1.005`

4. **Other ownership types** (excluding HM, HP/HPL/HGU, HGB) → 1.01
   - `hpm_sertifikat = hpm × 1.01`

5. **HGB rows**
   - Not matched in masks → remain `hpm_sertifikat = hpm`

### Output
`hpm_sertifikat` is an engineered "ownership-adjusted" price per meter feature.

---

## 11. Parcel Shape Scoring Feature

### Objective
Transform categorical parcel shape (`bentuk_tapak`) into a numeric score that depends on land-use class and parcel size.

### 11.1 Scoring Matrix by Land-use and Size Bucket

A nested `scoring_map` defines:

**Structure**:
```
reclassify class (e.g., premium / komersial / hijau / industri, etc.)
  → bentuk_tapak category (Persegi, Persegi Panjang, Trapesium, Ngantong, Letter L, Kipas, Tidak Beraturan)
    → size bucket score
```

**Size buckets**:
- `<500` (luas_tanah < 500)
- `500-10000` (500 ≤ luas_tanah ≤ 10000)
- `>10000` (luas_tanah > 10000)

### 11.2 Scoring Function

**Function**: `score_bentuk(row)`

**Steps**:
1. Normalize:
   - `reclassify` → lower().strip()
   - `bentuk_tapak` → title().strip()
2. Determine `size_key` from `luas_tanah`
3. Lookup score in `scoring_map`
4. If `luas_tanah` missing or key not found → return NaN

### 11.3 Apply to Dataframe

**Feature created**: `score_bentuk`

### Output
`score_bentuk` provides a numeric proxy for parcel efficiency/market preference based on shape, conditioned on land-use + parcel size.

---

## 12. Economic Index Feature (PDRB ADHB Normalization)

### Objective
Attach a macro-economic growth proxy to each observation based on city (`wadmkk`) and year (`tahun_pengambilan_data`).

### 12.1 Build PDRB Index (Base Year = 2010)

**Input**: `pdrb_adhb` table with column `Tahun` + city columns.

**Method**:

Choose base year: **2010**

For each city column (all columns except `Tahun`):
```
index_city = (PDRB_city_year / PDRB_city_2010) × 100
```

**Output**: New columns added to `pdrb_adhb`: `index_<city>` (base=100 at year 2010)

### 12.2 Map City Label to Correct Index Column

A mapping dictionary translates `wadmkk` names into the matching index column:

**Examples**:
- "Kota Administrasi Jakarta Selatan" → "index_Jakarta Selatan"
- "Kota Depok" → "index_Kota Depok"
- etc.

### 12.3 Assign Index to Each Row

**Feature created**: `index_pdrb_adhb`

**Method**:

For each record:
1. Locate `city = wadmkk`
2. Locate `year = tahun_pengambilan_data`
3. Fetch corresponding `pdrb_adhb[index_city]` for that year
4. If no match → None

### Output
`df['index_pdrb_adhb']` as a numeric macro index per observation.

---

## 13. Log Transform Features

### Objective
Reduce skewness and stabilize variance for distance and scale variables (common for pricing models).

### 13.1 Log-transform Distance Features

**Selection rule**:
```python
distance_cols = [col for col in df.columns if col.startswith("distance")]
```

**Features created** (for each distance column):
```
ln_<distance_col> = log1p(distance_col)
```

**Note**: `log1p(x)` is used to handle zero distances safely.

### 13.2 Log-transform Core Continuous Variables

**Features created**:
```
ln_hpm_sertifikat = log(hpm_sertifikat)
ln_luas_tanah = log(luas_tanah)
```

### Output
Log-scale versions of the main price metric and land area.

---

## 14. Outlier Removal

### Objective
Remove extreme observations using a two-stage strategy:
1. Manual ID exclusion list
2. District-class quantile filtering

### 14.1 Manual Outlier Exclusion by Object ID

**Input**: `ids_to_drop` list

**Action**: Remove rows where `id` is in `ids_to_drop`

### 14.2 Checkpoint Dataset Creation

**Action**: `df_checkpoint = df.copy()`

### 14.3 Define Grouping Key (District × Class)

**Features created** (helper):

#### wadmkd_agga_score_hyb
```
wadmkd_agga_score_hyb = str(reclassify_num) + wadmkd
```
This acts as a group identifier for "district + land-use class"

#### object_id_hyb
```
object_id_hyb = id
```

### 14.4 Quantile Fence Filtering Within Each Group

**Method**:

For each unique `wadmkd_agga_score_hyb` group:
1. Use `district_price = ln_hpm_sertifikat`
2. Define fences:
   - `lower_fence = quantile(district_price, 0.05)`
   - `upper_fence = quantile(district_price, 0.95)`
3. Flag outliers if:
   - `ln_hpm_sertifikat < lower_fence OR > upper_fence`

**Stored outputs** (auditability):

Dictionaries for borders:
- `up_value[group] = upper_fence`
- `lower_value[group] = lower_fence`

Outlier list:
- `is_outlier_hyb` stores IDs outside the fences

### 14.5 Annotate and Remove Outliers

**Features created** (diagnostic):
- `upper_border` (per row)
- `lower_border` (per row)
- `is_outlier_hyb` (0/1)

**Action**: Keep only rows where `is_outlier_hyb == 0`

### Output
A cleaned dataset with district-class-adjusted trimming (5–95% band).

---

## 15. Final Feature Generation for Modeling

### Objective
Produce final engineered predictors used directly in the ML model.

### 15.1 Accessibility-adjusted Road Width

**Feature created**: `lebar_jalan_new`

**Rule**:
- If `is_accessible == 0` → `lebar_jalan_new = 0`
- Else → `lebar_jalan_new = lebar_jalan_di_depan`

This encodes "road width only matters when accessible."

### 15.2 City Dummy Variables (Jakarta Admin + Bekasi City)

**Binary features created**:
- `is_jakarta_selatan`
- `is_jakarta_pusat`
- `is_jakarta_barat`
- `is_jakarta_utara`
- `is_jakarta_timur`
- `is_bekasi_kota`

**Rule**: 1 if `wadmkk` equals the respective city string (case-insensitive), else 0.

### 15.3 Normalize Reclass Labels into Short Codes

**Feature created**: `reclassify_nn`

**Mapping**:
- `slum+slum-to-normal` → `sstn`
- `normal+normal-premium` → `nnp`
- Others remain as-is but normalized: strip + lowercase

### 15.4 One-hot Style Land-use Flags

**Binary features created**:
- `is_premium`
- `is_industri`
- `is_komersial`
- `is_hijau`
- `is_sstn`
- `is_nnp`

**Rule**: Compare `reclassify_nn` to each target code.

### 15.5 Commercial Area Split: Jakarta vs Non-Jakarta

**Binary features created**:

#### is_komersial_jakarta
```
= 1 if (reclassify_nn == 'komersial' AND wadmpr == 'DKI Jakarta')
```

#### is_komersial_non_jakarta
```
= 1 if (reclassify_nn == 'komersial' AND wadmpr != 'DKI Jakarta')
```

---

## Final Deliverable of Feature Engineering Stage

At the end of the pipeline, your modeling table includes:

### Macro Feature
- `index_pdrb_adhb`

### Transforms
- `ln_distance_*` (for all distance columns)
- `ln_hpm_sertifikat`
- `ln_luas_tanah`

### Outlier Control Outputs (Optional Diagnostics)
- `upper_border`
- `lower_border`
- `is_outlier_hyb` (used to filter)

### Model-ready Engineered Predictors

**Road accessibility**:
- `lebar_jalan_new`

**City dummies**:
- `is_jakarta_*`
- `is_bekasi_kota`

**Land-use flags**:
- `is_premium`
- `is_industri`
- `is_komersial`
- `is_hijau`
- `is_sstn`
- `is_nnp`

**Commercial split**:
- `is_komersial_jakarta`
- `is_komersial_non_jakarta`

---

[← Back to Overview](index.md) | [Features →](features.md)
