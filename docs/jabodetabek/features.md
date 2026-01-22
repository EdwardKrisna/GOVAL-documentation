# Feature Dependencies & Computation Paths

Complete dependency mapping for all final modeling features in the Jabodetabek Property Valuation v4 pipeline.

---

## Table of Contents

- [Final Features Summary](#final-features-summary)
- [Feature Dependency Details](#feature-dependency-details)
  - [Distance Features](#distance-features)
  - [Core Continuous Features](#core-continuous-features)
  - [Accessibility & Road Features](#accessibility--road-features)
  - [Zone/Location Binary Features](#zonelocation-binary-features)
  - [City Dummy Variables](#city-dummy-variables)
  - [Land-use Type Flags](#land-use-type-flags)
  - [Scoring Features](#scoring-features)
  - [Macro Economic Feature](#macro-economic-feature)
  - [Supporting Features](#supporting-features)
- [Complete Dependency Graph](#complete-dependency-graph)

---

## Final Features Summary

### 1. Distance Features (Log-transformed)
- `ln_distance_to_big_city`
- `ln_distance_to_small_city`
- `ln_distance_to_road`
- `ln_distance_to_main_road`

### 2. Core Continuous Features (Log-transformed)
- `ln_hpm_sertifikat`
- `ln_luas_tanah`

### 3. Accessibility & Road Features
- `lebar_jalan_new`
- `is_accessible`

### 4. Zone/Location Binary Features
- `is_the_most_premium`
- `is_premium_area_2`
- `is_pantai_indah_kapuk`
- `is_rural`

### 5. City Dummy Variables
- `is_jakarta_selatan`
- `is_jakarta_pusat`
- `is_jakarta_barat`
- `is_jakarta_utara`
- `is_jakarta_timur`
- `is_bekasi_kota`

### 6. Land-use Type Flags
- `is_premium`
- `is_industri`
- `is_komersial`
- `is_hijau`
- `is_sstn`
- `is_nnp`
- `is_komersial_jakarta`
- `is_komersial_non_jakarta`

### 7. Scoring Features
- `score_posisi`
- `score_bentuk`

### 8. Macro Economic Feature
- `index_pdrb_adhb`

### 9. Supporting Features (used in feature engineering)
- `reclassify_num`
- `reclassify_nn`
- `sisi`
- `jarak_sisi`

---

## Feature Dependency Details

### Distance Features

#### ln_distance_to_big_city

**Direct Dependencies**: `distance_to_big_city` (computed feature)

**Computation Path**:
```
Step 4: Spatial distance calculation
  ← geometry (point from df)
  ← bc_new_gdf (big city centers POI)
  ← Requires: latitude, longitude (from source data)

Step 13: Log transform
  ln_distance_to_big_city = log1p(distance_to_big_city)
```

**Source Columns Required**:
- `latitude`
- `longitude`

---

#### ln_distance_to_small_city

**Direct Dependencies**: `distance_to_small_city` (computed feature)

**Computation Path**:
```
Step 4: Spatial distance calculation
  ← geometry (point from df)
  ← sc_new_gdf (small city/activity centers POI)
  ← Requires: latitude, longitude

Step 13: Log transform
  ln_distance_to_small_city = log1p(distance_to_small_city)
```

**Source Columns Required**:
- `latitude`
- `longitude`

---

#### ln_distance_to_road

**Direct Dependencies**: `distance_to_road` (computed feature)

**Computation Path**:
```
Step 4: Spatial distance calculation
  ← geometry (point from df)
  ← road (all roads GeoDataFrame)
  ← Requires: latitude, longitude

Step 13: Log transform
  ln_distance_to_road = log1p(distance_to_road)
```

**Source Columns Required**:
- `latitude`
- `longitude`

---

#### ln_distance_to_main_road

**Direct Dependencies**: `distance_to_main_road` (computed feature)

**Computation Path**:
```
Step 4: Spatial distance calculation
  ← geometry (point from df)
  ← main_road_jabodetabek (filtered roads)
  ← Requires: latitude, longitude

Step 13: Log transform
  ln_distance_to_main_road = log1p(distance_to_main_road)
```

**Source Columns Required**:
- `latitude`
- `longitude`

---

### Core Continuous Features

#### ln_hpm_sertifikat

**Direct Dependencies**: `hpm_sertifikat` (adjusted price feature)

**Computation Path**:
```
Step 2: Compute hpm
  hpm = f(harga_penawaran, diskon, jenis_objek, luas_tanah, kemungkinan_transaksi_bangunan)

Step 10: Adjust by ownership
  hpm_sertifikat = hpm × adjustment_factor
  ← Requires: reclassify, kepemilikan

Step 13: Log transform
  ln_hpm_sertifikat = log(hpm_sertifikat)
```

**Source Columns Required**:
- `harga_penawaran`
- `diskon`
- `jenis_objek`
- `luas_tanah`
- `kemungkinan_transaksi_bangunan`
- `kepemilikan`

**Intermediate Dependencies**:
- `reclassify` (from Step 8)

---

#### ln_luas_tanah

**Computation Path**:
```
Step 13: Log transform
  ln_luas_tanah = log(luas_tanah)
```

**Source Columns Required**:
- `luas_tanah`

---

### Accessibility & Road Features

#### lebar_jalan_new

**Direct Dependencies**: `is_accessible`, `lebar_jalan_di_depan`

**Computation Path**:
```
Step 7: Compute accessibility
  is_accessible = f(distance_to_road, sisi, lebar_jalan_di_depan, is_pantai_indah_kapuk)

Step 15: Conditional road width
  lebar_jalan_new = lebar_jalan_di_depan if is_accessible == 1 else 0
```

**Source Columns Required**:
- `lebar_jalan_di_depan`
- `luas_tanah`
- `panjang_maks_kebelakang`
- `lebar_frontage`
- `bentuk_tapak`
- `latitude`
- `longitude`

**Intermediate Dependencies**:
- `distance_to_road` (Step 4)
- `is_pantai_indah_kapuk` (Step 5)
- `sisi` (Step 7)

---

#### is_accessible

**Direct Dependencies**: `distance_to_road`, `jarak_sisi`, `is_pantai_indah_kapuk`

**Computation Path**:
```
Step 7.4: Compute sisi
  sisi = f(bentuk_tapak, sqrt_luas, panjang_maks_kebelakang, lebar_frontage, ratio_luas, ratio_sisi)

Step 7.5: Compute accessibility
  jarak_sisi = sisi + 0.5 × lebar_jalan_di_depan
  is_accessible = 1 if (distance_to_road ≤ jarak_sisi OR is_pantai_indah_kapuk == 1)
```

**Source Columns Required**:
- `luas_tanah`
- `panjang_maks_kebelakang`
- `lebar_frontage`
- `bentuk_tapak`
- `lebar_jalan_di_depan`
- `latitude`
- `longitude`

**Intermediate Dependencies**:
- `distance_to_road` (Step 4)
- `is_pantai_indah_kapuk` (Step 5)

---

### Zone/Location Binary Features

#### is_the_most_premium

**Computation Path**:
```
Step 5.1: Spatial intersection with premium zones
  Check if point intersects marking polygons: {menteng, kebayoran, pondok indah}
  is_the_most_premium = 1 if within polygon, else 0
```

**Source Columns Required**:
- `latitude`
- `longitude`

**External Data Required**:
- `mark_4.geojson` (premium marking polygons)

---

#### is_premium_area_2

**Computation Path**:
```
Step 5.1: Spatial intersection with secondary premium zones
  Check if point intersects marking polygons: {segara city, harapan indah, bsd, sgs, lippo karawaci}
  is_premium_area_2 = 1 if within polygon, else 0
```

**Source Columns Required**:
- `latitude`
- `longitude`

**External Data Required**:
- `mark_4.geojson` (premium marking polygons)

---

#### is_pantai_indah_kapuk

**Computation Path**:
```
Step 5.1: Spatial intersection with PIK zones
  Check if point intersects marking polygons: {pik, pik2}
  is_pantai_indah_kapuk = 1 if within polygon, else 0
```

**Source Columns Required**:
- `latitude`
- `longitude`

**External Data Required**:
- `mark_4.geojson` (premium marking polygons)

---

#### is_rural

**Computation Path**:
```
Step 5.2: Spatial join with rural polygons
  is_rural = 1 if point is within rural polygon, else 0
```

**Source Columns Required**:
- `latitude`
- `longitude`

**External Data Required**:
- `rural_2.geojson` (rural polygons)

---

### City Dummy Variables

#### is_jakarta_selatan

**Computation Path**:
```
Step 15.2: City dummy creation
  is_jakarta_selatan = 1 if wadmkk == "Kota Administrasi Jakarta Selatan" (case-insensitive), else 0
```

**Source Columns Required**: `wadmkk`

---

#### is_jakarta_pusat

**Computation Path**:
```
Step 15.2: City dummy creation
  is_jakarta_pusat = 1 if wadmkk == "Kota Administrasi Jakarta Pusat", else 0
```

**Source Columns Required**: `wadmkk`

---

#### is_jakarta_barat

**Computation Path**:
```
Step 15.2: City dummy creation
  is_jakarta_barat = 1 if wadmkk == "Kota Administrasi Jakarta Barat", else 0
```

**Source Columns Required**: `wadmkk`

---

#### is_jakarta_utara

**Computation Path**:
```
Step 15.2: City dummy creation
  is_jakarta_utara = 1 if wadmkk == "Kota Administrasi Jakarta Utara", else 0
```

**Source Columns Required**: `wadmkk`

---

#### is_jakarta_timur

**Computation Path**:
```
Step 15.2: City dummy creation
  is_jakarta_timur = 1 if wadmkk == "Kota Administrasi Jakarta Timur", else 0
```

**Source Columns Required**: `wadmkk`

---

#### is_bekasi_kota

**Computation Path**:
```
Step 15.2: City dummy creation
  is_bekasi_kota = 1 if wadmkk == "Kota Bekasi", else 0
```

**Source Columns Required**: `wadmkk`

---

### Land-use Type Flags

#### is_premium

**Direct Dependencies**: `reclassify_nn`

**Computation Path**:
```
Step 8: Reclassify land-use
  reclassify = f(kondisi_wilayah_sekitar_reclass, predicted_class, prediction_class, area_komersial)

Step 15.3: Normalize to short codes
  reclassify_nn = normalize(reclassify)

Step 15.4: Create binary flag
  is_premium = 1 if reclassify_nn == "premium", else 0
```

**Source Columns Required**:
- `kondisi_wilayah_sekitar`
- `predicted_class`
- `prediction_class`
- `lebar_jalan_di_depan`
- `latitude`
- `longitude`

**Intermediate Dependencies**:
- `distance_to_main_road` (Step 4)

---

#### is_industri

**Direct Dependencies**: `reclassify_nn`

**Computation Path**:
```
Step 8: Reclassify land-use
Step 15.3: Normalize to short codes
Step 15.4: Create binary flag
  is_industri = 1 if reclassify_nn == "industri", else 0
```

**Source Columns Required**:
- `kondisi_wilayah_sekitar`
- `predicted_class`
- `prediction_class`
- `lebar_jalan_di_depan`
- `latitude`
- `longitude`

---

#### is_komersial

**Direct Dependencies**: `reclassify_nn`

**Computation Path**:
```
Step 8: Reclassify land-use
Step 15.3: Normalize to short codes
Step 15.4: Create binary flag
  is_komersial = 1 if reclassify_nn == "komersial", else 0
```

**Source Columns Required**:
- `kondisi_wilayah_sekitar`
- `predicted_class`
- `prediction_class`
- `lebar_jalan_di_depan`
- `latitude`
- `longitude`

---

#### is_hijau

**Direct Dependencies**: `reclassify_nn`

**Computation Path**:
```
Step 8: Reclassify land-use
Step 15.3: Normalize to short codes
Step 15.4: Create binary flag
  is_hijau = 1 if reclassify_nn == "hijau", else 0
```

**Source Columns Required**:
- `kondisi_wilayah_sekitar`
- `predicted_class`
- `prediction_class`
- `lebar_jalan_di_depan`
- `latitude`
- `longitude`

---

#### is_sstn

**Direct Dependencies**: `reclassify_nn`

**Computation Path**:
```
Step 8: Reclassify land-use
Step 15.3: Normalize to short codes
  reclassify_nn = "sstn" if reclassify == "slum+slum-to-normal"
Step 15.4: Create binary flag
  is_sstn = 1 if reclassify_nn == "sstn", else 0
```

**Source Columns Required**:
- `kondisi_wilayah_sekitar`
- `predicted_class`
- `prediction_class`
- `lebar_jalan_di_depan`
- `latitude`
- `longitude`

---

#### is_nnp

**Direct Dependencies**: `reclassify_nn`

**Computation Path**:
```
Step 8: Reclassify land-use
Step 15.3: Normalize to short codes
  reclassify_nn = "nnp" if reclassify == "normal+normal-premium"
Step 15.4: Create binary flag
  is_nnp = 1 if reclassify_nn == "nnp", else 0
```

**Source Columns Required**:
- `kondisi_wilayah_sekitar`
- `predicted_class`
- `prediction_class`
- `lebar_jalan_di_depan`
- `latitude`
- `longitude`

---

#### is_komersial_jakarta

**Direct Dependencies**: `reclassify_nn`, `wadmpr`

**Computation Path**:
```
Step 8: Reclassify land-use
Step 15.3: Normalize to short codes
Step 15.5: Create Jakarta commercial split
  is_komersial_jakarta = 1 if (reclassify_nn == "komersial" AND wadmpr == "DKI Jakarta"), else 0
```

**Source Columns Required**:
- `kondisi_wilayah_sekitar`
- `predicted_class`
- `prediction_class`
- `lebar_jalan_di_depan`
- `wadmpr`
- `latitude`
- `longitude`

---

#### is_komersial_non_jakarta

**Direct Dependencies**: `reclassify_nn`, `wadmpr`

**Computation Path**:
```
Step 8: Reclassify land-use
Step 15.3: Normalize to short codes
Step 15.5: Create non-Jakarta commercial split
  is_komersial_non_jakarta = 1 if (reclassify_nn == "komersial" AND wadmpr != "DKI Jakarta"), else 0
```

**Source Columns Required**:
- `kondisi_wilayah_sekitar`
- `predicted_class`
- `prediction_class`
- `lebar_jalan_di_depan`
- `wadmpr`
- `latitude`
- `longitude`

---

### Scoring Features

#### score_posisi

**Direct Dependencies**: `reclassify`, `posisi_tapak`

**Computation Path**:
```
Step 8: Reclassify land-use
  reclassify = f(kondisi_wilayah_sekitar_reclass, predicted_class, prediction_class, area_komersial)

Step 9: Apply position scoring
  score_posisi = scoring_dict[reclassify.lower()][posisi_tapak.title()]
```

**Source Columns Required**:
- `kondisi_wilayah_sekitar`
- `predicted_class`
- `prediction_class`
- `posisi_tapak`
- `lebar_jalan_di_depan`
- `latitude`
- `longitude`

---

#### score_bentuk

**Direct Dependencies**: `reclassify`, `bentuk_tapak`, `luas_tanah`

**Computation Path**:
```
Step 8: Reclassify land-use
  reclassify = f(kondisi_wilayah_sekitar_reclass, predicted_class, prediction_class, area_komersial)

Step 11: Apply shape scoring
  size_bucket = determine_bucket(luas_tanah)
  score_bentuk = scoring_map[reclassify.lower()][bentuk_tapak.title()][size_bucket]
```

**Source Columns Required**:
- `kondisi_wilayah_sekitar`
- `predicted_class`
- `prediction_class`
- `bentuk_tapak`
- `luas_tanah`
- `lebar_jalan_di_depan`
- `latitude`
- `longitude`

---

### Macro Economic Feature

#### index_pdrb_adhb

**Direct Dependencies**: `wadmkk`, `tahun_pengambilan_data`

**Computation Path**:
```
Step 12: Build PDRB index
  For each city: index_city = (PDRB_city_year / PDRB_city_2010) × 100

  Map wadmkk to index column
  Fetch index value for corresponding year
  index_pdrb_adhb = pdrb_adhb[index_city][year]
```

**Source Columns Required**:
- `wadmkk`
- `tahun_pengambilan_data`

**External Data Required**:
- `pdrb_adhk (1).xlsx` (sheet: adhb)

---

### Supporting Features

#### reclassify_num

**Direct Dependencies**: `reclassify`

**Computation Path**:
```
Step 8.1-8.4: Build reclassify
  reclassify = f(kondisi_wilayah_sekitar_reclass, predicted_class, prediction_class, area_komersial)

Step 8.5: Encode to numeric
  reclassify_num = mapping[reclassify]
  where mapping = {hijau: 1, slum+slum-to-normal: 2, normal+normal-premium: 3,
                   premium: 4, industri: 5, komersial: 6}
```

**Source Columns Required**:
- `kondisi_wilayah_sekitar`
- `predicted_class`
- `prediction_class`
- `lebar_jalan_di_depan`
- `latitude`
- `longitude`

---

#### reclassify_nn

**Direct Dependencies**: `reclassify`

**Computation Path**:
```
Step 8: Build reclassify
Step 15.3: Normalize to short codes
  reclassify_nn = normalize(reclassify)
  Mappings:
    "slum+slum-to-normal" → "sstn"
    "normal+normal-premium" → "nnp"
    others → strip + lowercase
```

**Source Columns Required**:
- `kondisi_wilayah_sekitar`
- `predicted_class`
- `prediction_class`
- `lebar_jalan_di_depan`
- `latitude`
- `longitude`

---

## Complete Dependency Graph

```
SOURCE COLUMNS
│
├─► latitude, longitude
│   ├─► geometry (point)
│   │   ├─► distance_to_big_city ──► ln_distance_to_big_city
│   │   ├─► distance_to_small_city ──► ln_distance_to_small_city
│   │   ├─► distance_to_road ──► ln_distance_to_road, is_accessible
│   │   └─► distance_to_main_road ──► ln_distance_to_main_road, area_komersial
│   │
│   ├─► is_the_most_premium (via spatial join with mark_4.geojson)
│   ├─► is_premium_area_2 (via spatial join with mark_4.geojson)
│   ├─► is_pantai_indah_kapuk (via spatial join with mark_4.geojson)
│   └─► is_rural (via spatial join with rural_2.geojson)
│
├─► harga_penawaran, diskon, jenis_objek, luas_tanah, kemungkinan_transaksi_bangunan
│   └─► hpm
│       └─► hpm_sertifikat (+ kepemilikan, reclassify)
│           └─► ln_hpm_sertifikat
│
├─► luas_tanah
│   ├─► ln_luas_tanah
│   ├─► sqrt_luas ──► sisi ──► jarak_sisi ──► is_accessible ──► lebar_jalan_new
│   └─► score_bentuk (+ reclassify, bentuk_tapak)
│
├─► lebar_jalan_di_depan
│   ├─► lebar_jalan_new (via is_accessible)
│   ├─► rad_komersial ──► area_komersial ──► reclassify
│   └─► jarak_sisi ──► is_accessible
│
├─► panjang_maks_kebelakang, lebar_frontage, bentuk_tapak
│   └─► sisi ──► jarak_sisi ──► is_accessible ──► lebar_jalan_new
│
├─► kondisi_wilayah_sekitar, predicted_class, prediction_class
│   └─► reclassify
│       ├─► reclassify_num
│       ├─► reclassify_nn
│       │   ├─► is_premium, is_industri, is_komersial, is_hijau, is_sstn, is_nnp
│       │   ├─► is_komersial_jakarta, is_komersial_non_jakarta (+ wadmpr)
│       │   └─► (used in hpm_sertifikat adjustment)
│       ├─► score_posisi (+ posisi_tapak)
│       └─► score_bentuk (+ bentuk_tapak, luas_tanah)
│
├─► wadmkk
│   ├─► is_jakarta_selatan, is_jakarta_pusat, is_jakarta_barat
│   ├─► is_jakarta_utara, is_jakarta_timur, is_bekasi_kota
│   └─► index_pdrb_adhb (+ tahun_pengambilan_data, pdrb_adhb table)
│
├─► wadmpr
│   └─► is_komersial_jakarta, is_komersial_non_jakarta
│
├─► wadmkd
│   └─► wadmkd_agga_score_hyb (outlier removal grouping)
│
├─► tahun_pengambilan_data
│   └─► index_pdrb_adhb (+ wadmkk, pdrb_adhb table)
│
├─► posisi_tapak
│   └─► score_posisi (+ reclassify)
│
├─► bentuk_tapak
│   ├─► sisi ──► jarak_sisi ──► is_accessible
│   └─► score_bentuk (+ reclassify, luas_tanah)
│
├─► kepemilikan
│   └─► hpm_sertifikat (adjustment factor + reclassify)
│
└─► id
    └─► (manual outlier exclusion)
```

---

[← Back to README](README.md) | [← Workflow](WORKFLOW.md) | [Data Requirements →](DATA_REQUIREMENTS.md)
