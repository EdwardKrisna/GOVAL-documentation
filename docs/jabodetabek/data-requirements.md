# Data Requirements

Complete specification of all required data inputs for the Jabodetabek Property Valuation v4 feature engineering pipeline.

---

## Table of Contents

- [Primary Dataset](#primary-dataset)
- [External Geospatial Data](#external-geospatial-data)
- [Macro Economic Data](#macro-economic-data)
- [Point of Interest (POI) Data](#point-of-interest-poi-data)
- [Data Quality Expectations](#data-quality-expectations)
- [Summary](#summary)

---

## Primary Dataset

**File**: `jbdtb_100_1.xlsx`

This is the main property observation dataset containing all property characteristics, pricing, and location information.

### Required Columns (20 total)

#### Spatial Coordinates (2 columns)

| Column | Type | Description | Example |
|--------|------|-------------|---------|
| `latitude` | Float | Latitude coordinate in decimal degrees (WGS84) | -6.2184799 |
| `longitude` | Float | Longitude coordinate in decimal degrees (WGS84) | 106.8305723 |

**Requirements**:
- Must be valid WGS84 coordinates (EPSG:4326)
- Should cover Jabodetabek region
- No NULL values allowed

---

#### Property Characteristics (6 columns)

| Column | Type | Description | Valid Range |
|--------|------|-------------|-------------|
| `luas_tanah` | Float | Land area in square meters | 30 - 10,000 m² |
| `lebar_jalan_di_depan` | Float | Width of road in front of parcel (meters) | 2 - 80 m |
| `panjang_maks_kebelakang` | Float | Maximum depth of parcel (meters) | ≥ 0 |
| `lebar_frontage` | Float | Frontage width (meters) | ≥ 0 |
| `bentuk_tapak` | String | Parcel shape category | See below |
| `posisi_tapak` | String | Parcel position/location type | See below |

**`bentuk_tapak` valid values**:
- `persegi` (square)
- `persegi panjang` (rectangle)
- `trapesium` (trapezoid)
- `ngantong` (irregular/pocket)
- `letter l` (L-shaped)
- `tidak beraturan` (irregular)
- `kipas` (fan-shaped)

**`posisi_tapak` examples**:
- Position values vary by land-use class
- Used in position scoring logic

---

#### Pricing & Transaction (5 columns)

| Column | Type | Description | Valid Range |
|--------|------|-------------|-------------|
| `harga_penawaran` | Float | Asking price (IDR) | > 0 |
| `diskon` | Float | Discount percentage | 0 - 100 |
| `jenis_objek` | Integer | Object type | 1 (land only) or 2 (land + building) |
| `kemungkinan_transaksi_bangunan` | Float | Estimated building transaction value (IDR) | ≥ 0 |
| `kepemilikan` | String | Land title/ownership type | See below |

**`kepemilikan` valid values**:
- `hak milik (hm)` - Freehold
- `hak guna bangunan (hgb)` - Building use right
- `hgb diatas hpl` - HGB on top of HPL
- `hak pakai (hp)` - Use right
- `hak pengelolaan (hpl)` - Management right
- `hak guna usaha (hgu)` - Cultivation right

**Derived field**:
- `hpm` (harga per meter) is computed from these columns if missing

---

#### Land-use & Classification (3 columns)

| Column | Type | Description | Valid Values |
|--------|------|-------------|--------------|
| `kondisi_wilayah_sekitar` | String | Surrounding area condition/context | See reclassification mapping |
| `predicted_class` | String | AGGA baseline classification | hijau, perumahan, komersial, industri, etc. |
| `prediction_class` | String | Model predicted classification | hijau, perumahan, komersial, industri, etc. |

**`kondisi_wilayah_sekitar` categories**:
- Hijau (green/vacant)
- Kosong Pertanian (agricultural)
- Perumahan (residential) - various subtypes
- Komersial (commercial) - various subtypes
- Campuran (mixed-use)
- Industri (industrial) - various subtypes
- Lainnya (other)

**Requirements**:
- No NULL values allowed for these columns
- Values normalized and reclassified in Step 8

---

#### Administrative (3 columns)

| Column | Type | Description | Example |
|--------|------|-------------|---------|
| `wadmkk` | String | City/Municipality (Kabupaten/Kota) | "Kota Administrasi Jakarta Selatan" |
| `wadmpr` | String | Province | "DKI Jakarta" |
| `wadmkd` | String | District (Kecamatan) | "Kebayoran Baru" |

**Requirements**:
- Used for city dummies, economic index mapping, and outlier grouping
- Must match PDRB table city names

---

#### Temporal & Identifier (2 columns)

| Column | Type | Description | Valid Range |
|--------|------|-------------|-------------|
| `tahun_pengambilan_data` | Integer | Year of data collection | 2017 - 2025 |
| `id` | String/Integer | Unique property identifier | Unique per record |

**Requirements**:
- `tahun_pengambilan_data` used for de-duplication and economic index
- `id` used for manual outlier exclusion

---

## External Geospatial Data

### 1. Road Network GeoDataFrame

**Variable name**: `road`

**Description**: Complete road network for Jabodetabek region

**Format**: GeoDataFrame (LineString geometries)

**Required attributes**:
- Geometry column (LineString)
- Road class/type column (for filtering main roads)

**Road classes for main roads** (`main_road_jabodetabek` subset):
- `primary`
- `secondary`
- `motorway`
- `trunk`
- `primary_link`
- `secondary_link`
- `motorway_link`
- `trunk_link`

**CRS**: Should be provided in EPSG:4326 (will be converted to EPSG:32748)

---

### 2. Premium Zone Polygons

**File**: `mark_4.geojson`

**Description**: Polygons defining premium/strategic areas with marking labels

**Format**: GeoJSON (Polygon geometries)

**Required attribute**:
- `marking` column with zone identifiers

**Zone categories used**:

1. **Most premium core areas**:
   - `menteng`
   - `kebayoran`
   - `pondok indah`

2. **Secondary premium areas**:
   - `segara city`
   - `harapan indah`
   - `bsd`
   - `sgs`
   - `lippo karawaci`

3. **PIK zones**:
   - `pik`
   - `pik2`

**CRS**: EPSG:4326 (will be converted to EPSG:32748)

---

### 3. Rural Area Polygons

**File**: `rural_2.geojson`

**Description**: Polygons defining rural/non-urban areas

**Format**: GeoJSON (Polygon geometries)

**Required**: Geometry column only (no specific attributes needed)

**CRS**: EPSG:4326 (will be converted to EPSG:32748)

---

## Macro Economic Data

**File**: `pdrb_adhk (1).xlsx`

**Sheet**: `adhb`

**Description**: PDRB (Regional Gross Domestic Product) at current prices by city and year

### Required Structure

| Column | Type | Description |
|--------|------|-------------|
| `Tahun` | Integer | Year |
| `<City 1>` | Float | PDRB value for City 1 |
| `<City 2>` | Float | PDRB value for City 2 |
| ... | ... | ... |

**Example columns**:
- `Tahun`
- `Jakarta Selatan`
- `Jakarta Pusat`
- `Jakarta Barat`
- `Jakarta Utara`
- `Jakarta Timur`
- `Kota Bogor`
- `Kota Depok`
- `Kota Tangerang`
- `Kota Tangerang Selatan`
- `Kota Bekasi`
- `Kabupaten Bogor`
- `Kabupaten Tangerang`
- `Kabupaten Bekasi`

### Requirements

- Must include **year 2010** (base year for indexing)
- Must cover years 2017-2025 (matching property data range)
- City names must map to `wadmkk` values in property dataset

### City Name Mapping

The pipeline expects specific mappings from `wadmkk` to PDRB column names:

| wadmkk (Property Data) | PDRB Column |
|------------------------|-------------|
| Kota Administrasi Jakarta Selatan | Jakarta Selatan |
| Kota Administrasi Jakarta Pusat | Jakarta Pusat |
| Kota Administrasi Jakarta Barat | Jakarta Barat |
| Kota Administrasi Jakarta Utara | Jakarta Utara |
| Kota Administrasi Jakarta Timur | Jakarta Timur |
| Kota Depok | Kota Depok |
| Kota Bekasi | Kota Bekasi |
| ... | ... |

---

## Point of Interest (POI) Data

These are manually constructed reference point datasets.

### 1. Big City Centers

**Variable name**: `bc_new_gdf`

**Description**: Major city center points (CBD, major commercial hubs)

**Format**: GeoDataFrame (Point geometries)

**Example**: BEJ (Jakarta Business District)

**Requirements**:
- Point geometries
- CRS: EPSG:4326 (will be converted to EPSG:32748)

---

### 2. Small City / Activity Centers

**Variable name**: `sc_new_gdf`

**Description**: Secondary activity centers and landmarks

**Format**: GeoDataFrame (Point geometries)

**Examples**:
- Shopping malls
- Stadiums
- Botanical gardens
- Universities
- Hospitals

**Requirements**:
- Point geometries
- CRS: EPSG:4326 (will be converted to EPSG:32748)

---

## Data Quality Expectations

### Completeness

**Critical fields** (no NULL values allowed):
- `latitude`, `longitude`
- `luas_tanah`
- `harga_penawaran`, `diskon`, `jenis_objek`
- `kondisi_wilayah_sekitar`
- `predicted_class`, `prediction_class`
- `wadmkk`, `wadmpr`, `wadmkd`
- `tahun_pengambilan_data`

**Nullable fields** (will be handled in preprocessing):
- `panjang_maks_kebelakang` (fills 0)
- `lebar_frontage` (fills 0)
- `hpm` (will be computed if missing)

### Valid Ranges

Applied in Step 2.2:
- `100,000 ≤ hpm ≤ 100,000,000` (IDR)
- `30 ≤ luas_tanah ≤ 10,000` (m²)
- `2 ≤ lebar_jalan_di_depan ≤ 80` (m)
- `2017 ≤ tahun_pengambilan_data ≤ 2025`

### Coordinate Quality

- Coordinates must be in Jabodetabek region
- Suspicious coordinate clusters will be removed (Step 6.2):
  - (106.8305723, -6.2184799)
  - (106.8280988, -6.2184283)
  - 30m buffer around these points

### Duplicates

- Duplicate coordinates are handled in Step 6.3
- Most recent observation per location is retained

---

## Summary

### Minimum Required Data Files: 6

1. **Primary Dataset**: `jbdtb_100_1.xlsx` (20 columns required)
2. **Road Network**: GeoDataFrame with road classification
3. **Premium Zones**: `mark_4.geojson` with marking labels
4. **Rural Areas**: `rural_2.geojson`
5. **Big City Centers**: POI GeoDataFrame
6. **Small City Centers**: POI GeoDataFrame
7. **PDRB Table**: `pdrb_adhk (1).xlsx` (sheet: adhb)

### Total Source Columns: 20

From `jbdtb_100_1.xlsx`:
1. latitude
2. longitude
3. luas_tanah
4. lebar_jalan_di_depan
5. panjang_maks_kebelakang
6. lebar_frontage
7. bentuk_tapak
8. posisi_tapak
9. harga_penawaran
10. diskon
11. jenis_objek
12. kemungkinan_transaksi_bangunan
13. kepemilikan
14. kondisi_wilayah_sekitar
15. predicted_class
16. prediction_class
17. wadmkk
18. wadmpr
19. wadmkd
20. tahun_pengambilan_data
21. id (optional, for manual outlier removal)

### Coordinate Systems

**Input CRS**: EPSG:4326 (WGS84)
- All input data should be in geographic coordinates

**Processing CRS**: EPSG:32748 (UTM Zone 48S)
- All spatial data converted to projected CRS for distance calculations
- Ensures accurate metric distances

---

[← Back to Overview](index.md) | [← Features](features.md) | [Quick Reference →](quick-reference.md)
