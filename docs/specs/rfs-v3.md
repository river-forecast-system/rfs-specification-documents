# RFS V3

This is a draft of the RFS v3 specifications. It is not complete and is subject to change.

Documentation for the River Forecast System version 3, RFS v3.

- Model Version: 3.0
- Launch Date: <mark>1 March 2027</mark>
- End Date: 

| Property                       | Value                                    |
|--------------------------------|------------------------------------------|
| Reference IFS Version          | 50r1                                     |
| IFS Resolution                 | O1280 grid, 0.1 degree, 9km equator      |
| Forecast Length (Lead Time)    | 15 days                                  |
| Forecast Time Step             | 3 hours                                  |
| Forecast Compute Start Time    | Typically before 0600 UTC                |
| Forecast Compute End Time      | Typically before 1200 UTC                |                         
| Retrospective Start Time       | 1940-01-01                               |
| Retrospective Update           | Daily, 00:00 UTC, available by 01:00 UTC |
| Retrospective Native Time Step | 1 hourly average                         |

## Journal Papers

- <mark>TBD</mark>

## Conceptual explanation of the model

RFS produces 4 main datasets:

1. Hydrography and Model Configs (static)
2. Retrospective simulation and derivatives
3. 15 Day Forecasts and derivatives
4. Flood extent and depth maps

RFS data is available in the following ways:

1. AWS S3 buckets
2. Code packages
    1. Python: `pyrfs`
    2. JavaScript: `jsrfs`
3. Web apps: https://hydroviewer.geoglows.org
4. Lambda functions deployable to user's cloud account (AWS, GCP, Netlify, Vercel, etc)
5. ArcGIS Living Atlas web map layer

## Migrating from v2 to v3

This is the migration guide for consumers moving from v2 to v3, and it is the only place in this document where v3 is described in terms of v2. Every other section describes v3 on its own terms, so a
reader with no v2 history can skip ahead from here.

### Renames

These account for most of the mechanical work in porting v2 code.

| v2                                                         | v3                    | Notes                                                                     |
|------------------------------------------------------------|-----------------------|---------------------------------------------------------------------------|
| `rivid` (forecasts from RAPID), `river_id` (retrospective) | `riverId`             | One name in every product, where v2 used a different spelling per product |
| `Qout`                                                     | `Q`                   | One name in every product, including the ensemble arrays                  |
| `return_period`                                            | `recurrence_interval` | Now `float32`, not int, because the axis carries a 1.5 year value         |

### Reading discharge values

- **Discharge is bitrounded `float32`. Precision is now relative rather than absolute, see [Encoding and compression](#encoding-and-compression)
- **`riverId` order is now a formal promise.** The axis is topologically sorted and identical across every product. It is not ascending id order, so an id cannot be located by binary search on the
  published array. The new `riverIndex` attribute on the streams gives a river's position directly, which avoids a lookup when downloading
- **Flow duration curves are `float32` and bitrounded** like every other discharge array. v2 stored them as `float64`
- Streams carry additional attributes, and some renamed ones, which make the data more self documenting and convenient to use

### Where the data live

- All v3 data are under a single bucket, `s3://river-forecast-system/v3/`, alongside migrated v1 and v2 backups.
- Within each product, subdivisions use hive partition style names
  - Hydrography is organized in 3 digit computational group numbers `group=XXX`
  - Forecasts are organized by a sequence of year, month, day dividers: `year=YYYY/month=MM/day=DD`
  - Flood maps are organized by 1x1 degree tiles labeled by latitude and longitude: `lat=YYY/lon=XXX`
- Monthly and yearly averages are available in timeseries and timestep chunked forms in a single zarr
- The retrospective simulation is updated daily and is typically available by 01:00 UTC
- Forecast warnings are published as `alerts.csv`, formatted for CAP alerts, instead of a warnings parquet file
- Forecast flood extents are published as a single vector `fim.geo.parquet` rather than a raster set
- Reference flood maps are stored by tile as rasters
- Global mapping hydrography is published as PMTiles for web clients and a file geodatabase for Esri clients

### New in v3

- Confluences and lakes, published per computational unit as new hydrography products
- The 1.5 year recurrence interval and a derived `annual_exceedance_probability` array on the return periods

### Discontinued in v3

- The forecast records product
- The "52nd ensemble member" IFS HRES, so the `member` axis runs 1..51 with nothing off axis. HRES was discontinued after becoming identical to the control forecast
    - [https://www.ecmwf.int/en/about/media-centre/focus/2024/plans-high-resolution-forecast-hres-and-ensemble-forecast-ens](https://www.ecmwf.int/en/about/media-centre/focus/2024/plans-high-resolution-forecast-hres-and-ensemble-forecast-ens)
- The https://geoglows.ecmwf.int web portal is retired

### Model changes behind the numbers

These change the values themselves rather than how they are read, so a difference between v2 and v3 at a given river is expected.

- Forecasts and retrospective simulations are tightly coupled using shared initializations
- Reduction in total stream count from about 6.8 million to about 5.5 million, for computational efficiency and accuracy, better treatment of lakes and reservoirs, and reduction in total storage
  burden. Streams in the middle of deserts, within lakes, in oceans, and in other inappropriate areas were removed from the v2 hydrography
- Upgrading to use the newest IFS version, 50r1
- Runoff volumes are aggregated to catchment scale on the native octahedral, or reduced gaussian, mesh native to the IFS version instead of being resampled to a uniform lat/lon grid
- Upgrade computations to use river-route (Python) instead of RAPID (user compiled Fortran) for matrix muskingum routing, with synthetic reach subdivision for greater computational stability
- Application of bitrounding to 15 bits, which compresses much better, but data are modified after computation

## Possible Changes

These are under consideration for v3 but are not committed:

- <mark>Implement non-linear routing???</mark>
- <mark>Extended range 45 day forecasts and the 3-6 month statistical outlook derived from them???</mark>
- <mark>Hydrology Early Warnings for All</mark>

## Available Products

### Organization on S3

```text
s3://river-forecast-system/
├── v1/
│   └── backups....
├── v2/
│   └── backups....
└── v3/
    ├── hydrography/
    │   ├── group=0/                                # the 0 group is the global catch-all so we don't make near-duplicate names
    │   │   ├── streams.pmtiles                     # for mapbox-gl-js, leaflet, etc
    │   │   └── streams_map_optimized.gdb.zip       # for esri map layers
    │   └── group=XXX/
    │       ├── catchments_XXX.geo.parquet          # TDX-Hydro derived
    │       ├── confluences_XXX.geo.parquet         # TDX-Hydro derived
    │       ├── lakes_XXX.geo.parquet               # TDX-Hydro derived
    │       ├── streams_XXX.geo.parquet             # TDX-Hydro derived - epsg4326, modified copy of tdx-hydro region
    │       ├── streams_simplified_XXX.geo.parquet  # TDX-Hydro derived - epsg4326, extreme lat/lon coordinate simplification
    │       ├── streams_mapping_XXX.geo.parquet     # TDX-Hydro derived - epsg3857, coordinates rounded to 0 decimal places
    │       ├── params_XXX.parquet                  # for river-route
    │       └── synthetic_rating_curve.parquet      # from ARC, for routing, FIM? Q_baseflow depends on a modeled value!
    ├── retrospective/
    │   ├── hourly.zarr/
    │   ├── daily.zarr/
    │   ├── monthly.zarr/                           # holds both Q and Q_timesteps
    │   ├── yearly.zarr/                            # holds both Q and Q_timesteps
    │   ├── return-periods.zarr/
    │   ├── maximums.zarr/
    │   └── fdc.zarr/
    ├── flood-maps/
    │   └── lon=XXX/
    │       └── lat=YYY/
    │           ├── arc/
    │           │   ├── fim.tiff
    │           │   ├── velocity.tiff
    │           │   ├── depth.tiff
    │           │   ├── c2f_config.yaml             # arc/c2f config used to make this
    │           │   └── impact? DEM? burned DEM? TBD
    │           └── fldpln.zarr/
    │               ├── library/
    │               ├── streams/
    │               └── zarr.json
    ├── forecasts15/                                # 15 day @ 3 hour forecasts, lead_time coordinate alongside absolute time
    │   └── year=YYYY/
    │       └── month=MM/
    │           └── day=DD/
    │               ├── alerts.csv                  # for CAP alerts
    │               ├── maps/                       # summary files
    │               │   ├── esri_animation_tables/  # for animated forecast layer (arcgis living atlas)
    │               │   │   ├── YYYYMMDDHH.csv
    │               │   │   └── x120...
    │               │   ├── timeseries/
    │               │   │   ├── styles.bin
    │               │   │   └── styles.json
    │               │   ├── max-flow/
    │               │   │   ├── styles.bin
    │               │   │   └── styles.json
    │               │   ├── below-q95/
    │               │   │   ├── styles.bin
    │               │   │   └── styles.json
    │               │   └── time-to-peak/
    │               │       ├── styles.bin
    │               │       └── styles.json
    │               ├── next-init-files/
    │               │   └── group=XXX/
    │               │       └── warmstate_YYYYMMDDHHMM_groupXXX.parquet
    │               ├── discharge.zarr/
    │               └── fim.geo.parquet             # vector extent polygons, faster/smaller, single file
    └── forecasts45/                                # 1-3 month outlooks, assuming this works out
        └── year=YYYY/
            └── month=MM/
                └── day=DD/
                    ├── summary.csv
                    └── discharge.zarr/
```

### Hydrography

| Property            | Value       |
|---------------------|-------------|
| Approximate Streams | 5.5 Million |
| Reference Product   | TDX-Hydro   |

Hydrography and the routing configs derived from it are divided by computational unit. A computational unit is called a **group** and is written into the bucket as a `group=XXX` hive partition. The
`group=0` partition is a global catch-all, used so that global products do not need near duplicate names of their own.

| File                              | Partition   | Format     | Description                                                                     |
|-----------------------------------|-------------|------------|---------------------------------------------------------------------------------|
| `streams.pmtiles`                 | `group=0`   | PMTiles    | Global stream tiles for mapbox-gl-js, leaflet, and similar clients              |
| `streams_map_optimized.gdb.zip`   | `group=0`   | File GDB   | Global streams for Esri map layers                                              |
| `streams_XXX.geo.parquet`         | `group=XXX` | GeoParquet | Stream center lines, epsg4326, a modified copy of the TDX-Hydro region          |
| `streams_mapping_XXX.geo.parquet` | `group=XXX` | GeoParquet | Streams in epsg3857 with coordinates rounded to 0 decimal places                |
| `catchments_XXX.geo.parquet`      | `group=XXX` | GeoParquet | Drainage area of each stream, TDX-Hydro derived                                 |
| `confluences_XXX.geo.parquet`     | `group=XXX` | GeoParquet | Junctions of the stream network, TDX-Hydro derived                              |
| `lakes_XXX.geo.parquet`           | `group=XXX` | GeoParquet | Lakes and reservoirs, TDX-Hydro derived                                         |
| `params_XXX.parquet`              | `group=XXX` | Parquet    | Routing parameters for river-route                                              |
| `synthetic_rating_curve.parquet`  | `group=XXX` | Parquet    | Synthetic rating curves from ARC, used for routing and flood inundation mapping |

<mark>`Q_baseflow` in the synthetic rating curves depends on a modeled value.</mark>

<mark>Connectivity is not a separate file in this layout. Confirm whether it is carried inside `params_XXX.parquet` or still needs its own file.</mark>

### Flood Forecast Products

The daily 15 day forecast is published under `forecasts15/`, partitioned `year=YYYY/month=MM/day=DD/`. Each day's partition holds

| File                                                                | Format     | Description                                                       |
|---------------------------------------------------------------------|------------|-------------------------------------------------------------------|
| `discharge.zarr/`                                                   | Zarr v3    | Forecasted discharge for 15 days, 50+1 ensemble                   |
| `alerts.csv`                                                        | CSV        | Alert records, formatted for CAP alerts                           |
| `fim.geo.parquet`                                                   | GeoParquet | Vector flood extent polygons for all groups                       |
| `maps/esri-animation-tables/YYYYMMDDHH.csv`                         | CSV        | Summary table per forecast timestep (120) for ArcGIS Living Atlas |
| `maps/timeseries/styles.{bin,json}`                                 | bin + JSON | Map styleset for the forecast timeseries                          |
| `maps/max-flow/styles.{bin,json}`                                   | bin + JSON | Map styleset for maximum forecasted flow                          |
| `maps/below-q95/styles.{bin,json}`                                  | bin + JSON | Map styleset for flows below the Q95 low flow threshold           |
| `maps/time-to-peak/styles.{bin,json}`                               | bin + JSON | Map styleset for time to peak                                     |
| `next-init-files/group=XXX/warmstate_YYYYMMDDHHMM_groupXXX.parquet` | Parquet    | Routing warm state per group, the initialization for the next run |

The forecast discharge store contains

1. `Q` (member, time, riverId): 3 hourly discharge for each of the 50+1 IFS forecast members
2. `Qpercentiles` (percentiles, time, riverId): 3 hourly discharge ensemble at deciles 0 (min), 10, 20 ... 50 (median) ... 90, 100 (max)
3. `Qmean` (time, riverId): 3 hourly discharge ensemble mean

Request parameters for ECMWF IFS data:

- Stream: enfo
- Types:
    - pf, perturbed forecast, 50 members
    - cf, control forecast, 1 member
- Variables:
    - https://codes.ecmwf.int/grib/param-db/
    - Runoff: 205.128
- Grid:
    - Grid should be the native resolution of reduced gaussian grid/mesh. It should not be regridded or resampled.

Example MARS request

```text
retrieve,
class=od,
date=2025-07-15,
expver=1,
levtype=sfc,
number=1/2/3/4/5/6/7/8/9/10/11/12/13/14/15/16/17/18/19/20/21/22/23/24/25/26/27/28/29/30/31/32/33/34/35/36/37/38/39/40/41/42/43/44/45/46/47/48/49/50,
param=205.128,
step=0/1/2/3/4/5/6/7/8/9/10/11/12/13/14/15/16/17/18/19/20/21/22/23/24/25/26/27/28/29/30/31/32/33/34/35/36/37/38/39/40/41/42/43/44/45/46/47/48/49/50/51/52/53/54/55/56/57/58/59/60/61/62/63/64/65/66/67/68/69/70/71/72/73/74/75/76/77/78/79/80/81/82/83/84/85/86/87/88/89/90/93/96/99/102/105/108/111/114/117/120/123/126/129/132/135/138/141/144/150/156/162/168/174/180/186/192/198/204/210/216/222/228/234/240/246/252/258/264/270/276/282/288/294/300/306/312/318/324/330/336/342/348/354/360,
stream=enfo,
time=00:00:00,
type=pf,
target="output"
```

### Extended Range Forecast Products

Produced monthly on the 1st and the 15th. These may be generated a few days after the nominal date so that the retrospective initialization is available for the dates being forecasted for.

Extended range forecasts are published under `forecasts45/`, partitioned `year=YYYY/month=MM/day=DD/` on the same convention as the daily forecasts.

| File              | Format  | Description                                                         |
|-------------------|---------|---------------------------------------------------------------------|
| `discharge.zarr/` | Zarr v3 | Forecasted discharge for 45 days, 100 perturbed + 1 control members |
| `summary.csv`     | CSV     | Summary of the extended range forecast                              |

The extended range store contains

1. `Q` (member, time, riverId): daily average discharge for each of the 100+1 IFS forecast members
2. `Qpercentiles` (percentiles, time, riverId): daily discharge ensemble at deciles 0 (min), 10, 20 ... 50 (median) ... 90, 100 (max)
3. `Qmean` (time, riverId): daily discharge ensemble mean

<mark>What other derivatives or summaries should be made?</mark>

1. Expected volumes? Or could be computed on the fly, allowing for easy use for reservoir management
2. Comparisons to normal, or some sort of classification system

A statistically generated 3-6 month outlook uses the 45 day simulation together with groundwater baseflow. This is a method to calculate, not necessarily a dataset that is cached.

1. Based on historical daily discharge data from ERA5
2. So far: plots observed past 9 months, then forecast of next 3 months
3. Forecast includes median, 25-75 percentile, and wettest/driest 3-month period
4. Trying methods of scaling/stretching the historical median to see how well they forecast, in the process of deciding which metrics are most telling
5. Plans to incorporate other variables such as the 45 day forecast and groundwater

Request parameters for the 45 day forecasts:

- Stream: eefo
- Types:
    - pf, perturbed forecast, 100 members
    - cf, control forecast, 1 member
- Variables:
    - https://codes.ecmwf.int/grib/param-db/
    - Runoff: 205.128
- Grid:
    - Grid should be the native resolution of reduced gaussian grid/mesh. It should not be regridded or resampled.

Example MARS request

```text
retrieve,
class=od,
date=2025-07-15,
expver=1,
levtype=sfc,
number=1/to/100,
param=205.128,
step=0/to/1104/by/24,
stream=eefo,
time=00:00:00,
type=pf,
target="output"
```

### Retrospective Simulation Products

| Product Type         | Time Step       | Format  | Frequency | Description                                                    |
|----------------------|-----------------|---------|-----------|----------------------------------------------------------------|
| Hourly Discharge     | hourly average  | Zarr v3 | Daily     | Hourly average simulation, native resolution                   |
| Daily Discharge      | daily average   | Zarr v3 | Daily     | Daily average simulation                                       |
| Monthly Discharge    | monthly average | Zarr v3 | Monthly   | Monthly average simulation, `Q` and `Q_timesteps` in one store |
| Yearly Discharge     | yearly average  | Zarr v3 | Yearly    | Yearly average simulation, `Q` and `Q_timesteps` in one store  |
| Maximums Discharge   | annual maximum  | Zarr v3 | Yearly    | Annual maximums from hourly and daily averages                 |
| Return Periods       |                 | Zarr v3 | Once      | Return periods from multiple methods                           |
| Flow Duration Curves |                 | Zarr v3 | Once      | Flow duration curves                                           |

Routing warm states are not stored under `retrospective/`. They are written per group with the forecast that consumes them, at
`forecasts15/year=YYYY/month=MM/day=DD/next-init-files/group=XXX/warmstate_YYYYMMDDHHMM_groupXXX.parquet`.

### Flood Maps

Flood maps are tiled, partitioned `flood-maps/lon=XXX/lat=YYY/`. Each tile holds the ARC/Curve2Flood raster products and the FLDPLN library used by the flood worker.

| File                  | Format  | Description                                                 |
|-----------------------|---------|-------------------------------------------------------------|
| `arc/fim.tiff`        | GeoTIFF | Flood inundation extent                                     |
| `arc/depth.tiff`      | GeoTIFF | Inundation depth                                            |
| `arc/velocity.tiff`   | GeoTIFF | Flow velocity                                               |
| `arc/c2f_config.yaml` | YAML    | The ARC/Curve2Flood configuration used to produce this tile |
| `fldpln.zarr/`        | Zarr v3 | Per tile FLDPLN library, read directly by the flood worker  |

<mark>Impact layers, DEM, and burned DEM per tile are still TBD.</mark>

The daily forecast also publishes vector flood extents as a single `fim.geo.parquet` per forecast, which is faster and smaller than a raster set for map clients. See
[Flood Forecast Products](#flood-forecast-products).

The internal layout of `fldpln.zarr` is documented in [Dataset Structure and Schematics](#flood-mapslonxxxlatyyyfldplnzarr).

### Web Maps

1. Daily forecasted flood maps [https://www.arcgis.com/home/item.html?id=8f0573e0c0b9491dbeafde9c72ccf02b](https://www.arcgis.com/home/item.html?id=8f0573e0c0b9491dbeafde9c72ccf02b)
2. Return period flood maps and/or forecasted flood maps (ESRI)
    1. Return period flood maps
    2. Daily forecast flood maps, from the 90th percentile of the ensemble maximum
3. Maps for the extended range forecast
    1. HydroSOS style maps showing what basins fall into different categories of wet, dry, normal, etc. Similar to what is currently produced for the retrospective data, but based on forecasts
    2. Potentially a map highlighting areas forecasted to be significantly lower than before. Similar to the highlighted streams for the daily forecast maps but focused on low flows and larger areas
       instead of individual streams
4. Charts, tables, etc. for the extended range forecast

### Summary Table

Note: All times are given in UTC.

| Product Type                | Category           | Format     | Update Frequency         | Updates Available | Size               |
|:----------------------------|:-------------------|:-----------|:-------------------------|:------------------|:-------------------|
| Hydrography (`group=0`)     | Model Sources      | PMTiles    | None                     | N/A               |                    |
| Hydrography (by group)      | Model Sources      | GeoParquet | None                     | N/A               |                    |
| Routing Configs (by group)  | Model Sources      | Parquet    | None                     | N/A               |                    |
| Forecast 3-hourly Discharge | Forecasts          | Zarr v3    | Daily @ 00:00            | 6am-12pm          | 150 GB             |
| Esri Animation Tables       | Forecasts          | CSV        | Daily @ 00:00            | 6am-12pm          | 120 x 120 MB       |
| Map Stylesets               | Forecasts          | bin + JSON | Daily @ 00:00            | 6am-12pm          |                    |
| Alerts                      | Forecasts          | CSV        | Daily @ 00:00            | 6am-12pm          | 500 MB             |
| Warm States                 | Forecasts          | Parquet    | Daily @ 00:00            | 6am-12pm          |                    |
| Hourly Discharge            | Retrospective      | Zarr v3    | Daily @ 00:00            | by 1am same day   | 10 TB              |
| Daily Discharge             | Retrospective      | Zarr v3    | Daily @ 00:00            | by 1am same day   | 500 GB             |
| Monthly Average Discharge   | Retrospective      | Zarr v3    | Monthly on 5th at 00:00  | by 1am same day   | ~20 GB             |
| Yearly Average Discharge    | Retrospective      | Zarr v3    | Yearly on Jan 5 at 00:00 | by 1am same day   | ~2 GB              |
| Annual Maximums Discharge   | Retrospective      | Zarr v3    | Yearly on Jan 5 at 00:00 | by 1am same day   | ~1 GB              |
| Return Periods              | Retrospective      | Zarr v3    | None                     | N/A               |                    |
| Flow Duration Curves        | Retrospective      | Zarr v3    | None                     | N/A               |                    |
| Forecast Flood Extents      | Flood Maps         | GeoParquet | Daily @ 00:00            | 6am-12pm          | <5 GB              |
| Flood Map Tiles (ARC)       | Flood Maps         | GeoTIFF    | None                     | N/A               | <10 GB             |
| FLDPLN Libraries            | Flood Maps         | Zarr v3    | None                     | N/A               |                    |
| Extended Forecast Discharge | Extended Forecasts | ??         | Monthly 1 and 15         |                   | TBD ballpark 150GB |

## Dataset Structure and Schematics

Every store is **Zarr v3**. The river axis is named `riverId` and discharge is named `Q`. The chunking shown in the schematics below is a placeholder to tune.

### Datatypes

Every array gets an explicit dtype at write time, nothing is left for xarray to infer. The `encodings` dict in `generate_v3_examples_data.py` is the source of truth. `encodings_for_dataset` raises on
any variable missing from it, so a renamed or newly added variable fails the write instead of silently taking a default.

| Array                                                   | Dtype     | Bitrounded | Notes                                    |
|---------------------------------------------------------|-----------|------------|------------------------------------------|
| `riverId`                                               | `int32`   | no         | ids exceed int16, well inside int32      |
| `time`                                                  | `int32`   | no         | integer hours, see below                 |
| `member`                                                | `int32`   | no         | forecasts only, 1..51                    |
| `lead_time`                                             | `int32`   | no         | forecasts only, `units "hours"`          |
| `Q`, `Q_timesteps`                                      | `float32` | **yes**    | m3 s-1                                   |
| `daily`, `hourly`                                       | `float32` | **yes**    | maximums.zarr annual maxima              |
| `{gumbel,logpearson3,lognormal,weibull}_{hourly,daily}` | `float32` | **yes**    | return-periods.zarr, 8 arrays            |
| `max_simulated_hourly`, `max_simulated_daily`           | `float32` | **yes**    | return-periods.zarr                      |
| `Qpercentiles`, `Qmean`                                 | `float32` | **yes**    | forecasts, reduced across `member`       |
| `percentiles`                                           | `int32`   | no         | forecasts only, deciles 0..100 by 10     |
| `recurrence_interval`                                   | `float32` | no         | **float**, not int, the axis carries 1.5 |
| `annual_exceedance_probability`                         | `float32` | no         | derived `1 / recurrence_interval`        |
| `p_exceed`                                              | `int32`   | no         | fdc.zarr, percent 0..100                 |
| `hourly_annual`, `daily_annual`                         | `float32` | **yes**    | fdc.zarr flow-duration curves            |

Only discharge valued arrays are bitrounded. Do not bitround `annual_exceedance_probability`, derived from an exact axis value, or `Q_timesteps`, a rechunked copy of an already rounded `Q`. Every
value on the `recurrence_interval` axis, 1.5 included, is exact in float32, so an equality match against a literal is safe. Do not round-trip the axis through an integer type.

### Coordinate variables

- `riverId`
    - int32
    - The order of the ids is the same in every Zarr. To get this list, read the coordinate array off any store or refer to the hydrography datasets.
    - **The axis is in topological order, not ascending id order.** It follows the same hydrologically meaningful division as the computational units, so a river's *position* on the axis is meaningful
      and an id cannot be located by binary search on the array as published. That position is the `riverIndex`, and it is the join key shared by the vector tiles, the map style tables, and every
      Zarr.
- `time`
    - int32, counted in hours, including on the daily, monthly and yearly axes, which are still expressed in hours rather than in their own step unit.
    - Left aligned time windows. That is, the corresponding value applies from the stated time step, t, until the start of the next time step, t+1.
    - Uses a unit string of style "\<interval\> since \<reference time\>", e.g. `hours since 1940-01-01T00:00:00+00:00`, with `calendar: proleptic_gregorian`.
    - **The reference time is per store, not shared.** Read `units` off the array and parse it, do not assume a common epoch. The string always carries an explicit UTC offset, which a reader must
      honor rather than letting a local time default apply.
    - `time` is written as an integer type, never as `datetime64[ns]`. Writing the latter re-encodes the integers into the datetime container, lands `units` in the attributes, and blocks any later
      rewrite of the store.
- `member`
    - int32. 1..51 for the 15 day forecast (50 perturbed + 1 control) and 1..101 for the 45 day forecast (100 perturbed + 1 control).
- `lead_time`
    - int32, units `hours`. A coordinate **on the time dimension, not a dimension of its own**. The time delta since initialization.
    - Left aligned on the same convention as `time`: **120 values**, running 0, 3, 6, ... 357, where the value 357 covers hour 357 through hour 360. The 15 day horizon is 120 intervals, not 121
      instants.
- `recurrence_interval`
    - Are exactly the float values: [1.5, 2, 5, 10, 25, 50, 100]. Float rather than integer because of 1.5. Every value is exact in float32, so equality matching against a literal is safe.
- `annual_exceedance_probability`
    - float32 on the `recurrence_interval` axis, `1 / recurrence_interval`.
- `p_exceed`
    - int32, exceedance probability in percent, 0..100. 101 values, so Q95 is row 95.
- `percentiles`
    - Are exactly the integer values: [0, 10, 20, 30, 40, 50, 60, 70, 80, 90, 100]. Evenly spaced deciles, so 0 is the ensemble minimum, 50 the median, and 100 the maximum.

### Data variables

- Discharge
    - **Q** in every store. In the forecasts it carries a `member` dimension, there is no separate variable name for ensemble discharge.
    - **Q_timesteps** in the monthly and yearly stores: the same values as `Q`, rechunked for all-rivers-few-timesteps access (map styling) where `Q` is chunked for one-river-whole-series access
      (plots). One store serves both patterns.
    - Q values are always left aligned on their time intervals per the definition of the time coordinate variable. That is, Q is the average over the ***following*** 1 hour, 3 hours, 24 hours, 1
      calendar month, or 1 calendar year.
    - Units: cubic meters per second (m3/s), as `m3 s-1`.
    - Carries `aggregation_method` ("mean", or "max" on the maxima series) and `keepbits`.
- Ensemble summaries, forecasts only, reduced across `member` at each time step
    - **Qpercentiles** (percentiles, time, riverId): the ensemble distribution on the `percentiles` coordinate.
    - **Qmean** (time, riverId): the ensemble mean. Distinct from `Qpercentiles` at 50, which is the median. The two diverge whenever the ensemble is skewed, which is most of the time on a flood peak.
- Annual maximums, `maximums.zarr`
    - **hourly**, **daily**: the annual maximum of the retrospective series at each native step, one value per year. The series the return period fits are derived from, kept so the fits stay
      reproducible.
- Return Periods, `return-periods.zarr`
    - **max_simulated_hourly**, **max_simulated_daily**: the largest value in the retrospective record at each step. Split on the same hourly/daily axis as the fits, because the hourly record's
      maximum is by definition greater than or equal to the daily record's.
    - Each distribution is fit against both maximum series and stored as a separate array: **gumbel_hourly**, **gumbel_daily**, **logpearson3_hourly**, **logpearson3_daily**, **lognormal_hourly**,
      **lognormal_daily**, **weibull_hourly**, **weibull_daily**.
    - A distribution is always paired with the series it was fit to, so that a consumer never has to guess which one it holds.
- Flow Duration Curves, `fdc.zarr`
    - **hourly_annual**, **daily_annual** on the `p_exceed` axis. `hourly_annual[95]` is the flow the reach exceeds 95% of the time.

### Fill values and completeness

Discharge arrays are complete: every river on the `riverId` axis has a value at every step of the `time` axis, and the coordinate arrays themselves have no gaps. The fill value is `NaN`, so a NaN
indicates a real failure to be investigated, not an expected absence.

### Encoding and compression

- Write an explicit dtype on every array at store creation time. Discharge valued arrays are `float32`, coordinate and index arrays are `int32`.
- Bitround discharge to **15 keepbits**, round-to-nearest-even on the low mantissa bits. This bounds relative error at `2^-16` ≈ 1.5e-05.
- Write `keepbits` into each discharge array's attributes, and reapply the same value when appending. Bitrounding is idempotent only if keepbits is unchanged.
- Compress every array, discharge and coordinate alike, with `blosc(cname="zstd", clevel=5, shuffle="shuffle")`.
- **Blosc is the only permitted compressor.** Browser clients ship blosc alone in their WASM codec builds, so an array written with any other codec fails to decode there.
- Do not add a separate shuffle codec entry. Blosc applies shuffle inside its own frame.
- Leave `typesize` unset so each array derives it from its own itemsize.

### retrospective/hourly.zarr, daily.zarr

```text
daily.zarr/
├── zarr.json                 title, license
├── riverId/       int32   (riverId,)          chunks (all,)
├── time/          int32   (time,)             chunks (all,)          units "hours since 1940-01-01T...+00:00", proleptic_gregorian
└── Q/             float32 (time, riverId)     chunks (all_time, N)   units "m3 s-1", aggregation_method "mean", keepbits 15
                                               ^ whole series in one chunk: read pattern is one river, all time
dims: time=31602 (1940-01-01..now, daily) · riverId=N

hourly.zarr/                  identical shape/attrs, time on hourly axis — largest store
                              note the daily axis is still counted in hours, not days
```

### retrospective/monthly.zarr, yearly.zarr

```text
monthly.zarr/                 two chunkings of the same values, one store serves both access patterns
├── zarr.json
├── riverId/       int32   (riverId,)
├── time/          int32   (time,)             units "hours since 1940-01-01T00:00:00+00:00"
├── Q/             float32 (time, riverId)     chunks (all_time, N)   one river, whole series -> plots
└── Q_timesteps/   float32 (time, riverId)     chunks (few, N)        all rivers, few timesteps -> map styling
                                               ^ same values as Q, rechunked; not re-bitrounded
dims: time=1038 months · riverId=N

yearly.zarr/                  same, time = one value per year
```

### retrospective/return-periods.zarr, maximums.zarr

```text
return-periods.zarr/          no time dim — the interval is the axis. client zips recurrence_interval x gumbel_* -> {2: q, 5: q, ...}
├── zarr.json                            title, description (Gumbel/GEV-1, method of moments), license
├── riverId/                     int32   (riverId,)
├── recurrence_interval/         float32 (recurrence_interval,)   [1.5, 2, 5, 10, 25, 50, 100]
│                                        ^ float, not int — 1.5 is bankfull-ish, the most frequent alert tier
├── annual_exceedance_probability/ float32 (recurrence_interval,)   1 / recurrence_interval
├── gumbel_hourly/               float32 (recurrence_interval, riverId)      each of the four distributions is fit
├── gumbel_daily/                float32 (recurrence_interval, riverId)      against BOTH maximum series, so the
├── logpearson3_hourly/          float32 (recurrence_interval, riverId)      array name always carries the series
├── logpearson3_daily/           float32 (recurrence_interval, riverId)      it was fit to
├── lognormal_hourly/            float32 (recurrence_interval, riverId)
├── lognormal_daily/             float32 (recurrence_interval, riverId)
├── weibull_hourly/              float32 (recurrence_interval, riverId)
├── weibull_daily/               float32 (recurrence_interval, riverId)
├── max_simulated_hourly/        float32 (riverId,)                 largest value in the record, hourly series
└── max_simulated_daily/         float32 (riverId,)                 ...and daily — the hourly max is always >= it
                              there is no series-agnostic "gumbel" array — pick a series explicitly

maximums.zarr/                annual maxima the fit above is derived from, kept so it stays reproducible
├── zarr.json
├── riverId/         int32   (riverId,)
├── time/            int32   (time,)                      one value per year, still counted in hours
├── daily/           float32 (time, riverId)              aggregation_method "max", keepbits 15
└── hourly/          float32 (time, riverId)              aggregation_method "max", keepbits 15
```

### retrospective/fdc.zarr

```text
fdc.zarr/                     flow duration curves. no time dim — exceedance probability is the axis.
├── zarr.json                 title, description (percentile of the exceedance distribution), license
├── riverId/         int32   (riverId,)
├── p_exceed/        int32   (p_exceed,)              0..100 percent, 101 values -> Q95 is row 95
├── hourly_annual/   float32 (p_exceed, riverId)      chunks (all, N)   units "m3 s-1", keepbits 15
└── daily_annual/    float32 (p_exceed, riverId)      chunks (all, N)   units "m3 s-1", keepbits 15
                              ^ whole curve for one river in one chunk: the read pattern is one river,
                                every exceedance level — same shape of access as return-periods.zarr
```

`p_exceed` is *exceedance*, not a plotting position. `hourly_annual[95]` is the flow the reach exceeds 95% of the time, that is, a low flow threshold. The `below-q95` map styleset is built from
exactly that row.

### forecasts15/year=YYYY/month=MM/day=DD/discharge.zarr

```text
discharge.zarr/
├── zarr.json       title, initialization_time "2026-07-10T00:00:00Z"
├── riverId/        int32   (riverId,)                    chunks (all,)
├── member/         int32   (member,)                     1..51 — 50 perturbed + 1 control
├── time/           int32   (time,)                       chunks (all,)   units "hours since 2026-07-10T00:00:00+00:00"
├── lead_time/      int32   (time,)                       coordinate ON the time dim, NOT its own dim
│                                                         timedelta since init, units "hours" -> 0, 3, 6, ... 357
│                                                         left aligned like time: 357 covers hour 357..360, so the
│                                                         15 d horizon is 120 intervals, not 121 instants
├── percentiles/    int32   (percentiles,)                [0, 10, ... 100] deciles: 0 = min, 50 = median, 100 = max
├── Q/              float32 (member, time, riverId)       chunks (51, 120, N)   units "m3 s-1", keepbits 15
│                                                         ^ whole ensemble x whole horizon per river bucket = 1 request
├── Qpercentiles/   float32 (percentiles, time, riverId)  chunks (11, 120, N)   member-wise reduction of Q
└── Qmean/          float32 (time, riverId)               chunks (120, N)       NOT Qpercentiles[50] — mean != median
dims: member=51 · time=120 (15 d @ 3 h) · percentiles=11 · riverId=N

forecasts45/.../discharge.zarr    same layout, longer + coarser time
```

### flood-maps/lon=XXX/lat=YYY/fldpln.zarr

`fldpln.zarr` is not gridded and is not read with xarray. It is a flat, sorted, per river contiguous set of run arrays sliced by offsets carried in the group attributes, read by the flood worker
directly.

```text
fldpln.zarr/                  per-tile FLDPLN library
├── zarr.json                 attributes carry the index:
│                               schemaVersion  "tiles-1.0"   (reader rejects non tiles-1.x)
│                               grid           { gRow0, gCol0, ... }   tile origin in the global grid
│                               rivers         { comid[], visitStart[], visitCount[],   -> streams/ rows
│                                                pixStart[], pixCount[],                -> library/ per-pixel rows
│                                                relStart[], relCount[] }               -> library/ per-relation rows
├── streams/                  one row per stream-pixel visit, in baked headwater->outlet path order
│   ├── fsp_local/  (visit,)         flood source pixel index, tile-local
│   ├── row/ col/   (visit,)         pixel position, tile-local (+ grid origin -> global)
│   ├── bed/        (visit,)         bed elevation
│   ├── q_baseflow/ (visit,)         below this Q the reach is unflooded
│   ├── q/          (visit, stage)   discharge ladder (30-pt rating curve)
│   └── wse/        (visit, stage)   water-surface elevation per ladder step
└── library/                  per floodable pixel, then per (pixel, source) relation
    ├── pix_row/    (pix,)           floodplain pixel position, tile-local
    ├── pix_col/    (pix,)
    ├── fill_mm/    (pix,)   uint16  mm, lossless quantization -> client fround(mm/1000)
    ├── rel_count/  (pix,)           how many relation rows below belong to this pixel
    ├── fsp_local/  (rel,)           which stream pixel floods it
    └── dtf_mm/     (rel,)   uint16  depth-to-flood threshold, mm

kernel:  depth(fpp) = max over relations (DoF - DTF) + fill      no raster ships, bed elevation cancels out
```

## Input Datasets

### ECMWF IFS

See the MARS requests in [Flood Forecast Products](#flood-forecast-products) and [Extended Range Forecast Products](#extended-range-forecast-products).

### ERA5 and ERA6

<mark>TBD</mark>

## Implementation Details

### Project Organization

This suite expects a machine with a certain directory structure: a home directory with subdirectories for IFS, ERA5, forecasts which has subdirectories for each YMD, and retrospective. This is the
working layout on the compute machine. The final products it produces are uploaded to the layout described in [Organization on S3](#organization-on-s3). The computational unit is called a group there
and in the published data. The scripts and intermediate files below still use the older VPU naming.

```text
/$HOME
    /ifs
        yyyymmdd.grib
    /era5
        yyyymmdd.nc
    /forecasts
        /YYYYMMDD
            /vpus
                /vpu=101
                    /volumes
                        volumes_$vpu_$ens.nc  # 1 per ensemble member
                    /discharge
                        discharge_$vpu_$ens.nc  # 1 per ensemble member
                    nces_avg_$vpu.nc  # 1 per VPU, ensemble average
                    map_tables/
                        YYYYMMDDHH.parquet  # 1 per timestep of the forecast
            # Final products, uploaded to forecasts15/year=YYYY/month=MM/day=DD/
            discharge.zarr
            alerts.csv
            fim.geo.parquet
            maps/
                esri_animation_tables/
                    YYYYMMDDHH.csv  # 1 for each timestep of the forecast
                timeseries/
                max-flow/
                below-q95/
                time-to-peak/
            next-init-files/
                /group=XXX
                    warmstate_YYYYMMDDHHMM_groupXXX.parquet
        # For example
        /20250101
        /20250102
        /20250103
        /20250104
        /20250105
    /retrospective
        hourly.zarr
        daily.zarr
        monthly.zarr
        yearly.zarr
        maximums.zarr
        return-periods.zarr
        fdc.zarr
```

### Log/Status Feeds

In addition to writing logging messages to disc, status information is sent to the following locations:

- Teams channel webhook
- AWS CloudWatch logs

### Summary of computational steps

Each day, the following steps are performed in order:

#### Phase 0: Preparation

1. Set environment variables
    1. YMD - The date of the day to be processed, usually today. In YYYYMMDD format.

#### Phase 1: Download runoff data

Runoff data are cached. Check if the data are available, otherwise download them.

1. Download the latest ECMWF IFS runoff grid/mesh data for the date specified by YMD.
2. Download the latest ERA5 runoff grid/mesh data for the date specified by YMD.

#### Phase 2: Daily forecast computations

1. VPU level computations. Parallelize these jobs by VPU, but do not change the task order.
    1. Calculate catchment level volumes (python/calculate_catchment_volumes.py)
    2. Route the volumes (python/route.py)
    3. Concatenate ensemble members into a single file (bash/concat_member_discharges.sh)
    4. Generate a table of summarized flows used to style the web map layer (python/generate_vpu_map_tables.py)
    5. Issue alerts by filtering the map summary tables (TODO)
2. Concatenate the VPU level results. Tasks may be computed in any order and/or simultaneously.
    1. Concatenate the VPU level routed discharge. Makes a Zarr dataset. (python/concatenate_vpu_discharge.py)
    2. Concatenate the VPU level map tables. Makes a directory of CSVs. (python/concatenate_vpu_map_tables.py)
    3. Concatenate the VPU level alerts (TODO)

#### Phase 3: Update retrospective simulation

1. Download the latest day's ERA5 runoff data
2. Calculate catchment level volumes
3. Route the volumes, initialized from the last time step of the retrospective simulation.
4. Resample the hourly discharge to daily average and, when necessary, monthly and yearly averages.

#### Phase 4: Synchronize forecast initialization to retrospective simulation

1. Get the initialization from the last retrospective simulation updates
2. Get the first 24 hours of forecasted catchment volumes for the 5 days between the last retrospective timestep and the present date.
3. Reroute the forecasted volumes, but initialize at the last retrospective timestep.
4. Calculate the ensemble average of the rerouted forecasted volumes

#### Phase 5: Export results to S3 archives

All exports land under `s3://river-forecast-system/v3/`, see [Organization on S3](#organization-on-s3).

1. From the daily forecast computations, to `forecasts15/year=YYYY/month=MM/day=DD/`
    1. Routed discharge - `discharge.zarr`
    2. Esri animation tables and map stylesets - `maps/`
    3. Alerts - `alerts.csv`
    4. Vector flood extents - `fim.geo.parquet`
    5. Routing warm states per group - `next-init-files/group=XXX/warmstate_YYYYMMDDHHMM_groupXXX.parquet`
2. From the retrospective simulation update, to `retrospective/`
    1. Routed discharge in hourly, daily, monthly, yearly averages - Zarr
3. From the rerouted forecasted volumes
    1. Rerouted forecasted discharge, 5 days only - Zarr

#### Phase 6: Cleanup

1. Delete forecast directories dated __older than 5 days__
2. Delete runoff data dated __older than 5 days__

## Changelog
