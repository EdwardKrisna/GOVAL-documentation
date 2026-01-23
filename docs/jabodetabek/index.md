# Jabodetabek Property Valuation Model v4

## Overview

This repository contains the feature engineering pipeline and documentation for the Jabodetabek Property Valuation Model v4. The system implements a comprehensive geospatial and rule-based feature engineering workflow to predict property values (price per square meter) across the Greater Jakarta metropolitan area.

## Key Capabilities

- **Geospatial Feature Engineering**: Distance calculations to key landmarks, roads, and activity centers
- **Zone-based Classification**: Premium area marking, rural area detection, accessibility scoring
- **Multi-source Data Integration**: Combines property data, road networks, administrative boundaries, and economic indices
- **Smart Land-use Reclassification**: Rule-based reconciliation of multiple classification sources
- **Parcel Geometry Analysis**: Shape and position scoring based on land-use context
- **Economic Indexing**: PDRB-based macro-economic adjustment features

## Documentation Structure

This documentation is organized into the following files:

### 📘 [Workflow Documentation](workflow.md)
Complete step-by-step feature engineering pipeline covering:
- Data loading and spatial setup
- Preprocessing and data cleaning
- CRS reprojection
- Distance feature engineering
- Zone marking and accessibility features
- Land-use reclassification
- Scoring features (position, shape, ownership)
- Outlier removal strategies

### 📊 [Feature Dependencies](features.md)
Comprehensive mapping of all final features including:
- Feature dependency chains
- Source column requirements
- Computation paths
- Complete dependency graph

### 📦 [Data Requirements](data-requirements.md)
Detailed specification of all required data including:
- Source data schema (20 required columns)
- External geospatial data files
- Macro-economic data requirements
- Data quality expectations

### ⚡ [Quick Reference](quick-reference.md)
At-a-glance reference for:
- Final feature list (42 features)
- Feature categories and types
- Common use cases
- Model input summary

## Quick Start

### Input Data Requirements

**Primary Dataset**: `jbdtb_100_1.xlsx`
- Property observations with coordinates, pricing, and characteristics

**External Spatial Data**:
- Road network GeoDataFrame
- Premium zone polygons (`mark_4.geojson`)
- Rural area polygons (`rural_2.geojson`)
- City center POI datasets

**Macro Data**:
- PDRB economic index table (`pdrb_adhk (1).xlsx`)

### Feature Engineering Pipeline

The pipeline consists of 15 major steps:

1. **Data Loading & Spatial Setup** - Convert tabular data to spatial points
2. **Data Preprocessing** - Compute target variable (HPM) and filter outliers
3. **CRS Reprojection** - Transform to metric coordinate system (EPSG:32748)
4. **Distance Features** - Calculate proximity to key locations
5. **Zone Marking** - Identify premium, rural, and special areas
6. **Data Cleaning** - Remove duplicates and invalid coordinates
7. **Accessibility Engineering** - Compute parcel accessibility from roads
8. **Land-use Reclassification** - Harmonize classification labels
9. **Position Scoring** - Score parcel position by land-use context
10. **Ownership Adjustment** - Adjust pricing for certificate types
11. **Shape Scoring** - Score parcel geometry efficiency
12. **Economic Indexing** - Attach macro-economic growth proxies
13. **Log Transformations** - Stabilize variance for modeling
14. **Outlier Removal** - District-class quantile filtering
15. **Final Feature Generation** - Create model-ready predictors

## Final Model Features

The pipeline generates **42 model-ready features** across 8 categories:

- **Distance Features** (4): Log-transformed distances to key locations
- **Core Continuous** (2): Log-transformed price and land area
- **Accessibility & Roads** (2): Accessibility flags and adjusted road width
- **Zone/Location Flags** (4): Premium, PIK, rural area indicators
- **City Dummies** (6): Jakarta administrative areas + Bekasi
- **Land-use Flags** (8): One-hot encoded land-use categories
- **Scoring Features** (2): Position and shape scores
- **Economic Index** (1): PDRB-based macro indicator

## Target Variable

**`ln_hpm_sertifikat`**: Log-transformed, ownership-adjusted price per square meter

This represents the discounted property price per square meter of land area, adjusted for:
- Certificate type (ownership rights)
- Building value (for land+building properties)
- Ownership premium/discount by land-use type

## Key Technical Highlights

### Spatial Processing
- Utilizes EPSG:32748 (UTM Zone 48S) for accurate metric distance calculations
- Spatial indexing (`sindex.nearest`) for efficient nearest-neighbor queries
- Buffer-based coordinate error detection and removal

### Rule-based Intelligence
- Multi-source land-use reconciliation (context + AGGA + ML predictions)
- Commercial influence zones based on road width
- PIK-specific accessibility override rules
- Shape factor adjustments by parcel geometry type

### Data Quality Controls
- Range-based outlier filtering (HPM: 100K - 100M)
- District-class quantile trimming (5th-95th percentile)
- Duplicate removal (keeping most recent observations)
- Invalid coordinate cluster exclusion

## Use Cases

This feature engineering pipeline supports:

- **Property Valuation Modeling**: Regression models for price prediction
- **Market Segmentation**: Land-use and zone-based analysis
- **Spatial Analysis**: Distance-decay and accessibility studies
- **Temporal Analysis**: Economic index enables time-series modeling
- **Urban Planning**: Premium zone identification and accessibility mapping

## Technical Stack

- **Spatial Processing**: GeoPandas, Shapely
- **Data Manipulation**: Pandas, NumPy
- **Coordinate Systems**: EPSG:4326 (WGS84) → EPSG:32748 (UTM 48S)
- **Distance Calculations**: Spatial indexing with nearest-neighbor queries

## Version

**Current Version**: v4

## License

-

## Contact

edwardokrisnasembiring@mail.ugm.ac.id

---

**Navigation**:
[Workflow](workflow.md) | [Features](features.md) | [Data Requirements](data-requirements.md) | [Quick Reference](quick-reference.md)
