# Bayit Dashboard — Complete Project Specification

## Purpose

Find the best property in Richmond, BC to build a synagogue. The City of Richmond's Zoning Bylaw 8500 permits Religious Assembly use on specific parcels. This project identifies those parcels, enriches them with public data, and provides interactive tools to filter and rank candidates.

---

## Data Sources

### 1. ParcelMap BC (Primary — Land Parcels)

- **What:** Every land parcel in Richmond, BC — boundaries, ownership, lot size
- **Source:** BC Data Catalogue WFS (Web Feature Service)
- **URL:** `https://openmaps.gov.bc.ca/geo/pub/ows`
- **Layer:** `pub:WHSE_CADASTRE.PMBC_PARCEL_FABRIC_POLY_SVW`
- **License:** Open Government Licence – British Columbia
- **Volume:** ~128,000 parcels in Richmond
- **Key fields:**
  - `PID` — Parcel Identifier (9-digit, unique per lot)
  - `OWNER_TYPE` — Private, Local Government, Crown Agency, Federal, Crown Provincial
  - `FEATURE_AREA_SQM` — Lot area in square meters
  - `MUNICIPALITY` — Used to filter to "Richmond, City of"
  - `WHEN_UPDATED` — For incremental sync
  - Geometry — MultiPolygon boundaries in EPSG:4326
- **Fetch strategy:** Richmond bbox divided into 10x5 = 50 tiles (WFS caps at 10k features per request). CQL filter: `MUNICIPALITY='Richmond, City of' AND BBOX(SHAPE,xmin,ymin,xmax,ymax,'EPSG:4326')`
- **Update frequency:** ParcelMap BC updates roughly monthly. Incremental sync possible via `WHEN_UPDATED >= <date>`.

### 2. Assembly Zoning CSV (Primary — Which Parcels Allow Synagogues)

- **What:** 574 rows listing every Richmond address where Religious Assembly is a permitted use under Bylaw 8500
- **Source:** City of Richmond planning department analysis
- **File:** `Properties zoned to permit Religious Assembly.csv`
- **Columns:** `Address`, `Zoning` (comma-separated zone codes)
- **Unique addresses:** 561 (some addresses have multiple zone codes)
- **Zone code distribution:**
  - **CDT1** — City Centre District: 215 mentions (commercial towers, mixed-use)
  - **CA** — Commercial Activity: 214 mentions (strip malls, office parks)
  - **ASY** — Assembly: 85 mentions (purpose-built for gathering/worship)
  - **CC** — Community Commercial: 43 mentions (neighborhood commercial)
  - **CEA** — Community Entertainment Area: 17 mentions (casino/entertainment district)
  - Plus combo zones: AG1, CG1, RS1/E, RS1/F, ZHR10, RAM1, SI, IB1, IL, 062
- **Processing:** Each address is geocoded via Nominatim, then spatially matched to ParcelMap BC parcels (point-in-polygon). Of 561 addresses, 168 match to parcel polygons; 393 are shown as point markers.
- **Update frequency:** Only changes when City of Richmond amends Bylaw 8500. Check annually.

### 3. BC Agricultural Land Reserve (ALR)

- **What:** Provincial boundaries of protected farmland where non-agricultural development is effectively prohibited
- **Source:** BC Data Catalogue WFS
- **Layer:** `pub:WHSE_LEGAL_ADMIN_BOUNDARIES.OATS_ALR_POLYS`
- **Volume:** 16 polygons covering ~60% of Richmond
- **Purpose:** Any parcel intersecting ALR cannot realistically host a synagogue, regardless of zoning
- **Update frequency:** Rarely changes. Check annually.

### 4. OpenStreetMap Excluded Areas

- **What:** Polygons and points of parks, schools, hospitals, fire stations, police stations, kindergartens, childcare centres, cemeteries, places of worship, community centres
- **Source:** OSM Overpass API
- **Volume:** ~12,000 features (596 polygons + ~5,600 buffered point nodes)
- **Purpose:** Identify parcels that are already occupied by public infrastructure and cannot be purchased/redeveloped
- **Categories:**
  - Parks: 253
  - Schools: 115
  - Places of worship: 106 (existing churches, temples, mosques)
  - Sports facilities: 55
  - Cemeteries: 48
  - Community centres: 12
  - Hospitals: 7
  - Fire/police stations, kindergartens, childcare: various
- **Update frequency:** OSM data is community-maintained. Refresh quarterly.

### 5. TransLink GTFS (Transit Proximity)

- **What:** Bus and SkyTrain stop locations in Richmond
- **Source:** TransLink static GTFS feed
- **URL:** `https://gtfs-static.translink.ca/gtfs/google_transit.zip`
- **Volume:** ~1,000 stops in Richmond area
- **Purpose:** Transit proximity is valuable for a synagogue (congregants who don't drive on Shabbat)
- **Key data:** Stop name, coordinates, route type (1=SkyTrain, 3=Bus)
- **Update frequency:** TransLink updates GTFS roughly quarterly.

### 6. Future Data Sources (Not Yet Integrated)

| Source | What | Why | Status |
|--------|------|-----|--------|
| **BC Assessment** | Assessed land/improvement values, property class codes | Know the price — vacant land vs improved | Requires BC OnLine API access |
| **MLS/Repliers** | Active real estate listings | Know what's for sale and at what price | Requires Repliers API key or broker sponsorship |
| **Richmond Zoning Polygons** | Spatial zoning district boundaries | Spatial join instead of address lookup | Not publicly published; requires FOIPPA request or RIM network |
| **Building Footprints** | Existing building outlines | Distinguish vacant lots from built parcels | Not publicly available for Richmond |
| **Google Places API** | Current business/POI at each address | Better "what's there now" data than OSM | Requires API key, has cost |
| **BC Land Title Survey Authority** | Legal ownership details, title history | Know the actual owner name, encumbrances | Requires LTSA account, per-search fees |

---

## Filter Strategy

### Tier 1: Hard Exclusions (Apply First)

These eliminate parcels that are legally or practically impossible:

| Filter | Default | Rationale |
|--------|---------|-----------|
| **Exclude ALR** | ON | Provincial law prohibits non-agricultural development. No exceptions for worship. |
| **Exclude unusable land** | ON | Parks, schools, hospitals, fire halls, police stations, cemeteries, existing worship sites — these are not available for purchase. |
| **Exclude road parcels** | ON (auto) | Thin elongated parcels that are road rights-of-way, detected by compactness ratio > 12. |
| **Exclude Crown Agency** | ON (auto) | BC Housing, health authorities, transit authority land — not available for private development. |

### Tier 2: Practical Filters (Narrow the Search)

These focus on what's realistic:

| Filter | Default | Rationale |
|--------|---------|-----------|
| **Minimum lot area** | 1,000 m² | A modest synagogue (100-seat sanctuary, social hall, offices, 30-space parking) needs at minimum ~1,500 m². Below 1,000 m² is too small. |
| **Maximum lot area** | 50,000 m² | Parcels over 50,000 m² are typically infrastructure, parks, or industrial land — not realistic acquisition targets. |
| **Hide private land** | ON | Private parcels not listed for sale require unsolicited offers to unknown owners. Focus on government-owned or for-sale land first. |

### Tier 3: Assembly-Specific Filters (Fine-tune)

These let you explore specific subsets of the 561 assembly-zoned properties:

| Filter | Default | Rationale |
|--------|---------|-----------|
| **Zoning code** | All | ASY zones are purpose-built for assembly (easiest permitting). CA and CDT1 allow it but are primarily commercial (may need rezoning effort). CC is neighborhood commercial. CEA is the casino district. |
| **Owner type** | All | "Local Government" (City of Richmond) land may be available for community lease or below-market sale. "Private" requires market-rate purchase. |
| **Exclude occupied** | OFF | When ON, hides parcels with existing buildings (apartments, retail, restaurants, hotels, schools). Shows only vacant or undeveloped land. |
| **Only confirmed parcels** | OFF | When ON, hides geocoded point markers and shows only the 168 parcels with confirmed polygon boundaries from ParcelMap BC. |

### Filters We Did NOT Build But Should Consider

| Filter | Value | Data Source | Implementation |
|--------|-------|-------------|----------------|
| **Transit proximity** | Within 400m of a bus stop, within 800m of SkyTrain | TransLink GTFS (already loaded) | Spatial buffer query: `ST_DWithin(parcel.geom, transit.geom, 400)`. Critical for Shabbat observance. |
| **Distance to existing synagogue** | > 2km from Beth Tikvah (9711 Geal Rd) | Hardcoded point | Avoid cannibalization. `ST_Distance` calculation. |
| **Flood hazard zone** | Exclude high-risk | Richmond flood maps (not yet loaded) | Richmond is a floodplain; some areas have higher risk. Data available from City of Richmond. |
| **Assessed land value** | Under $X million | BC Assessment (not yet integrated) | Know the price before investigating. Requires BC OnLine access. |
| **Currently for sale** | MLS listings only | Repliers API (not yet integrated) | Show only parcels with active listings — immediate opportunities. |
| **Walkability score** | Minimum score | Walk Score API | Congregants walking on Shabbat need sidewalks, low traffic. |
| **Eruv boundary** | Inside/near existing eruv | Community data (manual) | An eruv allows carrying on Shabbat. Proximity matters. |
| **Vacant vs improved** | Vacant only | BC Assessment property class code | Class 1 = Residential, Class 6 = Business. Vacant land has different class codes. |
| **Neighborhood character** | Residential adjacency | Zoning of neighboring parcels | A synagogue in a residential neighborhood is different from one in a commercial strip. |
| **Lot shape/frontage** | Minimum frontage, reasonable depth-to-width ratio | Geometry analysis | Narrow lots or irregular shapes are harder to build on. |

---

## Ranking Strategy: Finding the Best Property

After filtering, rank remaining candidates by a weighted score:

### Proposed Scoring Model

| Criterion | Weight | Scoring |
|-----------|--------|---------|
| **Zoning** | 25% | ASY = 100, CA = 70, CDT1 = 60, CC = 50, CEA = 30 |
| **Ownership** | 20% | Local Government = 100, Federal = 80, Private (for sale) = 60, Private (not listed) = 20 |
| **Lot size fit** | 15% | 1,500–3,000 m² = 100, 1,000–1,500 = 70, 3,000–5,000 = 60, 5,000+ = 40, <1,000 = 0 |
| **Transit access** | 15% | <200m bus = 100, <400m bus = 80, <800m SkyTrain = 90, >800m = 30 |
| **Vacancy** | 10% | Vacant lot = 100, Parking lot = 80, Low-value improvement = 50, Built = 20 |
| **Neighborhood** | 10% | Residential adjacency = 100, Mixed = 70, Industrial = 30 |
| **Price** | 5% | Below median $/m² = 100, At median = 50, Above = 20 |

### Ideal Property Profile

The perfect synagogue site would be:
- **Zoned ASY** (no rezoning needed)
- **Owned by City of Richmond** (community lease possible)
- **1,500–3,000 m²** (right size for building + parking + garden)
- **Within 400m of a bus stop** (Shabbat access)
- **Vacant or low-value improvements** (no demolition needed)
- **Not in ALR** (legally buildable)
- **In a residential neighborhood** (fits community character)
- **Affordable** (below $5M assessed value)

---

## Data Pipeline Architecture

```
┌─────────────────────────────────────────────┐
│  Original Data Sources                       │
│  ┌──────────────┐  ┌──────────────────────┐ │
│  │ ParcelMap BC  │  │ Assembly CSV         │ │
│  │ (WFS, 128k)  │  │ (Bylaw 8500, 561)   │ │
│  └──────┬───────┘  └──────────┬───────────┘ │
│  ┌──────┴───────┐  ┌──────────┴───────────┐ │
│  │ ALR Boundary │  │ OSM Excluded Areas   │ │
│  │ (WFS, 16)    │  │ (Overpass, 12k)      │ │
│  └──────┬───────┘  └──────────┬───────────┘ │
│  ┌──────┴───────┐                           │
│  │ TransLink    │                           │
│  │ (GTFS, 1k)  │                           │
│  └──────┬───────┘                           │
└─────────┼───────────────────────────────────┘
          │
          ▼  (Refresh Cells in Notebook)
┌─────────────────────────────────────────────┐
│  GeoParquet Files (on GitHub)                │
│  parcels.parquet .............. 6.6 MB      │
│  assembly_candidates.parquet .. 0.04 MB     │
│  alr_boundary.parquet ......... 0.4 MB      │
│  excluded_areas.parquet ....... 4.4 MB      │
│  transit_stops.parquet ........ 0.1 MB      │
│  TOTAL ........................ 11.5 MB     │
└─────────┬───────────────────────────────────┘
          │
          ▼  (Cell 2: Load Data)
┌─────────────────────────────────────────────┐
│  GeoPandas DataFrames (in memory)            │
│  ┌─────────────────────────────────────────┐│
│  │ assembly_enriched GeoDataFrame          ││
│  │  = assembly_candidates                  ││
│  │  + parcels (owner, area, geometry)      ││
│  │  + ALR intersection status              ││
│  └─────────────────────────────────────────┘│
└─────────┬───────────────────────────────────┘
          │
          ▼  (Cell 3: Dashboard)
┌─────────────────────────────────────────────┐
│  Lonboard Map + ipywidgets Filters           │
│  128k parcel polygons (base layer)           │
│  + 168 assembly polygons (gold overlay)      │
│  + 393 assembly points (gold dots)           │
│  + 7 interactive filter controls             │
└─────────────────────────────────────────────┘
```

---

## Richmond Bounding Box

```
Northwest: 49.23°N, 123.30°W
Southeast: 49.08°N, 123.00°W
Center:    49.166°N, 123.137°W (No. 3 Road / Brighouse area)
```

All data is in EPSG:4326 (WGS84). All area calculations in square meters.

---

## Zone Code Reference (Richmond Bylaw 8500)

| Code | Full Name | Assembly Status | Typical Use |
|------|-----------|-----------------|-------------|
| ASY | Assembly | **Permitted** | Purpose-built for gathering, worship, community halls |
| CA | Commercial Activity | **Permitted** | Offices, retail, restaurants along arterials |
| CDT1 | City Centre District | **Permitted** | High-density mixed-use towers (City Centre / No. 3 Road) |
| CC | Community Commercial | **Permitted** | Neighborhood shops, services |
| CEA | Community Entertainment Area | **Permitted** | Casino/entertainment district (south Richmond) |
| AG1/AG2 | Agricultural | Prohibited | Farmland (mostly in ALR) |
| RS1/RS2 | Single Family Residential | Prohibited | Houses |
| RC1/RC2 | Compact Lot Residential | Prohibited | Townhouses |
| RM1-RM4 | Multi-Family Residential | Conditional | Apartments (may allow assembly with special permit) |
| C1-C6 | Commercial | Permitted | Various commercial types |
| I1/I2 | Industrial | Prohibited | Warehouses, manufacturing |
| PA1/PA2 | Public/Institutional | Permitted | Government, schools, hospitals |

---

## How to Refresh Data

### Quick Refresh (Quarterly)
1. Open the notebook in Colab
2. Run **Refresh Cell A** (parcels — 5-10 min)
3. Run **Refresh Cell C** (ALR — 10 sec)
4. Run **Refresh Cell D** (excluded areas — 30 sec)
5. Run **Refresh Cell E** (transit — 10 sec)
6. Re-run Cell 2 and Cell 3

### Full Refresh (When CSV Changes)
1. Upload new CSV to `data/` directory
2. Run **Refresh Cell B** (geocoding — 10 min)
3. Then run all other refresh cells
4. Commit updated parquet files to GitHub repo

### To Persist Refreshed Data
```bash
cd bayit-notebook
git add data/*.parquet
git commit -m "chore: refresh data from sources"
git push
```
