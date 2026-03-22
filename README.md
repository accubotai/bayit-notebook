# Bayit Dashboard — Google Colab Edition

Interactive Jupyter notebook for finding synagogue sites in Richmond, BC.

## Quick Start

1. **Upload data files** to Google Drive at `My Drive/bayit-dashboard/data/`
2. **Open in Colab**: [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/accubotai/bayit-notebook/blob/main/bayit_dashboard.ipynb)
3. Run the first three cells

## Data Files

The `data/` directory contains GeoParquet files:
- `parcels.parquet` — 128k Richmond parcels from ParcelMap BC
- `assembly_candidates.parquet` — 561 properties zoned for Religious Assembly
- `alr_boundary.parquet` — Agricultural Land Reserve boundaries
- `excluded_areas.parquet` — Parks, schools, hospitals from OSM
- `transit_stops.parquet` — TransLink bus stops

## Data Refresh

The notebook includes cells to refresh data from original sources:
- ParcelMap BC WFS (parcels)
- Nominatim geocoding (assembly addresses)
- BC Data Catalogue WFS (ALR)
- OSM Overpass API (excluded areas)
- TransLink GTFS (transit)
